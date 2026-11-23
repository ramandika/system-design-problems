# 03 — Architecture

## High-level diagram

```
                                ┌──────────────────────────────────────────────┐
                                │                  CLIENT (mobile/web)          │
                                │  tile cache (LRU) · search · route render ·   │
                                │  navigation state · GPS telemetry             │
                                └───────┬──────────────┬───────────────┬───────┘
                                        │ tiles        │ search/geocode│ directions
                                        ▼              │               │
                    ┌──────────────────────────────┐   │               │
                    │ EDGE/CDN  (worldwide POPs)   │   │               │
                    │ tile cache, hot-region POPs  │   │               │
                    └───────┬──────────────────────┘   ▼               ▼
                            │ miss (5%)      ┌─────────────┐   ┌────────────────────┐
                            ▼                │ GEO-FRONTEND│   │ DIRECTIONS SERVICE │
                    ┌──────────────┐         └──────┬──────┘   │  alternatives,     │
                    │ TILE SERVICE │                │          │  turn geometry     │
                    │ raster+vector│                ▼          └───┬────────┬───────┘
                    │ origin,      │        ┌──────────────┐       │        │
                    │ per-region   │        │ SEARCH/GEO   │       ▼        ▼
                    └──────┬───────┘        │ FST prefix + │  ┌─────────┐ ┌──────────────┐
                           │                │ fuzzy index  │  │ ROUTING │ │ ETA SERVICE  │
                           │                │ shards,      │  │ CRP graphs│ │ (ML model +  │
                           │                │ POI store    │  │ per region│ │ speed layer) │
                           │                └──────────────┘  └────┬────┘ └──────┬───────┘
                           │                                       │             │
                           │          ┌────────────────────────────┘             │
                           │          ▼                                          │
                           │   ┌─────────────────┐    weights refresh   ┌─────────┴───────┐
                           │   │ ROAD GRAPH STORE│◄────────────────────│ SEGMENT-SPEED │
                           │   │ packed graph +  │    (30–60 s)        │ STORE (RAM)   │
                           │   │ CRP hierarchy,  │                     └───────┬───────┘
                           │   │ per region      │                             │
                           │   └───────┬─────────┘                             │
                           │           │ map-edit triggers                     │ re-extract + re-customize
                           ├───────────┘                                       │
                           ▼                                                   ▼
              ┌──────────────────────────────────────────────────────────────────────────┐
              │              MAP-DATA PIPELINE (offline, region-sharded)                  │
              │  source imagery/surveys → feature extraction → road graph build →         │
              │  tile render farm (raster pregeneration) → tile store → CDN backfill      │
              └──────────────────────────────────────────────────────────────────────────┘

   ┌───────────────────────────────────────────────────────────────────────────────────────┐
   │                    LIVE TRAFFIC PIPELINE (realtime, region-sharded)                    │
   │  GPS pings (2.5M/s peak) → ingest bus (partitions = geo shards) →                      │
   │  coarse filter → MAP MATCHER (HMM per session-window) → segment aggregator →           │
   │  SEGMENT-SPEED STORE ─→ CRP customization (minutes) ─→ routing + ETA                   │
   └───────────────────────────────────────────────────────────────────────────────────────┘
```

## Components

**Edge/CDN + tile service** — serves ~95% of tile reads; the origin tier is sharded per
region and stores immutable tiles keyed by `(x, y, z, style-version, epoch, locale)`.
See 04.1 (and the CDN mechanics in [09.cdn](../09.cdn/04-deep-dives.md)).

**Geo-frontend** — API routing, auth, per-product throttling, and the location prior:
attaches the caller's coarse region (derived from the first request's IP/last GPS fix) to
search and directions calls so downstream shards can be selected directly.

**Search & geocode service** — regional shards each hold an FST-based prefix/fuzzy index
over their POIs and addresses; a query carries an S2-cell bias that keeps fan-out to 1–2
shards. POI detail lookups hit the POI store (key-value, by place ID). See 04.2.

**Directions service** — request → region-pinned routing cluster running CRP-style
hierarchical searches; returns 3 alternatives + geometry; asks the ETA service to decorate
each leg. Cross-region routes stitch at highway tiers (04.6).

**Routing graphs (CRP)** — metric-independent partition + hierarchy preprocess once;
edge weights re-customized from live speeds every few minutes. Query search runs
bidirectionally over the packed graph + shortcuts. See 04.3.

**ETA service** — base ETA from live segment speeds along the candidate route, corrected
by an ML model (departure time, day-of-week, historical profiles, incident features).
Model serving follows the pattern in [14.ml-production-system](../14.ml-production-system/04-deep-dives.md).
See 04.4.

**Live traffic pipeline** — ingest bus → filter → map matching (HMM per session window) →
per-segment aggregation → segment-speed store; also feeds the congestion layer shown on
tiles. See 04.5.

**Segment-speed store** — small (≈5 GB hot set), RAM-resident, region-sharded by S2 cell;
the join point between telemetry (write side) and routing/ETA (read side). Versioned:
each routing query pins a weights snapshot so geometry and ETA stay consistent.

**Map-data pipeline** — the offline half: imagery/survey ingestion → feature extraction →
graph build → tile rendering (pregeneration for low zoom + hot regions; on-demand with
request coalescing for the tail). Publishes new tile epochs and triggers CRP
re-customization region-by-region.

## Data flows

1. **Tile flow (read-dominated, immutable)**: client → CDN (hit: done, ~95%) → origin tile
   service (region shard) → 25 KB tile. Cache key encodes data epoch, so a map edit rolls a
   new epoch only for affected tiles/regions.
2. **Search flow**: query + location bias → regional shard FST (prefix + fuzzy) → top-k
   scored by (text match, distance prior, prominence) → hydrate place records → render.
3. **Directions flow**: origin/destination → map-match endpoints to road segments →
   CRP query with current weight snapshot → 3 alternatives → ETA decoration → response.
4. **Traffic flow (the loop)**: driving user → ping every 5 s → ingest partition (by geo
   shard) → map matcher assigns ping to segment → aggregator computes rolling speed per
   segment → speed store + CRP weight refresh → next user's directions/ETA reflect it.
   This loop is what makes F4/F5 possible; its budget is ping→weights ≤ 60 s. The F4
   "≤5 s" bound sits on top of that: once weights change, the navigating client re-polls
   route + ETA on a ~5 s cadence, so the client-visible refresh is 5 s behind the pipeline.
5. **Map-edit flow**: edit approved → feature/graph update → region re-customization →
   affected tiles rerendered → epoch bump in that region → CDN sees new keys.

The two pipelines and the three serving paths are **independent failure domains**: a
traffic-pipeline outage degrades ETA freshness (routing still works on stale/historical
weights); a map-data pipeline outage freezes tiles but not traffic; a CDN outage is a
client-visible tile outage but search/directions keep answering.
