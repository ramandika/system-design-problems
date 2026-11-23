# 04 — Deep dives

## 1. The delivery protocol: at-least-once wire + client-side dedup = "exactly-once" as-feel

A message must arrive **once, ordered, even across flaky mobile networks.** Pure
"exactly-once" is not achievable on an unreliable network; the real design:

- **Wire protocol is at-least-once**: message goes out with a unique ID per
  (device, conversation) pair; sender *retries with backoff* until it gets an ACK.
- **Server stores message with a conversation-local sequence number**; delivery is ordered
  by that sequence.
- **Client dedupes by message ID** before rendering. Receiver gets the sequence + dedup key;
  duplicates are discarded, order is enforced, gaps are detected (client can ask for a
  specific sequence range to catch up).

- **Trade-off**: client adds dedup logic + gap-filling complexity; in exchange, transport
  stays simple (retry blindly) and correctness is provable at the app layer.
- Contrast with YouTube's resumable upload (idempotent chunks) — same idea: make wire-level
  retry safe, get user-level "exactly once".

## 2. The connection layer at 100M sockets

Persistent connections, ~100M concurrent. Design notes:

- **Gateway tier terminates sockets**: non-blocking, epoll-based, per-socket buffers sized
  small (idle sockets are the norm — a typical user sends a handful of messages a day).
- **A "who is connected where" registry** (Redis or in-memory cluster) maps user_id →
  gateway instance, updated on connect/disconnect. Lookups are by user_id → route pushes.
- **Keep-alive tuning matters for phones** (battery + mobile carrier NAT timeouts):
  send keep-alives less often than the NAT idle timeout, not constantly. Trade the battery
  life of 2B users against a slightly higher reconnect rate when NATs time out.
- **Reconnect storms**: a whole region coming back online simultaneously will send millions
  of TLS handshakes in a few seconds → mitigate at the edge (TLS session resumption,
  randomized exponential backoff from the client on reconnect).
- **Gateway restarts** (deploys, OOM) drop up to ~1M sockets each; clients reconnect with
  backoff. Deploy canary + drain is part of the protocol design, not an ops afterthought.

**What the 2012–2014 evidence actually teaches.** WhatsApp's famous per-server numbers
were won mostly in the VM, not the app: going ~200k → 2M conns/server was "all contention
fixes," and *most required patching the BEAM VM itself* — scheduler spin counts, the
timeofday lock, the timer wheel, ets/mnesia partitioning (Rick Reed, Erlang Factory 2012).
After those fixes a single box held ~2.3M concurrent TCP sockets at ~58% CPU busy (~42%
idle) across 24 cores — the same Jan 2012 numbers quoted in [estimates](02-scale-estimates.md),
which come from the WhatsApp blog and Reed's 2012 talk; the design lesson for any stack: at 100M+ connections the
bottleneck is contention in shared runtime structures (locks, timers, queues), not
"threads vs coroutines." If your runtime doesn't let you split those structures per
core, you'll cap out lower — and patching a GC-language runtime is a worse place to be
than BEAM/Lua/Go where per-core structures are reachable or already default.

Discord's later Erlang session layer corroborates the same physics from the other side:
one VM held up to 500k live sessions, but a single `send/2` fanout to a 30k-member guild
cost 900ms–2.1s at peak (30–70µs *per send*), fixed by Manifold — group destination
PIDs by remote node, hash with `:erlang.phash2/2`, fan out via per-core worker processes.
Same lesson: per-core batching/routing structures beat any amount of "faster sends."

## 3. Offline queue + ordering per conversation

Store-and-forward for offline receivers. Design:

- **Store key = conversation_id, order key = sequence number**, monotonically increasing
  per conversation. Strict per-conversation ordering is easy; global order (across
  conversations) isn't needed (see requirements).
- **Delivery on reconnect**: on socket open, deliver everything parked for that account in
  sequence order, then live stream. No user-visible reorder, no gaps.
**TTL (say 30 days)**: after that, messages expire (or the client, when it eventually
  reconnects, re-pulls from a slower "cold" path). Trade-off: infinite queueing = unbounded
  storage for inactive users; users who don't come back for months would cost real money
  for messages they'll most likely never read.

**Documented store analogue (Discord)**: WhatsApp's actual store is not public, so the
best-documented comparable is Discord's message store (Discord blog, 2022–23): wide-column
store, partition key = channel ID + static time-window bucket, chronologically sortable
snowflake IDs, quorum reads/writes. May 2022: Cassandra 177 nodes → ScyllaDB 72 nodes,
p99 historical reads 40–125ms → 15ms, p99 inserts → steady 5ms. Two transferable details:
(a) the **data-services layer between API and DB** — one gRPC endpoint per query type,
consistent-hash-routed so a channel's traffic hits one instance, with request coalescing
(n concurrent readers of the same row → one DB query); (b) the migration *shrank* the
fleet ~2.5× (177 → 72 nodes) while cutting p99 — fewer, cheaper nodes with lower tail
latency, an argument
against "just add Cassandra nodes." Latency numbers are Discord's own measurements, not
an independent Cassandra-vs-ScyllaDB benchmark.

## 4. Group messages: bounded fanout (vs Twitter's unbounded)

A group message lands in up to ~1024 recipients' queues. That's fine synchronously: 1024
per-member writes with a group fanout service is a bounded, sharded job — orders of magnitude
smaller than Twitter's celebrity fanout (100M timelines for one post — see
[Twitter deep dives](../02.twitter/04-deep-dives.md)).

- **Trade-off**: per-conversation sequence numbers per group need one authoritative writer
  (the group message store holds the group log, with the fanout service expanding per
  member); group chats with 1024 members in one shard's log are fine; very large communities
  or broadcast-style groups would need a different model (Twitter-like on-read or push-based
  streaming into per-member timelines).
- **Receipts in groups**: read receipts for 1024 members in groups would be a fanout
  explosion; real systems batch/aggregate receipts rather than fan out one per read.

## 5. E2E encryption and what it removes from design

The server never sees plaintext. Consequences — this is an architecture constraint, not
just a crypto spec:

- **No server-side content inspection**: spam filtering can't look at text; it must use
  graph/metadata signals (frequency of sends, account age, report rates) — see
  [Twitter's rate limiting](../02.twitter/04-deep-dives.md) for the shape of that problem.
- **No server-side dedupe on ciphertext of the same plaintext across users** (each pair has
  its own keys). Media dedup works only per (content hash + per-recipient key).
- **Server is a router + mailbox, nothing else** — a useful simplification in every other
  design decision (no "smart" features that need plaintext on the server: no server-side
  search, no translation, no server-side summarization. Real systems push these to clients).

**How the key agreement earns forward secrecy asynchronously** (X3DH, the Signal spec
WhatsApp builds on): the recipient pre-publishes key bundles to the server — identity key
IK, signed prekey SPK, and a stock of one-time prekeys OPK — so two parties can
authenticate and derive a shared secret *without ever being online simultaneously*:

```
DH1 = DH(IK_A, SPK_B)   # mutual authentication of A and B
DH2 = DH(EK_A, IK_B)    # mutual authentication
DH3 = DH(EK_A, SPK_B)   # forward secrecy
DH4 = DH(EK_A, OPK_B)   # forward secrecy; included when a one-time prekey survives
SK  = KDF(DH1 || DH2 || DH3 || DH4)
```

The design pressure this creates on the *server* design: the key server is an
availability-critical, integrity-critical mailbox for prekey bundles — it must never be
able to substitute its own keys (or it breaks E2E), and one-time prekey exhaustion must
degrade gracefully to a valid 3-DH agreement (losing DH4's extra secrecy, not failing the
session). Depleting one-time prekeys under load is a real capacity coupling between the
crypto design and your prekey-refill/write rates.

**Caveat on "proven"**: WhatsApp's implementation is closed-source. The strongest public
guarantee (Albrecht–Dowling–Jones, EUROCRYPT 2025) is a formal proof over a
*reverse-engineered* description of the multi-device group-messaging subset, in a
model-relative variant of DOGM with device revocation — rigor, but not an audit of the
shipped product.

## 6. Presence, typing, receipts — ephemeral by design

These state signals are high-churn, low-value-if-lost. Design: keep them in an in-memory
presence cluster only, propagate best-effort, never persist. If presence is lost (gateway
crash) it self-heals on next update.

- **Trade-off**: presence may briefly be stale (wrong "online" badge) vs the cost of adding
  a durable write path for a signal nobody cares about 30 s later. Pick ephemeral.
- Note the contrast: this is the same call as WhatsApp's messages in reverse — messages are
  MUST-NOT-LOSE, presence is MUST-NOT-COST-ANYTHING. Knowing which is which per feature is
  basic design discipline.
