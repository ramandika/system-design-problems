# ZooKeeper across the case catalog

Where ZooKeeper — and the etcd/Chubby-class coordination tier it defines — appears in the
50 study cases, what role it plays, and how deep each case goes. Companion to
[CONCEPTS.md](CONCEPTS.md) rows "Replication / quorum / consensus (failover, anti-entropy)"
(10/43 cases) and "Lease / lock / fencing tokens" (9/43 cases).

Headline: **Coordination is not a workload in this catalog — it is a trust budget.** Every
appearance is the same move run from the other side of the data path: a tiny consensus-owned
tier decides things that are *slow and rarely decided* — who is master, who holds this lease,
which ranges this node may hand out — and everything hot avoids it. Only one case studies the
service itself; one case runs its actual product analog (Bigtable's Chubby); one case tells the
"external coordination failed at scale" story (Kafka→KRaft, told in [11](11.zookeeper/README.md));
the rest borrow its disciplines — leases, fencing tokens, watch-then-re-read, off-the-hot-path —
without deploying the service. etcd appears exactly once, in [11](11.zookeeper/01-requirements.md)'s
list of Raft-embedded systems ("etcd, Consul, TiKV, Kafka's KRaft"); no case designs with etcd as
a component — the etcd class is represented through its design stance, consensus as a small
embedded library, not as a running tier.

- Cases that mention ZooKeeper / etcd / Chubby-class coordination: **11 / 50**
- Plus one case that borrows the lease-and-fencing discipline without naming any coordination
  service: **1** ([04](04.deployment-system/README.md)) → **12 / 50 total**
- The case itself: **1** (11)
- Designed coordination tier inside another system's architecture: **2** (12, 13)
- Borrowed disciplines, no service on the diagram: **7** (02, 40, 04, 37, 47 substantive;
  28, 27 as one-liners / by-reference)
- "No coordination in the data path" as an explicit counter-pattern: **2** (49, 10)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [11.zookeeper](11.zookeeper/README.md) | The case itself: the service, ZAB, sessions/watches/recipes, and the Kafka KRaft story | Full case |
| [12.bigtable](12.bigtable/README.md) | Chubby: master election, root-tablet pointer, per-server liveness locks, schema/ACLs — deliberately off the data path | Deep |
| [13.kafka](13.kafka/README.md) | Controller/quorum "pattern like ZK": a metadata-only quorum the rest depends on; the ZK→KRaft story lives in [11](11.zookeeper/04-deep-dives.md) | Moderate |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Counter-pattern: no consensus service in the data path; no leader election even between the HA pair | Brief |
| [10.distributed-cache](10.distributed-cache/README.md) | Scope boundary: cross-shard transactions are "a job for the origin stores or a coordination service" | One-liner |
| [02.twitter](02.twitter/README.md) | Snowflake machine-ID uniqueness via ZooKeeper / manual assignment | Brief |
| [40.tinyurl](40.tinyurl/README.md) | Code allocator: pre-allocated ID ranges from a coordinator, ZooKeeper-style; never on the redirect path | Moderate |
| [04.deployment-system](04.deployment-system/README.md) | Leases + fencing tokens for runner claims and rollout leadership — the discipline, not the service | Moderate (not named) |
| [37.ticketmaster](37.ticketmaster/README.md) | Explicitly kept off the hot path; seat holds are leases, not locks; fencing tokens on strip failover | Moderate |
| [47.rate-limiter](47.rate-limiter/README.md) | Quota authority as coordination plane (batched lease grants); config broadcast borrows 11's sequencing ideas | Brief–moderate |
| [28.uber](28.uber/README.md) | Trip ownership: "owner leases state, log supplies recovery" — same shape as 11-style coordination | Brief |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Cross-reference: session/lease liveness, reconnect-storm herds, fencing tokens at tiny scale | Brief (relationship table) |

## Per-case notes

### The case itself

**[11.zookeeper](11.zookeeper/README.md)** — The design thesis: correctness-critical at *tiny*
scale (3–5 servers, ~500 MB heap, throughput ceiling is a disk fsync, not a CPU), because
**every other system's safety depends on it**. Scale unit is the ensemble, not the user:
~200 independent ensembles ≈ 1,000 servers for a big fleet (estimates anchored to the 2010
ATC paper), an ensemble serving ~50K ops/s at 95% reads, ~2,500 writes/s hitting a quorum
fsync ([02-scale-estimates.md](11.zookeeper/02-scale-estimates.md)).
Key decisions in [04-deep-dives.md](11.zookeeper/04-deep-dives.md): ZAB vs Raft (ZAB wins
zxid-as-single-comparable-ordering and a pipelined quorum fsync; Raft wins simplicity — newer
systems pick it because consensus is no longer their whole reason to exist); reads may be
slightly stale by design, with `sync()`/own-write as the escape hatch; the 40-second GC-pause
story — session expiry is probabilistic, so fencing tokens at the guarded resource, not a
longer timeout; watches are one-shot invalidation signals, on purpose; the herd-free
sequential-node election recipe; and the flagship boundary story — Kafka ran on ZK 2011–~2022
(membership, ISR, controller election, ACLs, quotas) until the coordination pattern itself
became the bottleneck (two consistency domains, minutes-long controller failover reloading full
metadata, every metadata write paying an fsync quorum) and KIP-500/KRaft moved metadata into
Kafka's own Raft-ordered log.

### Designed coordination tier (inside another system's architecture)

- **[12.bigtable](12.bigtable/README.md)** — the product analog Chubby (5 Paxos replicas,
  quorum 3), used for exactly four things: one-master lease file, bootstrapping the root
  tablet pointer, per-tablet-server liveness locks, and the schema/ACL home
  ([04-deep-dives.md §3](12.bigtable/04-deep-dives.md)). The load-bearing decision is what
  Chubby does *not* do: the read/write path is coordinator-free (client caches tablet
  locations; a server keeps serving while its lock lives). Trade-off: negligible Chubby QPS
  in steady state vs membership propagating at lease/gossip speed — and Chubby is an
  availability single point of failure (publicly reported: 0.0047% of server-hours lost to
  Chubby unavailability across 14 clusters, 0.0326% worst). The case draws the same line as
  11's KRaft story from the other side: Chubby-class coordination for assignment and lock
  semantics, hot metadata (tablet locations) pushed into Bigtable's own replicated table.
- **[13.kafka](13.kafka/README.md)** — the controller/quorum component is described
  explicitly as "pattern like ZK: a metadata service the rest depends on"
  ([03-architecture.md](13.kafka/03-architecture.md)): a small metadata-only quorum doing
  partition assignment and leader election, while brokers stay thin and consumer-side
  membership lives in `__consumer_offsets` inside the log. The ZK history and the KRaft
  replacement are told in [11](11.zookeeper/04-deep-dives.md), not here — the case inherits
  the corrected design without telling the failure story twice.

### Borrowed disciplines, no service on the diagram

- **[02.twitter](02.twitter/README.md)** — one line that carries the rare-decision rule:
  Snowflake's 64-bit IDs sort by time, and **machine-id uniqueness is via ZooKeeper / manual
  assignment** ([04-deep-dives.md](02.twitter/04-deep-dives.md) §3). Exactly the kind of
  decision to pay consensus for: per-machine, made once at fleet bootstrap, never on the
  tweet write path.
- **[40.tinyurl](40.tinyurl/README.md)** — the ID-generation dive ([04-deep-dives.md](40.tinyurl/04-deep-dives.md))
  picks pre-allocated ranges: each allocator node takes a block (~1M integers) from an
  assigned coordinator, "ZooKeeper-style coordination, see ../11.zookeeper/04-deep-dives.md
  on the sequencer/coordinator recipes", then encodes base62 with a random-permutation
  scramble. The wins track 11's contract exactly: no write-time coordination, survives
  coordinator outages for a long window (blocks are big); the loss is honest (wasted codes,
  one more moving part). Allocator is not on the redirect path.
- **[04.deployment-system](04.deployment-system/README.md)** — the lease/fencing discipline
  without any coordination service: scheduler claims are leases (token, TTL 60 s, heartbeat)
  so a dead runner's job is re-claimable, and every claim carries a **fencing token**
  (monotonic per job) so a runner reporting after reassignment loses
  ([04-deep-dives.md](04.deployment-system/04-deep-dives.md)). One leader per rollout, single
  writer per environment routing weight, fenced. Duplicate delivery is made harmless by the
  discipline instead of by consensus.
- **[37.ticketmaster](37.ticketmaster/README.md)** — the clearest "coordinate in a bespoke
  service instead" case. Explicit rejection: *no distributed-lock-per-seat via an external
  coordinator* during the hot path — a network RTT on the single tightest bottleneck,
  "[though] the same lease/TTL semantics apply — see the lock-service warnings in
  [11.zookeeper](11.zookeeper/04-deep-dives.md)" ([04-deep-dives.md](37.ticketmaster/04-deep-dives.md)).
  Holds are *leases, not locks* ("a lock without expiry is a dead-system-maker"), and strip
  primaries carry a fencing token (epoch bumped on failover) that the log consumer checks —
  "exactly the same idea" as 11's dive 3.
- **[47.rate-limiter](47.rate-limiter/README.md)** — two coordination surfaces, both batched
  off the enforcement path: the quota authority is "the coordination plane for local
  buckets" (lease grant/refresh as deltas every ~500 ms per node, not per request), and the
  config broadcast to the fleet "uses the coordinator/sequencing ideas of
  [11.zookeeper](11.zookeeper/04-deep-dives.md)" — numbered immutable revisions with
  last-known-good pinning ([03-architecture.md](47.rate-limiter/03-architecture.md)).
- **[28.uber](28.uber/README.md)** — one-line by-reference: trip ownership is "assigned by
  trip-id hash; if the owner dies, the next instance rebuilds from the DB checkpoint + event
  log tail" — "the same 'owner leases state, log supplies recovery' shape as
  [Case 11](11.zookeeper/README.md)-style coordination and [Case 13](13.kafka/README.md)
  partition recovery" ([04-deep-dives.md](28.uber/04-deep-dives.md)).
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — relationship-table row
  ([05-related-cases.md](27.ev-charging-network/05-related-cases.md)): borrows 11 for
  "session/lease liveness, half-open-vs-dead, and reconnect-storm herd handling at tiny
  scale, with fencing tokens" — the liveness side of the service, not a deployed tier.

### The counter-pattern: coordination off the data path, or absent

- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — a named non-requirement:
  "No coordination in the data path — no consensus service between alert evaluation and
  scrape", with the cross-ref "same pattern as [ZooKeeper](11.zookeeper/01-requirements.md)'s
  'off the hot path' — availability is per-target, not global"
  ([01-requirements.md](49.monitoring-metrics/01-requirements.md)). The HA pair goes further:
  two identical servers both evaluate and fire, and duplication is collapsed downstream —
  "no leader election between the two Prometheis"
  ([03-architecture.md](49.monitoring-metrics/03-architecture.md)), liveness decided by
  redundancy and dedup instead of consensus.
- **[10.distributed-cache](10.distributed-cache/README.md)** — a scope line that frames what
  coordination is *for*: multi-key atomicity across shards is explicitly out of scope,
  "a job for the origin stores or a coordination service"
  ([01-requirements.md](10.distributed-cache/01-requirements.md)) — i.e. a cache tier is the
  wrong place to be paying consensus on anything.

## The portable pattern

Every substantive use reduces to one contract, cross-linked to
[11.zookeeper](11.zookeeper/README.md) as the reference implementation:

1. **Coordinate only what changes slowly and is decided rarely**: master election, cluster
   membership, config/ACLs, per-node ID ranges, work-claim grants (11, 12, 02, 40, 04).
   The 2010-paper numbers put the ceiling at thousands of ops/s per ensemble — an fsync
   quorum per decision. A coordination write is priced like a fsync, so it must not sit on
   any per-request path.
2. **Keep the QPS uncoupled from the data QPS.** Cold-path-only: Chubby touched at discovery
   time (12), the allocator off the redirect path (40), lease grants batched every ~500 ms
   (47), no coordinator between scrape and alert (49), no cross-shard CAS via coordination
   (10). The trade to accept: membership churn propagates at lease/gossip speed (12), which
   is fine precisely because the coordinated facts change slowly.
3. **Lease/ephemeral liveness is probabilistic — pair it with fencing.** The 40 s GC pause
   with a 30 s session timeout produces two owners; ZooKeeper cannot reach into the frozen
   process, so the *guarded resource* rejects stale tokens (11 dive 3: Kafka/HDFS added
   fencing mechanisms after exactly this bug class; same in 12's two-stop-serving rule, 04's
   monotonic claim tokens, 37's strip-primary epoch, 27's fencing at tiny scale).
4. **Watches are one-shot invalidation hints, not streams.** Zero per-subscription server
   state, so a slow client just misses events — clients must re-read and re-arm, and hot
   paths must minimize watchers-per-path via the sequential-predecessor recipe (11 dive 4).
   The fleet-wide echo: batched config broadcast with numbered revisions and jittered
   re-establishment (47; WhatsApp-style reconnect storms in 11 and 27).
5. **Decide by writing, not by reading.** Coordination reads can be slightly stale by design
   (read scaling is the only lever, via observers); anything safety-relevant observes through
   a sequential *creation* (a write) or sync()/own-write, never from a possibly-lagging read
   (11 dives 2 and 6). Stale "no leader exists" read → two leaders is a recipe bug, not a
   service bug.
6. **The migration out is always the same.** When the decision stream itself becomes a
   high-rate, ordering-sensitive workload — Kafka's per-partition metadata (11's KRaft
   story), Bigtable's hot tablet locations (12) — move it into your own replicated
   store/log and keep the external coordinator for bootstrap, assignment and lock semantics.
   The cost is running the consensus protocol yourself; Kafka accepted it only after ~a
   decade of operating the external boundary.
