# 02 — Scale estimates

All numbers are rough estimates for a launch-and-scale phase (~1M DAU growing to ~10M),
with publicly reported facts marked. The most important estimates here are about
**GPU-minutes**, not QPS — see the framing note at the end.

## Assumptions

- **DAU**: 1M at launch, design headroom to 10M (a product like this grows on virality;
  capacity comes from the fleet, which is bought, not architected).
- **Generations per DAU**: ~3 submitted/day (people iterate; ~40% of submissions are
  discarded or abandoned by the user mid-job).
- **Clip**: 10s average, 1080p output.
- **Compute per clip**: diffusion video generation is publicly characterized as taking
  minutes of accelerator time per clip; OpenAI leadership described Sora training as
  "massively compute-intensive" at launch. Estimate: **~2–4 H100-minutes per 10s 1080p
  clip** at the denoise+decode stages, with headroom language of "up to ~10" for long/
  4K/complex jobs. (This is the load-bearing assumption; every derived number below scales
  linearly with it.)
- **H100 cloud cost**: ~$2–3/hour rental (publicly quoted range) → ~$0.03–0.05 per
  GPU-minute.

## The headline: QPS is trivial, GPU-seconds is everything

- Submissions: 1M DAU × 3/day = 3M jobs/day ≈ **35 submissions/s average** (÷86400);
  peak ~5×, and product-launch days spike 10–50× for hours. This is *pathetic* QPS — one
  modest API pod handles it.
- The real load: 3M jobs × 3 H100-min ≈ **9M H100-min/day ≈ 150K H100-hours/day** ≈
  **~6,250 H100s running 24/7** for 1M DAU (150000/24 = 6,250). At 10M DAU: ~62,500 GPUs. This is why
  every design decision here is a scheduling/cost decision first: the fleet is the product's
  COGS. At $2–3/hr, 150K GPU-hours ≈ **$300–450K/day** (estimate) — for a 1M-DAU product.
  Compare: YouTube's transcode fleet (see
  [YouTube deep dives](../01.youtube/04-deep-dives.md), dive 7) is measured in "1.1M
  machine-CPU-hours/day" but at CPU-class economics and paid back by delivery savings;
  here the GPU spend IS the product, amortized over zero repeat views.

## Storage/day

- **Final artifacts**: 1080p H.264 at ~5 Mbps → 10s clip ≈ 5 Mbps × 10s / 8 ≈ **~6 MB**.
  3M clips/day → **~18 TB/day of publishable artifacts**. Even with 2 renditions each
  (~1.4×) that's ~25 TB/day — round it to **~25 TB/day**. Trivial by video-platform
  standards (YouTube ingests ~orders of magnitude more per day).
- **Intermediates (the hidden cost)**: checkpointed latents + decoded keyframes kept per
  job for resume/edit (deep dives 3, 5). Estimate ~50–100 MB per job retained for ~7 days
  (latents for the full clip at the working resolution plus metadata) → 3M × 75 MB ≈
  **~225 TB/day ingested, retained about a week ≈ ~1.6 PB steady-state**. This is
  *temporary* by design: on retention expiry it is deleted, not archived.
- **Prompts/telemetry**: negligible (KBs per job).

**Lifecycle rule**: final artifacts live as long as the user's library (plus legal hold);
intermediates die after a short TTL unless a pending edit references them. The 225 TB/day
number is the cost of editability, and the TTL converts it from a growth curve into a
fixed steady state.

## Serving bandwidth

- Suppose ~50% of finished clips pass screening and get published/downloaded at least
  once, and the average published clip is watched/downloaded ~10× over its life
  (share-driven, feed-driven): 1.5M published/day × 10 views × 6 MB ≈ **~90 TB/day of
  delivery** — a rounding error next to YouTube's EB/day, but still CDN-shaped (global,
  spiky, cacheable). Use a CDN; the design problem is not delivery scale.

## Queue & scheduling sizing

- Average job: ~3 GPU-min; a job occupies one GPU (or one MIG-slice for the fast-draft
  phase, deep dive 2). ~6,250 GPUs running ~fully busy at 1M DAU means the queue is rarely
  empty: **expected wait is the queue-length signal**, and launching more marketing than
  GPUs is a product decision that manifests as ETA in the UI.
- **Dedup/cache upside**: identical prompts are common (trends). A prompt-hash cache that
  returns an existing screened artifact saves an entire generation (~$0.09–0.15, estimate).
  If ~10% of jobs hit the cache, savings are real but bounded — it is a
  cost optimization, not an architectural pillar (unlike YouTube, where caching is the
  whole delivery story).

## Sanity checks

- Per-user GPU cost: 3 jobs/day × 3 min ≈ 9 GPU-min/day/user ≈ **~$0.27–0.45/day/user
  (estimate)** at $0.03–0.05/GPU-min. That is a subscription-priced product's problem, not a
  free-tier's problem — and it explains the credits/limits structure of the real product.
- Consistency: 3M jobs/day at 35/s average, ~6,250 GPUs — each GPU must average one
  ~3-min job roughly continuously; utilization target ~85–90% after maintenance/drain
  windows, so fleet ≈ 6,250/0.875 ≈ ~7,150 provisioned machines. Numbers close over the
  estimates, which is the point of showing the arithmetic.
- The 10–50× launch spike cannot be absorbed by pre-provisioning (100× is not a fleet-
  sizing question); it is absorbed by queueing with honest ETAs plus per-user limits —
  decided explicitly in requirements, revisited in deep dive 1's failure modes.
