# 02 — Scale estimates (from the device's point of view)

The server tiers are given (see [Case 02](../02.twitter/04-deep-dives.md) for timeline
sizing at the server's scale); here we size what one device holds, renders, and transmits,
because on mobile *the device's limits are the real limits*. Assumptions are
order-of-magnitude estimates; change one and redo the arithmetic.

## Assumptions

- **Installed base ~100M, DAU ~50M** (rough estimate for a global consumer feed; publicly
  reported social apps sit at this scale — treat as an estimate, not a measurement).
- **Feed opens: 8/DAU/day** → 50M × 8 = **400M opens/day**.
- **Items consumed per open: ~30** (scroll depth median; heavy users far more, but the
  median carries the arithmetic) → 400M × 30 = **12B item-impressions/day** ≈ 240
  items/DAU/day — a scroll-heavy consumer profile, an estimate.
- **Composition**: ~60% of items are image-first, ~4% are video-first (~15% of video items
  get watched ≥ 3 s), the rest text/UI-light cards.
- **Image transfer size**: ~60 KB average for an optimized variant at ~1080 px display
  width (WebP/HEVC-class compression; estimate — device-width variants typically land in
  the tens-of-KB range, not hundreds).
- **Display**: full-width tiles, 1080 px wide, ~700 px average rendered height,
  ARGB_8888 decode at display size.

## Device-side numbers (the ones that drive the design)

- **Feed data is trivial**: a feed row (IDs, score, variant URLs, counts, timestamps) is
  ~400 B. Persisting the last 1,000 items = 1,000 × 400 B ≈ **400 KB**. The feed corpus
  is *not* the scaling problem — the bitmaps are.
- **Bitmap cost per tile** (the number everything else serves):
  - decoded at **display size**: 1,080 × 700 px × 4 B ≈ **3 MB** ARGB_8888 (~1.5 MB if
    opaque photos allow RGB_565 — half the bytes per pixel);
  - decoded at **full source size** (a 4,000 × 3,000 shot): 4,000 × 3,000 × 4 B = **48 MB**
    — sixteen tiles' worth for one mistake. Decode-at-display-size is not an optimization,
    it is the design (04 dive 1).
- **On-screen demand**: ~2.5 tiles visible + ~1.5 in-flight decode/prefetch ≈ 4 × 3 MB ≈
  **~12 MB live bitmaps**. We set the memory bitmap cache cap at **~48 MB**
  (~16 display-size tiles — at ~2.5–3 tiles visible per screen that is ~5–6 screens of
  scroll-back, a generous buffer), the single largest memory consumer in the
  app; the rest of the app budget (~150 MB total RSS target, mid-tier estimate) covers the
  rest: UI, video player, image pipeline scratch, the OS's share of RSS.
- **Disk image cache cap: 512 MB** (scaled down on low-storage/low-end tiers). Inbound
  images/DAU: 12B × 60% = 7.2B impressions ÷ 50M DAU ≈ 144/day × 60 KB ≈ **9 MB/day**;
  a 30% repeat-view hit rate locally nets ~6 MB/day of new bytes → 512 MB holds ~85 days.
  Evicting it costs latency, never correctness (the CDN re-serves everything — the same
  cache-is-pure-performance rule as [Case 19](../19.frontend-chat-app/02-scale-estimates.md)).

## Network numbers (the ones that gate prefetch)

- **Feed page requests**: page size 20, ~1.5 pages/open × 1.2 (resume-refetch overhead)
  → 400M × 1.8 ≈ **~0.7B requests/day** → mean 0.7B/86,400 ≈ **8K QPS**, peak (4× mean
  rule-of-thumb) ~32K QPS. Small: the server tiers absorb this; the client's concern is
  RTT, ~200–400 ms on 4G (estimate), which is what the persisted-UI cold start hides.
- **Image requests**: 7.2B impressions × ~70% unique-enough-to-fetch ≈ **5B/day** → mean
  ~58K, peak ~230K QPS — absorbed by CDN edges (publicly reported CDN hit ratios are
  typically ≥ 90%; origin sees ~6K QPS). See [Case 09](../09.cdn/README.md) for the edge
  tier; client-side these requests are the prefetch policy's currency.
- **Egress to clients**: images 5B × 60 KB = **300 TB/day**; video ~72M watched plays
  (12B × 4% = 480M video impressions, × 15% watched) × 3 MB ≈ **216 TB/day** — each
  ≈ 3.5 GB/s and 2.5 GB/s mean respectively. Sum ≈ 516 TB/day ≈ 6.0 GB/s mean, 4× at
  peak — all CDN-edge egress.
- **Per-user bytes**: 516 TB/day ÷ 50M DAU ≈ **10.3 MB/day average user** (estimate;
  distribution is heavily skewed — the top 10% of scrollers plausibly see several times
  the median). This grounds N4's ≤ 15 MB/day budget and data-saver's ≥ 50% cut target.

## Frame arithmetic (drives dive 2)

- 60 Hz → 16.7 ms budget; 120 Hz → 8.3 ms. Fling at ~3,000 px/s ≈ 4 tiles/s crossing or
  newly entering the screen; each arrival needs layout + bind (few ms, estimate) — fits,
  *only if* decode and network callback dispatch never land on the main thread and only
  if the recycle pool is primed for the item view-type.
- A cold view-type inflate on a fling is the classic 10–50 ms jank spike (estimate) — hence
  stable view types, pre-inflation, and direction-aware prefetch ahead of the finger.

## Cold-start arithmetic (drives dive 3)

- Budget decomposition, mid-tier 4G (all estimates): process fork + first frame
  300–500 ms; app-level init (non-lazy) ≤ 150 ms; local restore (DB read of last window +
  cached bitmaps) ≤ 100 ms; network fetch page 1 ~200–400 ms RTT — *overlapped with the
  restore render*, not blocking it; decode/first paint of above-fold tiles ~50–150 ms.
  Sum, worst-path ≈ 900 ms with the RTT overlapped as stated (≤ ~1.3 s even counted
  serially) → the N2 ≤ 2 s target is achievable *only if* init stays off the
  critical path and the persisted window paints before the fetch lands.

## Consistency check

- 1,000 persisted rows = 400 KB; live bitmaps ~12 MB; disk cache 512 MB over ~85 days —
  the device is nowhere near storage limits on feed *data*. Every constraint that shapes
  the architecture is memory (bitmaps), latency (cold start, fling), and goodwill (battery,
  data plan). That's the takeaway: a mobile feed client's scale problem is a
  physics-of-the-device problem, not a terabyte problem.
