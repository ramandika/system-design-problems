# ML platform plumbing across the case catalog

Where the shared ML infrastructure — feature stores, model registries, GPU fleet management,
drift monitoring — appears in the 50 study cases, what role it plays, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md) (precompute-at-write union, Tier 2, 26% — the feature-availability
side of that union is this doc's territory) and to [VECTOR-SEARCH.md](VECTOR-SEARCH.md) (the
retrieval tier, which this doc deliberately does not cover).

Headline: **the ML cases share platform plumbing, not models.** Six of the seven cases that
carry ML reach the same three pillars independently — an immutable model registry with a
promotion workflow, a shared two-tier feature store, and a GPU fleet where *the queue is the
product* — and their failure stories are all about train/serve skew, registries, and fleet
preemption, almost never about model quality itself.

- Cases with designed ML-platform components: **5 / 50** (14, 15, 16, 17, 18)
- Invoke the platform as an externalized dependency or pattern reference: **3** (26, 29, 08 briefly)

## Summary table

| Case | Feature store | Model registry | GPU fleet | Drift/monitoring |
|---|---|---|---|---|
| [14.ml-production-system](14.ml-production-system/README.md) | Online KV (<2 ms p99) + offline point-in-time (parquet), `computed_at` stamps | Immutable versions, eval → shadow → partial → full | (background) | PSI/KS on every feature + score distributions; retrain triggers |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Shared two-tier store, same definitions train & serve | Weights + embeddings + ANN index ship as **one version** | — (serving-side case) | Shadow-serve, team-draft interleaving before A/B |
| [16.dalle](16.dalle/README.md) | — | Registry pins (weights, steps, seed policy) → cache keys + rollback | Reserved + spot pools, batch packing | Utilization/cost telemetry |
| [17.elevenlabs](17.elevenlabs/README.md) | — | Voice registry: consent state on every synthesis | Interactive pool (sticky, preemption-proof) vs batch preemptible queue | Metering + abuse telemetry |
| [18.sora](18.sora/README.md) | — | Per-job model version pinning as load-bearing invariant | Draft/main/post pools; checkpointing routine under preemption | Utilization drift exposes wasted money |
| [26.google-maps](26.google-maps/README.md) | Versioned features for ETA correction | Shadow rollout via the 14-pattern | — | Degraded fallback |
| [29.uber-eats](29.uber-eats/README.md) | — | Versioned model per merchant×daypart | — | Shadow rollout, degraded fallback |
| [08.chatgpt](08.chatgpt/README.md) | — | Canary-aware model router (borders [DEPLOYMENT-SAFETY.md](DEPLOYMENT-SAFETY.md)) | Fleet physics — see [GPU-INFERENCE-SERVING.md](GPU-INFERENCE-SERVING.md) | — |

## Per-case notes

### The platform case

**[14.ml-production-system](14.ml-production-system/README.md)** — the case every other ML case
cross-references. [03-architecture.md](14.ml-production-system/03-architecture.md) defines the full
component set: online feature store (KV cluster, packed rows, p99 < 2 ms), offline store
(point-in-time-correct snapshots in parquet, keyed event_day/model/feature-set-hash), model
registry (immutable versions; promotion workflow eval → shadow → partial → full; artifact-store
pointer), prediction log (what was actually served, one place, one schema), monitoring (PSI/KS
per feature and score distribution on the prediction log). Deep dives: train/serve skew is killed
by *shared definitions* and point-in-time correctness, not by more training; the freshness dive
weighs lambda vs kappa and lands on "lambda-ish with one store on top". Named failure mode
elsewhere in the repo: Kafka retention shorter than the training window
([KAFKA.md](KAFKA.md), case 14 section).

### Registry as the single source of "what's shippable"

- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** — retrain rots more than weights:
  the ANN index, embedding tables and weights ship **together as one registry version**, because
  the funnel happily ranks garbage when embeddings drift after a partial deploy. Rollout is
  shadow-serve against real traffic, then blue/green shard rotation; evaluation uses team-draft
  interleaving (high power, small traffic) before an A/B confirm.
- **[16.dalle](16.dalle/README.md)** — the registry determines cache semantics: a version is
  (weights, steps, seed policy), which is what makes the **exact-request dedup cache** hashable
  (`prompt ‖ params ‖ seed ‖ model version`) and rollback a pointer move. The semantic cache
  (reuse for *similar* prompts) is evaluated and rejected — wins hit rate, loses provenance and
  isolation, an unacceptable mix at a content boundary.
- **[17.elevenlabs](17.elevenlabs/README.md)** — a *voice* registry: owner, model reference,
  sharing scope and **consent state**, with authorization checked on every synthesis call. It
  also carries two artifact tiers — instant clone = speaker embedding (KBs, ready immediately);
  professional clone = fine-tuned checkpoint (~500 MB, async training job flips registry state to
  `ready`) — and because registry lookup is on the interactive hot path, the two tiers have
  different consistency stories.
- **[18.sora](18.sora/README.md)** — per-job model version pinning is the load-bearing invariant:
  a 3-minute generation job that survives preemption only replays if the pinned version is
  still resolvable; mutable "latest" checkpoints are the failure story.

### The GPU fleet: the queue is the product

The same economics recur three times and each case states them differently. Reserved capacity is
sized to (or below) average load; preemption is assumed, not feared — batch/interactive classes
split into pools with different preemption rules, sometimes explicitly asymmetric (17: the
*synthesis* pool is sticky and preemption-proof because users hear mid-sentence cuts; the batch
pool is preemptible at segment boundaries).

- **[16.dalle](16.dalle/README.md)** — fleet arithmetic: ~16K GPUs at average load, a 5× peak
  *without queueing* would want ~80K — so the second segment goes to a **spot pool at 60–70% off**
  with batch packing, and the queue is the honest response to scarcity.
- **[17.elevenlabs](17.elevenlabs/README.md)** — three schedulers: streaming synthesis (sticky),
  fast-batch (preemptible queue), document batch (durable queue with segment manifests and
  checkpoints). Preemption asymmetry stated as a product requirement, not an infra one.
- **[18.sora](18.sora/README.md)** — every submission is *priced* in expected GPU-minutes; user
  credits are the budget; queue ETA is the product response. **Draft passes on a cheap draft
  pool** reject garbage prompts before main-pool spend; preemptible fleets make checkpointing
  routine ("interruptions are the happy path, not the failure mode").

### Externalized / by-reference

- **[26.google-maps](26.google-maps/README.md)** and **[29.uber-eats](29.uber-eats/README.md)** —
  a serving model (ETA correction; prep-time per merchant×daypart) that explicitly delegates to
  the 14-pattern: versioned features, shadow rollout, degraded fallback to heuristic. No registry
  of their own is designed — the pattern is imported rather than re-derived.
- **[08.chatgpt](08.chatgpt/README.md)** — the canary-aware model router and registry semantics
  border deployment territory ([DEPLOYMENT-SAFETY.md](DEPLOYMENT-SAFETY.md)); its serving physics
  belongs to [GPU-INFERENCE-SERVING.md](GPU-INFERENCE-SERVING.md).

## The portable pattern

1. **Immutable versions, pointer-move promotion.** The registry is the answer to four different
   questions at once: rollback, cache keys, reproducibility, and what ships *together* with the
   model — embeddings, ANN index, consent state, seed policy (14, 15, 16, 17, 18).
2. **The registry is only as strong as its invariants.** Anything served alongside weights must
   be versioned with them, or retrain-driven rot is silent (15, 17, 18).
3. **Train/serve skew is a data-contract problem.** Point-in-time correctness + shared feature
   definitions beat ad-hoc fixes; drift *detection* (PSI/KS against the prediction log) is what
   makes retraining a trigger rather than a hope (14).
4. **Fleet sizing is queue design.** Reserved covers a baseline; spot/preemptible covers peaks
   at 60–70% off; the queue is the honest product response to scarcity, and pricing admission in
   GPU-minutes is the cost-control lever (16, 17, 18).
5. **Preemption policy follows the user experience.** Sticky pools where artifacts are streamed
   (synthesis), preemptible-at-job-boundary pools where work is chunked (17, 18).
6. **When the platform is a side concern, import the pattern, don't redesign it.** 26 and 29
   reference the 14-pattern explicitly instead of deriving their own — consistent with the
   repo's cross-reference conventions.
