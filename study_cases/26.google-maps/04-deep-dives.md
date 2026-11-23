# 04 — Deep dives

## 1. Tile serving: raster vs vector, and the cache-key explosion

The client needs ~300 tiles per session; the edge serves ~10M QPS of them. The first
decision — what is a tile — is also a CDN decision.

**Option A — raster tiles** (server-rendered 256×256 PNG/WebP):
- wins: client does nothing but blit pixels; identical output everywhere; trivial to serve
  from any edge proxy; rendering cost exists once, offline.
- loses: each map style and locale is baked in. Labels are pixels: the same tile at the
  same coordinates must be re-rendered per locale (English, Japanese, Arabic...) and per
  theme, multiplying the corpus and the cache-key space.

**Option B — vector tiles** (geometry + attribute protobuf/protobuf-like payload; the
Mapbox Vector Tile spec is the public analogue):
- wins: labels and styling are applied by the client, so **one tile serves every locale
  and style** — a massive cache-hit-ratio win at the edge; smaller payloads over dense
  land at high zoom; enables client-side dynamic features (building footprints, live POI
  markers) without new fetches.
- loses: client rendering cost (matters for low-end devices), per-platform rendering
  inconsistency, and a fat client dependency — the tile only becomes a map after the
  client does real work.

**Chosen: hybrid.** Raster for the low-effort surfaces (web embeds, slow devices, the
default path) rendered offline and pregenerated; vector for the interactive map client.
Trade-off made explicit: we accept renderer complexity and per-platform output drift to
collapse the cache-key space from `tile × locale × style × day` toward `tile × epoch`,
because at 10M QPS the hit ratio is the product. The same lesson as the CDN case —
immovable cache keys with bounded cardinality ([09.cdn](../09.cdn/04-deep-dives.md)) —
except here the key explosion is self-inflicted by rendering choices.

**Pregeneration vs on-demand.** Full-planet rendering cannot be re-run on every data change
(~10B tiles at ~25–40 KB ⇒ ~250–400 TB per render pass); pure on-demand would leave cold tiles at multi-second render
latency. Pregenerate: all tiles ≤ z12 (few, heavily shared), all hot regions at all zooms;
on-demand with request coalescing (n concurrent misses of the same key render once) for
the cold long tail, cached after first render. Trade-off: hybrid means two rendering paths
and eventual-format drift between them; a cold-region first-touch user pays an origin
render. Acceptable: cold-region traffic is rare by definition.

**Failure modes**:

- *Style deploy bumps the key epoch globally* → planet-wide cache invalidation → origin
  stampede. Mitigation: staged key rollover per region, pre-render hot-region tiles under
  the new key before switching traffic.
- *Edit storms* (a big road reclassification) invalidate thousands of tiles at once →
  render farm queue grows; serve stale-pixel tiles (previous epoch, flagged to clients for
  soft refresh) rather than 200ms blank tiles.

## 2. Search & geocoding over ~100M POIs: prefix index, fuzzy, location prior

Two coupled problems: text matching (typo tolerance, prefix completion) and relevance
(same name everywhere — "Main Street" exists in thousands of towns).

**Option A — flat inverted index + edit-distance at query time**: exact terms only;
fuzziness computed per query. Wins: tiny index. Loses: interactive prefix completion
(58K–230K QPS, p99 <200ms) cannot afford on-the-fly edit-distance over 100M terms.

**Option B — FST/trie with materialized prefix structure**: finite-state transducer over
all names/addresses; prefix completion walks the automaton and reads per-node top-k
(scored by global prominence); fuzzy matching compiles a Levenshtein automaton against
the FST so edit-distance-1/2 lookups cost a bounded automaton walk, not a scan. Wins:
completion and fuzzy at trie-walk cost, one structure serving both; FSTs share suffixes so
memory stays ~O(text) with a small constant. Loses: rebuild is not incremental — per-update
edits to a shared automaton are awkward, so shards rebuild on batch (minutes of lag), and
RAM per shard is a real budget line (index ~2–3× the 200 GB text corpus region-slice).

**Chosen: B, sharded by region.** A query carries the user's coarse location; each shard
owns an S2-cell slice of POIs, so fan-out is 1–2 shards (own region + immediate neighbors
for the ambiguity margin). Trade-off made explicit: we accept batch rebuild lag (a new POI
is searchable within ~an hour) and regional index duplication to keep every query
1-shard-local; the alternative, one global index, would put every "Alexanderplatz" query on
a single hot tier. Geocoding adds a parsing layer (address string → house number / street /
city structure) before the same index serves candidates — parsing is where international
format variance hurts; we rank parsed candidates with a distance prior rather than trying
to make parsing perfect.

**Failure modes**:

- *Typo floods* (one viral misspelling) — spell-correction candidates come from the same
  FST; a spike of identical fuzzy queries is cached at the geo-frontend keyed by
  (normalized query, region).
- *Closed POIs served live* — POI freshness is a data-pipeline problem, not index: user
  feedback and batch verification flip a `closed` flag; the index keeps them with a
  down-ranked filter rather than blocking on the refresh lag.
- *Ambiguity inversion* — a user searches "springfield" globally; without a location prior
  the ranking is arbitrary. The prior (distance from user, query-log popularity) is the fix;
  its failure mode is prior leakage (searching for a distant city fails) — supported by
  explicit city-name recognition so the prior relaxes when the query names a place.

## 3. Directions on the road graph: CRP-style hierarchical routing

Given: ~300M-edge graph, live weights, ~140K directions requests/s peak → **~420K routing
graph searches/s peak** (3 alternatives each, per 02), p99 <500 ms.

**Baseline sanity check**: bidirectional Dijkstra over 300M edges touches ~10⁵–10⁶ edges
per query — tens to hundreds of ms each, ×~420K graph searches/s. That is not a fleet-size
problem you can buy your way out of; the per-query work must shrink by orders of magnitude.

**Option A — Contraction Hierarchies (CH)**: node-ordering + shortcut precomputation makes
queries near-microsecond. But the preprocessing is part of the *metric*: weights are baked
into shortcuts, so every live-traffic refresh means re-running an hours-long ordering
phase. With weights changing every 30–60 s, CH in its classic form is dead on arrival for
live routing.

**Option B — CRP (Customizable Route Planning)** (Delling, Goldberg, Pajor & Werneck,
publicly published): split preprocessing into:

1. *Metric-independent phase* (rare): partition the graph into ~10K cells with small
   boundary cuts; build a hierarchy of increasingly-coarse partitions; precompute
   boundary-edge shortcut *structure* (which cells connect to which, with placeholder
   weights). Hours of compute, redone only when the graph changes.
2. *Customization phase* (frequent): given current segment speeds, recompute edge weights
   and then shortcut weights cell-by-cell (parallelizable per cell). **Minutes**, on
   commodity cores, for a whole region — designed exactly so live traffic can be baked in.
3. *Query phase*: bidirectional search that leaves the local cell only via precomputed
   boundary shortcuts, then hops the hierarchy. Milliseconds per query on a continent.

**Chosen: CRP-style.** Trade-off made explicit: we pay ~3–5× the raw graph size in
hierarchy structures (per 02: 10 GB graph × 3–5 → ~30–50 GB with partitions/shortcuts) plus a
two-phase pipeline whose consistency must be versioned; we buy the only known architecture
where every traffic refresh reaches *all* queries within minutes and each query is still
sub-millisecond. Speed profiles per time-of-day are a metric — they only feed the
customization phase, which is the whole point of the split.

**Alternatives** (3 diverse routes): run the hierarchy query once, then k-shortest
variants via penalty methods (banned-edge or via-node perturbations) selecting for
*diversity* (different corridor), not just k-th cost. Trade-off: penalty methods give a
good-enough diverse set cheaply versus exact Pareto/k-shortest enumeration, which is
expensive and mostly yields near-identical routes users can't tell apart.

**Failure modes**:

- *Customization overrun* (traffic burst + edit storm racing) → serve the previous
  **versioned weight snapshot**: a query must never mix snapshots mid-route. Freshness
  drops from 30 s to 10 min; correctness holds. Versioning weights is non-negotiable.
- *Partition-boundary artifacts* (unnatural route forced through a cell door) → small
  boundary cuts during partitioning + penalty on repeated cell-door traversal; watch the
  metric "routes crossing the same boundary >2×".
- *Traffic data gap* → fall back to per-time-of-day historical profiles (same customization
  machinery, different inputs); the route is worse, not wrong.

## 4. ETA prediction: ML on top of live traffic ingestion

The ETA pipeline is two layers with different jobs:

- **Layer 1 — live segment speeds** (the product of 04.5's map matching): rolling per-segment
  speeds with a 30–60 s refresh, plus per-time-of-day historical profiles filling gaps.
- **Layer 2 — route-level ML correction**: given the candidate route and departure time,
  features such as (departure hour × day-of-week × segment class), incident flags, weather,
  historical distribution of traversal times — predicting the remaining-time curve, not
  just a scalar. Serving follows the model-serving discipline in
  [14.ml-production-system](../14.ml-production-system/04-deep-dives.md): versioned
  features, shadow-scored model rollouts, and a fallback (Layer 1 + historical profile)
  when the model service is degraded.

**Ingestion throughput budget** (from 02): ~700K pings/s average, ~2.5M/s peak — but
*map matching is expensive*, so not every ping gets the full treatment:

- Coarse filter drops impossible pings (outdoors accuracy >200 m, speed > 250 km/h,
  stationary duplicates).
- Full HMM map matching runs on sessions whose pings disagree with the last known segment,
  on roads flagged as congestion-relevant, and on a sample of everything else; the rest are
  aggregated statistically into cell-level congestion signals (useful for the traffic *color
  layer*, not for rerouting).

Trade-off made explicit: full matching on every ping gives the densest ground truth but
budgets thousands to tens of thousands of extra CPU-cores at peak (arithmetic in 04.5); sampling + conditional
matching gives ~90% of ETA value for a fraction of the fleet. We pick conditional matching
and accept sparser coverage on rural roads — where there is no congestion to detect anyway.

**Freshness budget**: ping → matched segment → aggregated speed → CRP weights ≤ 60 s
(ingest bus is stream-ordered per region shard; the aggregator keeps rolling windows so
aggregation never waits for a batch boundary). The ETA error a user perceives is dominated
by *coverage* (does this street have data at all?) more than by the model; hence the
layer-1/layer-2 split rather than "one model over raw pings".

**Failure modes**:

- *New incident undetected* → ETA optimistic by 10–20 min; detection combines speed-drop
  alarms over matched pings with incident feeds; until detected, the historical profile is
  the honest prior.
- *Feedback loop collapse*: if users are routed around a slowdown, the slowdown's road
  receives fewer pings and "recovers" on paper. Mitigation: weight matched pings by
  route-plausibility, not by how many users we happened to send there — and treat a segment
  with a dropping sample count as *unknown*, not *fast*.

## 5. Map matching: GPS pings → road segments

A ping says "5.1 m accuracy, here-ish". Traffic needs "this car was on segment E, moving
at 34 km/h". This is noise-robust assignment over time, not one ping at a time.

**Option A — nearest segment per ping**: instant, cheap, and wrong — across highways
running parallel, underpasses, and urban canyons it zigzags between carriageways and
poisons segment speeds with the opposite direction's vehicles.

**Option B — Hidden Markov Model over a session window**: states = road segments near each
ping (candidates found via S2 cell → segment index, ~50 m radius); *emission* probability =
GPS error model (σ≈10–30 m by reported accuracy, worse in canyons); *transition* probability
= plausibility of moving between segments in elapsed time (route distance ≤ max plausible
speed × Δt). Viterbi over the window returns the most plausible trajectory. Wins: direction
and parallel-road disambiguation come free from the temporal structure. Loses: it needs a
*window*, so it adds latency and compute; and a bad window start can lock on the wrong
road until enough evidence accumulates.

**Chosen: B, batched per session window (~30–60 s, overlapping)**, with backward Viterbi
so late pings correct earlier assignments before speeds are published. Trade-off: speeds
carry a matching latency (~1 min) — acceptable against the 60 s freshness budget — in
exchange for direction-correct, highway-grade assignments.

**Compute check** (why the sampling in 04.4 exists): per window ~5–12 pings × ~10–50
candidate segments; one forward-backward pass is ~10³–10⁵ cell transitions. At 200M
sessions/day × ~24 windows = ~4.8B windows/day ≈ 56K windows/s average, that is a
multi-thousand-core fleet if applied universally; conditional + sampled matching (04.4)
keeps it bounded. Estimate throughout; the point is the order of magnitude, and it is why
"just HMM everything" is not a design.

**Failure modes**:

- *Urban canyon / tunnel drift* — GPS wanders; emission model must be honest about accuracy
  (wider σ where buildings tall) and the transition prior keeps the trajectory road-shaped;
  where no segment fits, publish *no* data rather than misassigning (a wrong segment is
  worse than a gap: it poisons trust in that road's speeds).
- *Map staleness* — a road closed for construction still exists in the graph; pings
  detouring around it show massive transition penalties and appear as "impossible" routes.
  Detection: sustained unmatched-ping clusters per cell are a signal to audit the graph —
  map staleness is *observed through the map matcher*, which is a nice property, not an
  accident.
- *Session storms* — a stadium emptying generates ~50K correlated sessions; ingest
  partitions are regional so the storm lands on one shard. Correlated sessions produce
  correlated speeds (a real jam — keep it) but the shard must handle 10× its average ingest
  (capacity rule: shard sizing for the event-peaked case, not the average).

## 6. Regional sharding by geography: storage, compute, and the leaky partition

Geography is the natural partition for everything data-ful: road graph regions, segment
speeds, POI index shards, tile origin shards, ingest partitions, map-matcher fleet — all
keyed by a hierarchy of S2 cells.

**Why shard by geography at all?** The graph (10 GB), speeds (5 GB), and even the POI
corpus (200 GB) are small enough to be "replicate everywhere" in raw terms — but their
*write and compute* paths are not:

- Traffic writes (2.5M pings/s peak) and map-matching CPU are both locality-pinned: a
  Munich ping should be matched on a Munich worker with the Munich graph and Munich speed
  store in RAM. Regional sharding keeps read/write hot sets CPU-cache and page-cache local.
- CRP customization is embarrassingly parallel **by region** — the partition *is* the
  parallelization unit.
- Operational isolation: a regional outage, edit storm or event storm (04.5's stadium)
  degrades one region, not the fleet.

**The leak: routes cross regions.** A LA→NY directions query is no single region's job.
Design: the road network is layered — minor roads exist only in their region shard;
arterials/highways form coarse upper tiers that are **globally replicated** (the top tiers
are a tiny fraction of edges — a few percent — so replication is cheap). A long route is
computed as: global-tier corridor → per-region sub-queries pinned to each traversed region.
Trade-off made explicit: the global tier is a consistency coupling (an arterial's weight
must be near-simultaneously correct everywhere — it is in the global copy) in exchange for
bounded fan-out on cross-region queries; the alternative (pure regions + cross-region RPC
during query) turns every long route into a distributed computation with tail-latency
multiplication across dozens of region hops.

**Boundary handling**: junctions get one authoritative home region; boundary segments are
read-replicated into neighbors. Splits: pick cell keys that can split (a growing region
like Delhi/Bay Area splits by lower-tier S2 cell without renumbering others) — sharding
schemes that assume fixed region counts get rewritten every five years.

**Non-uniformity is the real constraint**: density varies ~10⁴× (intersections/km² in
Manhattan vs outback). Cell-based partitioning must be **balanced by road count**, not by
area; hot regions get more replicas of graph + speeds. Failure to do this shows up as
"routing is slow in cities, the exact places users are".

**Failure modes**:

- *Regional shard loss* → routes degrade to global-tier-only paths (highway corridors,
  no local detail, "get off at the wrong exit" quality) — announced degrading, not silent.
- *Hot-region overload* (event + tourism + rush) → load-shed ETA decoration first (routing
  still answers with historical speeds), never shed the route itself; traffic-pipeline
  backpressure drops *sampling coverage* before dropping *freshness* on hot segments.
- *Split/merge mistakes* (a segment assigned to the wrong region after a boundary change)
  → caught by the matcher's unmatched-ping audit (04.5's staleness signal repurposed) —
  again an emergent health check from the telemetry loop.
