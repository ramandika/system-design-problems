# The relational tier across the case catalog

Where a Postgres/MySQL-class OLTP database appears in the 50 study cases, what role it
plays, and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) (storage pillars
union, Tier 2: "Relational database (incl. sharded)", 18/43 cases) and to
[CONCEPTS.md](CONCEPTS.md)'s two other storage unions — this tier is the transactional truth
the cache tier sits in front of and the event log flows out of.

Headline: **unlike Kafka, there is no flagship case here — the relational tier is mostly an
assumption, not a study.** No case investigates the database itself; grep sweeps for
"Postgres"/"MySQL" hit only **4 / 50** cases, and most of those are historical anchors to
real systems rather than designed components. The catalog writes "relational DB", "sharded
SQL", "system of record" — a default, not a decision. The interesting content lives in the
boundary decisions *around* the tier: what small mutable truth stays inside it, and how
everything read-shaped escapes it.

- Cases with a designed relational-tier component: **~10** (04, 28, 32, 34, 36, 38, 39, 42, 43, 37 as the inversion)
- Named Postgres/MySQL mentions: **4** (34 named and deployed; 01, 02, 30 as real-system anchors)
- Explicit "don't put this in a relational DB" decisions: **2** (39's cart, 40's lookup)
- Different plane entirely (client-local relational store): **4** (19, 21, 24, 25)
- Scope boundaries ("no relational joins here"): 40, 49

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [34.reddit](34.reddit/README.md) | The only named Postgres: sharded system of record; derived stores reconcile against it | Core component |
| [39.e-commerce](39.e-commerce/README.md) | Relational core of record (~5–10 TB) + event stream to denormalized read projections | Core component |
| [36.airbnb](36.airbnb/README.md) | Relational `availability` table chosen over an event log for the day grid; bookings are the ledger | Moderate |
| [42.linkedin](42.linkedin/README.md) | Invite state machine in a small relational table; accepted edges in KV blobs | Moderate |
| [04.deployment-system](04.deployment-system/README.md) | Control-plane DB, sharded by org; transactional outbox feeds the event bus | Core component |
| [43.zoom](43.zoom/README.md) | Control plane (meetings, roster, admission) as plain SQL | Brief |
| [38.leetcode](38.leetcode/README.md) | SQL nodes for users, problems, submission history; the verdict log carries the hot path | Brief |
| [32.spotify](32.spotify/README.md) | Social graph as "modest-scale relational data" | Brief |
| [28.uber](28.uber/README.md) | Trip DB = system of record, generic OLTP-grade durability | Brief |
| [37.ticketmaster](37.ticketmaster/README.md) | Inversion: log is source of truth, relational tables demoted to projections | Brief |
| [01.youtube](01.youtube/README.md) | Anchor: metadata lineage is Vitess (sharded MySQL); "shard a Postgres" named as a trap | Anchor |
| [02.twitter](02.twitter/README.md) | Anchor: users-DB read scaling via Vtgate proxies, not replica fan-out | Anchor |
| [30.facebook-messenger](30.facebook-messenger/README.md) | Anchor: inbox store on MySQL/MyRocks — engine choice as a cost problem | Anchor |
| [40.tinyurl](40.tinyurl/README.md) | Boundary: no relational read replicas masquerading as KV | Rejection |
| [21.frontend-calendar](21.frontend-calendar/README.md) | Relational logic moves to the device: SQLite as the local source of truth | Different plane |

## Per-case notes

### Transactional core of record

The recurring shape: a **small** relational core holds the mutable truth that needs
transactions, foreign keys or conditional updates; everything read-shaped is deliberately
outside it. Size is the tell — the relational core is never the big store.

- **[34.reddit](34.reddit/README.md)** — the catalog's only case that names Postgres as a
  deployed component ([03-architecture.md](34.reddit/03-architecture.md)): sharded by post
  ID, system of record for posts, comments, communities, users. Reads go to a Cassandra
  side-store for comment-tree/prefix scans, with an explicit write-ahead rule — "Postgres
  first, Cassandra second" ([04-deep-dives.md](34.reddit/04-deep-dives.md)). The trade-off
  is stated: the adjacency record inside Postgres is the integrity arbiter, so a bug in the
  derived-path writer can fragment a subtree silently; mitigation is that truth can always
  be repaired from Postgres. The tier's consistency cost is paid precisely so derived
  stores have something authoritative to reconcile against.
- **[39.e-commerce](39.e-commerce/README.md)** — the cleanest articulation in the catalog
  ([04-deep-dives.md](39.e-commerce/04-deep-dives.md)): Option B (one denormalized document
  store as truth) loses because "transactional edits against documents are how you get
  silently half-updated SKUs", and the order path needs relational constraints anyway.
  Chosen: Option C, a relational core of record (~5–10 TB per
  [02-scale-estimates.md](39.e-commerce/02-scale-estimates.md) — SKU core, price overlays,
  ATP, promo rules) with the search index and 400M PDP documents as event-fed projections.
  Same chapter rejects relational carts: ~150K writes/s at flash bursts on objects that
  expire worthless means paying relational-writer cost for abandoned-cart churn — carts go
  to a TTL'd KV store, checkout never trusts the cart store alone.
- **[36.airbnb](36.airbnb/README.md)** — a rare case of picking the relational row over an
  event log as write model ([04-deep-dives.md](36.airbnb/04-deep-dives.md)): Option A, a
  relational `availability` table with row-per-day and conditional writes, wins "a decade
  of tooling" (conditional `UPDATE ... WHERE state='open'`, foreign keys to bookings);
  Option B, an append-only event log per listing with the grid as a projection, wins an
  audit narrative — recovered anyway, because bookings/holds themselves are append-only
  ledger rows and the grid reconstructs from them. Chosen: A.
- **[04.deployment-system](04.deployment-system/README.md)** — the tier's atomicity
  boundary put to work ([03-architecture.md](04.deployment-system/03-architecture.md)):
  sharded SQL sharded by org, rows the *authoritative* state of runs/jobs/rollouts, and a
  **transactional outbox row** consumed by a relay into the event bus — status and its
  event publish atomically, so the bus is never authoritative. Sizing explicitly says the
  tier is not the problem: ~50 GB/day and ~1.5K writes/s is "sharded SQL handles this
  trivially" ([02-scale-estimates.md](04.deployment-system/02-scale-estimates.md)); the
  design pressure is consistency *with* the event stream, not volume.

### The money/path state machine (kept intentionally small)

- **[42.linkedin](42.linkedin/README.md)** — the state machine lives where it's small
  ([04-deep-dives.md](42.linkedin/04-deep-dives.md)): invite state (pending/accepted paths,
  transactional invites, "find pending" queries) in relational edge rows with both
  directions indexed; accepted-edge adjacency — hot, big, ~120B endpoints — in KV blobs,
  one blob per member. Explicit verdict: "the state machine lives where it's small, the
  graph lives where it's hot." This is the same relational-core-plus-KV-edges split as 34
  at a different shape.
- **[28.uber](28.uber/README.md)** — the money record is generic OLTP, not named:
  trip records at 20 GB/day (estimate) "retained years: OLTP-grade durability, this is the
  money record" ([02-scale-estimates.md](28.uber/02-scale-estimates.md)); the trip DB is
  quorum-replicated wide-column *or plain relational* by city+time
  ([03-architecture.md](28.uber/03-architecture.md)). The case never decides the engine —
  the requirement (money-record durability + reconstruct-trip) is the point.
- **[32.spotify](32.spotify/README.md)** — social graph as "modest-scale relational data,
  bounded per user" ([04-deep-dives.md](32.spotify/04-deep-dives.md)): the tier appears
  only to be dismissed — bounded-per-user tables stay small, so the fanout problem the case
  would otherwise inherit never materializes.

### Metadata and control planes

- **[43.zoom](43.zoom/README.md)** — control plane as one line
  ([03-architecture.md](43.zoom/03-architecture.md)): "meeting service (SQL): meetings,
  roster, admission" — authoritative low-volume state the media plane never touches.
- **[38.leetcode](38.leetcode/README.md)** — SQL nodes for users, problems, submission
  history ([03-architecture.md](38.leetcode/03-architecture.md)); the hot path (verdicts,
  ranks) deliberately goes to the partitioned log instead —
  [02-scale-estimates.md](38.leetcode/02-scale-estimates.md) keeps the 90-day hot
  submission record in a "SQL/row store" and notes it is trivial, while the verdict path is
  "an append-only stream rather than synchronous SQL writes".

### The inversion: relational as projection, not truth

**[37.ticketmaster](37.ticketmaster/README.md)** is the one case that demotes the tier.
The seat/order/hold/reap state machine lives in the partitioned event log ([03-architecture.md](37.ticketmaster/03-architecture.md) cites 13's "events as the source of truth,
relational tables as projections" pattern); what stays relational is only the cold,
steady-state catalog — "venues, maps, historical orders — ordinary RDBMS + cache, not sized
by the spike at all" ([02-scale-estimates.md](37.ticketmaster/02-scale-estimates.md)). The
hot relational table is banned outright: seat transitions are "a partitioned log, not a
relational hot table". If [34.reddit](34.reddit/README.md) shows the tier as the boundary
everything validates against, 37 shows it as a cold-side projection — same tier, opposite
side of the freshness line.

### Real-system anchors (why cases cite named engines at all)

- **[01.youtube](01.youtube/README.md)** — the two anchors are the catalog's best one-two
  on the tier's limits ([04-deep-dives.md](01.youtube/04-deep-dives.md)): the metadata
  transactional lineage is **Vitess**, YouTube's sharded-MySQL layer (open-sourced),
  while the dimensional warehouse is Bigtable — "don't design one store to do both". And
  [02-scale-estimates.md](01.youtube/02-scale-estimates.md) names the trap directly: people
  estimate QPS and think "shard a Postgres", but the load is bytes through a CDN (~200:1),
  which changes the architecture entirely.
- **[02.twitter](02.twitter/README.md)** — 2023-era anchor
  ([02-scale-estimates.md](02.twitter/02-scale-estimates.md)): the users-DB scaled reads
  toward millions of QPS not by adding MySQL replicas but with hundreds of stateless Vtgate
  (Vitess) proxies on Mesos. Lesson: read scaling for the tier is a proxy/routing job, not
  replica fan-out — worth remembering before "add read replicas" becomes the plan.
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — a MySQL engine note wearing
  a KV costume ([02-scale-estimates.md](30.facebook-messenger/02-scale-estimates.md)): the
  per-user inbox store went HBase → MySQL/MyRocks (~2018), a fight about write
  amplification and SSD write endurance at ~70 PB/year of small rows
  ([04-deep-dives.md](30.facebook-messenger/04-deep-dives.md) draws the same engine-cost
  lesson from Discord's Cassandra→ScyllaDB migration). MySQL-class, but deployed as a
  sharded row store, not a transactional tier — count it honestly as a historical anchor.

### Boundaries and the different plane

- **[40.tinyurl](40.tinyurl/README.md)** — the lookup is key-shaped, so the design
  explicitly rejects "read replicas of a relational DB masquerading as KV"
  ([03-architecture.md](40.tinyurl/03-architecture.md)). [49.monitoring-metrics](49.monitoring-metrics/README.md) draws the analytic edge of the same line: label
  matching over a small label set, "not arbitrary relational joins" — push relational work
  to systems built for it ([01-requirements.md](49.monitoring-metrics/01-requirements.md)).
- **[21.frontend-calendar](21.frontend-calendar/README.md)** — all the relational reasoning
  moves into the client: the local store is the only source of truth for rendering, and it
  is relational (SQLite on mobile, IndexedDB on web) *because* queries are range-based and
  need indexes on interval endpoints ([03-architecture.md](21.frontend-calendar/03-architecture.md)). Same vocabulary, different plane — three sibling cases
  ([19.frontend-chat-app](19.frontend-chat-app/README.md), [24.mobile-chat-app](24.mobile-chat-app/README.md), [25.mobile-newsfeed](25.mobile-newsfeed/README.md))
  land on local SQLite/IndexedDB for the same reason: interval indexes, no network round
  trip.

## The portable pattern

The relational tier in this catalog is defined by how *small* it is kept, and by one
contract for everything outside it:

1. **The strong-consistency boundary is tiny.** What lives in the tier is exactly what
   needs transactions, foreign keys or a conditional-update arbiter — SKU core + price +
   ATP (~5–10 TB, 39), adjacency records (34), invite state (42), run/job rows (04). It is
   sharded by a plain domain key (post ID, org, member) and is never the throughput
   bottleneck (04 says it out loud). Everything read-shaped lives elsewhere.
2. **The tier is the atomicity unit for publishing.** Derived planes get their consistency
   from it, not from the log: "Postgres first, Cassandra" write-ahead (34) and the
   transactional outbox row that makes status and its event one transaction (04). If the
   log is authoritative anyway (37), the tier demotes to cold-side projections — only the
   hot path never goes through a relational write.
3. **Small mutable truth and hot bulk truth split; the state machine stays with the
   relational rows** — invites in a table, accepted-edge graph in KV blobs (42); the
   relational grid with an append-only booking ledger carrying audit (36). "The state
   machine lives where it's small, the graph lives where it's hot."
4. **Projections and caches offload reads, reconciled against the tier** — the PDP/search
   projections fed from the event spine (39), feed caches and Cassandra prefix scans
   (34), the ordinary RDBMS+cache catalog (37). The failure mode is always drift in the
   derived copy, and the repair is always "rebuild from the relational truth" (34's daily
   reconciler, 39's rebuild tooling).
5. **Named escapes exist for when the tier itself is the constraint**, and each case picks
   one honestly: read-plane routing (Vtgate proxies over replica fan-out, 02), engine swap
   for write amplification (HBase→MyRocks, 30), or admitting the load was never
   transactional to begin with ("shard a Postgres" trap, bytes went to the CDN, 01). What
   no case does is push the byte-heavy paths through the tier.
