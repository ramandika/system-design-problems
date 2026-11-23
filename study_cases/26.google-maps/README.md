# Case 26 — Google Maps

**The system**: a global maps platform for ~1B daily users. Four surfaces work together:
map display (pan/zoom over pre-generated tiles), search & geocoding over ~100M POIs, and
directions with live ETAs fed by a live traffic pipeline ingesting GPS pings from millions
of concurrent navigation sessions.

**Why it's hard**:

- Three load shapes in one product: tile reads are **edge-dominated and enormous** (~10M QPS
  scale of mostly-cacheable static blobs), search is **interactive and sparse**, and routing
  is **compute-heavy** — every directions query runs graph search over a ~300M-edge planet
  road graph, with 3 alternatives each, under 500ms.
- The road graph has **live weights**: congestion changes edge costs continuously, so any
  precomputed routing accelerator must be re-customizable in minutes, not rebuilt in hours.
- ETA quality is a loop: you estimate the ETA, the user drives it, their pings correct the
  traffic layer, which changes every other user's ETA. The correctness of the loop depends
  on **map-matching** raw GPS noise onto road segments.
- Geography itself is the partition: data and compute are sharded by region, but routes
  cross regions, so the partition has to be leaky in a controlled way.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**:

- Serving a giant static corpus through a CDN with cache keys that don't explode (vs the
  media CDN in [09.cdn](../09.cdn/README.md) — same physics, nastier keys).
- Approximate/indexed lookup at interactive rates (FST/prefix index over 100M names) — a
  different flavor of the read-path problem in [02.twitter](../02.twitter/README.md).
- Precomputation with fast re-customization (CRP-style routing) — the "precompute the hard
  part offline, refresh the cheap part online" pattern.
- ML serving inside a tight loop ([14.ml-production-system](../14.ml-production-system/README.md)):
  the ETA model's features are themselves pipeline products.
- Map client + driver telemetry loop (compare the consumer/driver split in
  [22.frontend-uber](../22.frontend-uber/README.md)).

**Try it first**: before reading 04, answer: (1) raster or vector tiles at 10M QPS — what
does the choice do to your CDN cache-key space, (2) why can't you just run Dijkstra with
live traffic weights at ~420K routing queries/s, (3) how does a GPS ping at "somewhere on this city
block" become an authoritative speed for one road segment — and how often?

**Primary sources**: routing precomputation is publicly documented in Delling, Goldberg,
Pajor & Werneck, "Customizable Route Planning in Road Networks" (crp, 2014/2017); Google
publicly reports Maps at over 1B monthly users. All scale numbers in this case are
estimates built up from assumptions in [02](02-scale-estimates.md), not leaked figures;
where a comparable is public (OSM corpus sizes, Mapbox vector tile spec as the vector-tile
analogue) it is labeled as such.
