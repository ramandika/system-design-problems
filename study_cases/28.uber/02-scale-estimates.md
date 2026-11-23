# 02 — Scale estimates (back-of-envelope)

All numbers are **rough estimates** derived from the stated assumptions, and are
deliberately the *server-side mirror* of [Case 22](../22.frontend-uber/02-scale-estimates.md),
which derives the same traffic from the client. Where a public figure is referenced it
is marked "publicly reported ~X"; everything else is arithmetic on assumptions.

## Assumptions

- "Publicly reported ~" ~130M monthly active riders; ~20M trips/day (publicly reported
  order of magnitude for recent years) → use **20M trips/day**, mean trip 20 min.
- **30M rider DAU** (≈ MAU/4, estimate), 4 app sessions/day, 4 min each → 480M
  rider-minutes/day.
- **4M driver DAU**, 6 h average online → 1.44B online-driver-minutes/day.
- Peak ≈ 2× daily average (evening rush), same convention as all cases in this repo.

## Derived concurrency

- **Trips in progress**: 20M × 20 min = 400M trip-min/day ÷ 1440 = **~280K concurrent
  trips** average, **~560K at peak** (2×).
- **Drivers online**: 1.44B ÷ 1440 = **~1M online** average; drivers actually driving
  ≈ trip-minutes/1440 = 400M ÷ 1440 = **~280K** (~28% occupancy; the rest are waiting
  for offers).
- **Rider clients connected**: 480M ÷ 1440 = **~330K concurrent**, of which ~280K
  mid-trip and ~50K browsing (~15% — this follows from trip-minutes vs app-minutes,
  400/480 ≈ 83% trip share, as in Case 22).

## Write QPS

**Position updates (the giant write path)**. Cadence policy from deep dive 1: moving
driver every 3 s, waiting driver every 15 s, backgrounded paused.

- 280K/3 s + 720K/15 s = 93K + 48K ≈ **140K updates/s average**, ~280K/s peak (2×).
- ~100 B/update on the wire → **14 MB/s ≈ 112 Mbps ingest** average, ~224 Mbps peak.
  Bandwidth is trivial by modern standards; the *state-update cost* (index mutation,
  fan-out) is the real expense.
- Per-trip high-fidelity stream: ~400 samples per 20-min trip, ~40 KB/trip if logged raw.

**State-transition writes (small, correctness-critical)**. ~10 lifecycle events/trip →
20M × 10 = **200M events/day ≈ 2.3K events/s average**, ~5K/s peak ≈ **1 MB/s** (~460 KB/s average at ~200 B/event).
Never confuse this with the position stream (deep dive 5: different channels).

- Ride requests: 20M/day ÷ 86,400 ≈ **230 req/s**, ~460/s peak.
- Driver offer responses: ≤ 2 offers/trip (some re-dispatch), so **~460/s average, ~1000/s peak**.
- Driver status changes: 4M drivers × ~10 online/offline transitions/day = 40M/day ≈
  **460/s** — plus abnormal disconnects counted as implicit offline (~2×).

## Read QPS

- **Fare/ETA quotes**: riders open the app and pull quotes before requesting. 30M DAU ×
  4 sessions × 3 quote refreshes ≈ 360M/day ≈ **4.2K QPS**, ~8K peak. Each quote fans
  out to 5–10 internal ETA calls (routing engine, out of scope, Case 26) → 20–80K
  internal ETA QPS — the ETA service, not this system, is the fan-out hotspot.
- **Nearby-driver map views**: ~50K browsing clients polling/pushed-to every 4 s →
  **~12.5K query/s** against the geo index, ~25K peak. Each query reads ~200–400 cells'
  latest positions (deep dive 2).
- **Surge lookups**: every quote and every match — ~4.5K/s of point lookups, trivial;
  surge tiles are small and cacheable ([Case 10](../10.distributed-cache/README.md)
  read-locality patterns).
- **Trip state reads**: active-trip state is served on reconnect (assume 10% of trips
  have a client reconnect → 2M/day ≈ 23 QPS, ~46 peak — trivial; snapshot reads are
  cheap and correctness-critical).

## Storage

- **Raw position stream**: 140K/s × 86,400 ≈ **12B samples/day** × ~100 B ≈ **1.2 TB/day**
  (estimate). Retention: hot latest-position in memory; full stream 24–72 h in a log/KV
  tier; then downsample to per-trip traces (~40 KB/trip ≈ 800 GB/day) and 15-min
  rollups. Most of that 1.2 TB is *deleted quickly by policy* — it exists for replay,
  debugging, and ML backfill, not for serving.
- **Trip records**: 20M/day × ~1 KB (route summary, fare breakdown, actors) ≈
  **20 GB/day**, retained years: OLTP-grade durability, this is the money record.
- **Lifecycle events**: ~200M events/day × ~200 B ≈ **40 GB/day** (2 KB/trip — same
  accounting as [Case 22](../22.frontend-uber/02-scale-estimates.md)), retained ~1 year
  for audit (legality requirement: pricing decisions must be reconstructible).
- **Surge tiles**: ~city-scale, a few hundred tiles/city × ~1KB of live parameters ≈
  tens of MB in memory globally + a small time-series history.
- **Active in-memory state**: 1M online drivers × ~200 B (latest position, heading,
  status, cell) ≈ **200 MB** — the entire hot supply state fits in RAM of one beefy
  server; geography, not capacity, is why it is sharded (deep dive 2).

## Bandwidth summary

- Inbound: ~112 Mbps position ingest average (~224 Mbps peak) + small event traffic.
- Outbound: ~330K clients × ~0.5 KB/s ≈ **165 MB/s ≈ 1.3 Gbps average**, ~2.6 Gbps peak
  (same per-client number as [Case 22](../22.frontend-uber/02-scale-estimates.md) —
  one system, two views of it).
- Internal: matching fans out ETA calls and offer pushes; event log replicates
  the ~40 GB/day of transition events across regions — a few Mbps, negligible.

## What the numbers dictate

- The write path is one dominant stream — design it as a *pipeline* (gateway → log →
  index/rollups), not as per-update DB writes.
- All correctness-critical writes are 3–4 orders of magnitude smaller than telemetry —
  they can afford synchronous persistence, quorum replication, and idempotency keys.
- The capacity problem is not storage or bandwidth; it is **update fan-in to a
  mutable in-memory index** and **match latency under re-dispatch** — hence deep dives
  1–3.
