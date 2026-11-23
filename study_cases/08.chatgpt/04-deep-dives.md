# 04 — Deep dives

## 1. GPU inference pool & batching: why batching is the whole ballgame

**The physics**: for a ~100B-class model, one GPU's memory bandwidth streams model weights
from HBM to compute units on *every token generated*. Generating one token for one user
touches ~hundreds of GB of weights. Generating the 41st token for a second concurrent user
touches **the same weights** — the marginal cost of decoding a batch of N sequences
simultaneously is memory bandwidth, which is nearly flat with N until KV cache/activation
memory saturates. This is the opposite structure of every web-service bottleneck in
[Twitter](../02.twitter/04-deep-dives.md) and YouTube (there: capacity ∝ number of servers ∝
requests; here: capacity has a "free lunch" dimension called batch size).

- **Option A — one request per GPU** (static, dedicated): wins: simple placement, predictable
  per-request latency, no interference. Loses: GPU utilization ~5% (bandwidth-bound decode
  leaves compute idle; weights fetched for a single stream); fleet × ~10 bigger; economically
  impossible at our scale (a 20K-GPU fleet becomes 200K).
- **Option B — static batching**: collect N requests, run the whole batch to completion
  together. Wins: utilization jumps (batch 50+), placement simple. Loses: the batch waits
  for its *slowest* member — a 2,000-token answer holds 60 faster requests hostage; short
  requests get padded with idle decode steps; TTFT for queued requests is destroyed because
  admission happens only between batch lifetimes. With ~30 s mean residency and 500K
  in-flight, batch-slot stranding wastes double-digit percentages of fleet.
- **Option C — continuous (iteration-level) batching**: at every decode step, finished
  sequences leave and admitted requests join mid-flight. Wins: near-full utilization with
  no stranding; queue entry every ~50 ms token step (the paged-attention/vLLM result —
  published, orders-of-magnitude throughput gain over static serving). Loses: much harder
  scheduler (per-step admission, per-sequence KV management, preemption bookkeeping), and
  interference: a new request's *prefill* (compute-heavy, hundreds of ms) lands inside an
  ongoing decode step and spikes everyone's token latency.
- **Chosen: C, with prefill/decode separation to manage the interference.** The scheduler
  admits prefills as a resource class of their own ("chunked/deferred prefill"): a joining
  request's prompt is processed in compute slices interleaved with decode steps, bounded so
  the *ongoing* streams keep their ~20 tokens/s (our token-rate SLO) while TTFT absorbs the
  wait. Trade-off made explicit: we accept scheduler complexity and a slightly worse TTFT
  for *newly admitted* requests in exchange for stable token rates for everyone already
  streaming and maximal throughput. Option B's simplicity buys nothing at this scale: a
  system that can only admit between batches cannot meet "p99 TTFT ≤ 2 s" with 500K
  in-flight.

**Prefill/decode disaggregation as a further split**: prefill wants compute-dense scheduling,
decode wants memory-resident batching — different fleet tuning.

- Disaggregated pools (prefill workers hand KV to decode workers) win utilization per pool
  and independent scaling (our arithmetic says decode : prefill ≈ 3.3 : 1 — one shared pool
  would size to the wrong mix half the time). Loses: KV transfer is heavy (multi-GB across
  nodes), adds a network hop and a failure mode inside the request path.
- We pick disaggregation *selectively*: long prefills (>~2K tokens) go to a dedicated
  prefill pool with transfer; short prefills run fused on the decode pool (transfer cost
  outweighs the gain below a threshold). This is a tunable split, re-derived from
  measured cost profiles per model version.

**Failure modes**: scheduler livelock (admission keeps joining work that starves decode
steps — protect decode with a hard per-step compute budget); a runaway long-generation
request blocking nothing in continuous batching but squatting KV memory (idle-stream
timeout and migration/recompute); hot GPU groups getting all traffic via affinity and
running hot (affinity is a hint, always fall back to recompute elsewhere).

## 2. Streaming down a persistent connection: the first-token latency budget

Every response is a stream over a client connection held for the request's ~10–60 s
lifetime. The connection layer itself is the WhatsApp pattern — ~100M sockets, gateway
tier, reconnect storms, keep-alive tuning for phone radios (see
[WhatsApp deep dives](../03.whatsapp/04-deep-dives.md), section 2; not re-derived here).
What's *new* here is the latency budget on top of it: perceived responsiveness is decided
by **TTFT** and then sustained **token rate**.

**Budget breakdown for p50 TTFT ≤ ~500 ms, client send → first token, unloaded path**
(estimates; queue wait under overload adds before this):

```
gateway + auth                     ~ 20 ms
conversation load + serialize      ~ 30 ms   (store read of last N turns)
admission check                    ~  5 ms
inline moderation classifier       ~ 30 ms
scheduler placement                ~ 10 ms   (affinity table lookup)
prefill of ~1,200-token prompt     ~250-400 ms across one GPU group  ← dominates
first decode step + network        ~ 20 ms
```

- The budget is *prefill-dominated*: this is why deep dive 3's prefix reuse matters — reuse
  turns a 400 ms prefill into a ~50 ms incremental one, the single biggest TTFT lever.
- **Option A — generate then stream everything at once** (non-streaming response): wins:
  trivial protocol, allows full-output moderation *before* delivery. Loses: product is dead
  on arrival — perceived latency jumps from 0.5 s to ~30 s; UX research across all chat
  products (publicly observable) says streaming is table stakes.
- **Option B — stream everything, moderate nothing in-line**: wins: maximum latency budget
  for compute. Loses: policy violations stream to the user with no gate; abuse harvests
  unlimited free text.
- **Chosen: stream + a fast in-line gate on the *input* only** (deep dive 6 covers why
  gating the output in-line is a losing trade). Trade-off: we spend ~30 ms of the TTFT
  budget on input classification, and accept that problematic *outputs* are handled
  post-hoc, in exchange for streaming's interactivity.
- **Token rate is a second SLO, and batching threatens it**: a request admitted into a
  100-way batch at 40 tokens/s each is *fine* (user reads at ~5–10 tokens/s); but if the
  batch grows past the memory-bandwidth knee, everyone's rate collapses together. The
  scheduler therefore enforces a **per-stream minimum token rate**, and admission pauses
  before it would violate it — degradation shows up as lengthening TTFT (queues), which
  backpressures naturally, rather than as a broken reading experience for streams in flight.
- **Continuity across devices/networks**: the stream is server-driven; a mid-stream
  reconnect (phone blip) re-issues the turn or resumes from the last delivered token
  offset — the client keeps a cursor; tokens are idempotent-per-offset. At-least-once
  delivery + client-side dedup/order by offset, exactly the delivery-protocol shape of
  [WhatsApp deep dive 1](../03.whatsapp/04-deep-dives.md).

**Failure modes**: gateway crash mid-stream (client auto-resumes or restarts the turn —
GPU work already spent is written off; generation is *not* checkpointed per-token because
saving incremental KV checkpoints to disk for every stream costs more than the losses);
token-rate collapse under overload (mitigated by the minimum-rate admission guard);
slow-reader hold — a user who stops reading doesn't stop the stream (they might stop it —
the stop button is a first-class cancellation that frees the batch slot *immediately*;
forgetting this strands GPU memory for the full max-token horizon).

## 3. Conversation context: KV-cache reuse vs recompute

Every turn re-sends the conversation. Turn 20 of a chat re-prefills ~19 prior turns worth
of tokens — at our arithmetic (~1.2K input tokens mean, history-dominated) that is wasted
GPU-compute on text the model already processed moments ago. The state to reuse is the
**KV cache**: per-token, per-layer ~4 MB (est., 02's arithmetic) that prefill computed and
decode reads.

- **Option A — recompute everything each turn**: wins: nothing to manage; any worker can
  serve any request; trivially correct under failures. Loses: at ~60–80% of input tokens
  being repeat history (est.), recompute-everything redoes 2.5–5× the prefill work (only
  20–40% of input tokens is novel), and prefill is ~5K of ~22K GPUs (02's arithmetic), so
  ~35–90% added fleet cost, and TTFT pays
  full prefill every turn.
- **Option B — durable KV cache store** (persist KV to disk/object storage between turns):
  wins: reuse across days, any worker can pick it up. Loses: 32 GB per 8K-context
  conversation — ~10M active conversations is ~300 PB at max context, and even at the mean
  ~1.2K-token context (~5 GB each, 4 MB/token) ~50 PB of high-value storage, plus transfer
  time (loading tens of GB eats most of the TTFT budget); the reuse window
  (seconds-to-minutes between turns) doesn't justify durability.
- **Option C — in-memory prefix cache + affinity routing**: keep hot KV resident on GPU
  (paged, LRU) and route follow-up turns to the workers that already hold the prefix.
  Wins: reuse is *free* when it hits (no transfer — the KV is already where it needs to
  be); TTFT on hit ~1/8 of full prefill (est.). Loses: routing is now stateful (affinity
  table, worker failure invalidates the entry → recompute fallback); load can skew to
  cache-holding workers; cache invalidation on conversation delete/edit.
- **Chosen: C, with A as the universal fallback.** The scheduler keeps an
  approximate affinity table (conversation prefix id → worker set, TTL'd, probabilistic —
  not a strict directory service). If affinity misses or the worker died: recompute. No
  user-visible failure exists at a *cache* miss — misses cost money, not correctness.
  Additional lever: shared prefixes (system prompts, common instruction templates) reuse
  across *users* via hash-matched prefixes. Trade-off accepted: we spend scheduler complexity
  and accept fleet skew risk; in exchange the dominant cost (prefill of repeated history)
  drops materially, which our fleet arithmetic says is a prefill-pool-wide saving, and TTFT
  improves *and* costs drop — rare alignment, which is why this is the most valuable
  optimization in the system.

**Memory pressure is the limiter**: with ~4 MB/token (est.), an 80 GB GPU holds 80 GB / 4 MB
≈ ~20K tokens of KV per *tenant-free* GPU — batching 40 streams at 1.5K context each =
40 × 1.5K ≈ 60K tokens of KV ≈ 60K × 4 MB ≈ ~240 GB. This does not fit an 80 GB GPU; hence
**paged KV allocation** (allocate KV in fixed-size pages per
sequence, on demand rather than reserving max-context up front — the published paged-attention
design), quantized KV (fp8 halves it; a quality-throughput trade), and sliding-window or
summarization of very long histories (context beyond the window is *dropped by design* —
a product-visible trade: the model "forgets" beyond the window; alternatives like map-reduce
summarization add a second model pass on the serving path). Even fp8 + a ~4K sliding window
does not get a 40-stream batch under 80 GB at 4 MB/token; the rest of the gap is model-shape:
KV-per-token assumes full multi-head KV, and this model class uses grouped-query attention,
which shrinks KV per token roughly 8× to ~0.5 MB/token (estimate, same tensor arithmetic with
~8 KV heads instead of 64). At that rate the 60K-token batch is ~30 GB and 40–80-way batching
(02's per-GPU decode assumption) fits. The 4 MB/token figure is the conservative no-GQA
upper bound; the memory-pressure machinery above is what the bound forces.

**Failure modes**: affinity hot spots (a viral conversation can't happen — one user — but a
popular shared system prompt can; mitigate: replicate hot prefixes); worker loss deleting
resident KV (recompute fallback, correctness unaffected); cache poisoning is impossible by
construction (KV derived from the *stored* conversation text, never mutated in place —
edit/branch a turn ⇒ new prefix id, new cache lineage).

## 4. Per-tier rate limiting and admission control: metering tokens, not requests

The expensive unit is GPU-seconds ≈ tokens; the request is nearly free. So limits must be
in **tokens** (per minute/day per tier), not requests.

- **Option A — request-count quotas** (like Twitter's post/hour):
  [Twitter's rate limiting](../02.twitter/04-deep-dives.md) meters writes because each write
  fans out O(followers)... here, metering requests is trivially gamed: one request with a
  100K-token prompt (or an unbounded "continue" loop) burns the fleet the same as a hundred
  short ones. Loses on its face.
- **Option B — pure token quota enforced post-hoc** (count after generation): wins: exact
  metering of actual spend. Loses: the damage is already done — an abusive prompt already
  consumed its GPU-seconds before the counter trips; a quota that only bites after the fact
  does not protect capacity, it merely bills.
- **Option C — pre-admission reservation with token accounting up front** (estimate the
  request's spend from input tokens + a generation cap, reserve against the quota, reconcile
  with actuals when the stream completes): wins: capacity protection before GPU work; the
  reconciliation catches underestimates. Loses: estimates are approximate (stop-generation,
  timeouts) and refunds must be handled — accounting complexity on every request.
- **Chosen: C.** Free tier: conservative per-window token budgets, lower priority in every
  queue, smaller default model (cheaper cost profile). Paid tier: larger quotas, priority
  admission, access to premium models. Quota state is a coarse, sharded counter service —
  it must survive ~17K admissions/s (est.) but can be eventually-consistent within a window
  (a user briefly exceeding their floor is tolerable; the counter converges).

**Admission control vs overload** is tiered, in order:

1. **Queue with visibility**: new requests queue; the client shows position/ETA. Queues
   convert overload into TTFT, not errors — and TTFT backpressure (deep dive 2) is a sane
   failure mode.
2. **Shed free-tier**: free requests get a fast "capacity" response *before* GPU work.
3. **Downgrade model**: at pressure, route paid-tier-eligible-for-downgrade traffic to the
   smaller model (a *quality* trade used deliberately as a pressure valve).
4. **Priority preemption**: queued paid work jumps queued free work. Preempting
   *in-flight* generation is the last resort (and only re-queues free-tier streams, which
   may be resumed from KV or restarted) — never preempt a paid stream mid-answer.

- Explicit trade: we accept approximate fairness at tier boundaries (eventual-consistent
  counters, occasional refund weirdness, a free user over their floor for seconds) in
  exchange for O(1) admission checks and no distributed synchronization on the hot path.
  A perfect, strongly-consistent token ledger at 17K admissions/s would put a global
  coordination step in front of every prompt — the cure kills the patient.

**Failure modes**: counter-service outage (fail *open* to a degraded local allowance rather
than fail closed and lock out all users — with async reconciliation catching the drift);
retry storms at limit boundaries (client retry must honor the same idempotency key so a
retried turn does not get metered twice — see the delivery-protocol pattern in
[WhatsApp](../03.whatsapp/04-deep-dives.md)); quota-reset stampede at midnight UTC windows
(spread resets).

## 5. Model rollout & canary serving

Dozens of model versions, continuous training improvements, one product surface. Rollout is
a routing problem lived entirely in the scheduler: registry state maps model+version →
checkpoint distribution and traffic percentage; the scheduler pins each request to a version
at admission.

- **Option A — big-bang deploy**: ship to all GPUs at once. Wins: simple, no mixed-fleet
  routing. Loses: any quality regression or serving-perf regression (a slower version
  silently halves decode throughput) hits 100% of traffic instantly; rollout duration is
  bounded by checkpoint-load time, not evidence.
- **Option B — full shadow mode**: candidate gets a mirror of all traffic (real prompts,
  outputs discarded); promote when evals pass. Wins: zero user risk during evaluation;
  production-distribution signals (latency, token rate) measured at full fidelity. Loses:
  doubles inference cost during evaluation — at our fleet size, doubling the most expensive
  resource for days is not a rounding error, it's a budget line.
- **Option C — percentage canary with layered evals**: 0% (shadow on a small traffic slice)
  → 1% → 5% → 25% → 100%, each gate requiring (a) offline eval suites, (b) online
  metrics — quality via side-by-side user preference sampling, serving metrics via the
  version's own token-rate/TTFT histograms, safety-refusal deltas. Automatic rollback on
  guardrail breach. Wins: bounded blast radius, statistical evidence at each gate, cheap
  (canary cost ∝ canary %). Loses: slow (days), and small-sample quality signals early
  (a 1% slice with 17K req/s still yields ~170 req/s ≈ plenty, to be fair).
- **Chosen: C, with a cheap shadow layer** — a tiny (<0.1%) always-on shadow for hotfix
  validation before the canary ladder starts. Trade-off accepted: rollouts are measured in
  days, and the registry becomes a *critical availability dependency* (schedule can't run
  without a version map) — so registry state is versioned, small enough to cache everywhere,
  and defaults to the last-known-good distribution on registry outage (fail-safe > fail-open:
  wrongly serving the old version is far safer than serving nothing).

**Mixed-fleet serving economics**: different versions have different cost profiles (a newer
checkpoint may use more tokens-per-answer or decode slower). The canary router must evaluate
*cost per quality point*, not just quality — a regression in decode speed is a fleet-size
regression. This is the same discipline as serving ML under SLOs (see
[14. ML production system](../14.ml-production-system/04-deep-dives.md)).

**Failure modes**: checkpoint-hot-swap is disruptive (KV layouts change across versions —
in-flight streams finish on the old version; scheduler version-pins per request, never
mid-stream); canary metrics lag product regressions (complement automated gates with
human side-by-side sampling); a "good" model with subtly worse safety refusals passes
quality gates (safety-refusal-rate is a first-class canary metric, not an afterthought).

## 6. Moderation: in-line vs async for content that streams

The structural problem: moderation wants to *inspect the whole output and block it*; the
product streams tokens as they are generated, so by the time a classifier sees the full
response, most of it is already on the user's screen. You cannot recall a streamed response.

- **Option A — moderate everything in-line, output too** (buffer full output, classify,
  release or refuse): wins: no policy-violating text ever delivered; the only real
  guarantee. Loses: destroys streaming (TTFT → full-generation latency, ~30 s mean) and
  doubles the cost of every request (full decode + full classification before serving).
- **Option B — classify input in-line only, audit output fully async**: wins: streaming
  preserved; budget tiny (~30 ms input classifier). Loses: violating *outputs* are delivered;
  the async trail is reactive (suspension, tuning) — the harm window already happened.
- **Option C — streaming partial moderation**: classify the *input* in-line; during decode,
  run a lightweight streaming classifier over token windows and *halt mid-stream* (replace
  the tail with a refusal notice) when confidence crosses threshold. Wins: streaming kept,
  worst content truncated before it fully streams. Loses: the halt is user-visible and
  mid-sentence; classifier errors make the product feel censored at random; a halting
  stream has already burned the decode spend up to that point.
- **Chosen: B + C** — in-line input gate (hard, pre-GPU: a refused prompt costs ~zero
  GPU-seconds, which also makes it an abuse-cost control), streaming halt only for the
  highest-severity categories (the rare, unambiguous ones), full async review of samples
  and flagged outputs feeding policy, model tuning, and the canary safety metrics (deep
  dive 5). Trade-off made explicit: we accept that most violating *outputs* are handled
  after delivery — because the alternative that fully prevents them (Option A) removes the
  product's defining interaction. The in-line budget is spent where it prevents the most
  GPU waste and the clearest harm: on the input.
- **Why input gating is cheap insurance here and not in Twitter**: a blocked prompt costs
  ~5 ms of CPU; an *answered* bad prompt costs ~30 GPU-seconds. Moderation at the admission
  gate is simultaneously the safety control and the cost control — the same "apply limits
  before the expensive fanout" logic as [Twitter's rate limiting](../02.twitter/04-deep-dives.md),
  where the resource being protected differs (there: O(followers) writes; here: O(600 tokens)
  of decode).

**Failure modes**: classifier false positives blocking legitimate prompts invisibly (every
inline refusal is logged and sampled; an appeals path exists — silent rejection is the one
unrecoverable failure); streaming classifier thrash (halting too often → tune confidence
thresholds per severity, not globally); async review backlog (flagged stream volume scales
with traffic — review is capacity-planned like any other tier of the system, and sampling,
not exhaustive review, is the design: exhaustive review of 0.9T tokens/day is impossible
by arithmetic, so the design must be honest that output *enforcement* is probabilistic,
with hard in-line gates only for input and the top-severity tail).
