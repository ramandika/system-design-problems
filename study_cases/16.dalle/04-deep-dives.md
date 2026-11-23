# 04 — Deep dives

## 1. The async job model end-to-end (submit → queue → GPU → deliver)

The submit API must be cheap and instant; generation is tens of GPU-seconds. The job is
therefore a durable entity with a state machine, and the whole design problem is who owns
transitions between states.

- **Durable first**: submit writes the job record (prompt, params, seed, model version,
  owner, state) before enqueueing. If the enqueue fails, the record exists and a sweeper
  re-enqueues — a submit that returns success never silently does nothing.
- **Claiming is leasing, not consuming.** A GPU worker leases a job with an idempotent
  client-generated ID and a lease timeout. If the worker dies mid-batch, the lease expires
  and the job returns to the queue with attempt count + 1. At-least-once execution;
  **exactly-once effects** come from idempotency: uploads are keyed by (job ID, image
  index) so a retry overwrites instead of duplicating; billing keys on the job state
  transition, not the GPU call. This is the same wire-protocol shape as
  [WhatsApp's at-least-once + client dedup](../03.whatsapp/04-deep-dives.md) — make retry
  safe at the edge, get user-level "exactly once".
- **Delivery: poll or webhook?** Poll = status reads from a cheap cached read model;
  simple, works everywhere, adds load as users mash refresh. Webhook = push on state
  change; needs a registered endpoint (API users) and inherits the dedup/retry problem.
  **Chosen: polling as the universal baseline, webhooks as an API-user upgrade** — both
  emit from the same state machine, and events are at-least-once with client-side dedup.
- **Seeds assigned at submit, not at generation.** Deterministic (prompt, params, seed,
  model version) → the job record becomes the cache key for deep dive 5 and the replay
  artifact for support/regeneration.

- **Trade-off**: the durable state machine adds a store + a sweeper + lease machinery
  around what could naively be "queue → worker → push result"; in exchange, worker death,
  duplicate delivery, and re-execution are all benign, and billing correctness has a
  single source of truth. Without it, a GPU crash loses a *billed* job — a direct money
  and trust bug.

**Failure modes**: sweeper bug double-enqueues → attempts counter caps retries; queue
store partition → new submissions stall but already-queued jobs drain on the other side;
model version rolled mid-batch → mixed-version results → pin version per job and finish
the batch on the pinned version.

## 2. GPU scheduling under bursty load (reserved + spot + batching)

Fleet arithmetic (02): ~16K GPUs at average load; a 5× peak *without queueing* wants ~80K.
Sizing is thus a three-layer decision:

- **Reserved/baseline pool** sized near average (~1×), committed capacity, paid tier
  routes here first. Predictable latency for paying users.
- **Spot pool** for the second segment, at 60–70% off public-cloud rates: spot instances
  face preemption risk, which is survivable *only if jobs are leasable* (deep dive 1:
  lease expiry returns the job). Free tier goes here by policy.
- **Queue buffer** absorbs everything above fleet capacity. Queue depth becomes the
  product surface: show position/ETA, don't hide it.

- Batching: GPUs want batches of ~4–8 images; arrival gives one job at a time of 4.
  **Trade-off: bigger batches raise throughput (better utilization, cheaper per image) and
  raise latency (the first image of a batch waits for the last slot)**. Chosen: cap batch
  wait at ~2–5 s via a micro-batcher, and let the free tier fill leftover slots — paid
  jobs don't wait for free ones, free jobs get free capacity.
- Priority inversion risk: a free job leased to a spot worker that gets preempted while
  paid jobs queue — bounded by routing tiers and preemption-aware scheduler re-shuffles.

- **Trade-off (the core one)**: fleet sized for peak ("no queue, ever") = ~5× rent forever;
  fleet sized for average = free-tier users see minutes of queue at peak. We pick the
  queue, *because* queueing pain lands on the free tier and latency promises land on
  paying users — the quota design makes the trade fair. (The GPU analog of Twitter's
  celebrity write offload: [Twitter deep dive 1](../02.twitter/04-deep-dives.md) also chose
  to keep a class of worst-case work off the hot path — celebrity writes served on read —
  rather than provision 100M-follower fanout.)

**Failure modes**: autoscaler lag (minutes) vs spike (seconds) → queue rides it; spot
mass-preemption during a cloud zone event → free tier drains slowly, reserved tier
unaffected by design; micro-batcher head-of-line blocking → cap wait, never let a free
image delay a batch slot.

## 3. Diffusion inference cost per image

Where ~12 GPU-s goes (estimate, 1024×1024): ~50 denoising steps × ~0.2–0.25 s/step
(batched) + text-encode/startup. The cost is *per step per pixel area*; everything below
attacks one of those terms.

- **Fewer steps**: a distilled sampler (e.g. few-step teacher-student distillation) cuts
  ~50 → ~4–8 steps at some quality cost. Second-order: fleet 16K → ~3K (02). This is a
  model-version decision made jointly with product, not eng preference.
- **Batching**: 8-image batch amortizes per-step overhead — throughput gain with the
  latency trade of deep dive 2.
- **Resolution**: 1024² has 4× the pixels of 512²; generating at lower
  latent + upsampling is cheaper per image at a fidelity cost.
- **Regeneration economics**: the sampler is stochastic (seeded); failures and
  content-filter rejections burn the full cost. Reducing the rejection rate is itself a
  GPU-cost lever (prompt-side filtering before spend, deep dive 4).

- **Trade-off**: cost/quality is explicit and quantifiable here — "Option A: 50 steps,
  ~12 GPU-s, best quality; Option B: 8 steps, ~2 GPU-s, slightly softer fine detail; we
  pick B for the free tier and A for paid HD" — tiered quality by tier monetizes exactly
  the thing the GPU fleet costs.
- Contrast with LLM serving ([ChatGPT](../08.chatgpt/04-deep-dives.md)): same rented-GPU
  economics, but text tokens are small and continuous-rate; images are **bulk units**
  (a generation is atomic, seconds-long) — you can't stream-half an image, so queueing
  granularity and batch packing matter far more here.

**Failure modes**: model regression doubles effective step time → capacity quietly halves,
needs a GPU-throughput regression gate at deploy; OOM at large batch → capped batch size
per device type; silent numerical drift with bf16 optimizations → visual-quality check in
the deploy pipeline, not just throughput.

## 4. Safety filter: in-line vs after-the-fact

Two filters, two moments, two different jobs:

- **Prompt-side (pre-spend)**: cheap text classifier + rule layer at the gateway, before
  any GPU. Lossy and gameable (adversarial phrasing evades it), but it keeps ~x% of
  bad spend off the GPU and returns a fast, actionable rejection.
- **Image-side (pre-delivery)**: NSFW/violence classifier run on generated images
  *in the result path*, before any URL a client can fetch exists. GPU cost is small next
  to generation but sits on the critical path (adds ~a few hundred ms; runs on the same
  worker CPU/GPU host batch-amortized).

Why in-line on images: an image URL that leaks unmoderated for even minutes is not
"eventually moderated" — it was delivered, screenshotted, re-hosted, and re-uploaded
forever. So:

- **Option A — filter only after the fact** (generate → serve → scan): wins ~latency on
  the hot path and simpler delivery; loses an unbounded abuse window on every image and
  turns the blob store into an archive of policy violations.
- **Option B — everything in-line** (classify every image before delivery): wins the
  guarantee ("no unmoderated URL ever exists"); loses latency (bounded, small) and some
  GPU/CPU spend.
- **Chosen: B for the delivery gate, with A's spirit elsewhere** — the *prompt* filter is
  cheap-and-lossy (a miss there costs GPU money, not a violation leak); a small
  borderline band routes to human review, while the job's state stays `pending-review`
  and the client sees a hold, not a URL.

- **Trade-off**: false-positive cost is real (a legitimate "medical illustration" prompt
  rejected is a support ticket); false-negative cost is a regulatory/trust incident. The
  two moments let us tune them independently — prompt-side loose (save GPU money), image-
  side tight (protect delivery). A single merged filter can't make both errors small at
  once.

**Failure modes**: classifier latency spike → delivery stalls (visible queue growth,
not violations leaking — correct degradation); classifier regression at deploy → shadow
scan first, gate second; human review backlog → jobs expire out of review after a horizon
with a refund, never silently dropped.

## 5. Result storage, CDN, and dedup by exact request

**Serving shape** (02): ~40 TB/day egress vs ~150 TB/day written. Generated-once,
read-many, hot right after creation: textbook CDN material
([CDN case](../09.cdn/04-deep-dives.md)) with blob-store mechanics
([blob store](../07.blob-store/04-deep-dives.md)). Full-res sits in origin object store
with TTL/lifecycle tiers; derived sizes are proactively pushed to CDN on job completion
(creation is the known access event) rather than pulling on first miss.

**Dedup — same prompt, same params.** Many users ask for identical or near-identical
things (the same meme prompt hits thousands of times). The key is
hash(prompt ‖ params ‖ seed ‖ model/version), assigned at submit (deep dive 1):

- **Option A — exact-request cache**: a second user's identical request looks up the
  completed result and pays zero GPU; store the mapping (hash → result) with a TTL.
  Wins: direct GPU-money savings per hit and free headroom at peak; loses: a hit is
  per-user privacy-visible if cached results leak between users (mitigate: serving tier
  re-copies the blob under the new owner's namespace — a metadata copy, not a generation),
  and seeds are user-random per account.
- **Option B — semantic cache** (embed prompts, reuse for "similar" prompts): wins far
  more hits; loses the correctness core — two similar prompts *should* produce different
  images, and silently returning another user's variant breaks the product promise.
- **Chosen: A, exact-keys only.** Semantic near-matching is rejected explicitly: the cost
  of a wrong serve is a user-visible correctness bug; the hit-rate gap doesn't justify it.

- **Trade-off**: exact-key caching adds a lookup + mapping table (small KV, TTL'd, ~1 KB
  keys) in exchange for skipping whole GPU batches on repeat requests; we cap the win by
  serving a *content-shared blob under per-owner metadata*, keeping account isolation.
  Estimated hit rate modest (tens of % for viral prompts — estimate), but each hit is a
  full ~12 GPU-s avoided — a few % of fleet for near-zero engineering risk.

**Failure modes**: dedup table hot key (viral prompt) → consistent-hash + replica read
path; TTL too long → serving a result generated by a since-patched model version (key
includes model version — solved); deleted-user blobs → origin lifecycle rules with
tombstones in history only.

## 6. Prompt injection and abuse at the content boundary

Prompts at a content boundary are adversarial inputs in the classic sense, plus the
service has a *billable meter* worth farming:

- **Jailbreak phrasing** ("ignore the policy...", role-play framing, character-substitution
  in obfuscated terms) targets the prompt classifier; multi-turn attempts probe
  rejection reasons. Defense-in-depth: layered classifiers with different blind spots,
  plus the image-side gate (deep dive 4) as the second net that doesn't depend on the
  first. Rejection UI is coarse and stable — explaining *why* failed becomes an evasion
  oracle.
- **Quota farming**: free-tier quotas per account; botfarms multiply accounts. This is the
  account-level abuse problem — rate-limit on account, IP, device and account-graph
  features (signup velocity, payment instrument reuse), with the quota counter strongly
  consistent at submit. The shape of the problem (metered writes, skewed abusers) is
  exactly [Twitter's rate limiting and abuse](../02.twitter/04-deep-dives.md) dive —
  apply limits *before the expensive act* (there: fanout; here: GPU spend).
- **Inference-data poisoning via feedback loops** ("thumbs-up" on adversarial content
  steers future models) and **extraction probes** (reproduce memorized training content).
  Mitigations live in the safety pipeline + human review, not in serving architecture —
  but the review queue capacity is an architecture sizing input.
- **Embedded attacks in prompts**: crafted text strings targeting the *pipeline* (control
  characters, prompt-injection against downstream text-processing steps, e.g. prompts
  crafted to break classifiers). Treat prompt bytes as untrusted data everywhere: never
  concatenate verbatim into logs, analytics, or classifier contexts unsanitized.

- **Trade-off (the standing one)**: tighter filters → fewer violations reaching delivery,
  more false rejects (legitimate prompts bounce, users churn); looser → abuse and
  violations leak. We spend on *layered* filtering rather than on a maximally strict
  single filter, because layers have complementary blind spots and a false positive at
  one layer can be appealed (human review), while a false negative at all layers is what
  actually matters.

**Failure modes**: moderation bypass published (a filter evasion goes viral) → rate of
violations spikes; mitigations are operational: shadow-evaluate new classifiers against
live traffic, keep the image-side gate model-swappable, and make quota enforcement
fail-closed (gateway degrades to rejects, not un-metered generations).
