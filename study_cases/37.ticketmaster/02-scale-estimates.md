# 02 — Scale estimates

## The anchor events (publicly reported, roughly)

- Ticketmaster publicly reported roughly **3.5 billion system requests** on the day of
  the Taylor Swift Eras Tour presale (November 2022), and said bot traffic was a
  several-fold multiple of normal. Treated as an order of magnitude for a "worst
  on-sale day", not a per-show spec.
- Major stadium on-sales are routinely reported to sell out in **minutes**; press
  coverage of arena tours regularly cites single-digit-minute sellouts. Estimates
  below assume a hot arena show with single-digit-minute sellout.
- Arena capacity roughly **15,000–25,000 seats**; stadium tours roughly 50,000–80,000.
  Estimate below uses a 20,000-seat arena show, the harder per-seat ratio.

All figures below are estimates re-derived from those anchors. Adjust the assumptions
and redo the arithmetic.

## The core mental shift: QPS in minutes, not days

Normal traffic modeling divides daily volume by 86,400 seconds and adds a peak factor of
maybe 5–10×. That method is useless here. On-sale demand arrives at a **pre-announced
instant** (10:00:00 sharp), so the arrival process is closer to a synchronized start
than to a Poisson dayload — the peak factor against the daily average is *tens to a few
hundred* — the anchor arithmetic below computes ~60× — and the honest way to size the system is **population × action rate
during the spike window**, with the window measured in minutes.

**Sanity anchor**: 3.5 billion requests in one day, if perfectly spread, is
3.5e9 / 86,400 ≈ **40,500 req/s sustained all day**. But it is not spread: assume ~40%
arrive in the first 10 minutes → 1.4e9 / 600 s ≈ **2.4 million req/s peak** in the hot
window. The ratio between the "average day" number and the real peak is ~60× — and the
system must be sized for the second number at second zero.

## Show setup (estimates)

- 20,000 seats; ~8 price tiers by section; ~60% of demand wants the 3 tiers closest to
  the stage (front-heavy demand — this matters for hot keys).
- Interested population for a super-hot show: ~5M known-fanbase visitors; assume ~1.5–3M
  actually show up at 10:00 within a ~15-minute window. Use **2M arrivals in 15 minutes**
  as the design case.

### Arrival math

- 2M arrivals over 15 min ≈ 2.2K *users* per second arriving. But user *actions* are the
  load: pre-on-sale, each visitor polls the waiting-room/event page; typical client
  poll intervals 2–5 s (impatient users, refreshes, tabs) → assume one request every 3 s
  per connected visitor at rest.
- Instantaneous on-system population builds toward ~2M over the ~15-minute window,
  front-loaded (most of it in the first few minutes). Waiting-room polling load: 2M users
  × (1 req / 3 s) ≈ **~650K req/s** if served
  naively per-user. This is the single most important number to engineer away (below).

### Inventory math — the scarcity ratio

- Sellout is capped by the venue: 20,000 seats / ~6 min ≈ **~55 seats/s** of
  *successful* purchase rate. That is the only rate the inventory can support, ever.
- Demand pressure: ~2M users each wanting ~3 seats ≈ 6M seat-demands against 20K seats
  → **~300× oversubscription**. Every design decision follows from this ratio.
- Successful conversions are therefore a rounding error of the traffic: ~55 *seats*/s
  ≈ ~15–20 completed *orders*/s (groups average ~3 seats) × ~40 request-paths per
  completed order (map views, holds, checkout steps, payment)
  ≈ ~600–800 req/s of *genuinely useful* work. The other ~98% of requests must be absorbed
  or diverted before they touch anything expensive.

## Where every request-category lands (the sizing)

*(estimates; each line follows from an assumption stated)*

1. **Waiting-room polling (~650K/s raw).** Served by the waiting-room tier, which holds
   admission state in Redis (one sorted set + per-user token). Key optimization: each
   client receives a signed position estimate and a scheduled next-check time — the
   server pushes position only on *change*, and clients poll no faster than their
   scheduled slot. Assume 20% of users ignore backoff and repoll at 1 s while the rest
   honor it at 1/10 s → 2M × 0.2/1 s + 2M × 0.8/10 s ≈ 400K + 160K ≈ ~560K req/s —
   that arithmetic shows
   backoff alone doesn't save you. Real solution is tiered: position served from an
   in-memory count against a static page (no per-user state), position *precision*
   bucketed (e.g., "you're ~position 400,001–410,000"), so any replica can answer any
   user. Target: **≤80K req/s** at the waiting-room service itself, with the rest
   served at CDN/edge from a nearly-static asset + coarse counters.
2. **Seat-map reads (only admitted users).** Admit ~150K users into the active-sale
   population at once (a tuned number — see 04 dive 1). Each admitted user loads the
   seat map and refreshes availability every ~3 s → 150K × (1/3) ≈ **50K req/s** map
   reads. Map payload: per-section aggregate bitmaps, cached per (event, tier); a
   section's bitmap changes on every hold/sale, so serve map from a read cache with
   ~1 s staleness — acceptable, because the authoritative hold step re-validates.
   50K reads/s of cached aggregates is trivial for a cache tier
   (cross-ref [10.distributed-cache](../10.distributed-cache/04-deep-dives.md)).
3. **Hold attempts (the contention point).** Admitted users attempt holds frequently
   (fast refresh loops, groups retrying): assume each admitted user attempts a hold
   every ~10 s under contention → 150K/10 = **15K hold attempts/s**, of which at most
   ~55 *seats*/s can convert (groups average ~3 seats → ~15–20 successful holds/s,
   see item 4). Holds are atomic operations on
   the event's inventory state — this is the hot-shard bottleneck, sized and solved in
   04 dive 6. Budget: the inventory tier must absorb ~15–20K atomic seat operations/s
   *for one event*.
4. **Checkout completions.** ~55 seats/s ≈ ~15–20 completed orders/s at peak (groups
   average ~3 seats), each with authorization + capture against payment rails. Trivial
   volume — but zero-tolerance for double-charge (04 dive 4).
5. **Catalog tier (everything else).** Non-on-sale events continue at their own
   traffic, unaffected; enforce this with per-event admission control so one show's
   spike cannot starve others' pages.

```text
   2M arrivals @ t=0..15min        (design case, estimate)
        │
        ├─ ~650K req/s polling raw → edge-served static + coarse counters (target ≤80K/s hit origin)
        │
        ├─ ~150K admitted to active sale
        │     ├─ ~50K req/s seat-map reads (cached aggregates, ~1s stale)
        │     └─ ~15K/s hold attempts  → ATOMIC ops on ONE event's inventory (hot shard)
        │            └─ successful conversions capped at ~55 seats/s = 20K seats / 6 min
        │
        └─ ~15–20 orders/s completions → payment, capture exactly once
```

## Storage and state (unexciting but bounded)

- Seat maps: a 20K-seat map is ~20K × ~30 B (section, row, seat, state) ≈ **~600 KB**
  per event as bitmaps/vector; a day of on-sales (say 200 hot events) is ~120 MB of
  hot state total. Inventory state fits in memory; this is a compute/atomicity problem,
  never a storage one.
- Hold records: short-lived and bounded by the seat count, not the population —
  concurrent held groups ≈ ~15–20 successful holds/s × ~7–9 min TTL (04 dive 2) ≈
  ~9K holds (estimate, before the cap) — but held seats are always capped by the
  ~20K-seat map (≤ ~6.7K concurrent 3-seat groups); at ~200 B per seat that's
  ~2–4 MB, churning every few minutes.
- Order + immutable seat-transition event log: ~20K seats + failed attempts of the day;
  write volume ~10^4 events/s during spikes (estimate: the ~15K hold attempts/s set the
  floor, since failed attempts are logged too; successful seat transitions are only
  ~15–20/s), retained for dispute replay — a
  partitioned log, not a relational hot table (cross-ref [13.kafka](../13.kafka/04-deep-dives.md)).
- Steady-state catalog (venues, maps, historical orders) — ordinary RDBMS + cache, not
  sized by the spike at all.

## What the numbers buy (design consequences)

- **Waiting room is not optional.** At 2.2K users/s arriving and 300× oversubscription,
  there is no hardware fleet that serves everybody the seat map honestly. Admission
  control is the architecture.
- **The scarce resource is atomic seat operations on one event**, not reads, not
  bandwidth, not storage. Sizing the inventory tier correctly is dive 6.
- **Purchase throughput is fixed by the venue** (~55 seats/s). Any engineering effort
  spent making purchases "faster" is effort spent on the wrong bottleneck — it should
  go into making admission fair and holds honest instead.
- **Everything expensive must be behind admission**, because admitted population
  (150K) is 13× smaller than arriving population (2M) and that ratio is what all the
  downstream numbers rest on.
