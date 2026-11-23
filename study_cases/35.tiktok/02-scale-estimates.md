# 02 — Scale estimates

Public anchor: TikTok publicly reported ~1B monthly users in 2021 and the figure has
grown since; public measurements of similar apps put mobile data use at roughly
**0.7–1 GB/hour** of scrolling. Everything below scales from these anchors plus explicit
assumptions; change one and redo the arithmetic.

## Assumptions

- **MAU ~1B (publicly reported), DAU ~500M (estimate** at a 0.5 DAU/MAU ratio typical of
  engagement-heavy social apps**)**.
- **Sessions: 8/DAU/day × 12 min avg → ~96 min/DAU/day** (consistent with the ~90–100
  min/day figures that circulate publicly for similar apps; treat as an estimate).
- **Impressions**: assume ~70% of session time is active scrolling and the swipe cadence
  is one new video every ~7 s → (96 × 0.7 × 60) / 7 ≈ **~600 impressions/DAU/day**.
  At 500M DAU → **300B impressions/day** ≈ 3.5M impressions/s mean, ~10–14M/s peak.
- **Feed fetches**: the client pulls a batch of ~10 candidates each request (prefetch
  ahead of the finger) → 300B / 10 = **30B feed requests/day** → mean
  30B / 86,400 ≈ **350K QPS**, peak (3–4×) ≈ **1–1.4M QPS**.
- **Videos**: mean length ~25 s; effective average bitrate ~2.5 Mbps (720p-class
  rendition; the ladder matters, see dive 2) → full video ≈ 25 s × 2.5 Mbps ≈ **8 MB**.
  The heavy swipe-away skew means a mean impression consumes only ~25% of the bytes →
  **~2 MB/impression**.

## Egress (the number that dominates cost)

- **300B impressions/day × 2 MB ≈ 600 PB/day ≈ 7 TB/s mean egress**, ~20+ TB/s peak —
  nearly all at CDN edges ([Case 09](../09.cdn/README.md)); with ≥ 90% edge hit ratios
  the origin sees ~6 PB/day per percent of miss (estimates).
- **Cross-check, top-down**: ~96 min/day × 0.7–1 GB/h ≈ 1.1–1.6 GB/DAU/day → 550–800
  PB/day. The bottom-up 600 PB/day lands inside that band, so the impression × bytes
  assumptions are mutually consistent (estimates throughout).
- N4's per-DAU budget: 600 PB ÷ 500M ≈ **1.2 GB/DAU/day** — the ladder decision (dive 2)
  is worth tens of percent on this single number, which is why it is a first-order
  design lever, not a media-pipeline detail.

## Upload and processing scale

- Assume ~6% of DAU publishes on a given day (estimate) → ~30M uploads/day → mean
  ~350 uploads/s, peak ~1.5K/s.
- Raw ingest: 30M × 8 MB (original) ≈ **240 TB/day** through the upload service.
- Transcode: assume the ladder costs ~3× realtime in CPU-seconds across all renditions
  (estimates; per-title encoding spends more) → 30M × 25 s × 3 = 2.25 × 10⁹ CPU-s/day
  ≈ **~26K cores sustained**, bursty at peak hours.
- New stored data: original ~8 MB + ladder ~10 MB + thumbs/sprites ~0.3 MB ≈
  **~18 MB/video** → **~540 TB/day** written into the blob store
  ([Case 07](../07.blob-store/README.md)).
- Retention shape: the engagement long tail means most uploads accumulate near-zero
  views after their first weeks. Assume ~70% of uploads expire from hot tiers after
  ~7–30 days and only surfaced/engaged videos persist. Upper bound hot tier with
  90-day retention = 90 × 540 TB ≈ **49 PB**; with expiry the steady state plausibly
  sits at **~15–25 PB hot**. Raw write volume is 540 TB/day ≈ ~200 PB/year; the archive
  (long-term tier) grows at ~60–200 PB/year depending on what fraction of
  expired-from-hot uploads are kept rather than deleted (all estimates — the point is
  the *order*, hundreds of TB daily writes, tens of PB hot).

## Feed computation scale (drives dive 1)

For one feed request serving 10 items, assume a funnel:

| Stage | Candidates | Model | Per-request work |
|---|---|---|---|
| Recall (indexes + ANN + follow + trending) | ~10K | none (retrieval) | 10K fetches |
| Seen/dedup/rules pre-filter | ~6K | none | set ops only |
| Coarse rank (tiny model) | ~6K | cheap | ~6K scores |
| Fine rank (heavy model) | ~600 | expensive | 600 scores + feature fetches |
| Slate/re-rank (diversity, quotas) | top ~150→10 | light heuristics | ~150 ops |

At 30B fetches/day:

- **Coarse-rank scoring ≈ 30B × 6K = 1.8 × 10¹⁴ evaluations/day ≈ 2B scores/s mean
  (≈ 2 × 10⁹; ×3–4 peak)**. This is the number that forces the coarse ranker to be tiny and forces
  rules/dedup to run *before* any ML — a microsecond of wasted per-candidate work costs
  a fleet.
- **Fine-rank scoring ≈ 30B × 600 = 1.8 × 10¹³/day ≈ 200M heavyweight scores/s** —
  batched GPU/CPU inference; this matches the scale discussed in
  [Case 15](../15.ml-ranking-system/02-scale-estimates.md).
- **Item-feature lookups ≈ 30B × 6,600 = 2 × 10¹⁴/day (same order as coarse
  scoring)** unless item features are cached
  in-process — hence item features live near the ranker, user features are computed
  once per request and shared across candidates.

## Telemetry write scale (drives dive 4)

- Events per impression: play start, progress heartbeats, completion, swipe-away
  metadata, plus sparse social actions (like/share/follow/report), ≈ **7 events per
  impression** → 300B × 7 ≈ **2.1T events/day** → mean **~24M events/s**, peak
  ~75–100M/s (estimates).
- Client batching (~50 events per POST) keeps the *request* rate at ~500K QPS mean —
  the event rate is the pipeline's problem, not the edge load balancer's.

## Cold-start budget (drives dive 5)

- Exploration share: to give every new video an initial audience, reserve a
  guaranteed burst of ~300 impressions per video in its first day: 30M × 300 =
  **9B impressions/day ≈ 3% of all impressions** — the price of keeping the catalog
  open. Cheap in aggregate; the hard part is which videos get *extended* beyond the
  burst.

## Video-metadata read scale

Every impression carries metadata in its fetch response: 30B requests × 10 items =
300B metadata reads/day ≈ 3.5M QPS — served overwhelmingly from caches since video
metadata changes slowly (view/like counters are approximate and lag; see dive 4).
