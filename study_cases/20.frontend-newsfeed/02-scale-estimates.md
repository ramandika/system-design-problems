# 02 — Scale estimates

The ranking backend has its own math ([Case 02](../02.twitter/02-scale-estimates.md)).
Here we size the **client-originated load** — what user devices fetch, render, hold, and
beacon — because at feed scale the client *is* a distributed system of hundreds of
millions of nodes and its choices set the backend's QPS. Assumptions are order-of-magnitude
estimates; change one and redo the arithmetic.

## Assumptions

- **DAU ~200M** — an order-of-magnitude design point; publicly reported scale for large
  feed products ranges from tens of millions to billions of DAU, so this sits mid-range.
- **Sessions**: 3 feed sessions/user/day → 600M sessions/day. Session ~5 min.
- **Consumption**: ~120 items fetched per session (infinite scroll), of which ~100
  genuinely *viewed* (viewport dwell) — the fetched-vs-viewed gap matters for prefetch waste.
- **Pagination**: initial page = 20 items; continuation pages = 15 items. Calls/session =
  1 + ⌈100/15⌉ = 1 + 7 = **8 feed API calls/session** (20 + 7 × 15 = 125 ≈ the ~120 fetched).
- **Hydrated item payload**: ~3 KB JSON (text + media metadata + interaction state +
  impression IDs). Wire size after gzip (JSON gzips ~4:1): ~750 B/item.
- **Media**: ~25% of items carry a server-rendered display-size image (~150 KB CDN),
  ~2% autoplay video (~2 MB, estimate).
- **JS**: ~150 KB gzipped on the critical path (shell + list + store); 500–800 KB total
  across lazily loaded chunks (ads SDK, video, comment UI) per session.

## Feed API load (the client sets the backend's QPS)

- Calls: 600M sessions × 8 = **4.8B feed page requests/day** → average 4.8B / 86,400 s ≈
  **~55K QPS**; peak-hour multiplier ~3× → **~165K QPS**. Sanity check: Twitter publicly
  reported >300K QPS of poll-based timeline reads (2012, QCon) — so at this design point
  we land in the same order of magnitude, which is the point of the arithmetic.
- Bytes: ~120 items/session × 600M sessions = **72B items/day**; raw egress 72B × 3 KB ≈
  216 TB/day; after gzip ~4:1 → 72B × ~750 B ≈ **~54 TB/day of API egress** (the heavier
  20-item initial page is included in the ~120). This is why the page token must be compact
  and why the client must *not* re-fetch pages it already holds (dive 3), and it prices
  every unnecessary prefetch: one wasted 15-item continuation page ≈ 15 × ~750 B ≈ ~11 KB —
  one wasted page per session across 600M sessions ≈ ~6.6 TB/day.

## Media load (CDN, per CLS-free design)

- Per session: images = 120 × 25% × 150 KB ≈ 4.5 MB; video = 120 × 2% × 2 MB ≈ 4.8 MB →
  **~9 MB media/session** (estimate).
- 600M sessions × 9 MB ≈ **~5 PB/day CDN egress**. Three client decisions cut it
  materially: decode-at-display-size (sends only what's needed), media disk cache with
  per-URL LRU evicting re-fetches (~10–20% hit rate assumption → ~0.5–1 PB/day saved,
  estimate), and *not prefetching* video beyond the scroll direction (dive 3).
- This magnitude is exactly why delivery is CDN-offloaded (see [Case 09](../09.cdn/README.md))
  and why the client treats bandwidth as a user cost, not a free resource.

## Render cost: why the window, again

- Naïve mount: a 1 h power-scroll session at the assumption rate (~100 viewed / 5 min ≈
  20 items/min) walks ~1,200 items. On one per-item unit — ~1,000 DOM nodes × ~1 KB
  retained/node including framework wrappers (engine-dependent estimate) — that is
  ~1,200 items × ~1 MB ≈ **~1 GB RAM** in the worst naive case: dead on arrival against
  the 250 MB budget, hence dive 1.
- Windowed steady state: ~40–60 mounted items × ~1 MB ≈ **~50 MB of live DOM** — the same
  per-item unit makes windowing a ~20× cut, which is exactly the margin over the budget.
- Height/size cache: one number per fetched item + key → ~10k items ≈ ~1 MB. Trivial.
- Scrolling is the only continuous "animation": at ~800 px/s typical scroll and ~400 px
  avg item height ≈ 2 items/second entering the window — batchable well inside 16.7 ms.
  A fling at ~3,000 px/s ≈ 8 items/s — still fine *iff* image decode stays off the
  main thread (see [Case 19's](../19.frontend-chat-app/02-scale-estimates.md) decode math:
  one naive 12 MP decode = ~48 MB RGBA vs ~0.5 MB at display size).

## Beacon load (the client is also a sensor)

- Impressions: 600M × 100 viewed = **60B impression events/day**. One beacon per event
  would be 60B requests — absurd. Batched (50 events/beacon, 2 flushes/min or on
  visibility-change): ~1.2B beacons/day × ~300 B ≈ **~360 GB/day** of analytics ingest —
  real money, and it is why impression batching + `visibilitychange` flush is a design
  decision, not an implementation detail. The ranking backend consumes this
  ([Case 15](../15.ml-ranking-system/README.md): training/eval signals).
- Interaction events are 100× rarer than impressions; batching covers them.

## Per-device budget (drives dives 1, 3, 6)

- Memory, 1 h session: DOM ~50 MB + store (paged log, worst case ~1,200 fetched items ×
  3 KB ≈ ~4 MB before trimming behind the window, ~2 MB after) +
  bitmap cache for near-viewport media ~10 items × ~1 MB ≈ ~10 MB + framework overhead
  ≈ **well under the 250 MB budget** — provided images decode at display size and the
  log trims far-behind pages, which dives enforce.
- Disk: media cache fills ~30 MB/day (3 sessions × ~10 MB), capped at 500 MB LRU → ~2
  weeks of history before steady evict. Purely latency; eviction never costs correctness.
- Critical-path JS: 150 KB gzip → parse/compile at estimated ~1 ms/KB on a low-tier
  phone ≈ **~150 ms of main-thread time before anything else can happen**, plus execute.
  Every KB on the critical path is paid at 200M DAU; that's the bundle-hygiene arithmetic
  behind dive 5.

## Consistency check

- Device-side: DOM ~50 MB, store ~2–4 MB, bitmaps ~10 MB — windowing keeps capacity
  nowhere near a limit. API-side ~55K QPS average and ~54 TB/day egress; media ~5 PB/day. Same takeaway
  as [Case 19](../19.frontend-chat-app/02-scale-estimates.md): frontend scale problems are
  *latency, jank, and load-multiplier* problems, not storage problems — but at feed scale
  the client's wastefulness is multiplied by 600M sessions/day, so prefetch discipline
  and beacon batching are backend-scale decisions made in frontend code.
