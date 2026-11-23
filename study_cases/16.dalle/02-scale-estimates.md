# 02 — Scale estimates

Assumptions (all rough estimates; adjust and redo the arithmetic if they change):

- ~10M DAU, ~30M MAU (estimate — no official figures to cite)
- ~2 generation requests per DAU/day → **20M requests/day**; N = 4 images per request →
  **80M images/day**
- Prompt ~100 chars ≈ ~200 bytes; job metadata record ~1 KB
- Output 1024×1024: lossless ~1.5–2 MB; compressed (WebP/JPEG q85) ~150–400 KB → use
  ~300 KB compressed, ~1.75 MB lossless
- Inference cost, per image (1024×1024, ~50 denoising steps, batched ~4–8 images per GPU
  batch on an A100-class device): step time ~0.2–0.25 s → **~12 GPU-seconds/image**
  (estimate; see deep dive 3 for where this number comes from and how to cut it)
- Burst profile: submission traffic is human-driven → peak ~3–5× average; the viral-moment
  case is a 10× spike for hours

## Request rates

**Submissions**: 20M/day ÷ 86,400 s ≈ **230 req/s average**, peak 5× → **~1.2K req/s**.
Trivial for the submit tier (each submit is: validate, moderate prompt, check quota, write
job row, enqueue — no GPU).

**Images out**: 80M/day ≈ **925 img/s average**, peak 5× → **~4.6K img/s**. This is the
number that consumes GPUs, and — the point — GPUs cannot scale at 5× in the moment.
Whatever we size the fleet to, the rest must queue (deep dive 2).

## GPU fleet sizing

Per GPU per day: 86,400 s × 0.70 utilization (the rest = batch fill/idle/failed jobs)
÷ 12 GPU-s/image ≈ **~5,000 images/GPU/day**.

Fleet at average load: 80M ÷ 5,000 ≈ **~16,000 A100-class GPUs, running flat out, 24/7**
(11.1K busy on the arithmetic: 925 img/s × 12 s ≈ 11.1K GPU-seconds/s; at 0.70 utilization
that's 11.1K ÷ 0.70 ≈ 16K, i.e. ~43% more than busy GPUs — matching the fleet above).

Sanity check the other way: a 5× peak would want ~80K GPUs *if we refused to queue*.
Nobody sizes for that; the fleet gets sized around ~2× average (~30K) or lower and peak
excess waits in queue — this single observation drives the whole async design.

**The dominant sensitivity**: per-image GPU cost. A distilled model at 8 steps cuts
~12 → ~2 GPU-s/image; the same 80M images/day then needs ~3K GPUs instead of 16K. A 6×
fleet swing from a model change is why inference cost (deep dive 3) is an architecture
axis here, not a detail.

## Storage

- Job metadata: 20M rows/day × ~1 KB ≈ **20 GB/day** (~7 TB/year) — easy.
- Images: this dominates. 80M/day × (1.75 MB lossless + ~110 KB derived thumbs)
  ≈ **~149 TB/day** if originals are lossless; **~33 TB/day** if originals are stored
  compressed and lossless is kept only for paid downloads.
- With free-tier 30-day TTL, at steady state the lossless-store keeps roughly
  30 × 149 TB ≈ 4.5 PB (most of it expiring); without TTLs it grows unbounded. Retention
  policy is doing ~10× of the storage-cost work; dedup (deep dive 5) comes second.

## Reads / egress

Serving skew: an image is generated once, viewed/re-shared several times. Estimate
~2.5 view events per image → 200M views/day; mixed sizes (thumb + full view) ~200 KB avg:

- **~40 TB/day egress** ≈ ~460 MB/s (~4 Gbps) average, ~3× at peak → ~12 Gbps.
- Put behind a CDN: cache-hit 95%+ (results are hot on creation, then lukewarm) → origin
  egress ~2 TB/day. See [CDN case](../09.cdn/04-deep-dives.md) and
  [blob store case](../07.blob-store/04-deep-dives.md) for the serving-tier patterns.

## Consistency check

- 20M req/day ÷ 10M DAU = 2 generations/user/day — matches assumption (heavy users +
  churn of inactive DAU balance out; fine as a rough average).
- 925 img/s ÷ 230 req/s ≈ 4 images/request — matches N=4.
- 16K GPUs × 5,000 img/GPU/day = 80M — round-trips exactly.

## Calibration against publicly listed pricing

API self-serve pricing for DALL·E 3 1024×1024 was publicly listed at roughly
**$0.04/image standard, ~$0.08 HD** (OpenAI API pricing page), i.e. ~$3.2M/day for our
80M retail images if every one sold at API rates (80M × $0.04) — **it wouldn't**: most of our volume is
free-tier, so treat $0.04 as a ceiling on willingness-to-pay per image, not revenue.

Cost side (estimate): A100-class GPU at ~$1.5–2/hour on public cloud → ~$36–48 per
GPU-day × 16K GPUs ≈ **~$600–750K/day** at average load. Even if half the images sold at
API rates (~$1.6M/day), revenue would be only ~2.4× GPU rent alone ($1.6M ÷ ~$700K) —
tight, and before
amortizing training, safety-filter GPUs, and over-provisioned peaks. That arithmetic is
the argument for distilled samplers (deep dive 3), spot capacity (deep dive 2), and
exact-request caching (deep dive 5): each cuts the numerator directly.

Cross-check the implied $/GPU-hour of inference: ~$700K/day ÷ 80M images ≈
$0.009 GPU-rent/image vs $0.04 list price — the gap is margin + training amortization +
the peak-vs-average overbuild, which is consistent with how it must work, and is all
estimate.
