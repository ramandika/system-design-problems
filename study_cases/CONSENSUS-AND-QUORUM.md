# Consensus and quorum across the case catalog

Where the algorithm pattern — Raft/Paxos/ZAB, quorum reads and writes, leader election as a
mechanism, split-brain fencing via epoch numbers, standby/witness replication, and the choice
*not* to run consensus — appears in the 50 study cases. Companion to [CONCEPTS.md](CONCEPTS.md)
row "Replication / quorum / consensus (failover, anti-entropy)" (union: 10/43 cases) and to
[ZOOKEEPER.md](ZOOKEEPER.md), which covers the *service* level: ZooKeeper/etcd/Chubby as
coordination tiers, watches, sessions/leases, and the KRaft migration story. The engine-tier quorum story is
also linked rather than repeated here: quorum reads/writes as a store-engine trade in
[CASSANDRA.md](CASSANDRA.md), Paxos/Chubby and the consensus-vs-serving split in
[BIGTABLE.md](BIGTABLE.md), primary-replica failover in [REDIS.md](REDIS.md).
This doc covers only
what stays after that boundary: the protocol geometry, its latency price, and every case's
deciding role.

Counts below are **mention-based** (grep over the corpus, read in context, false positives
filtered per §"Filtering") and are not the union counts in [CONCEPTS.md](CONCEPTS.md).

Headline: **consensus is almost never chosen in this catalog — it is either inherited from a
default (the KV/DB stores use quorum replication because their engine ships it) or rejected by
name.** One case studies the algorithm (11: ZAB vs Raft, leader-election recipes, quorum-fsync
arithmetic); six cases *run* a leader/quorum/fencing mechanism in one corner of the system;
six cases reject consensus explicitly (microsecond latency, rebuildable state, single-writer
discipline, or "display surfaces don't need single-copy truth"); five use quorum terms as
replication shorthand or brief mechanism notes. And the classic theory — solving for R + W > N
per key — is never actually worked: the only quorum *arithmetic* in the catalog is ISR
shrinking against `min.insync.replicas` in [13.kafka](13.kafka/README.md).

- Cases touching the algorithm pattern: **18 / 50**
- The case itself (algorithm level): **1** ([11](11.zookeeper/README.md); the service-level
  story lives in [ZOOKEEPER.md](ZOOKEEPER.md))
- Designed mechanism — a leader, quorum-failover, or fencing epoch runs inside the architecture:
  **6** (04, 10, 12, 13, 27, 37)
- Counter-pattern — consensus is named and excluded: **6** (07, 29, 38, 49, 50, 06)
- Theory discussion or brief mechanism mention: **5** (03, 05, 09, 19, 28)

## Summary table

| Case | Role | Depth |
|---|---|---|
| [03.whatsapp](03.whatsapp/README.md) | Replication shorthand: Discord's Cassandra/ScyllaDB analog uses quorum reads/writes | Brief (by-reference) |
| [04.deployment-system](04.deployment-system/README.md) | Leader-per-rollout + fencing epochs; 3-node quorum for the coordinator | Moderate |
| [05.discord](05.discord/README.md) | Quorum reads/writes as store design; rejects cross-region consensus per insert | Moderate (two-sided) |
| [06.apple-app-store](06.apple-app-store/README.md) | Rejects global catalog consensus; single logical writer + async replication | Deep (rejected) |
| [07.blob-store](07.blob-store/README.md) | Scope line: no consensus semantics on the flat-key API | One-liner |
| [09.cdn](09.cdn/README.md) | Leader-per-key election on the cache-miss path for hot objects | Brief |
| [10.distributed-cache](10.distributed-cache/README.md) | Controller quorum promotes replicas; quorum-write option priced; fencing vs split brain | Deep |
| [11.zookeeper](11.zookeeper/README.md) | The case itself: ZAB vs Raft, quorum-fsync math, election recipes | Full case |
| [12.bigtable](12.bigtable/README.md) | Quorum-write replication evaluated against async eventual; Paxos lives in Chubby | Theory (Chubby linked) |
| [13.kafka](13.kafka/README.md) | KRaft-style controller quorum; ISR quorum arithmetic; unclean leader election | Deep (two-sided) |
| [19.frontend-chat-app](19.frontend-chat-app/README.md) | Client-side Web Lock leader election for the outbox single-writer | Brief |
| [27.ev-charging-network](27.ev-charging-network/README.md) | One controller per site is the leader; failover to a conservative default, no quorum | Brief |
| [28.uber](28.uber/README.md) | Money-class trip writes afford quorum replication; telemetry does not | Brief |
| [29.uber-eats](29.uber-eats/README.md) | Order state machine is "a validating owner, not a consensus"; split-brain absorbed by state checks | Brief |
| [37.ticketmaster](37.ticketmaster/README.md) | Strip primaries carry fencing tokens (epoch bumped on failover) | Moderate |
| [38.leetcode](38.leetcode/README.md) | Rejects single-copy consensus for a display surface; staleness disclosed in-product | Moderate (rejected) |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | HA without consensus: two identical scrapers, no leader election between them | Deep (rejected) |
| [50.stock-exchange](50.stock-exchange/README.md) | Rejects per-message quorum commits; standby replay beats sync-replicate; fencing on failover | Deep (rejected) |

## Per-case notes

### The case itself (algorithm level)

**[11.zookeeper](11.zookeeper/README.md)** — the only case that studies consensus as the
product: ZAB vs Raft compared on ordering comparability (zxid) and pipelined quorum fsync;
an ensemble's throughput ceiling is a disk fsync, so ~50K ops/s at 95% reads collapses to
~2,500 writes/s once every decision pays that quorum — the catalog's cleanest statement of
what a consensus round costs, and why everything in this list keeps it off the hot path
([04-deep-dives.md](11.zookeeper/04-deep-dives.md)). Its "herd-free" sequential-node election
recipe is the catalog's only inside-protocol leader-election design. The service-level use
(that quorum inside Kafka, Bigtable, and coordination-as-trust-budget) is fully documented in
[ZOOKEEPER.md](ZOOKEEPER.md) — link, not restatement.

### Designed mechanisms

These cases run one consensus-adjacent mechanism *inside* an architecture that otherwise avoids
consensus: a leader that must exist, a quorum that must be reached on failover, or an epoch that
must kill a zombie.

- **[13.kafka](13.kafka/README.md)** — two layers, both quorum-shaped.
  *Control*: a 3–5 node KRaft-style metadata-only quorum assigns partitions and elects
  leaders ([03-architecture.md](13.kafka/03-architecture.md)); the service-tier reading of that
  quorum is [ZOOKEEPER.md](ZOOKEEPER.md)'s territory. *Data*: per-partition ISR replication
  ([04-deep-dives.md §3](13.kafka/04-deep-dives.md)) carries the catalog's only quorum
  arithmetic — with RF 3 and `min.insync.replicas = 2`, an `acks=all` write survives **one**
  dead broker (ISR = 2 = min.insync, writes still succeed) and fails only when **two** are out
  (ISR = 1 < 2); leaders advance the high-watermark to min(ISR acks). The unclean-leader-election
  fork — availability with data loss vs read-only until an in-sync replica returns — is framed
  as a per-topic product decision, not a protocol default.
- **[10.distributed-cache](10.distributed-cache/README.md)** — the most explicit quorum
  economics in the catalog. Per shard: 1 primary + 2 async replicas; a controller quorum
  promotes on heartbeat lapse (≤ 15 s). The quorum-write option is priced and rejected at
  tier level: WAIT-style sync-to-replica adds a cross-zone RTT — "a few ms typical, several
  times the 1.5 ms N1 op budget" — so it is chosen *only for the session class* (~10% of write
  volume), while the rebuildable object class stays async and accepts that a failover can lose
  acked-in-the-last-interval writes ([04-deep-dives.md](10.distributed-cache/04-deep-dives.md)).
  The failover quorum explicitly does *not* preserve unreplicated acknowledgments — it selects
  the newest surviving replica. Split brain (an isolated old primary still taking writes) is
  closed with controller-quorum fencing: the old primary stops serving when it cannot renew
  its lease.
- **[37.ticketmaster](37.ticketmaster/README.md)** — fencing epoch as a load-bearing
  component. Strip primaries hold a fencing token — "an epoch bumped on failover" — that the
  log consumer checks before accepting writes, so a swapped-out primary answering from stale
  memory can't serve ([04-deep-dives.md](37.ticketmaster/04-deep-dives.md)). Failure
  recovery is not consensus: a dead shard host is rebuilt by event-log replay on a standby.
- **[04.deployment-system](04.deployment-system/README.md)** — the lease/fencing discipline
  as component glue: every work claim (scheduler, rollout leader) carries a monotonic token,
  and a crashed controller's recovery *re-issues the fence with a new epoch*, invalidating
  in-flight verdicts from the crashed instance
  ([03-architecture.md](04.deployment-system/03-architecture.md),
  [04-deep-dives.md](04.deployment-system/04-deep-dives.md)). The fleet-rollout coordinator
  itself runs on a 3-node quorum so a node loss pauses rather than corrupts — the coordination
  *tier* is [ZOOKEEPER.md](ZOOKEEPER.md)'s story; what belongs here is the epoch bump: fencing
  is how single-writer survives crash recovery without a consensus round in the recovery path.
- **[12.bigtable](12.bigtable/README.md)** — two separable layers. Chubby is
  "a Paxos-replicated (5 replicas, quorum 3) lock/file service"
  ([04-deep-dives.md](12.bigtable/04-deep-dives.md)) — that is the coordination service
  (see [ZOOKEEPER.md](ZOOKEEPER.md)). The case also evaluates Multi-Paxos-class *data*
  replication as explicit Option A for multi-cluster sync: losing a region keeps acknowledged
  writes (RPO ≈ 0, automatic failover) but kills the flat-latency LSM write path —
  "every write pays cross-region latency" and availability now requires quorum-interconnect
  health. Chosen: async eventual replicas, last-timestamp-wins merge, operator-run fencing on
  failover — the case's stated reason (same numbers compared in
  [03.whatsapp/04-deep-dives.md](03.whatsapp/04-deep-dives.md)): which side you pick is a
  function of RPO your product absorbs, not a universal best.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — a leader with no quorum
  behind it: one controller per site is "the" writer for power-allocation decisions
  (the single-writer/fencing reasoning of [04](04.deployment-system/README.md) "applied to a
  cabinet, not a cluster"), and controller failover goes to a **conservative default** (all
  stalls capped to a safe fraction of site limit) *until a new leader computes*
  ([04-deep-dives.md](27.ev-charging-network/04-deep-dives.md)). The pattern: when the
  wrong-leader failure is physically dangerous, fail toward safe inaction, not toward
  election speed.

### Counter-patterns: consensus named and excluded

- **[50.stock-exchange](50.stock-exchange/README.md)** — the sharpest rejection in the
  catalog. Kafka-style 3-way quorum-committed inbound log is Option A and loses structurally:
  "per-message quorum commit is ~ms-class and bursty" against a ~50 µs matching budget
  ([04-deep-dives.md](50.stock-exchange/04-deep-dives.md), budget in
  [02-scale-estimates.md](50.stock-exchange/02-scale-estimates.md)) — the exchange even needs
  one symbol = one partition, so replication stays per-message anyway. Chosen: single-active
  sequencer + hot standby tailing the same journal — "no data structure to elect", failover is
  resume-at-sequence-N; total order is one stamp, not distributed agreement. Split brain
  (two writers after a failed failover) is fenced at the journal — the old active's dispatch
  credential is revoked and a journaled epoch/leadership event makes double-dispatch detectable
  in replay. The general rule stated: **replay beats sync-replicate whenever the reducer is
  deterministic** — and determinism is the only replication mechanism that also *proves* the
  standby is identical, which synchronous state plumbing cannot.
- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — consensus excluded in three
  places. Alerts: "HA without consensus" — two identical servers both scrap the same targets;
  leader-elected scraping (Option A) loses because a 30-second election gap is the scenario
  the HA pair exists to survive; duplicate delivery is deduped downstream by fingerprint
  ([04-deep-dives.md §1](49.monitoring-metrics/04-deep-dives.md)). Ingest: independent
  per-server WALs, "no consensus needed" ([03-architecture.md](49.monitoring-metrics/03-architecture.md)).
  Scaling: Prometheus refused to make single-server deployments into cluster deployments
  (consensus, rebalancing, coordinator SPOFs) — the long-term tier was forked into the
  ecosystem instead. The stated price: every consumer of cross-server data must know about
  the duplication.
- **[06.apple-app-store](06.apple-app-store/README.md)** — global consistency rejected by
  arithmetic on the *benefit*. Worldwide consensus per catalog write buys simultaneous
  availability of a release across ~175 storefronts — but the write path is gated by human
  review measured in hours, and only 71K metadata-path writes/day would pay for it; the loss
  side is worse than the cost: active-active with conflict resolution makes "approved in
  region A / pending in B" a trust-system conflict that needs a global arbiter anyway
  ([04-deep-dives.md](06.apple-app-store/04-deep-dives.md)). Chosen: a single logical writer
  (the release gate), async replication, eventual read models — plus the acceptance that the
  authoritative region's outage blocks releases worldwide, "the right failure direction".
- **[38.leetcode](38.leetcode/README.md)** — single-copy consensus *disclosed as not wanted*:
  "instead of chasing exact single-copy consensus for a display surface", divergent views (own
  rank 41 on the page, 37 in the list) are accepted within the freshness budget and said in
  product ("standings may lag by up to ~30 s") ([04-deep-dives.md](38.leetcode/04-deep-dives.md)).
  The truth source is the verdict log; rankings are projections, rebuildable when anti-cheat
  re-runs flip verdicts after the freeze.
- **[29.uber-eats](29.uber-eats/README.md)** — the order state machine is "a validating
  owner, not a consensus" ([04-deep-dives.md](29.uber-eats/04-deep-dives.md)): one service is
  the only writer of assignment transitions; dispatch failures don't split state, they re-offer
  from legal states. Split brain between dispatch and order state after a failover is bounded
  by state-machine legality (re-assignment legal for any order not yet PICKED_UP), not by an
  agreement protocol.
- **[07.blob-store](07.blob-store/README.md)** — the scope-boundary version: tenants may build
  consensus-and-queue systems on the flat-key API, but the API will not optimize semantics for
  it ([01-requirements.md](07.blob-store/01-requirements.md)); locking, appends, and directory
  rename are deliberately absent. Same stance as 10's "coordination is a job for the origin
  stores", one layer down.

### Theory and brief mentions

- **[03.whatsapp](03.whatsapp/README.md)** — the quorum-replication *facts* enter by
  reference to Discord's public store: wide-column, channel+time-bucket partitions, snowflake
  ID ordering, **quorum reads/writes**, publicly reported ~5 ms p99 inserts
  ([04-deep-dives.md](03.whatsapp/04-deep-dives.md)). WhatsApp's own design keeps only the
  consequence needed here: one authoritative writer per conversation sequence.
- **[05.discord](05.discord/README.md)** — two-sided. For the store itself, quorum reads/writes
  are inherited from the Cassandra/ScyllaDB engine
  ([02-scale-estimates.md](05.discord/02-scale-estimates.md)). For *placement* across regions,
  consensus is priced and rejected: a guild is owned by one region at a time because
  "cross-region consensus on every message insert would multiply latency by region-wide RTT
  for a feature that rarely exists"
  ([04-deep-dives.md](05.discord/04-deep-dives.md)) — accepting seconds of in-flight message
  loss on failover, papered over by snowflake-ID dedup.
- **[28.uber](28.uber/README.md)** — the sizing rule in one line: trip writes are 3–4 orders
  of magnitude smaller than driver telemetry, "they can afford synchronous persistence, quorum
  replication, and idempotency keys" ([02-scale-estimates.md](28.uber/02-scale-estimates.md));
  the trip DB is quorum-replicated, money-record durability, everything else a projection
  ([03-architecture.md](28.uber/03-architecture.md)). Quorum is affordable exactly where
  volume is small.
- **[09.cdn](09.cdn/README.md)** — leader election as a cache mechanic, not a state machine:
  on a hot-object miss (one URL at 10M+ req/s), the owning node takes a per-key distributed
  lock ("leader-per-key election"), fetches once, backfills concurrent waiters, and TTL jitter
  (+0–10%) desynchronizes expiries — a lock held only for the fetch duration, hits never touch
  it ([04-deep-dives.md](09.cdn/04-deep-dives.md)).
- **[19.frontend-chat-app](19.frontend-chat-app/README.md)** — the same leader election at
  browser-tab scale: two tabs double-draining the outbox is fixed with a Web Lock
  leader-election single-writer ([04-deep-dives.md](19.frontend-chat-app/04-deep-dives.md)).
  The smallest instance of the rule in the catalog — one writer, or else.

### Filtering notes (false positives excluded)

- `raft` hits that are the substring of "draft(s)":
  [15.ml-ranking-system](15.ml-ranking-system/README.md)
  (team-draft interleaving, a real design but not consensus), [16.dalle](16.dalle/README.md),
  [18.sora](18.sora/README.md), [33.instagram](33.instagram/README.md) (draft renditions) —
  excluded entirely.- `epoch` hits that are data version numbers (tile data epoch in
  [26.google-maps](26.google-maps/README.md), snapshot epoch in
  [36.airbnb](36.airbnb/README.md), permutation epoch in [40.tinyurl](40.tinyurl/README.md),
  score-formula epoch in [34.reddit](34.reddit/README.md), admission "issue epochs" in 37,
  app lifecycle/staleness epochs in [22.frontend-uber](22.frontend-uber/README.md) and
  [24.mobile-chat-app](24.mobile-chat-app/README.md)) — excluded: they tag *content*, they do
  not fence a stale writer. One borderline: [19](19.frontend-chat-app/README.md) uses a
  version epoch in a checkpoint to force full re-sync on mismatch — a freshness guard, not
  fencing; 19 is kept on its leader-election hit instead.
- [04.deployment-system](04.deployment-system/README.md)'s "quorum approval" for broker-access
  (a *human* quorum) — excluded.
- "leader/selection" hits unrelated to election (personalization selection in 31, 14;
  recurrence "masters" in 21; audio "masters" in 32) — excluded.

## The portable pattern

Every substantive run-in reduces to one decision, cross-linked to
[11.zookeeper](11.zookeeper/README.md) for the protocol math:

1. **Choose the writer, not the election.** Nine of the ten cases with a safety stake make
   total order cheap by having exactly one writer — the sequencer (50), the release gate (06),
   the strip primary (37), the rollout leader (04), the site controller (27), the
   conversation shard (03), the catalog authority (06), the order state machine (29) — and
   recover it by replay or lease, not by election speed. Leader election appears only where
   ownership is genuinely symmetric or numerous: partition leaders among brokers (13), the
   per-key coalescing owner (09), tabs on a client (19).
2. **Quorum is priced per write, so it is reserved for the class that cannot rebuild.**
   The catalog's version of R + W > N is a volume split, not an N-split: 10 routes only the
   ~10% session-class writes through quorum semantics (the object class is rebuildable, so
   losing acked writes on failover is "acceptable: origin rebuilds"); 28 sends only the
   3–4-orders-of-magnitude-smaller money records through quorum replication; 13 buys ISR
   quorum per topic. Nobody in this corpus works the classic per-key arithmetic — the trade
   always resolves through "what can be rebuilt from a log or origin".
3. **The durability knobs are a product decision, not an engine default.** 13's
   `acks`/`min.insync.replicas`/unclean-leader-election trade (two brokers out vs
   availability), 12's RPO ≈ 0 vs flat write latency, 05's region-affinity vs RTT — each is
   chosen per data class with the loss named. Quorum never enters this catalog silently.
4. **Split brain is fenced with a monotonic epoch at the guarded resource, and fencing is
   always mentioned after the zombie already exists.** Strip-primary epoch (37), journal
   epoch/leadership event (50), controller quorum lease-fencing (10), re-issued fence with a
   new epoch on recovery (04), the two-stop-serving Chubby rule and operator fencing (12).
   The why and the two-owner bug class live in [11's dive 3](11.zookeeper/04-deep-dives.md)
   and [ZOOKEEPER.md](ZOOKEEPER.md) — this doc records only that every case re-derives the
   same token shape.
5. **When the reducer is deterministic, replay beats sync-replicate.** 50 proves the limit
   case (identical code on identical sequence stream gives a standby that is not just
   up-to-date but provably identical, and it doubles as a divergence canary); 37 rebuilds a
   strip by journal replay; 06 rebuilds read models from the catalog event stream; 29's
   re-offer is legality-check replay. Synchronous quorum/state replication is bought only
   where state cannot be re-derived (28's trip ledger, 10's sessions).
6. **The boundary is the per-message quorum commit: ~ms-class, and everyone on the wrong
   side of their budget rejects it by name.** 50: ~ms against ~50 µs — structurally
   disqualified. 06: a worldwide-consensus RTT on 71K writes/day buying simultaneity nobody
   experiences over an hours-long human gate. 05: region-RTT multiplied onto every message
   insert for a feature that rarely exists. 49: a 30 s election window inside its most
   critical loop. 10: a cross-zone RTT that is ~2–3× the whole 1.5 ms op budget, tolerated
   for 10% of writes only. Consensus in this catalog costs about a disk fsync plus an RTT
   (11's ~2,500 writes/s ensemble is the ceiling made concrete); every case above it drew
   that line and stepped back.
