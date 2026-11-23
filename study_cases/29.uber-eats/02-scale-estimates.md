# 02 — Scale estimates

All numbers are estimates built from publicly known Uber ranges and labeled assumptions.
Uber has publicly reported roughly $75B annual gross bookings across all products in 2022,
rising to ~$140B in 2023, with the Eats/Delivery segment a large minority share — Delivery
segment gross bookings publicly in the ~$60-70B range (2023). Uber also reports roughly
1M+ active merchants and high single-digit billions of trips/orders per year across
products. We derive Eats-only numbers from those ranges rather than inventing them.

## Anchoring assumptions

- **A1. Orders/day ≈ 6M** (estimate). Reasoning: publicly reported Delivery-segment gross
  bookings of roughly $60-70B annually at an average order value in the $30-40 band implies
  roughly 1.5-2.3B orders/year → ~4-6.5M/day. We use 6M (an order value near the $30 end of
  the band yields ~6M/day).
- **A2. Consumer DAU ≈ 20M** (estimate). Monthly Eats consumers in the ~80-90M band (our
  estimate; Uber publicly reports ~150M monthly active consumers across the platform, Eats
  a large share). Food delivery is frequency-skewed, so assume ~1/4 of
  monthly actives order or browse on a given day, plus pure-browse visitors
  → 20M daily actives, of which ~30% place an order (6M orders/day, consistent with A1).
- **A3. Merchants ≈ 1M** (publicly reported order of magnitude).
- **A4. Couriers**: assume ~2.5M couriers earning via Eats each month, ~1.5M online at the
  dinner peak. A courier completes ~4-8 orders in an active day (estimate); 6M orders/day
  ÷ 5 ≈ 1.2M couriers online on average across the day, peaking ~1.5M — internally consistent.
- **A5. Trip shape**: ~45 min from order completion to delivery completion (pickup wait +
  travel), single dropoff for ~90% of orders, two-order batches for ~35% of assigned
  couriers at peak (see 04.2).

## Traffic

### Read path (dominant load)

- Menu/catalog requests: 20M DAU × ~15 catalog-related reads/session (restaurant discovery
  impressions, menu loads, item photos, modifier fetches) ≈ 300M reads/day
  → **300M / 86,400 ≈ 3.5K QPS average**.
- Peak: demand concentrates in lunch (~12:00-13:30) and dinner (~18:00-20:00) windows;
  assume ~30% of daily read volume in the dinner peak hour → ~90M/h ≈ **25K QPS peak**,
  essentially all cacheable at the edge (per-merchant menu + photo objects).
- Restaurant search/browse QPS rides the same reads and is included.

### Write path

- Order placement: 6M/day ≈ **70 QPS average**; peak hour ≈ 30% of daily orders →
  1.8M/h ≈ **500 QPS peak** (order placement, cart commits, payment authorizations).
- Order state transitions: ~8 transitions per order (accept, prep stages, assign, arrive,
  pickup, deliver, settle, rate) × 6M ≈ **~50M state events/day ≈ 580/s average,
  ~4-5K/s peak** on the event bus.
- Merchant/POS sync traffic is trivial relative to this (tablet pushes, not pulls).
- Courier job offers/decisions at peak: with ~2-5 dispatch rounds per unassigned order
  before assignment and ~500 orders/s peak inflow, dispatch produces
  **~1-2.5K offer-events/s** (estimate) — a write burst on the offer/decision path.

### Telemetry (the sleeper load)

- Courier location pings: 1.5M online couriers × 1 ping/4s ≈ **~375K pings/s at dinner
  peak**; at the ~1.2M online average (A4) ≈ ~26B pings/day. This is the highest-QPS stream
  in the system (~15× the 25K QPS read peak) and is *not* user-visible load — it is the
  dispatch + tracking fuel.
- ETA computations: dispatch scoring needs ETA between couriers and restaurants/restaurants
  and customers; at ~2 dispatch rounds/s per market with candidate sets of ~50, this is
  tens of thousands of small route queries/s — served from in-process caches seeded by the
  ETA service (see 04.2; models and serving patterns from
  [26.google-maps](../26.google-maps/README.md)).

### Connectivity

- Concurrent in-flight orders: orders/hour peak ≈ 1.8M with a ~45 min lifecycle
  (A5) → concurrent orders ≈ 1.8M × (45/60) ≈ **1.35M concurrent orders at dinner peak**
  (estimate).
- Tracking sockets: each in-flight order watched by ~3 parties (customer, courier,
  restaurant) → **~4M concurrent long-lived connections** at peak. Push model; per-order
  update rate ~1 msg/5-10s per viewer.

## Storage per day

| Store | Arithmetic | Volume | Retention |
|---|---|---|---|
| Order records + line items | 6M orders × ~3KB (items, modifiers, fees, snapshot of menu prices) | ~18 GB/day | hot 90d, then warehouse |
| Order/ledger event stream | ~50M events × ~500B | ~25 GB/day | immutable; ledger rows kept indefinitely at ~10B size (see
[06.apple-app-store](../06.apple-app-store/README.md)) |
| Courier location pings (raw) | 26B pings × ~150B (id, lat/lon 8B×2 as fixed ints, ts, heading, speed) | **~4 TB/day raw** | raw in stream log 72h, then dropped |
| Courier location (downsampled) | keep ~1 point/30s per active courier: 1.2M × 2,880 pts × ~100B | ~350 GB/day | 90d for analytics/incidents |
| Menu/catalog | 1M merchants × ~200KB average menu JSON (structure, not photos) | ~200 GB total corpus, churn ~1%/day → ~2-5 GB/day of versions | versioned, long-lived |
| Photos (blob CDN) | ~50 photos/merchant × ~2MB + item photos ≈ 150-250 GB total | mostly static; new/edited ~5-10 GB/day | indefinite |
| Tracking/push fanout state (Redis etc.) | ~4M sockets × ~2KB session state | ~8-10 GB live RAM footprint at peak | ephemeral |

Dominant daily writer is **courier telemetry**, by two orders of magnitude over the order
store — which is exactly why pings live on the stream log ([13.kafka](../13.kafka/README.md)
pattern) with downsampling at ingest, not in the order database.

## Bandwidth

- Catalog/photo read egress: 300M reads/day × average ~50KB of image payloads (progressive
  item photos dominate page weight; menu JSON ~20-100KB) ≈ **~15-20 TB/day**, ~95%+
  cacheable at CDN edges (see 04.4).
- Tracking push: 4M connections × ~10 updates/min (~1 msg/5-10s per viewer, above) ×
  ~300B ≈ ~40M msgs/min ≈ ~200 MB/s ≈ **~720 GB/h peak ≈ ~2-3 TB/day** of telemetry egress
  (6M orders × 45 min ≈ 270M in-flight order-minutes/day × 3 viewers) — small next to the
  read path, but latency-sensitive and connection-count-bound rather than byte-bound.
- Ingest: 26B pings/day × ~150B ≈ **~4 TB/day inbound** to the location pipeline, spread
  across couriers' mobile networks; batching on device (send 3-5 pings per payload with
  delta compression) cuts it ~3-5× at the cost of dispatch freshness — see 04.3.

## Consistency budget (what we can and cannot spend)

- Order transitions: strongly consistent, ordered per order — 500/s peak is cheap to make
  rigorous, so we spend here.
- Catalog: eventual with minutes of staleness — 25K/s peak of reads would be ruinous to
  make strongly consistent globally, so we cache at the edge and bust on versioned
  publishes.
- Courier positions: latest-wins, no persistence guarantee per ping; correctness comes from
  volume + freshness, not durability.
- ETA values: probabilistic estimates, never promises with a hard SLA — represented as a
  band with updates, because a wrong point estimate is worse than an honest interval.

## Scaling sensitivities

- Lunch+dinner means **5-10× over-average load for ~25% of the day**: capacity must be
  provisioned for the peak and shed/scale for the trough, and the tracking fanout must
  tolerate connect/disconnect storms when dinner rush starts.
- Read:write ratio ~50:1 on the catalog path → any work done at order time (price
  re-validation, tax, fee computation) should be precomputed or cached, because the write
  path is a rounding error next to the read path.
- Telemetry scales with *couriers online*, not orders — demand in a new city adds pings
  before it adds orders.
