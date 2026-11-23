# 02 — Scale estimates (back-of-envelope)

All numbers are **rough estimates** derived from the stated assumptions; where a public
figure is referenced it is marked "publicly reported ~X". The point of this file is to size
what *the client* must handle and what the client-driven traffic looks like at the server —
so the numbers below are stated per-client first, then aggregated.

## Assumptions (stated, then derived)

- Riders: ~130M monthly active users ("publicly reported ~X" figure for recent years);
  assume DAU ≈ 1/4 of MAU (130M / 4 = 32.5M), rounded down to **~30M rider DAU** (estimate,
  conservative).
- Trips: "publicly reported ~20M+ trips/day" → use **20M trips/day**, mean trip
  **20 min**.
- Drivers: assume **4M driver DAU**, mean online session **6 h/day**, of which ~28% is
  moving-in-traffic work that generates position fidelity. Why 28%: a moving driver is a
  driver on a trip, so moving-driver-minutes ≈ trip-minutes (400M below, out of 1.44B
  online-driver-minutes → 400/1440 ≈ 28%). (Estimate.)

## Derived concurrency (the numbers that matter to streams)

- Concurrent trips, daily average: 20M trips × 20 min = 400M trip-minutes/day;
  400M / 1440 min = **~280K trips in progress** at an average moment. Traffic is peaked:
  assume peak ≈ 2× average → **~500–600K concurrent trips at peak** (estimate).
- Concurrent online drivers: 4M × 6 h × 60 min = 1.44B online-driver-minutes/day;
  1.44B / 1440 = **~1M drivers online** on average; of these, driving ones ≈
  400M driving-minutes/day / 1440 = **~280K** (~28% "occupancy", plausible: the rest are
  waiting for offers — a driver app's waiting state is the cheaper telemetry state).
- Active rider clients in the app: 30M DAU × 4 sessions/day × 4 min = 480M rider-
  minutes/day → 480M / 1440 = **~330K concurrent rider clients** on average; during trips
  (a subset, ~280K) the client is in high-fidelity mode.

## Client→server: position updates (driver clients only)

Cadence policy (designed in deep dive 2): moving driver **every 3 s**, waiting driver
**every 15 s**, backgrounded/idle **paused**.

- Update rate: 280K/3 s + 720K/15 s = 93K + 48K ≈ **140K position updates/s** inbound.
- Wire size per update (lat, lng, heading, speed, accuracy, timestamp, id, flags,
  compressed): ~**80–120 B**; use 100 B → 140K × 100 B ≈ **14 MB/s ≈ 112 Mbps** ingest.
  The backend scales this ([Case 03](../03.whatsapp/02-scale-estimates.md) style gateway
  arithmetic), but the design constraint lands on the client: *each driver app sends a
  trivial 0.07–0.33 msg/s — and yet raw GPS-every-second would triple that for zero user
  benefit. The throttle is a battery decision, not a bandwidth decision.*

## Server→client: the vehicle stream (rider clients)

Two modes with very different payloads:

- **Browsing mode** (no active trip): the concurrency derivation above says ~85% of
  concurrent rider clients are mid-trip (280K trips / 330K clients), so browsing is
  **~15%** — not an independent split, it follows from trip-minutes vs rider-app-minutes
  (400M / 480M ≈ 83% trip share). Viewport vehicle batch every 4 s. Visible vehicles in
  a dense CBD viewport ~150–400; assume 200, each
  ~60 B (id, pos, heading, type) → 200 × 60 B = **12 KB per batch → 3 KB/s** per client.
- **Trip mode** (~85%): only *your* driver's car + route + ETA, every 2–3 s during
  pickup, 3–5 s during trip (the phase table in deep dive 2; pickup is the
  freshness-critical phase) ≈ 150 B per update → ~30–75 B/s; **~60 B/s** is the
  conservative figure carried through the rest of this file (route polylines are
  evented, not streamed).

Weighted per-client: 0.15 × 3 KB/s + 0.85 × 0.06 KB/s ≈ 0.45 + 0.05 ≈ **0.5 KB/s
average**. That is the bandwidth promise to keep even on weak 3G: ~0.5 KB/s ≈
1.8 MB/hour of app use (0.5 KB/s × 3600 s).

- Aggregate egress (server side): 330K × 0.5 KB/s ≈ **165 MB/s ≈ 1.3 Gbps** average,
  ~2.6 Gbps peak (peaks ride the ~2× concurrency peak). This is socket/gateway egress, not
  CDN — compare tile bytes, which are CDN-bound (below).

## Map tiles & static assets (CDN-bound)

- Per client session (~4 min): one zoom-lattice around user + route. ~40–80 tile requests
  (256–512 px tiles, vector or raster), ~15–40 KB each vector / raster mix →
  ~1.5–3 MB of tile bytes **per session**, almost all cacheable and delivered from a CDN
  ([Case 09](../09.cdn/README.md)); the backend serves only the dynamic layer.
- Client tile cache budget: **~200 MB** across home city + frequently visited cities
  (style pack 5–20 MB, label/street data, offline city packs 20–80 MB/city — estimates).

## State-machine event volume (small, but correctness-critical)

- Lifecycle events per trip (requested/matched/arrived/began/completed/charged/…):
  ~10 events × ~200 B → 2 KB/trip.
- Day total, server-side: 20M trips × 2 KB = **~40 GB/day** of event traffic. Trivial in
  bytes — these events are *rare and ordered*, which is exactly why they must ride the
  reliable channel with sequence numbers ([Case 03](../03.whatsapp/04-deep-dives.md)
  delivery protocol), not the lossy high-cadence position channel. **Different signals,
  different channels** is the load-bearing decision (deep dives 2–3).

## Client-side state/storage numbers

- Position samples retained during one trip: 20 min × 1/3 s = 400 samples × ~50 B ≈
  **20 KB/trip** — keep last N trips in local DB; megabytes, not gigabytes.
- Rebuild-on-reopen ("what request reconstructs the ride UI?"): one snapshot call of
  ~5–20 KB. At 20M trips/day and an assumed 10%-of-trips reopen rate → 2M opens/day →
  ~23 snapshot QPS average, ~46 QPS peak (same peak ≈ 2× average convention as above) —
  trivial capacity cost for a *huge*
  correctness win (deep dive 3), which is why the snapshot endpoint is non-negotiable.

## Render budget (why deep dive 1 exists)

- Frame budget at 60 fps = **16.7 ms**; the moving-car layer gets ≤ ~8 ms of it (budget
  split: UI/JS thread + raster). Size for the dense-CBD high end, ~400 visible markers:
  per-marker per-frame cost must be ≪ 20 µs — this rules out naive per-marker DOM updates
  and dictates canvas/batching + dead-reckoning (deep dive 1). The arithmetic:
  8 ms / 400 markers = 20 µs/marker/frame including projection math.
