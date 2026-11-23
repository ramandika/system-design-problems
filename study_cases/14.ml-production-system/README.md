# Case 14 — Production ML System (MLOps platform)

**The system**: the machine-learning platform *behind* a product — training pipelines, a
feature store, a model registry, CI/CD for models, online serving under latency SLOs, and
drift monitoring. The product (assume a ~100M-DAU consumer app whose feed is ranked by
learned models — see [15.ml-ranking-system](../15.ml-ranking-system/README.md) for one such
surface) is the customer; this platform is what lets dozens of teams ship models weekly
instead of quarterly.

**Why it's hard**:

- Machine learning systems fail **silently and statistically**: no exception is thrown when
  a feature distribution shifts or training and serving compute the same feature differently —
  the model just gets worse. Reliability here is *distributional*, not binary.
- The core invariant — "the model is trained on the same feature values it sees at serving
  time" — crosses a boundary Spark controls (batch, minutes-to-hours) and a KV store serves
  (online, milliseconds). Every skew bug in a ten-year career of ML is a violation of it.
- Models are **artifacts with dependencies** (weights + feature definitions + pre/post
  processing + eval harness), so "deploy a model" is a dependency problem, per
  [ChatGPT's inference lessons](../08.chatgpt/04-deep-dives.md) about serving under SLOs
  applied to a versioned artifact with a mutable feature catalog around it.
- Feedback loops: the model's own predictions generate tomorrow's training data (logged
  features), so the data pipeline, the serving stack and the monitoring stack read the
  *same firehose*.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: point-in-time correctness (the ML version of consistency),
compute-on-write vs compute-on-read (same split as [Twitter's timeline](../02.twitter/04-deep-dives.md),
re-applied to features), staged promotion with shadow traffic, distributional monitoring
(PSI/KS), petabyte-scale batch pipelines with late data.

**Try it first**: before reading 04, answer: (1) the offline feature says `p_click=0.13`
but at serving time the code computed `0.17` — where does that come from and how do you
make it impossible? (2) a retrained model is better on *training-week* data; what gate stops
you shipping it on noise? (3) the model quietly degrades for 3 days — what did you monitor,
and what triggers retraining?

**Primary sources / honest labels**: no real company's MLOps platform is documented end to
end, so this case is a **design built from public building blocks**: Feast/Tecton's published
feature-store design, MLflow/Kubeflow registry semantics, Uber's published Michelangelo and
feature-store posts ("~point-in-time joins", estimates here), and standard fairness/drift
literature on PSI/KS. **All scale numbers below are rough estimates constructed for
internal consistency, not measurements from any one system.**
