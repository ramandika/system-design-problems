# 01 — Requirements

## Functional

**F1 — Map display**: pan/zoom over the whole planet at zoom levels ~0–19, with road, POI
and label content drawn per locale and per map style (default, transit, satellite overlay).
Tiles must be cacheable by anyone (CDN, browser, offline pack).

**F2 — Place search & geocoding**: type-ahead completion over POI names and addresses
(~100M POIs, label: estimate); ambiguous-query ranking (a user in Berlin searching "Alexanderplatz"
should get the Berlin one); geocoding (address string → coordinate) and reverse geocoding
(coordinate → address), both locale-aware.

**F3 — Directions**: car / transit / cycling / walking routes between two points,
typically 3 alternatives, each with distance, time estimate and turn-by-turn geometry.
Car routing must use **live traffic** on real-time-relevant roads.

**F4 — Live ETA during navigation**: while the user is navigating, ETAs and the remaining
route are recomputed as traffic changes, with a freshness bound (seconds, not minutes).
Rerouting on a missed turn is a local computation (client-assisted).

**F5 — Traffic layer**: display congestion (green→red) per road segment, derived from the
same pipeline that feeds ETA.

**F6 — Telemetry ingestion**: navigation users' location pings are ingested continuously
(F4's raw input and F5's only large-scale source). Privacy constraints: pings are used
aggregated/at-segment-level for traffic; per-user trails are retained only as long as the
navigation session requires (policy, not a scale excuse).

**F7 — Place details**: hours, address, photos metadata, ratings summary for a POI. (The
reviews corpus itself is out of scope; the POI record and its freshness are not.)

## Non-functional

- **Latency**: tile fetch p95 <100ms from edge; search p99 <200ms for completions; first
  route response p99 <500ms (car, metropolitan area); ETA refresh during navigation ≤5s
  after a material traffic change.
- **Freshness**: segment speeds updated every 30–60s on traffic-bearing roads; map edits
  (new road, closed road) visible in routing/tiles within hours, tiles for hot regions faster.
- **Availability**: 99.99% for tiles and search; 99.9% for directions (degrade to
  traffic-free routing with historical speeds rather than failing).
- **Scale target**: ~1B DAU, 10M+ tile QPS at the edge, ~230K search QPS and ~140K
  directions QPS at peak (all estimates; arithmetic in [02](02-scale-estimates.md)).
- **Consistency**: none of this needs transactions. Tiles are immutable blobs per data
  epoch; search and geocode tolerate index lag (minutes); ETAs are best-effort fresh.
  The one *correctness* requirement: a route's turn-by-turn geometry must be consistent
  with the graph snapshot its ETA was computed from (no route along a segment whose
  "closed" flag the ETA already respected).

## Explicitly OUT of scope

- **Indoor maps / venue floor plans** — different data model, different acquisition problem.
- **Street View / satellite imagery acquisition and stitching** — the image corpus is
  treated as pre-existing content served by the tile CDN; the capture/stitch pipeline is
  its own system.
- **Transit schedule ingestion & agency contracts** — transit routing reuses the routing
  machinery, but GTFS data quality/agreements are out of scope (we assume schedule data arrives).
- **Ride/driver economies**: driver-side routing optimization, demand-heatmaps, matching
  supply to demand — see [22.frontend-uber](../22.frontend-uber/README.md); Maps is the
  substrate those systems build on, not the market itself.
- **Offline map packs** (download region, use without connectivity) — packaging/versioning
  problem, noted but not designed.
- **Embedded API partner platform** (keyed access, billing, quotas for third parties) —
  the multi-tenant/billing plane is orthogonal to the core design.
- **3D buildings, AR navigation, elevation terrain rendering** — client rendering features;
  the backend contract (vector tiles include what they include) is out of scope.
