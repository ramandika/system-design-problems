# 02 — Scale estimates

All numbers are **rough estimates derived from stated assumptions** unless marked
"publicly reported". Public anchors used, roughly: Amazon has publicly reported
**~300M+ active customer accounts** and (2021) **~200M Prime members**; third-party
estimates put Amazon's marketplace catalog at **several hundred million listings**
(~350–600M) with mean order volumes estimated on the order of **tens of millions
of orders/day**. Alibaba has publicly reported Singles' Day **peak order-creation
rates in the hundreds of thousands of orders per second** (publicly reported
~544K/s during the 2019 event). Adobe Analytics has reported US holiday online sales of
**roughly ~$10B+ in a single day** on Cyber Monday. Everything below is arithmetic
on those anchors plus stated guesses.

## Assumptions

- A1: Storefront scale "Amazon-class": **~300M active customer accounts**,
  **~100M DAU** (estimate: not every account shops daily; roughly a third of
  actives touch the site or app in a day).
- A2: Catalog **~400M listed SKUs**; roughly **30M** are actively sellable in a
  given region at any time (in stock, in-region, not suppressed); the browse and
  search load concentrates hard on a minority of them.
- A3: Engagement: **1.5 sessions per DAU** → 150M sessions/day; **~10 page
  views per session** → **1.5B page views/day**. Each page view fans out to ~6
  back-end API calls (price, availability badge, reviews, recommendations, similar
  items, promo eligibility) → **~9B API calls/day**.
- A4: Conversion: **~3% of sessions end in an order** (typical e-commerce
  conversion is a few percent) → 150M × 3% ≈ **4.5M orders/day**; mean **2.5 line
  items/order** → **~11M inventory line movements/day**.
- A5: Session economics: sessions are 10 page views but each session touches the
  cart ~3× (add, view, re-render) → ~450M cart mutations/day ≈ **~5K cart
  writes/s at the mean** (estimate).
- A6: Peak factors: within a day, traffic peaks ~**3×** the mean (evening hours);
  **Black Friday week runs ~5–10×** the baseline for days; flash sales create
  **100×+ instantaneous multipliers on specific SKUs** (not on the whole site).

## The read/write asymmetry (the headline)

- Reads: 9B API calls/day / 86,400s ≈ **~104K QPS average**, evening-hour peak
  ~3× ≈ **300K QPS**, Black Friday peak 10× ≈ **~1M QPS**.
- Writes-that-matter (order creation): 4.5M orders/day / 86,400s ≈ **~52
  orders/s average**; realistic burst ceiling for one retailer (doorbuster spikes;
  the global extreme is Alibaba's publicly reported ~544K/s peak, far above a
  single US retailer's burst — see the cross-check below) → size the
  transactional plane for **~2–5K orders/s**, ~200–500× below the read plane's
  ~1M QPS peak.

That is a read:order ratio of **~2,000:1 at the daily mean** and still **~400:1
at peak**. Consequence, stated as a design rule: *any mechanism that puts a
browse read on the transactional plane (synchronous inventory lookup, synchronous
promo check, synchronous price fetch) multiplies the transactional load by the
whole browse stream and is disqualified by arithmetic before any other argument.*

The write side is not uniform either: cart writes (~5K/s avg, ~15K/s peak;
concentrated flash-add bursts ~150K/s) and
inventory *display* writes (price/stock projection updates, ~10M SKU-events/day)
are "write-shaped but cache-destined" — they feed read models, not the order of
record. Only ~11M line-item decrements/day and 4.5M order rows/day are ledger-grade.

## QPS summary

| Tier | Avg/day | Evening peak | Black Friday / flash-sale peak |
|---|---|---|---|
| PDP + browse API calls (A3) | ~104K | ~300K | ~1M |
| Search queries (~30% of page views originate from a search query → 0.3 × 1.5B ≈ 450M searches/day) | ~5K | ~15K | ~50K |
| Cart mutations (A5) | ~5K | ~15K | ~150K (flash adds) |
| Checkout submissions (assume ~1.5× orders, incl. retries/failures) | ~78 | ~230 | ~3–7K |
| Order creates (A4) | ~52 | ~160 | ~2–5K |
| Inventory reservation writes (line items ×2 for reserve/commit) | ~250 | ~750 | ~10–25K |
| Promo evaluations (cart mutations + checkout submissions) | ~5K | ~15K | ~150K |

## Storage

- **Catalog relational store**: 400M SKUs × ~2KB row (attributes JSON, per-region
  overlay elsewhere) ≈ **800GB**; with per-region price/availability overlays
  (~30M active × 20 regions × 100B ≈ 60GB) and localized content
  (400M × 5 locales × 2KB ≈ 4TB) → **order of ~5–10TB**. Small; the interesting
  catalog data is the *index* and the *image corpus*.
- **Images**: 400M SKUs × ~5 assets × ~80KB master, ×~4 renditions ≈ 400M × 20 ×
  80KB ≈ **~600TB** in the blob store (immutable, CDN-served — same economics as
  the blob-store case; compare `../09.cdn/04-deep-dives.md`). Delivery egress is
  CDN-dominated and cheap per byte thanks to cache hits.
- **Search index**: inverted index over 400M docs with ~2KB of indexable text
  (title, bullets, facets) ≈ **~0.25–0.5GB per 100K dense docs (~2.5–5KB/doc)**
  scaling to **~1–2TB index** fleet-wide, sharded; plus per-region availability bitsets
  (~30M × 20 regions × bit-masks ≈ tens of GB) — see deep dive 1.
- **Orders**: 4.5M orders/day × ~5KB (header + line items + status history) ≈
  **22.5GB/day** → hot 90 days ≈ **2TB**; append-only, then archive to object
  storage. The ledger-grade part of the system is *tiny* relative to the read
  plane — this is the arithmetic that justifies the two-plane architecture.
- **Cart store**: ~500M live carts (guest TTL weeks + accounts) × ~1KB ≈
  **~500GB** hot, memory-resident in a distributed KV store (the whole thing fits
  in a mid-size cache cluster — compare `../10.distributed-cache/README.md`).
- **Event stream (the spine)**: impressions, clicks, cart events, order events,
  inventory moves, promo events ≈ **~15B events/day** × ~200B ≈ **~3TB/day**
  retained 7 days hot ≈ 21TB — a classic Kafka-tier pipeline (compare
  `../13.kafka/README.md`). Every read model (search index, availability
  projection, personalization features, analytics) is a projection over it.

## Black Friday / seasonal math (why the peak dominates design)

- Baseline: ~100K QPS reads, ~50 orders/s. **BFCM week: ~5–10× sustained for
  days** → ~500K–1M QPS reads and up to ~500 orders/s sustained. The 10× is
  *multi-day*, so cache pre-warming (not elastic burst capacity alone) is the
  primary lever: the marginal 900K QPS must be served at a cache tier or the
  origin fleet cost is an annual 10×.
- **Doorbuster concentration**: a $10 flash sale on 5K units of a hot SKU produces
  ~100× the *page views* of the SKU's normal rate within minutes, but only ~5K
  order wins. The ratio of losing-traffic to winning-writes is ~20:1 *per SKU* —
  the whole purpose of surge shaping (deep dive 6) is to stop those losers from
  reaching the transactional plane.
- Cross-check against public anchors: Alibaba's publicly reported ~544K/s 2019
  Singles' Day peak is ~36× that event's own mean (publicly reported ~1.3B orders
  over 24h ≈ 15K/s) — event-day concentration well beyond our 5–10× sustained
  multi-day multiplier; Adobe's ~$10B+/day US online figure implies
  (median ticket ~$50) roughly ~200M US transactions/day at peak ≈ 2.3K/s
  sustained nationally — consistent with our ~2–5K orders/s per-retailer burst
  ceiling.

## Capacity consequence summary

- Read plane: cache-first (CDN for media + page fragments; distributed cache for
  PDP fragments, facets, carts); origin fleet sized for **cache misses only**
  (assume 80–90% hit on PDP fragments, 95%+ on media) → origin read load ~10–20%
  of headline QPS.
- Transactional plane: sized for **~5K orders/s + headroom**, strongly consistent,
  small data, horizontally sharded by customer/order id. Anything that can be a
  projection or a cache *will* be — the ledger-tier fleet stays tiny on purpose.
