# 02 — Scale estimates

All numbers are **estimates** built from stated assumptions (shown arithmetic); where a
figure is public ("publicly reported ~X"), it is labeled. Design target: **1B DAU** —
Google publicly reports Maps at 1B+ *monthly* users, so 1B DAU is a deliberate upper
target, not an observed number.

## Users and sessions

- 1B DAU; average 3 sessions/user/day, ~8 min each (estimate) → **24B mapped-minutes/day**.

## Tile fetches (the dominant load by two orders of magnitude)

- A panning/zooming session fetches ~300 tiles across zoom levels (256×256 px; 60% never
  re-requested, cached client-side for revisits) → estimate.
- 1B users × 3 sessions × 300 tiles = **900B tile requests/day**.
- ÷ 86,400s = **10.4M QPS average at the edge**. Usage is skewed to daytime of local
  timezones, flattening the global curve; peak-to-average ~1.5× → **~16M QPS peak at edge**.
- CDN/edge hit ratio ~95% (tiles are immutable, hot tiles are hot) → origin tile load =
  5% × 900B = 45B/day = **~520K QPS to origin**, peak ~800K.
- Tile size ~25 KB average (raster PNG; vector tiles ~40 KB at feature-dense high zooms,
  see 04.1) →
  egress = 900B × 25 KB ≈ **22.5 PB/day at the edge** (CDN absorbs it); origin egress
  ≈ 45B × 25 KB ≈ **1.1 PB/day**. Even origin-only is a petabyte-scale daily problem.

## Search & geocoding

- ~5 search/completion actions per user per day (most sessions search once or twice;
  completions are cheap multi-request events) → 5B requests/day → **~58K QPS average**.
- Skew: peak-hour factor ~4× (evening leisure hours cluster globally) → **~230K QPS peak**.
- Each query hits 1–2 regional shards (location-restricted) — shard fan-out is 1–2, not N.

## Directions

- ~2 directions computations/user/day → 2B/day → **~23K QPS average**.
- Strong rush-hour skew (commutes concentrate in local 8–9am/5–6pm): peak factor ~6× →
  **~140K QPS peak**. Each returns 3 alternatives → **~420K routing queries/s peak** of
  underlying graph search. This is the compute bottleneck: ~420K concurrent graph searches
  that each must finish in <500 ms.

## Traffic ingestion

- Suppose 20% of DAU uses turn-by-turn navigation daily: **200M navigation sessions/day**
  (estimate). Average session 25 min → 200M × 25 = 5B navigation-minutes/day.
- Average concurrency = 5B nav-min ÷ 1440 min/day ≈ **3.5M users actively navigating at any
  moment** (rush-hour peak 3–5× that, ≈ 10–17M).
- Ping every 5 s while navigating →

  - pings/day = 5B nav-min × 60 s ÷ 5 s/ping = **60B pings/day**
  - pings/s average = 60B ÷ 86,400 ≈ **700K/s**; at rush hour (concurrency ~12M) =
    12M ÷ 5 ≈ **2.5M pings/s**.
- Ping payload ~200 B (position, speed, bearing, accuracy, session id) → raw ingest
  60B × 200 B = **12 TB/day**. Small data, enormous message rate — the bottleneck is
  fan-in, not bytes: a partitioned ingest bus keyed by road region (one partition per
  geographic shard) keeps the downstream map-matching workers locality-pinned.

## Data at rest (planet corpora, all estimates)

- **Road graph**: ~60M km of global roads (commonly cited estimate). Average segment
  (geometry node to junction) ~200 m → **~300M edge segments, ~150M junction nodes**.
- Packed storage: node ≈ 12 B (id + lat/lon as ints) → 150M × 12 B ≈ 1.8 GB; edge ≈ 24 B
  (target, length, road class, speed category, flags) → 300M × 24 B ≈ 7.2 GB.
  **Raw graph ≈ 10 GB** — almost embarrassingly small. The real cost is the CRP hierarchy
  built on top (partition tables, boundary shortcuts): ~3–5× on-disk → 10 GB × 3–5 =
  **~30–50 GB**, and
  ~3× in RAM for query-hot pages. Add per-region copies & replication → low-terabyte fleet-wide.
- **Live segment speeds**: 300M segments × ~16 B (speed, sample count, timestamp) ≈ **5 GB
  hot set** — fits fully in RAM per region tier; refreshing it is a throughput problem,
  not a size problem.
- **POI corpus**: 100M POIs × ~2 KB record → **~200 GB** primary; the search index
  (FST + per-shard posting structures) ~2–3× that ≈ 500 GB, sharded by region, replicated.
- **Tiles**: full-planet raster pregenerated z0–19 ~1B–10B distinct tiles after deduplicating
  ocean/empty (merging identical ocean tiles at low zoom; most land tiles at low zoom too).
  At ~25 KB → 1B × 25 KB = 25 TB, 10B × 25 KB = 250 TB → **~25–250 TB corpus** (estimate). Vector tiles only where features exist
  (populated land, z10–19) ~1–2B tiles × 40 KB ≈ **~40–80 TB**. Corpus grows slowly (map-change
  rate, see below), so tile **storage is cheap; tile invalidation discipline is the hard part**.

## Write paths (small but load-bearing)

- Map edits: ~1M edits/day (new/closed roads, POI changes) × ~5 KB ≈ **5 GB/day**, but
  each edit invalidates tiles at its location across zoom levels (≈ 20 tiles/zoom × 10 zooms)
  and triggers regional graph re-extraction + CRP re-customization for that region —
  cost is in the **pipeline trigger**, not the bytes.
- POI attribute drift (hours, phone): ~10M updates/day, indexed within the hour.

## Consistency of the top line

10M QPS tiles + 230K QPS search + 140K QPS directions + ~700K pings/s average ingest (2.5M/s at rush-hour peak; the ~3.5M figure above is
  *navigating users*, not messages). Three of the
four are fan-in problems; only tiles are fan-out, and those are CDN-shaped. If the design
gets tiles wrong, the bill is 22 PB/day of egress; if it gets routing wrong, p99 blows.
They fail differently — the architecture keeps them on independent paths deliberately.
