# 02 — Scale estimates (back-of-envelope)

## Anchors (public, roughly)

- **Listings**: publicly reported ~8M active listings.
- **Demand**: publicly reported ~491M nights booked in 2024 → call it ~0.5B nights/year.
- **Money**: publicly reported ~$81B gross booking value in 2024 → implies
  ≈ $81B / 0.491B ≈ **$165/night average** (derived, not reported).
- **Hosts**: publicly reported ~5M hosts; platform cumulative arrivals crossed ~2B guests.

Everything else below is our estimate built on those anchors.

## Derived demand (estimate)

- Assume average stay ≈ **3.5 nights** →
  stays booked/year ≈ 491M nights ÷ 3.5 ≈ **140M stays/year** →
  ≈ 140M ÷ 365 ≈ **~380K stays booked per day** (average; peak travel days ~3× →
  ~1.1M stays booked on a peak day — New Year week, August).
- Nights in progress on any given evening ≈ 491M nights ÷ 365 ≈ **~1.35M guests
  staying somewhere tonight**; arrivals and departures per day each ≈ stays booked/day
  ≈ ~380K (nights are consumed by stays in progress, and every stay contributes one
  arrival and one departure).
- Average order value ≈ 3.5 nights × $165 ≈ **$580 per stay**; at 140M stays/year the
  daily booked volume ≈ $81B/365 ≈ **~$220M booked per day** (average).

## Users and sessions (estimate)

- Assume a booked stay is preceded by ~4 active search sessions over a couple of days and
  the search→book conversion per session is ~4%: booking sessions needed per day ≈
  380K ÷ 0.02–0.04 → we take **~10M search sessions/day** as the working number
  (order-of-magnitude assumption, labeled).
- Assume each session runs 3 searches (query, refine filters, re-query) →
  **~30M search queries/day** → 30M ÷ 86,400 s ≈ **~350 QPS average**, ~3–5× peak-season
  concurrency → **~1–2K QPS peak** search QPS. Map panning adds client-issued re-queries
  on top (assume another ~1–1.5× effective).
- Listing detail views: assume 4 detail views per session → **~40M views/day** →
  **~460 QPS average, ~2K peak**.
- DAU: 10M sessions/day spread over ~1.3–2 sessions/user → estimate **~5–7M DAU**
  (guest-heavy; hosts are ~1M DAU-equivalents at most — 5M hosts mostly idle until they
  get a booking).

## Booking and payment calls (estimate)

- Booking *attempts*: not every attempt converts (failed auth, host declines, guest
  walks away at payment). Assume ~3 attempts per confirmed stay →
  380K × 3 ≈ **~1.1M booking attempts/day** → 1.1M ÷ 86,400 ≈ **~13 QPS average**;
  peak day ×3 with intra-day spike to ~5× → **peak under ~200 QPS** (13 × 3 × 5 ≈ 195).
  This is tiny QPS for
  the effort — the booking path is sized for *correctness and latency*, not volume.
- Payment authorizations ≈ booking attempts that reach payment ≈ ~1M/day ≈ same
  magnitude (~12 QPS average, peak in the low hundreds). Captures, refunds, payouts: refunds on, say, ~5% of stays → ~20K refunds/day; payouts batched (see payment dive) —
  ~5M hosts × weekly ≈ ~700K payout events/day worst case if uniform, realistically
  batched to far fewer daily jobs of millions of rows.

## Photos (the real bandwidth monster)

- Assume ~8M listings × ~20 photos ≈ **~160M original photos**; originals avg ~4MB →
  ~160M × 4MB ≈ **~640TB originals at rest** (still modest by blob-storage standards;
  photos are a bandwidth problem, not a storage one).
  Derived sizes are rendered from originals (precomputed common sizes + on-demand), so
  at-rest derived storage is ~3–5× originals ≈ ~2–3PB.
- Deliveries: a detail page loads ~12 image variants; 40M detail views/day × 12 ≈
  **~480M image requests/day** (search grid tiles add more small thumbs; count them inside
  this). → 480M ÷ 86,400 ≈ **~5.5K QPS average, well over 20K peak** — an order of
  magnitude above search QPS, hence entirely a CDN problem
  ([09.cdn](../09.cdn/02-scale-estimates.md)).
- Egress: avg delivered ~45KB (thumbs dominate by count) → 480M × 45KB ≈ **~21TB/day** →
  ≈ 250 MB/s ≈ **~2 Gbps average**, ~6 Gbps peak. Nearly all served from CDN caches;
  origin fetches only on misses (assume ~90%+ hit rate for popular listing imagery).

## Data at rest (order of magnitude)

- **Listings**: 8M × ~100KB row (desc, amenities, pricing rules, denormalized search
  fields) ≈ **~800GB**.
- **Photos**: ~640TB originals + ~2–3PB derivatives ≈ **~3PB** in blob storage.
- **Calendars**: 8M listings × 730-day horizon × ~16B/state ≈ **~100GB** — trivial in
  volume, maximal in correctness weight ([04 dive 2](04-deep-dives.md)).
- **Bookings**: 140M stays/year × ~2KB ≈ **~280GB/year**.
- **Reviews**: assume one review per ~65% of completed stays ≈ ~90M/year × ~2KB ≈
  **~180GB/year**.
- **Search index**: 8M listings × ~10KB indexed (text, geo cells, filter postings, date
  bitmap per month) ≈ **~80GB** × replica factor (5–10 replicas for fanout capacity) →
  still well under 1TB, small enough that the *entire* searchable corpus can live purely
  in RAM on moderate clusters — this shapes dive 1: we never shard by capacity, only by
  query load and geography.

## Consistency of the top line

- 491M nights/year ÷ 365 = 1.35M nights consumed/day → with 3.5-night avg stay,
  ~140M stays/year → 380K booked/day → at $580 AOV → ~$220M/day → ×365 ≈ $81B/yr. ✓
- 30M searches/day over ~10M sessions at ~4% search→booking conversion ≈ 400K booked
  stays/day vs 380K derived from nights: consistent within the slop of the assumptions. ✓

## What these numbers drive

- **Search is a memory-resident, replication-not-sharding problem** (~80GB corpus) — the
  engineering is in query shape (geo + date-range + filters + rank), not in distributed
  storage heroics.
- **Photos own the bandwidth budget** (5.5K QPS vs 2K peak search) → CDN-first, which is
  also where fake-listing detection hooks image reuse ([04 dive 6](04-deep-dives.md)).
- **Booking/payment QPS is trivial (~13 avg, low hundreds at peak)** but is the only path with an absolute
  correctness requirement (N3/N4) — architecture effort concentrates there, not on the
  hot QPS paths.
- **Escrow float is the business**: ~$220M/day moving through a hold state; payout timing
  is simultaneously a fraud control and a working-capital design.
