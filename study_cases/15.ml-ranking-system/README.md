# Case 15 — Ranking System (RecSys)

**The system**: the ranking layer behind a large feed product (YouTube-style home feed /
Twitter-style home timeline / search ranking). Given a user and a context, decide which
~50 items to show next, out of a corpus of ~500M items — in under 100 ms, fresh from what
happened 10 minutes ago. A three-stage funnel: candidate generation (retrieval) →
heavyweight ranker → re-rank + business filters, backed by online feature serving and a
training loop.

**Why it's hard**:

- The funnel is a latency budget problem: you cannot score 500M items with a good model,
  so retrieval must find the right ~2,000 in ~10 ms — and every shortcut you take to go
  fast is a way to systematically miss the best item (see deep dive 1 and 2).
- The ML and the infra fight each other: the model wants huge feature sets and daily
  retraining; the serving path wants 50 ms and feature values that are seconds old.
- The feedback loop poisons itself: the model ranks what users see, users watch what the
  model ranked, so training data is position- and selection-biased (see deep dives 4–6).
- Freshness vs stability: a video published 5 minutes ago must be rankable, but a serving
  index and embedding tables cannot be rebuilt every 5 minutes (deep dive 3).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: multi-stage funnels (millions → thousands → dozens) with an
explicit latency budget per stage, ANN retrieval (two-tower vs HNSW), online/offline
feature split, negative sampling and label delay, exploration/exploitation in production,
interleaving for unbiased A/B.

**Try it first**: before reading 03, answer: (1) you have a 100 ms budget and a 500M-item
corpus — where exactly do the milliseconds go? (2) a video went viral 5 minutes ago and is
not in your ANN index — how does the user see it? (3) your offline metric improved but the
live A/B did not — list three biases that explain it before you read deep dives 4–6.
