# Load balancing across the case catalog

Where load balancers, edge/entry tiers, health checks, stickiness and placement balancing
appear in the 50 study cases, what each case actually decides, and how deep it goes.
Companion to [CONCEPTS.md](CONCEPTS.md) — **which has no load-balancing row**; this doc is
the first pass at that union.

Headline: **load balancing is the catalog's most universal, least-studied tier.** 19 of 50
cases draw a balancer in the architecture, but not one studies a balancer as a product or
a named subsystem — the front-door tier gets exactly one diagram line and the prose moves
on. Every load-balancing decision that *does* get real analysis is written as a
**placement** decision rather than a scheduling decision: route by where work already sits
(8's KV affinity, 17's voice cache), move state by measured load (12's tablet gossip),
pin the hot tail to dedicated capacity (34), or pick-then-drain with a two-choice rule
(47). And the sharpest authored statements about balancing are all rejections:
"Throughput is bought per-symbol by design, not per-cluster by load balancing" (50),
sticky gateways traded for one store lookup (27), and "load-balancing dashboards across
both servers behind one VIP" named as the bug itself (49).

Mention-based counts (grep + verified in context across all 50 case dirs; these are
phrase/class counts set by this doc's scope and are **not** comparable to
CONCEPTS.md's union-concept counts, which have no load-balancing row):

- Cases where traffic distribution / balancing is part of the text: **19 / 50**
- Full case (balancer studies end-to-end as the subject): **0** — no such case exists
- Core component (an entry or balancing tier designed with a stated trade-off): **3** (03, 27, 43)
- Designed (a balancing/placement mechanism weighed in options-and-trades form): **7** (05, 08, 09, 12, 34, 47, 50 — 50 as a rejection)
- Moderate (a named anti-pattern / cautionary verdict): **1** (49)
- Brief (one to three lines, or health-check concept only): **6** (01, 11, 17, 26, 35, 48)
- Boundary (mentioned only to refuse the framing): **2** (07, 28), plus 30 folded into 03's note

False-positive filtering (counted nowhere above; greps were case-insensitive over all 50 dirs):

- `balance` → ledger/account balances in 24, 29, 36, 44; shard *rebalancing* (a store
  operation, not a traffic tier) in 02, 10, 15, 40; "unbalanced drafting" in 15. 12 balances
  *tablets* (store-side; counted, see boundary note there).
- `LB` in diagrams → "leaderboard service" in 38.leetcode; 38 has no balancer.
- `L4`/`L7` → 39.e-commerce's "L4" is an engineering-degradation-ladder level, not a tier.
  Real L7 usage: 09 (edge filter) and 05 (L7 LB).
- `drain` → outbox drain (19, 24, 41), GPU/queue drains (16, 31), "drains batteries" (28),
  flood drain burst (27). Real connection-drain content lives in 03 and is owned by
  DEPLOYMENT-SAFETY.md; see boundaries.
- `sticky` → experiment bucketing (20), UI stickiness (19), "cooperative-sticky
  rebalancing" (13, consumer groups — KAFKA.md owns).
- `consistent hashing` → a **sharding** technique in 02, 05 (data services), 10, 33, 40 —
  owned by the store docs (REDIS.md, CASSANDRA.md, REALTIME-TRANSPORT.md); excluded here
  per this doc's boundary, except where load balancing itself is the point (47).
- `gateway` → API/payment front-ends that terminate auth, not traffic (07, 39, 44, 47).
- Health-gated deploys (27 OTA waves, DEPLOYMENT-SAFETY.md cases) → rollout health, not
  pool health.
- Admission control / waiting rooms (37, 39, 08) → load *shedding* under flash load, a
  distinct concept with its own CONCEPTS.md pattern-map row; excluded.
- Fair scheduling in queues: 04's "drained weighted round-robin" per-tenant dequeue and
  38's round-robin queue discipline are queue fairness, not balancer scheduling; noted as
  boundary color, not counted.
- 31.netflix's anycast hit is inside its exclusions ("General CDN internals … —"); 32's
  gateway hit is the payment gateway.

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [01.youtube](01.youtube/README.md) | "Global LB / DNS geo-route" — one diagram line, never revisited | Brief |
| [03.whatsapp](03.whatsapp/README.md) | Stateless gateway fleet + user→gateway registry; canary + drain as protocol design | Core component |
| [05.discord](05.discord/README.md) | L7 LB sticky by user hash; registry vs deterministic gateway placement weighed | Designed |
| [07.blob-store](07.blob-store/README.md) | Chunk placement explicitly a reliability decision, "not a load-balancing one" | Boundary |
| [08.chatgpt](08.chatgpt/README.md) | Prefix-affinity routing of turns to KV-holding workers, recompute fallback | Designed |
| [09.cdn](09.cdn/README.md) | Anycast as the base balancer + DNS steering for the tail; origin-shield LB | Designed (CDN.md owns the system) |
| [11.zookeeper](11.zookeeper/README.md) | Connection-aware LB pins a client to a far-behind observer | Brief |
| [12.bigtable](12.bigtable/README.md) | Master tablet balancing by measured load, not tablet count | Designed (store-side) |
| [17.elevenlabs](17.elevenlabs/README.md) | Voice-affinity-biased GPU dispatch; degrade to load-on miss | Brief |
| [26.google-maps](26.google-maps/README.md) | Telemetry loop as an emergent shard health check | Brief |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Stateless acceptors; sticky gateways explicitly rejected | Core component |
| [28.uber](28.uber/README.md) | Regional gateways do backpressure, not balancing | Boundary |
| [34.reddit](34.reddit/README.md) | Balance-vs-locality shard split; pinning the hottest mega-communities | Designed |
| [35.tiktok](35.tiktok/README.md) | Client batching keeps the edge LB's request rate modest | Brief |
| [43.zoom](43.zoom/README.md) | Relay allocator picks an SFU by region + load, caps per node | Core component |
| [47.rate-limiter](47.rate-limiter/README.md) | Two-choice chunk pick (power-of-two-choices) + rebalance sweeper | Designed |
| [48.google-search](48.google-search/README.md) | Per-shard inclusion rate, not p99, as the health metric | Brief |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Balancing an active/passive HA pair behind one VIP = anti-pattern | Moderate |
| [50.stock-exchange](50.stock-exchange/README.md) | Capacity bought per-symbol, not per-cluster by load balancing | Designed (rejected) |

## Per-case notes

### The connection plane: sticky vs stateless gateways

This is where the catalog has genuinely argued about a balancer. [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) owns the connection systems; this doc owns the distribution
question — how a socket pins to a box, and what that pin costs.

- **[03.whatsapp](03.whatsapp/README.md)** — the reference shape: gateways are stateless
  acceptors, and a "who is connected where" registry maps user_id → gateway instance,
  "updated on connect/disconnect" ([04-deep-dives.md](03.whatsapp/04-deep-dives.md)).
  The balancer never needs affinity because push routing is a lookup, not a pin. Gateway
  restarts "drop up to ~1M sockets each"; clients reconnect with backoff, so "deploy
  canary + drain is part of the protocol design, not an ops afterthought."
- **[05.discord](05.discord/README.md)** — the counter-shape: the gateway tier stores
  real per-session state (auth, zlib context, shard assignment), so
  [03-architecture.md](05.discord/03-architecture.md) labels the ingress L7 LB
  "geo/sticky by user hash". [04-deep-dives.md](05.discord/04-deep-dives.md) weighs
  Option A (WhatsApp-style registry: "deploys and reshard are trivial", loses a hot
  metadata cluster and a registry outage = chat outage) against Option B (deterministic
  consistent-hash placement: O(1) publish routing, loses mass-stranded sessions on node
  death); chosen is a hybrid — large-count shard ring where a 1024-shard map balances
  publish routing, per-shard (not per-node) failover. Cost stated: "shard ownership moves
  are a coordinated event."
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — the stickiness
  trade-off stated in one line
  ([04-deep-dives.md](27.ev-charging-network/04-deep-dives.md)): stateless acceptors +
  device shadow cost "one extra store lookup per connection vs the alternative — sticky
  gateways — which turns every gateway deploy into a reconnect storm we'd create
  ourselves. We pick the lookup." Also the only case that names a classic L4 balancer
  job plainly: reconnect-storm cost is "(a) TLS handshakes amortized by load balancers".
- **[28.uber](28.uber/README.md)** (and **30.facebook-messenger**, which reuses 03's
  shape verbatim) — regional gateway tiers that exercise backpressure (drop position
  samples, never lifecycle events) rather than balancing; no distribution mechanism is
  designed. Boundary: the transport plane is REALTIME-TRANSPORT.md's.

**[43.zoom](43.zoom/README.md)** is the one case with a *balancer as a designed
controller*: the relay allocator "picks an SFU (region + load), caps sessions per node,
handles failover reassignment" ([03-architecture.md](43.zoom/03-architecture.md)) — a
spread-with-cap placement across media nodes per region, entered via anycast. The media
plane is REALTIME-TRANSPORT.md's; the allocator-as-balancer is this doc's. No case
examines round-robin vs least-conn as a balancer policy anywhere; the catalog's only
algorithmic scheduling content is below.

### Placement balancing: measured load and affinity

- **[12.bigtable](12.bigtable/README.md)** — the most explicit balancer-vs-load text:
  tablet moves are driven by "per-server load gossip (tablet count, RPS, disk bytes,
  Compaction debt) — not tablet count alone"
  ([04-deep-dives.md](12.bigtable/04-deep-dives.md)). Option A (balance by tablet count):
  "simple, wrong" — a hot 50 GB tablet and an idle one weigh the same. Option B (balance
  by measured load, chosen): "wins correctness of placement, loses simplicity of
  reasoning", and hot-tablet relief is bounded — one hot tablet is one scheduling unit;
  only a split relieves it. Boundary: this is the store's internal balancer; BIGTABLE.md
  owns the tablet engine, this doc claims only the by-measured-load decision.
- **[08.chatgpt](08.chatgpt/README.md)** — affinity routing as a designed option:
  keep hot KV resident on GPU and "route follow-up turns to the workers that already
  hold the prefix"; wins reuse for free ("TTFT on hit ~1/8 of full prefill", est.),
  "loses: routing is now stateful (affinity table ...), load can skew to cache-holding
  workers" ([04-deep-dives.md](08.chatgpt/04-deep-dives.md)). Chosen: an approximate,
  TTL'd affinity table with full-prefill recompute as the universal fallback — a
  probabilistic hint, never a strict directory, so a miss costs money, not correctness.
- **[47.rate-limiter](47.rate-limiter/README.md)** — power-of-two-choices in different
  clothes: a fat quota key's allowance is split into K chunks across shards, and a check
  "pick[s] two random chunks, spend[s] from the fuller one" — "with the same K chunks,
  load balances like power-of-two-choices — drained chunks are rarely picked once any
  chunk has tokens, so false rejections drop by orders of magnitude over pure-random
  pick" ([04-deep-dives.md](47.rate-limiter/04-deep-dives.md)); loses two reads and a
  "residual imbalance". Composed with a chunk-rebalance sweeper and the lease tier
  (C-first). The catalog's clearest statement that per-key hashing cannot fix skew:
  "no amount of consistent hashing fixes a single hottest key".
- **[17.elevenlabs](17.elevenlabs/README.md)** — brief: interactive dispatch is
  "voice-affinity-biased: prefer a GPU that already has the voice, degrade to a load-on
  miss with an admission check" ([04-deep-dives.md](17.elevenlabs/04-deep-dives.md)) —
  the same affinity-then-fallback shape as 08 at one-line depth.

### Skew: balanced placement vs pinned hot spots

- **[34.reddit](34.reddit/README.md)** — the balance-vs-locality trade-off stated as
  options ([04-deep-dives.md](34.reddit/04-deep-dives.md)): shard by community wins
  locality but a ~50M-subscriber flagship's "vote+read burst is a shard-killer"; shard
  by content wins "perfectly even load" but every community operation becomes
  scatter-gather. Chosen: both, split by plane — and for the unbounded tail, "the top
  ~1K communities by traffic are pinned to dedicated capacity", converting "an unbounded
  skew problem into a bounded, monitored one". Option B alone "wins load balance but
  loses the locality that moderation needs."
- **[50.stock-exchange](50.stock-exchange/README.md)** — balancing refused as a capacity
  unit: "Throughput is bought per-symbol by design, not per-cluster by load balancing";
  the static per-symbol grouping is "regime-dependent, needs repartition maintenance
  windows" ([01-requirements.md](50.stock-exchange/01-requirements.md),
  [04-deep-dives.md](50.stock-exchange/04-deep-dives.md)), and a single hot symbol's
  ceiling is one core — solved by batching/pipelining/throttling *inside* the core, never
  by moving the symbol. The sharpest boundary in the catalog: when fairness-per-key is a
  correctness contract, balance is refused.

### The edge tier: anycast and geo steering

- **[09.cdn](09.cdn/README.md)** — the edge is the balancer. Option A, "anycast with a
  global routing policy", wins BGP-native failover and DDoS spreading; Option B,
  "unicast with DNS/GSLB steering", wins "precise, per-client mapping balancing latency
  *and* load" but adds control-plane complexity and DNS propagation lag; chosen anycast
  as base "with unicast/DNS steering as an escape hatch" for the tail anycast serves
  badly ([04-deep-dives.md](09.cdn/04-deep-dives.md)). The origin-shield LB appears only
  as one Mermaid node ("Origin shield LB: TCP/keep-alive pool"). Boundary of boundaries:
  CDN.md owns the edge system end-to-end (anycast/POP/shield/steering all covered there);
  this doc claims only the reading that anycast *is* a load-balancing tier.
- **[01.youtube](01.youtube/README.md)** — "Viewer ──► Global LB / DNS geo-route"
  ([03-architecture.md](01.youtube/03-architecture.md)) as one line ahead of app
  servers; the watch path's real balancing work happens inside the CDN tier (CDN.md's
  territory). Unexamined, and characteristic of the whole doc.

### Anti-patterns and health checks

- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — the catalog's only named
  anti-pattern: an operator "fixes" an active/passive scrape pair by "load-balancing
  dashboards across both servers behind one VIP — now each panel render lands on a server
  with *different* staleness, and graphs flip between them; dashboards should pin one
  server" ([04-deep-dives.md](49.monitoring-metrics/04-deep-dives.md)). Balancing is also
  a deliberate non-goal in the base design: vertical federation "loses: no automatic load
  balancing (a domain outgrows its server, someone must re-cut)" — kept on purpose to
  avoid consensus and a coordinator SPOF. Meanwhile the monitor's own HA uses
  both-scrape-and-dedup, not a balancer, because alert delivery is idempotent.
- **[48.google-search](48.google-search/README.md)** — a health-metric trap generalized
  from hedge-dropping: a shard legitimately slow from real load gets dropped from every
  query and "systematically missing from results while the system 'looks' healthy
  (p99 is fine!)"; "shard-inclusion rate per shard is the health metric, not p99 alone —
  the same trap as a load balancer that health-checks after the pool has already drained"
  ([04-deep-dives.md](48.google-search/04-deep-dives.md)). No pool is balanced here; the
  case is cited only for its named LB trap.
- **[11.zookeeper](11.zookeeper/README.md)** — a connection-aware balancer as a
  correctness hazard: "an application pinned (by a connection-aware load balancer) to a
  far-behind observer" makes staleness worse than a follower's
  ([04-deep-dives.md](11.zookeeper/04-deep-dives.md)).
- **[26.google-maps](26.google-maps/README.md)** — health check as an emergent property:
  the segment-assignment matcher's unmatched-ping audit is "an emergent health check from
  the telemetry loop" ([04-deep-dives.md](26.google-maps/04-deep-dives.md)) — the same
  pull-model idea as 49's scrape-is-health, discovered rather than installed.
- **[35.tiktok](35.tiktok/README.md)** — one sizing insight: ~24M events/s compress to
  ~500K QPS of HTTP request work because "client batching (…~50 events per POST) [keeps]
  the *request* rate at ~500K QPS mean — the event rate is the pipeline's problem, not
  the edge load balancer's" ([02-scale-estimates.md](35.tiktok/02-scale-estimates.md)).

## The portable pattern

Load balancing as a *service* is invisible in this catalog; what recurs is a small set of
placement rules:

1. **Draw the front-door tier; refuse to design it.** Every case puts a global
   LB / edge entry ahead of tier one (01, 09, 05, 39's API gateway) and spends its
   analysis one tier lower. The one designed exception is where entry must fail over
   globally — anycast + steering (09).
2. **Balancing is really placement.** The studied decisions route to where work already
   sits (08's KV, 17's voice cache), move state by measured load rather than count (12),
   spread-with-caps under a hard per-node limit (43), or spend randomness to fix skew
   (47's two-choice).
3. **Stickiness is a trade, and the deciding variable is per-session state size.** Big
   per-socket state makes balancer affinity the design's spine (05: sticky by user hash,
   costs a coordinated reconnect on node death); small/absent state makes sticky a trap,
   traded for one lookup (27: "We pick the lookup") or a registry (03). The catalog never
   picks sticky for stateless sessions.
4. **Hash-only placement cannot save a hot key; pinning can.** Consistent hashing is a
   sharding tool (store docs own it); against a single hottest key it is explicitly
   insufficient (47), and the tail answer is dedicated capacity, not better spread (34).
5. **Health checks are per-subsystem semantics, not one shared liveness bit.** Scrape-as-
   health (49), shard inclusion rate rather than p99 (48), telemetry-loop emergent
   health (26). And never balance across an active/passive pair (49's VIP anti-pattern).
6. **When per-key capacity is a correctness contract, refuse balance** (50) — move the
   per-key work inside its unit and throttle there instead.

Where the pattern is the wrong tool:

- **Anycast/edge routing is a CDN product feature**, not a generic pattern — 09's POP,
  cache keys, shielding and steering are CDN.md's; a non-CDN service that copies them is
  rebuilding a CDN for no reason (31 keeps the whole question out of scope).
- **Connection-plane balancing is a socket question.** For socket gateways,
  REALTIME-TRANSPORT.md owns the systems (03, 05, 28, 43, 30); this doc only takes the
  distribution and stickiness decisions above. Failover of single-writer components
  (leader election, standby promotion) is CONSENSUS-AND-QUORUM.md and ZOOKEEPER.md's
  topic — a balancer never promotes stateful masters in any case here.
- **Partition→consumer assignment is the same pattern in different clothes** (13.kafka's
  rebalance, cooperative-sticky assignment) but KAFKA.md owns that model; it is not
  counted here.
- **Consistent hashing, hash rings and slots are store-side sharding** (10, 02, 33, 40,
  05's data services) — REDIS.md / CASSANDRA.md territory.
- **Admission control and flash-load shedding** (37's waiting room, 39's degradation
  ladder, 08's tiered admission) are a separate discipline: they decide *whether* a
  request enters, not *where* it lands, and CONCEPTS.md's pattern map already has that
  row. Connection drain during deploys belongs to DEPLOYMENT-SAFETY.md (03's
  "canary + drain" is listed there).
