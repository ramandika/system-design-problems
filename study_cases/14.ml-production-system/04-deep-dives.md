# 04 — Deep dives

## 1. Training/serving feature skew (THE dive)

Skew = the model was trained on feature values computed one way, and at serving time sees
values computed a different (or stale, or double-computed) way. It is the dominant silent
killer of ML systems: metrics look fine in eval, quality bleeds in production, nothing
throws. Skew has three root causes:

- **double computation**: the same feature implemented twice (Spark for training, Java for
  serving) — floating point, timezone, join-order, and filter-order differences leak in.
- **vintage mismatch**: training features are computed "as of" labeling time; serving reads
  current values. Training on `count_5min_clicks` captured *after* the label exists is
  label leakage that can't happen at serving — the model learns a signal it never sees.
- **missing-value drift**: a feature exists in batch but not online (cold user, eviction);
  the two sides impute differently.

**Option A — two implementations, careful review** (what most teams start with): Spark
train code + serving code, kept "in sync" by convention.
- wins: each implementation is fast and idiomatic for its runtime.
- loses: *nothing* enforces equality; every edit is a new skew chance. This is a
  review-process guarantee, and review is not a consistency model.

**Option B — one transformation, compiled for both sides**: feature definitions are
declared (source, window, TTL, impute) in one place; the platform generates/executes them
both in Spark and in the online runtime.
- wins: double computation becomes structurally impossible.
- loses: the portable-expression layer constrains transformations (custom window logic,
  UDFs are painful); and runtime equality still must be pinned down (e.g. float behavior,
  null semantics) — option B narrows the attack surface enormously but doesn't erase
  definition discipline.

**Option C — train strictly on what was served** (log-and-replay): the prediction log
records the exact feature values each request actually saw; training joins labels back to
logged features. Recompute offline only for features the log lacks (too new, too
high-cardinality).
- wins: skew between training set and serving becomes *unrepresentable by construction* —
  you trained on serving's output. Serving-time imputations and cold-start nulls are exactly
  what the model learned on. Vintage leakage is also killed, because log time is
  pre-label-attribution by construction.
- loses: offline data quality (bad fetch that day = trained on bad data unless the log
  stores enough raw inputs to recompute), and you cannot train on features that were not
  served — new features need shadow logging (serve-and-log without using the value) for a
  window before they can train.

**Chosen: C as the backbone, B underneath, A banned.** Log-and-replay guarantees the
invariant where it's cheapest (at the one join that produces training data); the shared
declaration layer keeps the offline/online *stores* consistent for backfills and for
features not yet logged; and the platform's regression harness diffs logged vs
recomputed values nightly to catch declarations that have drifted — an alarm, not a
guarantee.

- **Trade-off made explicit**: Option C wins training/serving fidelity and vintage safety,
  loses the freedom to train on never-served features and adds a "shadow logging window"
  step for any new feature; we pick C because ML failures here are silent and costly while
  its losses are visible and schedulable.
- **Failure modes**: log schema drift (a feature renamed upstream but not in the log →
  silent join drop → training set silently loses rows: enforce log schema versioning +
  drop-rate alarms); Kafka retention shorter than the training window (compaction lag eats
  data day; monitor log-coverage per day-part); logged *scores* without logged *features*
  (can debug what was predicted but not why — log both or the debugging power is fake).

## 2. Feature store freshness: batch + streaming under one roof

Features come from two clocks: batch tables land daily (warehouse); streaming aggregates
(`clicks_last_30min`) update continuously. The store must serve both without the two
*disagreeing*.

**Option A — pure batch (materialize everything daily)**: simple, exactly the warehouse's
numbers.
- wins: one processing model, correctness inherits from the warehouse, cheap.
- loses: streaming-relevant features are stale by up to a day — fatal for a ranking model;
  late-arriving events also leave yesterday's snapshot partially wrong with no self-heal.

**Option B — pure streaming (kappa)**: every feature is an unbounded stream aggregate.
- wins: one freshness story, no batch/stream split.
- loses: unbounded reprocessing over months of history is expensive and drift-prone; joins
  against the warehouse (big dimension tables, one-off backfills) are awkward; streaming
  aggregate state mutates with late data, so "the same feature value at the same point in
  time" is hard to reproduce — the exact property training data needs.

**Option C — lambda-ish split with one store on top**: warehouse = source of truth for
batch-computed features (rebuilt daily); streaming = incremental aggregates with a TTL
(hours), and at TTL expiry each streaming feature *hands over* to the batch value for that
bucket; the store records `computed_at` per value so "fresh" is metadata, not vibes.
- wins: each source does what it's good at (bounded windows streaming, expensive long
  windows batch), and the handover is audit-able.
- loses: the handover seam is where batch-vs-stream disagreement shows up (stream says 42
  clicks, batch computed 39 for the same window — late data, dedup differences); needs a
  reconciliation pass so that the *training-data* view is always the batch-corrected one.

**Chosen: C.** The reconciliation rule is one sentence: **serving optimizes freshness,
training optimizes correctness** — training-set builds may *only* use batch-final values
(the day-2-complete snapshot), serving may use either but stamps which it used. This is
deliberate: serving being 5 min stale is a quality cost; training on a mutable stream
aggregate is a correctness cost.

- **Trade-off made explicit**: the split buys bounded streaming state + warehouse-grade
  reproducibility, and costs a reconciliation component and handover edge cases; pick C
  because correctness for training is non-negotiable while 5-minute staleness is a
  quantifiable loss.
- **Failure modes**: TTL handover races (stream value outlives its batch replacement →
  stale value presented as fresh: stamp `computed_at`, alarm on max-age); streaming
  over-count under late dedup vs batch dedup (the 42-vs-39 case; nightly recomputation of
  recent streaming buckets shrinks it); watermark misfire drops events (monitor
  drops-per-window, not just lateness).

## 3. Model registry & promotion workflow

What must a "model version" identity capture? A model is not weights: it is weights **+
the feature-set definition hash it was trained with + the training code hash + pre/post
processing config + eval receipts**. Serving must be able to reconstruct the exact
environment: if the registry records only "v17", the eventual hotfix to serving may
invisibly change feature resolution for v17 and nobody can prove it.

Promotion pipeline (each transition writes evidence to the registry; the serving tier
refuses to load anything else):

```
candidate → [offline eval gate: beat champion on held-out window, no metric regression]
           → shadow → [mirror 100% traffic, compare score distributions, no serving skew]
           → partial 5–20% → [real-traffic experiment guardrail window]
           → production (pointer move) → champion
```

**Option A — manual promotion** (a human clicks "ship").
- wins: humans catch "looks fine on metrics, smells wrong" cases.
- loses: does not scale to daily retrains; manual steps drift from the recorded evidence;
  rollback-by-human at 3am is the slowest possible version of the fast operation you most
  need to be fast.

**Option B — fully automatic promotion on metric pass**.
- wins: cadence matches the retrain cadence; the registry becomes the single unambiguous
  "what's where" — machine-checkable.
- loses: an eval gate that *passes on noise* ships regressions no human would have; the
  gate becomes the system's weakest point instead of its strongest (deep dive 5's thresholds
  directly bound this).

**Chosen: B with typed exceptions.** Auto-promote through the whole pipeline when every
gate passes with margin; require explicit human sign-off for: first-ever versions of a new
model, any change to preprocessing/tokenization/embedding *dimension* changes, and metric
passes without margin. The exception list is data-shaped, not taste-shaped, so it's
implementable.

- **Trade-off made explicit**: automation wins cadence and auditability; the exception
  list claws back human judgment exactly at the cases whose *evidence* is weakest, which is
  where human attention has the highest expected value.
- **Failure modes**: pointer moves without immutability (someone overwrites the v17 artifact
  → "rollback" can't roll back; artifacts are content-addressed and immutable); gates
  passing on the same held-out window repeatedly (eval-set overfitting by selection: rotate
  the held-out window per candidate); promotion succeeds but the *feature set* changed
  silently (registry pins feature-set hash — a mismatch is a blocked load, not a warning).

## 4. Serving stack: model versioning, shadow traffic, rollback

**Artifact hot-swap**: the model server loads N versions in-process from a content-addressed
artifact store, and the routing config (champion/challenger/shadow splits) is a
fast-changing pointer table consulted per request (cached ~seconds).

**Option A — separate inference service per model over gRPC**:
- wins: independent scaling, per-model languages and hardware; blast radius per service.
- loses: one extra network hop on the hot path (tens of ms — against a 50 ms p99 budget
  that is enormous); version skew between feature service and model service becomes *another*
  consistency problem; fleet-count grows as O(models).

**Option B — one co-located serving binary, many in-process models**:
- wins: no hop; feature fetch and scoring are in one memory space; one deployment.
- loses: a crash or a bad artifact affects the whole server (mitigate: subprocess/isolate
  loading, per-version health gates before traffic shifts); a single heavy model can starve
  CPU (per-model concurrency limits).

For a platform whose hot models are dozens, not thousands, and whose SLO budget is tight,
**B with isolation** wins. (At [ChatGPT scale](../08.chatgpt/04-deep-dives.md) the calculus
differs — one enormous workload justifies dedicated services; a platform serving many small
models does not.)

**Shadow / challenger mechanics**: shadow = execute candidate on mirrored requests, log
scores, *don't return them* (no user risk, full distribution evidence, doubles scoring cost
temporarily); challenger/partial = a slice of real traffic gets real candidate scores under
an experiment flag. Rollback = repoint `champion` to the previous immutable version; servers
observe within seconds, load is already local for retained versions.

- **Trade-off made explicit**: shadow wins risk-free evidence and loses ~2× serving cost
  during the window; rollback-by-pointer wins seconds-level recovery and loses "one current
  artifact" simplicity (you must keep last-K versions warm or lose fast rollback). Both
  worthwhile: the failure they insure against (shipping a bad artifact) is the highest-cost
  failure the platform has.
- **Failure modes**: shadow silently erroring → "evidence" is an empty set that reads as
  "no problems" (shadow must report *coverage*, and low coverage blocks promotion);
  artifact load OOM on a fat embedding table (pre-flight load against a canary node before
  pointer move); hot-swap mid-request mixing versions on a multi-stage model (pin the
  version for the whole request, not per stage).

## 5. Drift detection: PSI/KS, thresholds, and when to retrain

Drift = today's distributions differ from the distribution the model was trained on. Tools:
**PSI** (Population Stability Index: bin each feature's distribution, `Σ (p_i − q_i) ·
ln(p_i / q_i)`) for categorical/binned features — cheap, interpretable, bin-boundary
sensitive; **KS statistic** for continuous distributions — nonparametric, no binning
choices. Both compare the *logged serving* distribution against the training baseline
snapshot, per feature, hourly (~10K comparisons/hour, from the shared log — see
[02](02-scale-estimates.md)).

**Option A — auto-retrain on threshold breach** (PSI > ~0.2 as rule of thumb).
- wins: reaction is fast, no human queue; drift often is seasonal and retraining quietly
  fixes it.
- loses: PSI fires on benign variation (a holiday, a UI change, a bot crawl — the
  distribution *did* shift, but retraining buys nothing and costs GPU + a promotion
  pipeline cycle); auto-retrain on noise turns the GPU pool into an on-call incident target.

**Option B — alert pages, human decides**.
- wins: humans can tell "seasonal drift" from "upstream feature broke" from "world changed",
  which metrics alone cannot.
- loses: drift accumulates silently between pages; at dozens of models, alerts fatigue into
  ignoring exactly the signal the platform exists to catch.

**Chosen: tiered response, auto limited to the safe ones.** Tier 1 (breakage-shaped: a
feature's null-rate or mean moves by more than a robust bound, i.e. the *pipeline* changed)
→ page a human; this is data-plane breakage, not model drift. Tier 2 (slow, broad
distribution movement across many features) → open an auto-retrain trigger **with the
guardrail gates of deep dive 3 applying as usual** — retraining costs a pipeline cycle, not
a regression. Tier 3 (business metric decay confirmed above drift) → the only tier that
mandates immediate human attention, often *revert* rather than retrain.

- **Trade-off made explicit**: auto-retrain wins reaction time, loses money and attention
  on benign shifts; we bound that loss by auto-retraining *only* on the tier where retraining
  is the right tool and letting gates veto the candidate.
- **Failure modes**: per-feature thresholds blind to joint shifts (individual PSI ~0.1 each,
  the *joint* moved — a correlated-shift detector on top of per-feature PSI); baselines
  going stale (baseline must be the *trained-on* snapshot pinned in the registry, not
  "last week's" which drifts along); threshold passes because bin count hides the shift
  (re-verify sensitive features at finer bins before closing an alert); and the classic
  trap: the *model itself* caused the distribution shift (a ranker that down-scores an item
  class starves its own future training data of exposure) — feedback-loop drift can't be
  detected by comparing to your own past outputs; only an experiment or a held-back slice
  sees it.

## 6. Training data pipeline at petabyte scale

30-day retrain ≈ ~15 PB compressed read per cycle (arithmetic in [02](02-scale-estimates.md)).
The pipeline is Spark: nightly incremental builds (1 day of data) + full point-in-time
rebuilds; columnar partitions keyed by (event_day, model, feature-set-hash).

**The core correctness problem is the point-in-time join** — training labels arrive
*time-later* than features, and features mutate continuously (a snapshot of "current"
feature values joined naively to two-week-old impressions *leaks the future* into training).
The builder must as-of-join: for each training row, feature values as they existed at the
row's prediction time — served-log values make this free (that's what was logged), snapshot
features need snapshot history keyed by event time, not "latest".

**Option A — full batch rebuild each night**: recompute the whole 30-day training set
nightly.
- wins: simplest semantics, immune to late-data corrections mid-window.
- loses: ~6 days of burst-pool Spark per cycle (est.) for mostly-unchanged data; can't also
  fit backfills.

**Option B — incremental append + late-data repair**: nightly build appends only the newly
complete day; a repair pass rewrites the trailing window when late events correct it.
- wins: ~5 h nightly hot path; backfills and full rebuilds run independently.
- loses: two build paths must produce byte-identical semantics; late data crossing the
  incremental boundary silently trains models on pre-repair data unless "day-2 complete"
  ordering is machine-enforced.

**Chosen: B, with A as the audited fallback** (any suspicion of incremental drift → one
full rebuild as ground truth for diffing — ~6 days of burst-pool time, so a deliberately
rare, audited event, not routine). This is the same shape
as Twitter's hybrid fanout
([02.twitter](../02.twitter/04-deep-dives.md)): pay full cost for the rare whole truth,
incremental cost for the common case — and keep a way to audit one with the other.

- **Trade-off made explicit**: incremental wins ~30× nightly cost (~5 h vs ~6 days of burst
  pool), loses semantic simplicity
  and adds a completeness contract; we pay for the contract with a per-day *closed* flag and
  a full-rebuild diff harness.
- **Failure modes**: label attribution gaps (a conversion attributed to the wrong impression
  window — attribution rules are data, version them and make them visible in the feature-set
  hash); skew in Spark for high-cardinality keys (popular items explode one join partition —
  salting/replicate-join for the hot keys); storage cost creep (every experimental training
  set materialized forever — tier old experimental sets out and keep only registries' worth);
  and the platform trap: a training set silently *shrinking* (a join dropping rows on a
  schema change) — row-count-per-day alarm on every training set, because a smaller training
  set produces a *successful-looking* run.
