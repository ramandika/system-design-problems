# 01 — Requirements

## Functional

- **Conversation turn**: user submits a message in a conversation; system appends it to
  history, generates a model response, and (usually) writes both to the conversation.
- **Streaming output**: tokens arrive incrementally (first token target well under a
  second of perceived action, then a steady token rate); the response renders progressively.
  The user can stop generation mid-stream.
- **Conversation history**: create, list, rename, delete conversations; read past
  conversations and continue them; history is the model's context and the user's archive.
- **Multi-turn context**: follow-ups must account for the whole prior conversation
  (bounded by a max context window for the model in use).
- **Tiers**: free tier (lower quotas, queue last, access to a smaller/cheaper default
  model), paid tier (higher quotas, priority admission, access to premium models). One
  account may run several models per conversation.
- **Safety**: refuse or redirect clearly harmful generations; keep a review trail for
  policy enforcement and abuse.
- **Model rollout**: multiple model versions live at once; a fraction of traffic serves
  candidate versions; users can sometimes select a model explicitly.

## Non-functional

- **Latency** (targets are estimates, set by product feel):
  - TTFT (time to first token, measured client send → first token; queue wait under
    overload adds on top): p50 ≤ ~500 ms, p99 ≤ ~2 s.
  - Token rate after first token: ≥ ~15–30 tokens/s sustained (fast enough to read along).
  - End-to-end for a 600-token answer: ~30 s mean but *interactive-feeling* throughout.
- **Availability**: generation path degrades before it fails. Under overload: queue, shed
  free-tier traffic, extend quotas — but a paid user mid-conversation should get an answer.
  Conversation *history* reads/writes must stay available even when GPUs are saturated
  (history is cheap storage; don't couple it to GPU availability).
- **Durability**: a completed response, once saved, is never lost; an in-flight response
  may be lost on crash (user retries) — MUST-NOT-LOSE applies after completion, not during
  generation (same discipline as [WhatsApp's presence vs messages split](../03.whatsapp/04-deep-dives.md)).
- **Consistency**: a user must see their own conversation exactly (read-your-writes,
  single-writer per turn). Cross-device: a turn started on one device continues on another
  (stream must be reconnectable or the turn resumable).
- **Cost**: GPU-seconds per prompt is the primary economic unit; every design decision
  (batching, cache reuse, routing) is judged partly in cost-per-token.

## Scope decisions

**In scope**: serving plane (gateway → scheduler → GPU pools), conversation store, KV-cache
management, tiered admission control, streaming protocol, model rollout plumbing, safety
checks (in-line + async), observability of token throughput and latency percentiles.

**Explicitly OUT of scope**:

- Model training, fine-tuning, eval jobs, and the data pipeline feeding them — training
  clusters are a different workload; the serving system only consumes checkpoint artifacts.
- Retrieval/agent/tool-calling features (search, code execution): modeled as an extra
  prefill step and a follow-up request; changes arithmetic, not architecture.
- Voice, image, and file inputs/outputs: text-only case; see [ElevenLabs](../17.elevenlabs/04-deep-dives.md)
  for the streaming non-text media pattern.
- Enterprise admin, SSO, billing internals: quotas are taken as input from a subscription
  system.
- The model internals themselves: we treat "a model version" as an opaque artifact with
  measurable prefill/decode cost profiles.
- CDN/edge caching of responses: by design there is no re-serving cached answers (each
  answer is generated); only static UI assets use a CDN.
