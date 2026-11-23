# 04 — Deep dives

## 1. The mailbox model: one row per message vs per message-recipient

The first storage decision in a server-backed messenger: where do history rows live? Two
coherent answers:

- **Option A — per-conversation canon** (WhatsApp/Discord shape): one append log per
  conversation; a member reads by reading the log. One copy per message total.
- **Option B — per-user mailboxes** (Messenger's shape): the fanout service writes one
  mailbox entry per recipient; history per user is *their* rows, shardable by user id.

**Trade-off.** Option A wins write efficiency (one row per send, no fanout) and group
storage, but every read path pays: "my threads, newest first, with unread counts" requires
scanning/bucketing every conversation the user belongs to; per-user retention (delete from
my side only), per-user read markers, and per-user search indexing are awkward bolt-ons to
a conversation-owned log, and shard key choice fights the schema (by conversation = hot
shards for popular threads; by user = a log you don't own). **Option B wins** exactly those
per-user properties: the mailbox shard keyed by user owns inbox, unread state, retention and
search for that user in one place, and the read path is a local index scan. It loses
write amplification — ~200B entries/day under [estimates](02-scale-estimates.md) arithmetic —
and multiplies storage by group size. **We pick B** for a system whose product surface is an
*inbox* (browsable, searchable, per-user) rather than a relay — that is the Messenger
question, not a religious one; a relay-only product should pick A, which is why WhatsApp
did. Note the same split at a different scale: Twitter's write-fanout-into-timelines is B
with followers instead of group-mates, and its read-fanout failure mode is the celebrity
case (see [Twitter deep dives](../02.twitter/04-deep-dives.md)); Discord's per-channel store
is A (see [Discord](../05.discord/04-deep-dives.md)).

**Failure mode of the chosen model**: group fanout storms (one busy group of 500 members →
500 mailbox writes per message; a spammy thread with many recipients saturates the recipient
shards, not the sender's). Mitigation is per-thread rate limits *plus* coalescing writes to
users who are not reading the thread (batch parked entries) — and admitting the failure is
*recipient-side*, so the abuse model keys on recipients, not senders (see
[abuse/anti-spam](#6-anti-spam-and-the-abuse-surface-at-messaging-recipient-scale)).

Publicly documented anchor for this design: Facebook engineering talks describe Messenger's
history in terms of per-user mailboxes fed by a unidirectional message log, with the inbox
built as mail-service views over those mailboxes — the HBase→MyRocks migration of the inbox
store (~2018) is then a storage-engine fight over write amplification, exactly the term this
dive used.

## 2. Group chat fanout at scale: the mailbox vs the channel

A group message with K members under the mailbox model does K×(a few) mailbox writes plus
K×device pushes. Group sizes here are bounded (tens-to-hundreds), so this is *bounded*
fanout — the Twitter celebrity problem is unbounded (100M timelines) and needs different
techniques (see [Twitter deep dives](../02.twitter/04-deep-dives.md#1-fanout-on-write-vs-fanout-on-read-the-dive)).
Design within the bound:

- Batch by destination (as Discord's Manifold does, per remote node — see
  [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md#2-the-connection-layer-at-100m-sockets)
  for the numbers): coalesce the per-recipient writes into batches per shard, amortize
  per-write costs.
- Async + parked-first: deliver live pushes only to *currently online* devices; everyone
  else gets the mailbox entry and picks it up at next sync — no blocking the sender on K
  deliveries.
- **Trade-off.** Push-to-all-devices synchronously wins instant cross-device glow, loses
  tail latency under per-recipient failure (one slow shard stalls the sender's ack path).
  We pick async-with-parked-entries: the sender's ack should depend only on the canon write,
  never on delivery to K participants. The receiver-visible guarantee ("ordered, exactly
  once") is enforced per-user by the mailbox seq, not by delivery success.
- Read-state in groups: per-member read cursors stored per member (their own mailbox row),
  never a shared group-wide counter — otherwise K members' reads contend on one row.

**Failure mode**: hot group threads create hot user shards (a thread whose members'
mailboxes all hash to few shards). Mitigation: shard by (user) with threads' load spread
by membership — and accept some residual skew; re-sharding a mailbox by (user, thread) is
the escalation path, paid for with cross-shard inbox transactions.

## 3. Inbox / thread organization and unread counters at web scale

The inbox is the product: ~600M DAU each loading a ~20-thread page several times daily, with
unread counts correct across all their devices. Three designs:

- **Option A — count-on-read**: unread = COUNT of messages after my per-thread read cursor.
  Always correct, but multiplies page loads by thread scans across 20+ threads; at this DAU
  that is the hot path's dominant cost. Loses on read latency.
- **Option B — materialized counters**: per-(user, thread) unread count maintained by the
  same transaction that appends the mailbox entry, decremented on read-cursor moves.
  Wins reads (thread list with badge numbers is a single indexed scan), loses on
  consistency-under-failover (a counter can drift from the data it summarizes, and a
  drifted badge is a user-visible bug with no self-heal).
- **Option C — hybrid**: counters for the *common* state, re-derived from the mailbox rows
  on thread open (scrollback render recomputes the true unread from cursor vs seq anyway).

**We pick C.** The mailbox entry and its counter live in one transaction, so drift only
appears from failover/backups; thread-open repair fixes it lazily without a background
reconciliation fleet. Option A's latency is paid only when the user actually opens a thread
(where Option B's correctness problem becomes visible anyway). Badges may briefly disagree
with the thread view; the badge is allowed seconds of staleness (see the consistency bar in
[requirements](01-requirements.md)).

Thread list ordering is by (latest activity, per user) — which means *the user's own*
reactions/reads/replays also reorder threads, so inbox row updates come from state changes,
not just from new messages. **Failure modes**: (a) inbox-page thundering herd on a big
reconnect (all online devices re-fetch thread lists simultaneously — cap by serving a
cached list with delta piggyback, same shape as the reconnect-storm backoff in
[WhatsApp deep dives](../03.whatsapp/04-deep-dives.md#2-the-connection-layer-at-100m-sockets));
(b) counter drift after a fanout-partial failure — repair happens on open, but a thread
nobody opens carries the wrong badge for days; keep a cheap async reconciler for unopened
threads older than N hours.

Historical anchor for scale: the inbox store's engine migration (HBase → MyRocks/MySQL) is
the public evidence this path matters: compaction/write-amplification characteristics of the
storage engine decide the cost of maintaining those counters, not just the rows.

## 4. Link previews and media pipelines: crawling as a side effect of text

A pasted URL is rendered to every participant as a title + thumbnail card — someone fetched
it server-side. This is a crawler attached to the chat hot path, and its load profile is
unusual: ~4B fetches/day from [estimates](02-scale-estimates.md), cache-friendly
(a handful of viral links dominate).

- Design: preview fetch is **async off the hot path** (the message text appears
  immediately; the card populates), backed by a content cache keyed by URL with short TTL —
  first fetcher pays, everyone else reads.
- **Trade-off.** Option A — fetch at send time, inline: instant cards, but the send path
  is coupled to the whole hostile web (slow sites, redirect chains, 404s); Option B —
  async + cache: fast sends, cards appear a beat later. **We pick B**: a few hundred ms of
  card-population delay is invisible next to a send that might wait on an arbitrary
  third-party web server, and the async worker pool can rate-limit itself independently.
- **Safety is the real constraint** (a messaging crawler is an SSRF factory): deny private
  IPs/link-local, cap redirects, cap body size, no JS execution, fetch from a dedicated
  egress range so the crawler's IP is rate-limited separately from user traffic — standard
  crawler hygiene, but the blast radius (any user can make the fleet fetch any URL) is
  attached to the hottest endpoint of a billion-user app.
- Media pipeline: upload → content-hash (dedupe) → derivative render (thumbnail, resize) →
  CDN. Same resumable-upload shape as [mobile-chat-app](../24.mobile-chat-app/04-deep-dives.md#2-resumable-photovideo-upload-on-flaky-mobile-networks);
  the Messenger-specific twist is *lifetime*: chat media is short-lived content with
  spiky-then-inert access — tiered storage/cold eviction beats CDN-permanent, unlike
  [YouTube](../01.youtube/README.md) catalog content. (Under E2EE, dedupe weakens and
  derivatives happen client-side — see [deep dive 5](#5-the-e2ee-migration-what-server-truth-buys-and-what-it-costs).)
- **Failure modes**: cache-poisoning a viral link serves attacker-chosen images to millions
  of thread contexts (mitigate by per-fetcher signing and short TTL); preview of a
  slow/hostile URL exhausting worker pool (isolate per-domain concurrency caps).

## 5. The E2EE migration: what server truth buys, and what it costs

Historical Messenger's server-side plaintext is what *enables* the rest of this case:
server-side search, ML ranking (message "smart replies" / content classifiers), moderation
and anti-spam on content, link crawling, media dedupe and derivative rendering. The
published trajectory — opt-in **Secret Conversations** (Signal protocol, 2016) → default
E2EE rollout begun December 2023 — means the features above must either move client-side,
run on metadata only, or die. This dive is a features-vs-crypto inventory, not a protocol
rehash (the protocol is WhatsApp's; see
[E2E encryption and what it removes from design](../03.whatsapp/04-deep-dives.md#5-e2e-encryption-and-what-it-removes-from-design)).

Per-feature fate under E2EE:

| Feature | Plaintext-era mechanism | Under E2EE |
|---|---|---|
| Server-side search | index over plaintext rows | client-side index per device (device-local, slow) |
| Link previews | server fetch URL | *client* fetches (leaks URL to recipient's network — acceptable) |
| Media dedupe | global content-hash | per-pair/per-upload only (WhatsApp's consequence, inherited) |
| Derivative rendering | server resize | client-side generation |
| Content moderation | classifiers on plaintext | metadata/report-based — the single sharpest loss |
| Unread/inbox/mailbox | server truth | survives: envelope/seq/flags are metadata, not content |
| Multi-device sync via mailbox | mailbox rows are canon | needs the encrypted-device-key-fanout machinery WhatsApp multi-device built (see [WhatsApp case](../03.whatsapp/04-deep-dives.md#5-e2e-encryption-and-what-it-removes-from-design)) |

**Trade-off.** Option A — migrate everything to client-side equivalents: users keep every
feature, but each client reimplements it (search index on a phone, previews leaking
network info, moderation collapsing to metadata), and devices with less capability lose
features. Option B — exempt some features (keep moderation via server-side processing of
encrypted content or trusted-compute): preserves safety, but "server-side processing"
against E2EE content is crypto-fiat and erodes the guarantee users were promised.
**We pick A for user-facing features, and pay the security bill with metadata-only
moderation** — the choice Messenger's actual rollout took (client-side migration for
search/previews; moderation shifting to metadata, reports, and unencrypted-adjacent
signals), because the product already had a decade of feature expectations that B would
quietly break, and B's half-measures are the failure mode that makes E2EE theater.

**Failure mode specific to the migration** — not crypto-failure but *migration* failure:
two regimes live in one fleet for years (some threads E2EE, some not, device version skew),
so a message's "is this decryptable everywhere?" depends on which thread regime it was sent
in and which device renders it. The state machine of features per (thread, protocol
version) is where incidents live; design rollout flags per thread, not per fleet.

## 6. Anti-spam and the abuse surface at messaging-recipient scale

Two external abuse paths distinguish this from a closed-friends product: platform/business
API writers, and content moderation across a searchable corpus.

- **Plaintext era**: content classifiers run server-side on the mailbox stream — the same
  plaintext that powers search powers abuse detection; per-sender limits live at the
  conversation service. This is the same shape as Twitter's ranking/rate-limit problem
  (see [Twitter deep dives](../02.twitter/04-deep-dives.md)) with one crucial difference:
  the *recipient* is the victim whose experience is hurt (unwanted thread message), not a
  follower feed — so limits and reputation key on recipient-reported signals, not just
  sender-graph signals.
- **Mailbox model's abuse coupling** (from [dive 1](#1-the-mailbox-model-one-row-per-message-vs-per-message-recipient)):
  a spam send *costs* the abusers nothing but costs K mailbox writes and K inbox
  reorders for recipients — the cost of abuse is externalized onto the storage fleet.
  Countermeasure shape: pre-write filters gate at fanout (cheap graph/metadata signals can
  run *before* K×fanout), because after the fanout the cost is sunk.
- **E2EE era**: content signals are gone; what remains is metadata (send frequency, account
  age, graph, recipient reports, unencrypted-adjacent features like group membership rate
  change). Detection quality drops — this is the sharpest product cost of the privacy
  migration, and public reporting of the rollout says the same thing.
- **Failure mode**: a third-party business integration leaking into unsolicited sends at
  API throughput (an app bug or a compromised token) — mitigate with per-integration
  budgets, per-recipient quotas, and revocation that can starve a single integration id
  without a fleet-wide deploy.
- **Trade-off.** Option A — block first, review on appeal (strict filters, false
  positives): safe fleet, angry users. Option B — deliver first, retract on report: good
  UX, sunk fanout cost that an already-delivered spam message paid. **We pick A at the
  pre-fanout gate for *prior known* abuser signals, B for novel content** — the gate is
  cheap enough before fanout and too expensive after it, so the aggressiveness budget is
  asymmetric by position: strict where the write is still cheap to block, liberal where
  it is already sunk.
