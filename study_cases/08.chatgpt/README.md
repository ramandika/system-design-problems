# Case 08 — ChatGPT

**The system**: LLM chat serving for hundreds of millions of users. Every request is a
conversation turn: the user's message plus the conversation history is sent to a GPU pool,
the model generates tokens, and those tokens stream back over a live connection, one by one,
as they are generated. Conversations persist, quotas are per-tier (free vs paid), and model
versions roll out continuously behind the same endpoint.

**Why it's hard**:

- It is a **compute-bound** system, unlike every other case here. Twitter and YouTube are
  storage/bandwidth-bound; WhatsApp is a network-programming problem. Here storage and
  bandwidth are trivial (~TB/day of chat text, ~MB/s of streaming egress) and every request
  occupies scarce, expensive GPU hardware for *tens of seconds*.
- **Scheduling is the system.** The central artifact is a GPU scheduler that batches many
  concurrent generations onto the same accelerator without letting one user's long prefill
  stall another user's next token. The whole product quality (latency, cost per prompt) is
  decided in that scheduler.
- State lives in two places: on disk (conversation history — durable, cheap) and in GPU
  memory (KV cache — ephemeral, precious, up to ~30 GB per request). Reconciling the two is
  core design, not an optimization detail.
- Responses stream out incrementally and **cannot be unsaid**: unlike a post or a video,
  a bad output already streamed is delivered. Moderation and canary rollouts therefore
  have a different shape than elsewhere.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: inference as the scarce resource (queueing, admission control,
batching), latency budgeting for a streaming response (TTFT vs tokens/s), cache reuse of
expensive derived state (KV cache vs conversation store), per-tier rate limiting where the
metered unit is tokens not requests, canary rollout of models, and in-line vs async
moderation for content that streams.

**Try it first**: before reading 03, answer: (1) why can't you just make each request a
separate GPU job — what does batching actually save?, (2) if the user asks a follow-up
question, why recompute the whole history at all — what could you reuse?, (3) a free user
hits their quota during a traffic spike — what exactly do you refuse, and how does the
refusal avoid wasting GPU cycles already spent?

**Primary sources** (publicly reported figures only; most serving internals are not public,
so the design is reconstructed from the published inference-systems literature —
vLLM/paged-attention papers, OpenAI/Sam Altman public statements on capacity and demand):
ChatGPT usage publicly reported at roughly ~800M weekly users and on the order of ~1B+
prompts/day (2025, self-reported figures); the GPT-3 paper gives the model-shape arithmetic
behind KV-cache sizing. All scale numbers below are *estimates* built to be internally
consistent, not internal data.
