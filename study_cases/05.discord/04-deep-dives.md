# 04 — Deep dives

## 1. Gateway layer: terminating millions of persistent sockets

Scale target (est. from §02): ~10M concurrent sockets fleetwide as the design envelope,
with every region provisioned to carry the full envelope alone (regional failover);
at ~100–500k sockets per gateway node that is ~20–100 nodes per region →
~75–400 nodes fleetwide (estimate). The gateway is
the hardest tier to scale horizontally because each socket is, by definition, stateful
per node.

**Option A — stateless gateway with a lookup registry** (WhatsApp's shape): any gateway can
answer any user; a "who is connected where" registry maps user_id → gateway route and
publishes are addressed by that lookup.
- wins: deploys and reshard are trivial; no need for consistent hashing between the socket
  and anything else.
- loses: every publish does an extra registry lookup; registry becomes a hot metadata
  cluster; a registry outage is a chat outage even with gateways healthy.

**Option B — deterministic placement (sharded gateway)**: user_id → gateway node via
consistent hash; a publisher knows where a session lives without looking it up.
- wins: no registry to maintain; publish routes are O(1) arithmetic.
- loses: node death strands every hash-assigned session at once (a coordinated mass
  reconnect instead of an independent one); the user↔node mapping is brittle under fleet
  resize; multi-session users (mobile + desktop) must be handled explicitly anyway.

**Chosen: a hybrid that matches Discord's *sharded* gateway design.** Sessions are assigned
to gateway *shards* by user_id (a fixed large-shard-count ring, e.g. 1024 shards, each shard
owned by one active gateway); guild publishes are fanned out to the shard set of their
member *subscribers*, not directly to nodes. Wins: publish routing is a topic → shard-id
map (no live registry for message routing), and shard-level failover is per-shard, not
per-node. Loses: shard ownership moves are a coordinated event (and the shard map itself
must be a fault-tolerant registry — an extra moving part, but one whose failure mode is
"slow reconnect", not "chat down").

Two physics lessons from prior cases carry over directly, and both are documented in
Discord's own history (see [WhatsApp §2](../03.whatsapp/04-deep-dives.md), which quotes the
Manifold numbers):

- **Runtime contention, not I/O, is the ceiling.** Discord's Erlang layer held ~500k live
  sessions per VM, but a single send/2 fanout to a 30k-member guild took 900ms–2.1s at peak
  (~30–70µs per send) purely from contention and per-process scheduling; the fix (Manifold)
  was to group destination PIDs by remote node and fan out via per-core worker processes.
  The design consequence for us: the gateway fanout must be **grouped by remote node and
  batched per-core**, not a naive per-socket send loop — a design constraint, not an
  optimization.
- **Reconnect storms** after a gateway deploy or a regional outage: TLS handshakes,
  READY payloads (full guild snapshot), and history fetches all arrive together. Client-side
  jittered exponential backoff plus server-side shard fail-slow (admit a bounded rate of
  new sessions per shard and let the rest back off) turn a thundering herd into a queue.

**Failure modes:** gateway OOM (deploy with drain + shard handoff, never abrupt); pub/sub
partition between a gateway and the bus (sessions keep sockets alive but stop receiving —
require heartbeats between gateway and bus, force reconnect on staleness); shard map
corruption (shard → node reassignment is the highest-stakes state; keep it in a
strongly-consistent small store, not the message store).

## 2. Broadcast fanout to 100k-member guilds without fanout storms

A message in a 100k-member channel is a one-write → 100k-delivery event. Twitter's hybrid
([Twitter §1](../02.twitter/04-deep-dives.md)) doesn't apply as-is: there, celebrity
fanout is the *exception*; here, huge guilds are a normal part of the workload, and unlike
Twitter, membership is uniform — you can't skip "the celebrity segment" because there
isn't one.

**Option A — fanout to all members on write (Twitter-style on-write):**
- wins: every member gets the message even offline and unread counts stay exact.
- loses: O(members) per message; at §02's numbers (~30% of peak traffic in huge channels)
  this is hundreds of millions of member-deliveries/s — the fleet cannot afford it, and
  offline members' message rows would be 98% wasted work.

**Option B — fanout on read (no push; clients poll history):**
- wins: write cost is O(1) — publish to subscribers of the channel topic only.
- loses: chat latency becomes poll latency; presence/typing can't be push-style; the
  UX regresses from "live" to "refresh".

**Chosen: membership ≠ delivery subscription.** Delivery targets are **connected socket
sessions subscribed to the channel's topic**, not guild members. A 100k-member guild's
channel typically has 1–5k live subscribers (estimate) — a 20–100× reduction. Offline
members and clients in other channels get message *existence* via a decoupled path:
- **unread counters** are maintained by fanout workers asynchronously, aggregated per
  (user, channel) in the metadata store, and pushed lazily/infrequently (or fetched on app
  foreground).
- **push notifications** (APNs/FCM) fire only on mention/ping/mention-role matches, which
  is a small fraction of messages; they resolve mentions eagerly and are rate-limited per
  user.
- **catch-up on reconnect/foreground** is a store read, not a replay through the bus.

Trade-off made explicit: we win a bounded, workload-shaped fanout (scales with *live
attention*, not membership size), and lose exact real-time unread counts for offline users
(counters are eventually consistent, reconciled on foreground). We pick this because the
alternative is a fleet sized to a delivery volume nobody actually consumes, and because a
briefly-wrong unread badge is recoverable state, not lost data — the same
must-not-lose/must-not-cost-anything split as [WhatsApp §6](../03.whatsapp/04-deep-dives.md)
§6, just applied to a different signal class.

**Anti-storm mechanics inside the send loop** (the Manifold lesson from §1):
group destinations by remote gateway node; one batch per node; per-core worker processes
drain the batch; per-socket backpressure applies beyond a buffer watermark — for a slow
socket the gateway *drops live pushes* rather than queuing, because the store has the
message and a reconnect/catch-up will re-sync. Losing a live push temporarily is
self-healing; queueing to a slow socket is how fanout storms become OOM cascades.

**Failure modes:** hot-guild event (a viral announcement in a 500k guild) → rate-limit
sends at the channel level (message/s cap), not just per-user, and degrade to a "channel
is busy, fetch history" client state instead of storming the bus; a fanout worker backlog
(detect via queue depth, re-publish idempotently — the event bus is at-least-once, clients
dedupe by message snowflake); bus topic explosion during a huge guild join (see §3).

## 3. Pub/sub topic routing across a fleet

Every chat event, presence blip, typing indicator, and voice state change is an event on
a pub/sub cluster. Topic space:

- `channel:{id}` — message create/delete/update, typing, channel edits.
- `guild:{id}` — presence changes, member join/leave, bulk channel updates.
- `user:{id}` — direct personal events (DM create, friend state, session invalidation).

**Option A — every session subscribes to its guild's full topic set** (naive):
- wins: no routing logic; any event reaches anyone who might care.
- loses: a user in 100 guilds, several of them 100k-member mega-guilds, would have a
  session subscribed to every channel in every guild; the bus would route an enormous
  fraction of events to sessions that will never display them — the same waste as
  Option A in §2, moved one level down.

**Option B — fine-grained subscriptions** (channel-level and presence-interest-level):
- wins: bus load tracks *what clients actually render*, i.e. real attention, not
  membership.
- loses: subscription churn (every join/leave/channel-switch is a subscription write);
  the routing table per subscriber is non-trivial state on the bus.

**Chosen: Option B with two stabilizations.** First, sessions subscribe to the *visible
render set* (the open channel + the sidebar's top-N channels), not to the whole guild;
guild-level topics carry only low-frequency state events. Second, presence interest is
**opt-in and bounded**: a client subscribes to presence for the member list it renders
(not "everyone in the guild"), and mega-guilds get presence digests (online counts, not
individual identities) for members beyond that boundary.

- Trade-off: we win bus capacity that scales with rendered attention instead of with
  membership; we lose live "just-joined" precision in mega-guild member lists (acceptable —
  a badge count updates on the digest interval).
- Routing implementation: consistent-hash partitioning of the topic space across bus
  nodes; each gateway holds a local subscription index subscriber → interested-topics
  (inverse index), and the bus holds topic → subscriber-set. Both are sharded by the same
  hash so a topic's state lives on exactly one bus node ("no split brain between routing
  and delivery").

**Failure modes:** subscription churn under mass join/leave (batch subscription writes;
subscriptions are per-shard, so a shard's churn doesn't hot the whole bus); bus node death
(recoverable: sessions reconnect and resubscribe from local state — the bus holds no
authoritative state); hot-topic skew when one mega-guild channel dominates (split the
topic, or throttle channel-rate at the source, per §2).

## 4. Voice/video via WebRTC SFUs

**Option A — mesh (each participant sends directly to every other):**
- wins: no central media infrastructure; simplest possible implementation; works for
  tiny groups.
- loses: uplink cost O(participants) per sender; a 50-person room means every sender
  uploads 49 copies of their audio — user bandwidth, mobile CPU/battery, and ICE/
  traversal per pair blow up.

**Option B — SFU (Selective Forwarding Unit): each participant sends one stream to an
SFU; SFU forwards to each other participant.**
- wins: uplink is O(1) per sender; SFU just relays (no transcode latency cost);
  participant count becomes an SFU capacity problem, not a client bandwidth problem.
- loses: SFU tier is a new stateful pool to operate and to allocate per room; SFU death
  mid-call is a user-visible outage (reconnect is not seamless like a chat retry).

**Chosen: Option B, with routing rules.** SFUs are per-room sticky
(one SFU per voice channel session), allocated by the voice backend by current load
(participant count and egress bitrate — a video SFU at 50 participants × ~1.5 Mbps
downstream is ~75 Mbps egress, the binding number, not CPU). SFU selection is regional:
an SFU in the same region as the room's median participant latency-wise.

- **Trade-off: no transcoding.** The SFU is a pure router — one send-rate per participant
  (the sender's single stream), not per-receiver adaptation, and no per-receiver quality
  levels on audio. Video gets simulcast (sender uploads 2–3 spatial layers; SFU picks per
  receiver) as the compromise; without it, a 500-person room cannot run video at all.
  Trade-off accepted: extra uplink (2–3× audio, 3–4× video) buys O(1) SFU forwarding complexity
  and no transcode latency.
- **Where signaling lives:** the voice backend does ICE/srtp handshake over the *existing
  gateway socket* (reusing auth and session context instead of a second control channel);
  media then switches to UDP directly to the SFU. Voice control events (someone joined,
  screen share started) ride the pub/sub bus as ordinary events.

**Failure modes:** SFU node death — room participants lose media and must re-ICE; the
voice backend must detect SFU liveness with the same heartbeat discipline as gateway
sessions and migrate an empty room to a new SFU automatically; participants already
connected are the only real cost (reconnect UX). NAT/firewall traversal with UDP blocked
(fallback to TURN relay is a capacity line, not an afterthought — TURN is ~2× SFU egress
for the relayed participants, cost-wise a real number to report to the fleet owner). Packet
loss / jitter — client-side jitter buffer and opus Forward Error Correction absorb it;
beyond ~5% loss, simulcast drops video layers before audio, because audio carries the
session.

## 5. Per-channel message ordering & storage

**Ordering.** Every message gets a **snowflake-style 64-bit timestamp-sorted ID**
(timestamp | worker | sequence; the same argument as
[Twitter §3](../02.twitter/04-deep-dives.md) — sortable IDs make pagination, dedup, and
"everything after X" arithmetic instead of a sort). Per-channel order is
strict as observed by clients: clients order by ID, and a per-channel monotonic write
sequence (assigned in the store's partition) resolves ties. **No global ordering across
channels is needed** — the only cross-channel ordering clients need is per-user
"which channel has new stuff", which is the read-state and unread-counter path, not a
total order.

**Option A — one row store, partition key = channel_id** (no time bucket):
- wins: trivially correct per-channel order, no range-split decision to make.
- loses: a ten-year-old public guild has billions of rows in ONE partition — the
  partition eventually outgrows any node, and compaction/repair of it degrades the whole
  cluster.

**Option B — partition key = channel_id + static time bucket** (bucket every N days):
- wins no partition grows without bound; old buckets go cold, can be tiered or archived;
  reads specify (channel, bucket) → bounded partitions; compaction per bucket.
- loses: writes and reads must know the current bucket (a small routing/lookup); time-based
  boundaries put two logical neighbors in different partitions (pagination crosses a
  bucket boundary needs a 2-partition read — handled by the data-services layer, invisible
  to clients).

**Chosen: Option B** — the shape Discord publicly uses (Cassandra/ScyllaDB, wide-column,
partition = channel + time bucket, snowflake ID ordering, quorum reads/writes; publicly
reported p99 inserts ~5 ms steady, historical reads ~15 ms after their migration (down
from 40–125 ms) — same
figures quoted and caveated in [WhatsApp §3](../03.whatsapp/04-deep-dives.md)). Trade-off
made explicit: we win bounded partitions (which at trillion-row scale is not optional) and
lose a two-level routing layer and an occasional cross-partition read — the data-services
consistent-hash routing shim (one gRPC instance per channel shard, request coalescing of
concurrent same-row readers) absorbs both.

**Retention.** Keep rows forever but tier storage by bucket age (recent bucket on the fast
store; cold buckets on cheaper storage; the bucket key is the tiering unit). Trade-off:
infinite retention, tiered, is affordable in a way that one homogeneous store holding
trillions of hot rows is not; we accept a "cold read is slow" SLA tail (documented in
01-requirements) rather than losing history.

**Multi-region.** Writes are region-affine (nearest region), metadata is replicated
eventually cross-region; a guild is owned by one region at a time (failover is a metadata
operation). Trade-off: we lose active-active writes for the message store, in exchange for
a partition-per-channel store that never has to reconcile two conflicting appends to the
same ordered log — cross-region consensus on every message insert would multiply latency
by region-wide RTT for a feature (geo-distributed concurrent writes to one channel) that
rarely exists. **Failure mode this accepts:** region failover can drop in-flight messages
in the seconds before switchover — clients retry on the new region; the snowflake ID +
client-side dedup by message ID makes retry idempotent at the store (same ID, same row —
no duplicate).

**Failure modes:** store partition outage (channel unavailable for write, but readable
from a replica: the API fleet marks the channel read-only and clients show "can't send, "
retrying with backoff — degrade, don't black out); snowflake clock skew (IDs are near-sorted;
per-channel monotonic sequence still gives strict local order — the snowflake is for
pagination, the sequence is for truth); data-services cache of the bucket router going
stale during a bucket rollover (bounded staleness = a millisecond window of writes into the
old bucket; recovery is a re-route, not data loss).
