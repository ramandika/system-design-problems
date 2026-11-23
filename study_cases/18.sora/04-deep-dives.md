# 04 — Deep dives

## 1. The cost asymmetry: why a generated clip is ~10^3× a served clip

Arithmetic, explicitly:

- **Serve**: one viewer watching 10s of a 6 MB clip ≈ 6 MB of CDN egress ≈ $0.0001–0.0006
  (public CDN egress rates, rough) and ~0 CPU-seconds of origin work.
- **Generate**: ~3 H100-minutes for a typical clip (estimate, [02](02-scale-estimates.md))
  ≈ $0.09–0.15 of raw compute at 02's ~$0.03–0.05/GPU-min; long/4K jobs at 02's "up to
  ~10 GPU-min" headroom reach ~$0.30–0.50. Idle, retries, and failed jobs sit on top.
- Ratio: **~10^3× in dollars per interaction** (~$0.09–0.15 typical vs
  $0.0001–0.0006 — ~150–1,500×; the
  serve side costs ~0 machine-seconds of origin work, so no machine-seconds ratio is
  meaningful) — and unlike
  a served view, a generation produces no reuse: its "value" is consumed once by one user.

Every architectural consequence flows from this, so write them down where design reviews
will see them:

- **Option A — generate everything submitted immediately** (best effort). Wins: simplest
  product surface, no queue UX. Loses: unbounded COGS; one viral moment with a "generate
  everything" meme and a bot farm is a budget-destroying event; no honest mechanism to
  shed load. **Option B — queue + cost-admission control**: every submission is priced in
  expected GPU-minutes, user credits are the budget, and the queue's ETA is the honest
  product response to scarcity. Loses: latency variability users notice; queue management
  complexity. **Chosen: B, unambiguously.** A system whose marginal cost is GPU-minutes
  cannot have an "unlimited free" plane; the queue *is* the product.
- Serve-side plane is deliberately severed from generation-side: CDN-backed library keeps
  playing when the GPU fleet is entirely down. This is the same "two planes" discipline as
  YouTube's operational metadata vs analytics warehouse split (see
  [YouTube deep dives](../01.youtube/04-deep-dives.md), dive 5) — don't let a slow/expensive
  plane drag a free one down.
- Draft-pass gating (deep dive 2's preview) is the cheap version of "don't pay for junk
  inputs": ~15s of draft compute kills the ~1/3 of prompts users abandon, for ~1/10th of
  a full job's cost.

**Failure modes in this dive**: (a) a bug that lets a finished artifact be served without
a screening verdict re-opens the moderation plane for the cost asymmetry's ugliest abuse —
treat "verdict exists" as an invariant, checked at publish; (b) a cache-key bug that
returns another user's artifact over prompt-hash collision leaks content — keys include
user scope or the cache only serves from *already-published* artifacts.

## 2. The chunked latent pipeline as a job DAG

The model does not produce 10s in one shot. Design: **split into temporal chunks**
(~2–5s each), generate chunks as independent (or boundary-conditioned) GPU jobs, then
assemble. This is YouTube's segment-transcode DAG turned inward onto the *generation*
step (same shape, opposite economics — see
[YouTube deep dives](../01.youtube/04-deep-dives.md), dive 1):

```
chunk DAG (10s clip, 2 x 5s chunks):
  plan prompt -> decompose -> [C1: denoise_steps x N] -> [assemble -> upscale -> encode]
                            -> [C2: denoise_steps x N]     (C2 conditioned on C1 boundary)
```

- **Option A — one monolithic GPU job**: temporal coherence is the model's own problem;
  chunk-boundary artifacts disappear. Loses: no parallelism (10s clip = whole 3 GPU-min
  on one machine, serialization only), and the retry unit is the whole clip — one CUDA
  fault at minute 2 pays the entire bill again. **Option B — chunked DAG**: near-linear
  latency win from parallel chunks (with the boundary-conditioning caveat below), a
  retryable unit of ~5s of work, and checkpoints smaller than the whole clip. Loses:
  **temporal coherence across boundaries** — the seam where chunk 1 hands off to chunk 2
  can visibly jump, unless the model was trained (or is conditioned) on overlapping
  boundary frames. **Chosen: B.** Generation latency is user-facing minutes already;
  halving it matters. Seam quality is a *model* capability problem that the *system* pays
  for in a small overlap-and-blend pass at assembly (re-generating just boundary frames,
  not whole chunks).
- **Chunk state machine per job**: `planned → running (per GPU) → checkpointed (per few
  steps) → done → assembled`. The orchestrator owns state; GPUs are stateless executors
  that pull work. A GPU that burns a job leaves no residue: its last checkpoint is the
  truth.
- **Stragglers**: one slow/failing chunk must not hold the clip hostage — speculative
  re-execution of the chunk on a second GPU after a threshold, first finisher wins
  (identical discipline to YouTube's straggler segments; here the waste is priced in
  H100-minutes, so the threshold is stricter — a straggler retry must cost less than
  the ETA pain it saves, and the metric to watch is wasted GPU-fraction).
- **Draft passes on the draft pool** cheaply reject prompts that produce garbage before
  the main pool is touched.

**Failure modes**: orchestrator decompose-bug producing a DAG where the "boundary
condition" chunk is missing (clip assembles with a visible seam; caught by a boundary-
frame classifier at assembly); priority inversion where internal eval jobs starve user
jobs (bounded: eval jobs get a separate queue with its own budget).

## 3. Resumable, retryable generation after partial failure

A 3-minute job on a machine that dies or is preempted for maintenance must not restart
from zero. Design:

- **Chunked denoising checkpoints**: within a chunk-job, snapshot latent state every
  ~k denoising steps (tuned so checkpoint I/O overhead is a few % of the step budget —
  the snapshot size is tens of MB, so every few seconds to the artifact store is viable).
  A resumed chunk-job reloads the latest snapshot and continues. Retention is keep-latest:
  each chunk-job's write supersedes the previous snapshot for its key (superseded step
  snapshots are deleted on write), and only the chunk's final snapshot survives into the
  7-day editability window of deep dive 4. Without that GC the per-step keys would
  accumulate ~36 snapshots × ~20 MB ≈ ~700 MB per chunk (multi-chunk jobs: GBs) — a few
  PB/day at 3M jobs/day, not 02's ~225 TB/day (estimate) — so keep-latest is what makes
  the 02 estimate hold.
- **Idempotent keyed execution**: every chunk-job has a stable id `(job_id, chunk_idx)`;
  artifact writes are content-addressed and keyed by that id + model version + step.
  Re-running overwrites the same key. This is at-least-once execution whose artifacts
  collapse to effectively-once — the exact wire/store split WhatsApp uses for messages
  (see [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md), dive 1: retry blindly at
  the transport, dedup provably at the app layer).
- **Model version pinning**: the job record pins the exact model version + params; a
  fleet-wide rollout mid-job must not silently mix versions into one clip. Resume only if
  the pinned version is still served on the pool; otherwise the job restarts from its
  last *version-matched* checkpoint, or fails honestly.
- **Option A — checkpoint only at chunk boundaries** (coarse). Wins: almost no I/O
  overhead, simple state machine. Loses: losing ~a full chunk's work on failure (~1–2
  GPU-min wasted per incident; at millions of jobs/day that's real fleet capacity).
  **Option B — checkpoint every k steps** (fine). Wins: failure loses ~k steps only.
  Loses: I/O overhead and checkpoint-write contention (mitigable: async writes off the
  GPU's critical path, compressed latent tensors on the store side). **Chosen: B with a
  calibrated k** — preemptible GPU fleets make interruptions routine, not exceptional, so
  the "exceptional cost" being guarded against is actually the common case.

**Failure modes**: (a) checkpoint poisoning — a checkpoint written by a buggy worker
version gets resumed forever by newer workers; guard with version-tagged checkpoints and a
max-resume age (resume older than X minutes from the best current checkpoint); (b) zombie
jobs — an orchestrator crash loses leases, GPUs keep working for a dead job; fix with
lease-based job claiming and TTL'd leases (the same claim/lease pattern the scheduler
would use anywhere); (c) silent double-generation when dedup-by-key is bypassed in an
edge path — the wasted money shows up as fleet utilization drift, so utilization vs
job-count is a monitored correlation.

## 4. Storage + CDN of large artifacts (small by video-platform standards)

From [02](02-scale-estimates.md): ~25 TB/day final artifacts, ~225 TB/day transient
intermediates, ~90 TB/day delivery. Nothing here is large by YouTube standards — but the
*shape* is different, and that shapes the tiering:

- **Two classes with opposite lifecycle**: finished artifacts are immutable, tiny, and
  long-lived → object store + CDN, ordinary (library links point at CDN paths; the clip
  is content-addressed so re-encoding renditions can be done idempotently). **Intermediates
  (latents, boundary frames) are large, mutable-resume targets, and short-lived** → separate
  store prefix with a TTL (~7 days, sized so an edit within that window reuses state, deep
  dive 5), deleted rather than archived. The 225 TB/day is the *cost of editability*, and
  the TTL turns it from a growth curve into a steady state — the same call as WhatsApp's
  30-day offline TTL (see [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md), dive 3),
  taken for different reasons (editability vs delivery probability).
- **Where checkpoints live matters for resume latency**: on a preemption, a resumed chunk
  job wants its latent snapshot in the nearest cheap tier — regional object store, not
  global. The global artifact CDN is for published output, not for machine-to-machine
  checkpoint reads (keeping the two paths separate prevents checkpoint I/O from competing
  with user playback).
- **CDN design** is exactly YouTube's stack (edge → shield → origin, negative caching for
  blocked artifacts) — see
  [YouTube deep dives](../01.youtube/04-deep-dives.md), dive 3. The one Sora-specific note:
  blocked/rejected artifacts must be *negative-cached* at the edge so probing abuse doesn't
  turn into origin reads of content the system decided not to publish.
- **Watermarking at publish**: visible + invisible watermark and a C2PA-style manifest are
  applied once, at publish, before the CDN sees the artifact — the CDN serves only
  watermarked variants, so there is no "unwatermarked variant accidentally cached at an
  edge" class of incident.

**Failure modes**: TTL bug that deletes checkpoints while an edit is pending (edit falls
back to full regen — acceptable, but log it: it means the TTL or edit-reference GC is
wrong); watermark pipeline outage (artifact not publishable rather than published
unmarked — fail closed; publishing unwatermarked synthetic video is a worse failure than
delaying publication).

## 5. Prompt refinement: edit = regenerate vs edit = patch

The user gets a clip, then tweaks the prompt. What happens next?

- **Option A — full regeneration**: re-run the whole DAG from the new prompt. Wins:
  trivially correct, no state to reuse, no dependency tracking. Loses: pays a full price
  (~3 GPU-min, ~$0.09–0.15) for a change that might affect one chunk; worse, the result
  is *plausibly a different clip entirely* — the user's mental model "my clip, but with X"
  breaks, which is the actual product failure. **Option B — chunk-level patch (chosen)**:
  reuse checkpointed latents of untouched chunks; regenerate only chunks whose segment of
  the prompt changed; the DAG for the edit is a copy with some chunks marked `reused`
  (deep dive 3's idempotency keys make a `reused` chunk a metadata pointer, not a GPU
  run). Wins: cost/latency roughly proportional to how much changed; and the *unchanged*
  chunks being bitwise-identical is what makes the edit feel like the same clip modified.
  Loses: dependency tracking (a change early in the clip can invalidate downstream chunks
  that conditioned on it — the orchestrator must recompute an invalidation closure), and
  a "patch hole" quality risk when the seam between reused and regenerated chunks is
  awkward (mitigated the same way as deep dive 2's boundaries).
- The edit DAG is the reason intermediates get a 7-day TTL (deep dive 4): without them,
  every edit silently degrades to Option A. The state is kept for exactly the window in
  which users iterate; after that, full regen is the correct answer and nobody notices.
- **Remixes/derivative prompts** (viral prompt shared by many users) sit in the same
  machinery: chunk-level reuse where prompts share a prefix, plus the prompt-hash cache
  from [03](03-architecture.md) for exact duplicates. Cache upside is bounded (deep dive
  1's note) — it optimizes cost, it does not remove it.

**Failure modes**: invalidation bug under-invalidating (user sees a stale chunk inside an
"edited" clip — a trust-breaking bug; guard with a per-edit diff manifest reviewed by the
orchestrator, and version the invalidation closure logic); over-invalidating (edit costs
a full job — invisible to the user as wrongness, visible in COGS; keep a per-edit
"reused-chunk ratio" metric and alarm when it collapses).

## 6. Moderating novel content before release

Every clip is content that has never existed and has no fingerprint to match against.
Takedown-after-publication is the cheap standard for uploads; here it is not
sufficient — the marginal cost of generating the content means abuse (harassment,
likeness violations, CSAM, dangerous instructions rendered as video) needs to be caught
*before* the artifact leaves the pipeline. Design as a four-stage gauntlet, ordered
cheapest-first:

1. **Prompt-side pass (cost ~0)**: text classifiers + policy rules on the prompt at
   intake. Catches the trivially bad and cheaply refuses obvious probe patterns. Blocks
   here are logged for model-improvement feedback.
2. **In-pipeline pass on latents (cheap early signal)**: during the draft pass, the
   model's coarse latents already sketch composition — a light classifier on decoded
   preview frames can flag "this is heading somewhere forbidden" before the expensive
   full render. Early cancellation saves real GPU-minutes on the worst-content jobs.
   This is a genuinely generation-native tool: the *pipeline itself* is a progressive
   preview of the content.
3. **Final artifact gate (mandatory)**: classifiers on the finished clip (video-level,
   frame-sampled) run before publish; nobody's art bypasses this. Verdicts: publish /
   human-review / block.
4. **Human review queue (borderline only)**: reviewers see artifacts with verdicts,
   context, and the prompt. The queue is capacity-constrained and publication
   backpressure is deliberate — when reviewers are saturated, clips wait in
   `ready-but-unpublished` (generation continues; *publication* slows). Never let review
   backlog push artifacts out unreviewed; fail closed.

- **Option A — post-publish takedown**. Wins: zero moderation latency in the happy path;
  scales with no queue. Loses: unacceptable here — synthetic likeness/violent content
  that got seen once cannot be un-seen, and there is no "same video as that other one"
  matcher for fast propagation checks. **Option B — pre-release gate (chosen)**: users
  wait (screening adds ~seconds-to-minutes p50; human queue adds hours for the
  borderline slice), and the moderation cost grows with generation volume (classifiers
  are cheap but not free — budget them in the same fleet accounting as deep dive 1).
  **Chosen: B**, because the product's failure mode is not "slow" but "a clip that should
  never have existed is public."
- **Provenance** (deep dive 4's watermark, C2PA manifest) is part of the moderation
  story, not a feature: a clip that leaves the pipeline must be identifiable as machine-
  generated forever, which bounds the downstream harm of any miss.

**Failure modes**: adversarial prompts engineered to pass the text stage and surface
content only visually (mitigation: stages 2–3 are exactly for this; accept that evasion
is an arms race, monitor for drift in what stage 3 flags after stage 1 cleared); human
reviewer queue poisoning (submitting borderline-but-fine content to saturate reviewers so
genuinely bad content flows through — detect by per-user review-consumption rate and
throttle generation for users who consume disproportionate review capacity); classifier
false positives nuking legitimate clips (appeals path with human review, and publish
verdicts to the user — a wrong block hurts less if the user sees why and can push back).

## 7. (Summary discipline) The three knobs everything else serves

Every dive above reduces to one of: (1) **budget** — GPU-minutes are the currency; every
feature (drafts, queue, cache, TTLs) is priced in them; (2) **resumability** — the pipeline
is a DAG of idempotent, checkpointed, version-pinned chunk-jobs; (3) **gating** — nothing
cost-free or content-bearing leaves the pipeline unverified. If a new feature can't state
its effect on all three, it hasn't been designed yet.
