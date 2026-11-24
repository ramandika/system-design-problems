# GPU & inference serving across the case catalog

Where model-serving physics — batching, KV-cache management, prefill/decode separation, dedup,
streaming synthesis — appears in the 50 study cases, what role it plays, and how deep each case
goes. Companion to [ML-PLATFORM.md](ML-PLATFORM.md) (registry, feature store, *fleet* economics —
this doc covers what happens on one request), [KAFKA.md](KAFKA.md) (the job-queue plane behind
generation workloads), and [DEPLOYMENT-SAFETY.md](DEPLOYMENT-SAFETY.md) (canary routing).

Headline: **serving a model is scheduling a scarce, stateful accelerator under a latency budget —
and every case that does it reaches the same separation: compute-bound work in pools, memory-bound
work in pools, and identity of the request as a dedup key.** No case studies the details here
except one; the others import fragments.

- Cases with designed inference-serving components: **4 / 50** (08 = the case itself; 16, 17 full
  components; 15 brief)
- 18's generation-orchestration (chunked latent DAG) is covered from the media angle in
  [MEDIA-PIPELINES.md](MEDIA-PIPELINES.md); its scheduling/pool decisions are in
  [ML-PLATFORM.md](ML-PLATFORM.md).

## Summary table

| Case | Serving shape | Memory management | Dedup / identity | Streaming behavior |
|---|---|---|---|---|
| [08.chatgpt](08.chatgpt/README.md) | The case itself: prefill pool + decode pool + scheduler | Paged KV cache, continuous (iteration-level) batching, KV transfer | Request routing, free-tier preemption | Token streaming with scheduler-controlled admission |
| [16.dalle](16.dalle/README.md) | Async job model: at-least-once queue, sweeper re-enqueues | — | **Exact-request cache** on `hash(prompt ‖ params ‖ seed ‖ model version)`; semantic cache rejected | Delivery of finished artifact (webhook/poll) |
| [17.elevenlabs](17.elevenlabs/README.md) | Streaming synthesis controller: boundary chunking + coalescing buffer | Two-tier voice artifacts (embedding vs checkpoint), per-GPU LRU voice cache | Voice identity via registry + consent check | Sentence/clause-boundary chunks with sequence numbers; client jitter buffer |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Model server: N versions in-process, shadow route | — | — | Synchronous scoring (no streaming) |

## Per-case notes

### The case itself

**[08.chatgpt](08.chatgpt/README.md)** — [03-architecture.md](08.chatgpt/03-architecture.md) splits
inference into a **prefill pool** (compute-bound: prompt → KV cache) and a **decode pool**
(memory-bound, paged KV cache, continuous batching). The **inference scheduler** is the core
component: places each admitted request on a prefill worker (with prefix affinity), transfers KV to
a decode worker (or keeps phases co-located), **admits new requests into running decode batches
continuously** rather than waiting for batch completion, and preempts free-tier work under
pressure. [04-deep-dives.md](08.chatgpt/04-deep-dives.md) works the memory side: KV-cache size is the binding
constraint, attacked by GQA-style KV-factor reduction (~8×), fp8 KV quantization, and
sliding-window/summarized context; prefill and decode are **disaggregated with chunked prefill**
because they bottleneck on different resources. Registry/canary routing of model versions borders
[DEPLOYMENT-SAFETY.md](DEPLOYMENT-SAFETY.md) and [ML-PLATFORM.md](ML-PLATFORM.md).

### Generation via identity dedup

**[16.dalle](16.dalle/README.md)** — serving starts before the GPU: the worker checks an
**exact-request cache** keyed by `hash(prompt ‖ params ‖ seed ‖ model version)` — the registry's
version pinning is what makes that key well-defined and rollback safe. The **semantic cache** is
evaluated and explicitly rejected: embedding-similar prompts reuse results across users, which
wins hit rate but mixes provenance/isolation — "unacceptable at a content boundary". The job plane
itself is the durable-queue pattern ([KAFKA.md](KAFKA.md) outbox family): record-then-enqueue,
sweeper re-enqueues on failure, at-least-once execution with attempt caps.

### Streaming synthesis as a serving problem

**[17.elevenlabs](17.elevenlabs/README.md)** — text streams in, audio streams out; the designed
component is the **streaming synthesis controller**: chunk at sentence/clause **boundaries**
detected per-language, with a small coalescing buffer so chunks are worth rendering, and sequence
numbers to keep client-side ordering. Boundary choices are the failure surface (a "Dr." can
mis-split; unbounded input needs a force-flush rule), and the split adds a small initial latency —
stated as the trade-off against chunking on fixed character counts (audible breaks). The client
side holds an adaptive **jitter buffer** (start near-empty, grow within a bound) and the server
keeps *no playback buffer* — backpressure is expressed by stream errors, not buffering. Voice
identity and artifact tiers live in the registry ([ML-PLATFORM.md](ML-PLATFORM.md)).

### Brief

**[15.ml-ranking-system](15.ml-ranking-system/README.md)** — the model server scores synchronous
requests with N versions in-process and a shadow route; no batching/KV machinery, because a ranker
is small and stateless — the boundary of this doc's applicability: the serving physics above
appears when the model is large *and stateful in memory* (KV cache, streamed output), not when it
is a pure function per request.

## The portable pattern

1. **Split by bottleneck, not by model.** Compute-bound prefill and memory-bound decode are
   separate pools because one saturates FLOPs and the other saturates HBM; co-locate only until
   interference is measured (08).
2. **Admit into running batches.** Batching is continuous (iteration-level), not request-level;
   a new request joins the next forward pass, so queueing latency is one iteration (08).
3. **Attack the KV cache with structure, then precision.** GQA / sliding window first, fp8 /
   quantized KV second — the two levers differ in risk and apply at different layers (08).
4. **Dedup by request identity before spending GPU.** An exact-request cache keyed by a
   registry-pinned version tuple is cheap and safe; semantic similarity is the tempting, weaker
   key — the cases reject it wherever provenance matters (16).
5. **Stream by linguistic structure, not byte counts.** Boundaries (sentences/clauses) are natural
   resumption and quality anchors; fixed-size chunks are audible. Client jitter buffer absorbs
   variance; the server buffers nothing (17).
6. **Preemption and quota are scheduler responsibilities.** Free-tier preemption under pressure
   (08), paid-tier budget admission priced in GPU-minutes
   ([ML-PLATFORM.md](ML-PLATFORM.md), case 18 economics).
