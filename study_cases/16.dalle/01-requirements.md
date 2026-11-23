# 01 — Requirements

## Functional

Core v1:

- **Submit generation**: prompt text (≤ ~1,000 chars), params (aspect/resolution from a
  fixed menu, style hints, N images per request, N ≤ 4). Returns a job ID immediately —
  **the submission is synchronous, generation is not**.
- **Job status**: poll or push (webhook) with states `queued → generating → done |
  rejected | failed`. Status includes queue position or ETA when queued.
- **Fetch result**: image URLs after completion; results are immutable artifacts with a
  metadata record (prompt, params, seed, model version, safety verdict, cost units).
- **History / gallery**: per-user list of past jobs with thumbnails; download original.
- **Quotas**: free tier (a few generations/day), paid tiers (more/day, larger batches,
  priority queue). Quota enforcement is authoritative at submit time.
- **Safety**: prompt-level moderation before spend; image-level moderation before
  delivery; a user-visible rejection reason (coarse, not filter-revealing).

Out of scope v1 (but noted):

- **Model training pipelines** — we operate inference over shipped model versions; training
  is a separate org/system (but model version pinning is in scope).
- **Inpainting / editing / variations on existing images** — new trust problems (user
  uploaded images), skipped in v1.
- **Video generation** — an order-of-magnitude jump in per-job GPU cost; different case.
- **Community feed / discovery of other users' generations** — makes every stored image
  public-by-default; v1 results are visible to their owner and whoever they share with.
- **Payments/billing internals** — we consume a billing service ("charge N units when a
  job completes"), we don't design the ledger.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Submit latency | p99 < 300 ms | submit does validation, moderation, quota check, enqueue — no GPU work on this path |
| Job completion | p50 10–30 s, p99 < 2–5 min under burst | queueing time dominates p99; the product promise is an async ETA, not a latency SLA |
| Never lose a paid job | paid jobs retried to completion or auto-refunded | the job record is durable before any GPU is touched; billing ties to completion, not submission |
| Safety decision before delivery | no unmoderated image ever reaches a URL the user can fetch | forces the post-generation classifier into the result path, see deep dive 4 |
| Availability split | submit: high; generation: degradable; serving: highest (CDN) | an outage of GPUs queues jobs, it must not fail submissions or break already-fetched URLs |
| Consistency | job state eventually consistent; quota strongly consistent | letting a user exceed quota on a race is cheap (one extra generation); losing a paid result is not — so we spend consistency budget on the job record, not the quota counter |
| Scale | ~10M DAU (estimate, not public) | drives the estimate sheet |
| Retention | free results ~30 days, paid ~1 year (product decision, estimate) | storage tiering; deleted jobs leave tombstones in history |

## Scope decisions that shape everything

1. **Async as the contract, not an optimization.** Generation takes tens of GPU-seconds;
   the API would be broken without a job model. Everything downstream (queue design,
   webhook/polling, ETA) follows from this — deep dive 1.
2. **GPUs are the only hard capacity.** Web pods, queues and stores scale elastically;
   the GPU pool scales in minutes at best. So the design *buffers* (queues) and
   *prioritizes* (paid over free) instead of promising latency — deep dive 2.
3. **Cost per image is a first-class design axis.** A 2× inference cost regression shows up
   as a 2× GPU fleet. Step count, batch size and exact-request caching are product
   decisions, not implementation details — deep dives 3 and 5.
4. **Moderation sits on both ends of the GPU.** Prompt filter before spend (cheap, lossy),
   image classifier before delivery (expensive-ish, must be near-complete) — deep dive 4.
