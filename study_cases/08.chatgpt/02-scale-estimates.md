# 02 — Scale estimates

All numbers are **estimates**, chosen to be internally consistent and consistent with
publicly reported magnitudes. Publicly reported anchors (self-reported, 2025-era): roughly
~800M weekly ChatGPT users and on the order of ~1B+ prompts/day. Everything derived below
shows its arithmetic; redo it if an assumption changes.

## Assumptions

- 500M DAU (estimate; ~60% of the reported weekly figure acting daily — generous but keeps
  arithmetic round).
- **1.5B prompts/day** across DAU → 3 prompts/day per daily user (plausible: casual users
  1–2, power users dozens).
- Per prompt (estimate): **~1,200 input tokens** (user message + serialized conversation
  history; history dominates after turn 2), **~600 output tokens**.
- Mean request residency: decode 600 tokens at ~20 tokens/s ≈ 30 s dominates; add prefill
  ~1–2 s of residency (estimate — includes chunked-prefill/queue wait, not pure compute;
  the pure prefill compute for the ~1,200-token prompt is ~250–400 ms, see 04 DD2's latency
  budget) ≈ **~30 s total** (long tail for long outputs).

## Request rate

- Average: 1.5B / 86,400 s ≈ **17.4K prompts/s**.
- Peak factor ~2.5–3× (evening concentration, product launches): **~45K prompts/s**.

## Token throughput (the real load unit)

- Prefill tokens/day: 1.5B × 1,200 ≈ **1.8T tokens/day**.
- Decode tokens/day: 1.5B × 600 ≈ **0.9T tokens/day**.
- Steady rates: prefill 1.8T / 86,400 ≈ **21M tokens/s**; decode 0.9T / 86,400 ≈
  **10M tokens/s**.

## GPU pool sizing (the distinctive estimate)

Rough per-GPU capabilities for a flagship ~100B+-parameter model class (estimates — real
numbers depend heavily on model, quantization, and batch shape):

- Prefill is compute-bound: ~10K prefill tokens/s per GPU in large batches (estimate).
- Decode is memory-bandwidth-bound: ~1.5K output tokens/s per GPU aggregate when batching
  ~40–80 sequences at once (estimate).

Fleet at target ~40% utilization (headroom for peaks and scheduling fragmentation):

- Prefill: 21M tokens/s ÷ 10K ≈ 2.1K GPUs at 100% → /0.4 ≈ **~5K GPUs**.
- Decode: 10M tokens/s ÷ 1.5K ≈ 7K GPUs at 100% → /0.4 ≈ **~17K GPUs**.
- **Total ~20–25K high-end GPUs**, plus reserve for peak 2.5× absorbed partly by queueing.
  Decode dominates ~3.3:1 — the cost center is generating tokens, not reading the prompt.

## Concurrency (what the scheduler actually sees)

- In-flight requests = arrival rate × residency: 17.4K/s × ~30 s ≈ **~500K concurrent
  active generations** at mean, ~1.3M at peak (45K/s × ~30 s).
- These are not 500K sockets — they are 500K *reservations of GPU memory and throughput*
  riding on top of a much smaller number of gateway connections (see below).

## Streaming bandwidth (surprisingly trivial)

- Per active stream: ~20 tokens/s × ~4 B/token ≈ 80 B/s.
- 500K streams ≈ **~40 MB/s egress**. Even at peak (~1.3M streams): ~105 MB/s. A single beefy edge
  cluster handles this. Compare YouTube (~PB/hour). **Bandwidth and storage are not this
  system's constraint; GPU-seconds are.** The load on the gateway tier is connection
  bookkeeping (~100M clients with SSE/WebSockets), the same physics as
  [WhatsApp's connection layer](../03.whatsapp/04-deep-dives.md) — ~100M connections at
  ~1M sockets/server ≈ ~100–200 gateway servers, cheap.

## KV-cache memory (why GPU memory is the scarce resource)

KV cache per token for a ~100B-class transformer (estimate: 2 tensors × hidden ~8K × ~120
layers × 2 bytes fp16) ≈ **~4 MB/token**. This is the no-GQA upper bound; grouped-query
attention (standard in this model class) cuts KV per token roughly 8× to ~0.5 MB/token
(estimate), which is what makes 40–80-sequence batching fit in 80 GB (deep dive 3).

- A single request at the 8K-token context limit: 8,000 × 4 MB ≈ **32 GB of KV cache** —
  a third of an 80 GB GPU just for *state*, per request.
- So even with batching, a GPU holds a handful of full contexts: **paged KV allocation,
  prefix reuse, and aggressive routing-by-prefix are requirements, not optimizations**
  (deep dive 3).

## Storage (surprisingly trivial too)

- 1.5B turns/day × ~3 KB/turn (text + envelope + index entry) ≈ **~4.5 TB/day raw**,
  ~15 TB/day with replication and search indexes. A modest object/key-value tier. The
  *conversation store* is a footnote; almost the whole capital budget is GPUs.
- KV cache is **not** stored: it is recomputed or reused within a request's lifetime and
  discarded (the only durable persistence is the text).
