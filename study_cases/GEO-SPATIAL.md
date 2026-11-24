# Geo & spatial indexing across the case catalog

Where geo / spatial indexing and location technology — geohash, S2 cells, H3,
quadtree/R-tree/kd-tree, cell-shard partitioning, radius queries, proximity matching,
map matching, geo-fences, raster/vector map tiles, tile serving — appear in the 50 study
cases, what role it plays, and how deep each case goes. Companion to
[CONCEPTS.md](CONCEPTS.md) (its "geo" row lives *inside* the 23/43 sharding/partitioning
union — that counts partitioning technique, not geo *queries*; the two numbers are
different unions and should not be added to each other).

This doc is the carved-out slice [SEARCH-INDEX.md](SEARCH-INDEX.md) deliberately excluded
from its count boundary: the **geo/spatial structure itself** (26's geo index shard and
S2-cell sharding, 36's geo postings, 28's spatial matching — named in SEARCH-INDEX.md's
false-positive/boundary list). Where a geo index sits *inside* a text engine
(26's POI search, 36's listing search), this doc owns the geo machinery and cross-links
SEARCH-INDEX.md for the engine around it.

Headline: **in this catalog geo technology serves exactly three masters, and every case
states which one it serves: cells as a *query-time index* ("find members of a cell"),
cells as *posting lists inside another engine*, and cells as a *sharding key* (locality,
parallelism, blast radius).** Only one case (26) makes geo the product; two design a geo
index component; the index-for-candidates-exact-scoring-second rule is stated almost
verbatim in both.

- Cases that use geo/spatial/location technology: **5 / 50** — *mention-based count,
  including one-line refs; distinct from CONCEPTS.md's sharding union of 23/43, which
  counts geo only as a partitioning technique across many more cases*
- Full case (the geo system *is* the product): **1** (26 — tiles, S2 sharding, map matching)
- A geo index is a designed architectural component: **2** (28, 36)
- Brief / client-side mention only: **2** (29, 22)
- False positives filtered out (grep hits that are not geo): **10 files**
  (03, 05, 25, 33, 37, 43, 48, 49, 50, 09 boundary — see note at the end)
- Zero cases in the catalog design **geo-fences**; "kd-tree" appears once, as a
  parenthetical alternative in 29. Neither term gets a per-case entry.

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [26.google-maps](26.google-maps/README.md) | The case itself: tile serving (raster vs vector), S2-cell sharding of everything data-ful, HMM map matching | Full case |
| [28.uber](28.uber/README.md) | Geo index shard (H3-style) + proximity matching engine + surge tiles, one geography three features | Deep (designed component) |
| [36.airbnb](36.airbnb/README.md) | Geo postings *inside* the text search engine; viewport → cell-set union | Deep (designed component) |
| [29.uber-eats](29.uber-eats/README.md) | Courier candidate prune by geohash/kd-tree cells; geo-sharded live supply store; map matching by reference | Brief |
| [22.frontend-uber](22.frontend-uber/README.md) | Client-side map: SDK/vector-vs-raster tile choice + corridor tile prefetching | Component (client-side) |

## Per-case notes

### The case itself

**[26.google-maps](26.google-maps/README.md)** — the only case where geo is the product.
Three geo surfaces in one system ([02](26.google-maps/02-scale-estimates.md)): tiles at
~10.4M QPS average / ~16M peak at the edge (~900B tile requests/day = 1B DAU × 3 sessions
× 300 tiles ÷ 86,400 s; ~22.5 PB/day egress), search over ~100M POIs (~230K QPS peak,
1–2 regional shards per query), and map matching at ~700K pings/s average / ~2.5M peak.
Key decisions in [04-deep-dives.md](26.google-maps/04-deep-dives.md):

- **Raster vs vector tiles (dive 1)**: raster (Option A) wins trivial serving and
  pixel-identical output, loses per-locale/per-style re-rendering — the cache key explodes
  to `tile × locale × style × day`. Vector (Option B) wins one tile for every locale/style
  and dynamic client-side features, loses client rendering cost and per-platform drift.
  Chosen: **hybrid** — raster for embeds/slow devices, vector for the interactive client —
  explicitly to collapse the cache-key space toward `tile × epoch`, because at 10M QPS
  "the hit ratio is the product". Pregeneration for tiles ≤ z12 and hot regions;
  on-demand with request coalescing for the cold tail. Failure mode: a style deploy bumps
  the key epoch planet-wide → staged per-region rollover with pre-rendered new keys.
- **S2 cells as the universal sharding key (dive 6)**: road-graph regions, segment
  speeds, POI index shards, tile origin shards, ingest partitions, and the map-matcher
  fleet are all keyed by a hierarchy of S2 cells — deliberately *as partitioning, not as a
  query structure*, because the raw corpora are tiny (graph ~10 GB, speeds ~5 GB, POIs
  ~200 GB) and the reason to shard is write/compute locality: a Munich ping is matched on
  a Munich worker with the Munich graph in RAM; CRP customization parallelizes by region;
  geo-sharding is operational isolation. **The leak is routes**: minor roads stay
  region-local, arterials/highways (a few percent of edges) are globally replicated as the
  top CRP tier, so a LA→NY route is a global-tier corridor plus per-region sub-queries —
  the alternative (pure regions + cross-region RPC at query time) multiplies tail latency
  across dozens of hops. Shards are balanced by road count, not area (density varies
  ~10⁴×; Manhattan vs outback), and split by lower-tier S2 cell without renumbering.
- **Map matching (dive 5)**: nearest-segment-per-ping (Option A) is instant and wrong —
  parallel carriageways and urban canyons zigzag assignments. Chosen: **HMM + Viterbi over
  ~30–60 s session windows**, candidates found via S2 cell → segment index at ~50 m
  radius; win is direction-correct assignments, cost is per-window compute (a multi-thousand-core
  fleet if universal — hence conditional + sampled matching, ~90% of value for a fraction
  of the fleet). Map staleness is *observed through the matcher* (sustained unmatched-ping
  clusters per cell ⇒ audit the graph). F2's place-search text engine (FST/trie,
  Levenshtein automata, S2-cell shard fan-out of 1–2) is indexed in
  [SEARCH-INDEX.md](SEARCH-INDEX.md); the S2-cell *slice* mechanism is this doc's.

### Designed geo components

**[28.uber](28.uber/README.md)** — the catalog's reference implementation of a geo index
shard and proximity matching. Sized in [02](28.uber/02-scale-estimates.md): 1M online
drivers updating at ~140K updates/s ([01](28.uber/01-requirements.md)), ~12.5K geo-index
query/s average / 25K peak (50K map-view clients × 1 query per 4 s), each reading
~200–400 cells' latest positions. Decisions in [dive 2](28.uber/04-deep-dives.md):

- **Index choice, all three options priced**: flat geohash grid (A) wins O(1) updates and
  free prefix sharding, loses non-size-uniform cells and neighbor expansion; S2/H3
  hierarchical hex (B) wins uniform neighbor distance and parent/child coarsening
  ("publicly reported" — Uber's H3 hexagonal indexing work), loses conversion complexity
  (boundaries shrink but do not vanish); in-memory quadtree/R-tree (C) wins exact
  bounding without expansion, loses rebalancing under 140K updates/s and a poor sharding
  story (a splitting node moves machines). **Chosen B with geohash-flat storage semantics**
  — `cell_id → set of driver ids` per region shard — because one geography serves
  *matching, map views, and surge tiles* at once, and hierarchical cells directly
  implement the expanding-window policy (start at res-8 ≈ 0.46 km edge / ~0.74 km², climb
  to coarser parents when empty).
- **"The index is a filter, never the answer"**: the residual boundary falseness (a driver
  90 m away in the adjacent cell, a driver 800 m away in the same cell) is fixed by
  scoring candidates on true distance/ETA after cell selection — the identical rule 36
  reaches independently.
- **Matching engine (dive 3)**: expanding-resolution geo query → filter → score by
  passenger ETA → offer, with the reserve living as a compare-and-swap
  (`available → offered-to(trip)`) **in the geo-index shard that owns the driver's cell** —
  single-writer by cell partition, so no distributed transaction. Batched assignment
  (Option C, bipartite over a ~2–5 s window) for dense contention, greedy fallback
  (Option A) when sparse — the pathological case for each is exactly the other's good case.
- **Surge tiles (dive 4)**: the *same* H3 cell hierarchy is the aggregation key for
  supply/demand ratios; surge (Option A precomputed tiles vs B compute-at-quote) is chosen
  as A — precomputed tiles with damping + hysteresis — because the control loop is
  stateful across quotes, not because inline math is expensive. Note the boundary between
  word senses: 28's "tile" is an **aggregation cell**, not a render tile (26, 22).
- Geo-sharding doubles as blast-radius sharding ([03](28.uber/03-architecture.md): a
  Mumbai spike cannot consume New York's matching budget); driver location streams feeding
  this index are the delivery-guarantee split detailed in
  [KAFKA.md](KAFKA.md) (lossy position samples vs must-not-lose lifecycle events) and
  [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md).

**[36.airbnb](36.airbnb/README.md)** — geo *postings inside the text search engine*, not a
separate geo database ([dive 1](36.airbnb/04-deep-dives.md)). Corpus ~80 GB, RAM-resident
across 8M listings ([02](36.airbnb/02-scale-estimates.md)), so this is a query-shape
problem, not a distributed-storage one. Decisions:

- Each listing is posted to the identifiers of its covering cells at **several
  precisions** (geohash/H3-like); a viewport query expands to a *cell set* — children over
  dense regions, parents over sparse ones — takes the union of postings, then
  point-in-polygon verifies against the true viewport shape. The explicit trade-off:
  finer cells make short posting lists but explode expansion near boundaries; coarser
  cells serve more per list but return junk to verify. Chosen: mid-precision
  (few-hundred-meter cells) with the **parent-inclusion trick** — parents only around
  viewport edges — "because post-verification of a few false candidates is far cheaper
  than expanding 50 children for sparse areas". Same engine as 39's postings and 26's POI
  index; the geo layer's role, cell-family trade-off, and engine placement are the
  SEARCH-INDEX.md / this-doc boundary.
- Hot metro cells make one posting list the hottest object (Lisbon in July) — solved by
  replica-fan by shard-per-city, the same playbook as 28's hot airport cell.
- Cache keys are normalized to a cell-tree closure before hashing, so panning queries
  (near-identical viewports at jittered offsets) hit a bounded key space; the price is
  slightly-wrong tile edges for panning users, accepted as bounded.

### Brief mentions

- **[29.uber-eats](29.uber-eats/README.md)** — courier allocation prunes to ~50 candidates
  per unassigned order by geo-proximity — "(kd-tree / geohash cells in the supply index)"
  ([dive 2](29.uber-eats/04-deep-dives.md)) — the one geo-structure sentence in the case;
  the interesting work is scoring and commit timing, not indexing. The **location
  pipeline** runs a *geo-sharded live supply store* (in-memory + Redis mirror,
  [03](29.uber-eats/03-architecture.md); store mechanics in
  [REDIS.md](REDIS.md)) — here geo is purely a **sharding key**: no radius query is posed,
  the cells just pin couriers to shards and give each order's dispatch loop a bounded read
  set. Last-mile GPS drift is named as map-matching ([dive 3](29.uber-eats/04-deep-dives.md)) with
  the algorithm handed to [26.google-maps](26.google-maps/README.md); the UI answers
  arrival states from the courier app, not position inference.
- **[22.frontend-uber](22.frontend-uber/README.md)** — the *client* half of the tile
  story. [Dive 5](22.frontend-uber/04-deep-dives.md): pick the map SDK once — platform
  native (A) wins basemap quality, loses style control and license economics; embedded
  vector-tile stack (B) wins full style control, smaller cacheable tiles and re-render
  from cache on style changes (dark/night mode at zero network), loses rendering/UX
  parity work and a tile-pipeline dependency; raster + DIY (C) dismissed. Chosen:
  **B-flavored** — vector tiles with a retained custom GL vehicle layer, because the
  map's look and the 60 fps vehicle layer are the product while basemap data is licensable.
  The mirror image of 26's server-side hybrid: what 26 ships as vector for its own first-party
  client, 22 buys as a stack to own the hot rendering path. Tile prefetching (rings +
  ~500 m route-corridor buffer, priority queue, ~200 MB LRU, incremental corridor diff on
  reroute) sits on top; ~1.5–3 MB of tile bytes per session ([02](22.frontend-uber/02-scale-estimates.md)),
  CDN-bound. Server-side counterpart: [28.uber](28.uber/README.md); the tile *bytes* still
  come from the 26-style tile/CDN path, out of scope in both cases.

## The portable pattern

Every substantive use reduces to one contract, with [26.google-maps](26.google-maps/README.md)
as the full-system reference and [28.uber](28.uber/README.md) as the index-shard reference.

1. **Grid cells first, exact distance second — the index is a filter, never the answer.**
   Cells select candidates cheaply; scoring is by true distance/ETA/point-in-polygon
   (28 dive 2, 36's post-verification, 26's 50 m cell → segment candidates before the
   HMM). Every case that designs a geo index states this rule, word for word.
2. **One geography, many features is the payoff that justifies the cell choice.** 28's
   H3 hierarchy serves matching expansion, map-view viewport reads, and surge-tile
   aggregation off the same `cell_id` namespace; picking a second structure per feature
   would lose the free reuse and make pricing and matching disagree about geography.
3. **Cell shape follows the query.** Hexes/H3 for uniform neighbor distance and clean
   parent/child coarsening (28); mid-precision cells as text-engine postings with the
   parent-inclusion trick (36); S2-cell slices as region shards (26); flat geohash where
   O(1) set mutation and string-prefix sharding matter more than uniformity (28's storage
   semantics, 29's supply store). No case reaches for an R-tree/quadtree at write-heavy
   scale — 28 prices Option C and rejects it on rebalancing alone.
4. **Geo-sharding is also blast-radius sharding**: a city maps to specific geo shards,
   matching capacity, and log partitions (28); 29's supply store and 26's regional
   isolation are the same move — the cell grid is the operational partition, independent
   of any query it serves.
5. **Freshness is a query field, not a background property.** 28's queries carry
   `max_age` and drivers older than ~10 s are filtered; 36 tracks a per-listing index
   epoch and audits snapshot age; 26 budgets ping → matched segment → speed → CRP
   weight ≤ 60 s; 29 filters stale couriers and surfaces "courier nearby" instead of a
   dead dot. A stale nearby-listing that cannot be acted on is worse than a shorter list.

**Boundaries:**

- **Grid-cell boundary errors — the three named ways an approximate cell query lies.**
  (a) *Hot border cells*: high-traffic places sit on cell edges and one cell's posting
  list or shard set becomes the hottest object — 28's airport pickup zone (fix: split to a
  finer resolution or hash by driver-id *within* the cell) and 36's hot metro cells
  (fix: replica-fan by shard-per-city). (b) *Neighbor-cell fan-out*: a radius/viewport
  query that honestly expands neighbors pays a multiplicative read cost — the flat geohash
  9-cell expansion 28 prices, the "expanding 50 children for sparse areas" 36 refuses to
  pay, and 26's 1–2 shard fan-out kept bounded by the "immediate neighbors for the
  ambiguity margin" rule. Parent-inclusion-at-edges is the standing fix: include coarse
  parents only where the query shape crosses children. (c) *Cell falseness*: a cell match
  proves "possibly within R", nothing more — hence rule 1; any system that treats a cell
  hit as a distance answer will dispatch a driver 800 m away over one 90 m away in the
  next cell.
- **When geo is really just a sharding key vs a query-time index.** The test: does any
  query ask "nearest / within R"? 28 and 36 say yes — the cell is a query-time index (a
  cell→members map, or postings inside an engine). 26 says no for most of its corpora —
  its S2 cells are a *partition key* that buys compute locality, CRP parallelism, and
  operational isolation, and the one place it needs a spatial lookup (map-matching
  candidates) is an explicit small index (S2 cell → segment list), not a general one. 29's
  supply store is the purest sharding-only use: couriers are placed by cell, but dispatch
  reads a bounded candidate set per order rather than posing a radius query. Keeping the
  two roles apart matters because an index-role geo store carries update-rate and query
  fan-out SLOs a sharding role does not, and upgrading a sharding key to an index is a
  different (and unpriced) design decision.
- **"Tile" means two different things, five cases apart.** A render tile (26 dive 1,
  22 dive 5) is a map image/vector payload served from a CDN with cache-key economics; an
  aggregation tile (28 dive 4) is a grid cell of live supply/demand state. Both derive
  from the same cell-grid instinct, serve nothing alike, and fail differently: render
  tiles fail as cache-stampede, aggregation tiles fail as control-loop staleness.
- **Routing/map-matching is a different specialty than geo indexing, and this catalog
  keeps them separate.** 26 owns road-graph routing (CRP) and HMM map matching; 28 and 22
  both consume it as a dependency (matching scores by ETA; the client renders, it does not
  route); 29 hands last-mile map matching to 26 on purpose. Geo indexing answers
  "which points are nearby"; routing answers "how long to get there" — no case merges the
  two subsystems.
- **No geo-fences anywhere.** Zero cases design geofence machinery (region enter/exit
  triggers); 50's order tiers and 09's per-geo POP announcements are the closest spatial
  partitioning, and both are network placement, not spatial indexing — see the false
  positives below.

**False positives filtered out (grep hits that are not geo):** 03.whatsapp's "DH3"
(cryptographic key schedule, not "H3"); 05.discord's "spatial layers" (simulcast video
layering); 25.mobile-newsfeed and 43.zoom's "tiles" (UI display/gallery tiles);
33.instagram's "image tile" (CDN asset); 37.ticketmaster's "spatial-fragmentation"
(inventory metaphor); 48.google-search's "term-proximity" (ranking feature);
49.monitoring-metrics' "stat tiles" (dashboard widgets); 50.stock-exchange's
"proximity" (colocation of exchange access tiers); 09.cdn's "per-geo" (network
announcement scope, no spatial structure). These files are excluded from the 5/50 count;
09.cdn is the only one of them worth a second look and still fails the geo-index test.
