# 01 — Requirements

## Functional

Core v1 (the platform's contract with its users — model-developing teams):

- **Define features once**: a feature definition (source table/stream, transformation,
  TTL) is declared in one place; both training and serving consume that declaration. No
  team writes a Spark version *and* a Java version of the same transformation by hand.
- **Build training sets**: assemble labeled, point-in-time-correct training data for a
  model from batch sources + streaming sources, at petabyte-per-cycle scale (arithmetic in
  [02](02-scale-estimates.md)).
- **Train**: run distributed training over that data on a shared GPU/CPU pool; emit a
  versioned artifact (weights, metrics, feature-set commit hash, code commit hash).
- **Registry and promotion**: register the artifact; run the promotion workflow
  (offline eval → shadow → partial rollout → full); record who/what approved each step.
- **Online serving**: for a predict request, fetch features, apply the model, return
  scores within SLO; support many model versions concurrently (champion/challenger, shadow).
- **Prediction logging**: log exactly what was served — model version, feature values
  fetched, raw inputs, scores — to become tomorrow's training data and monitoring input.
- **Monitoring**: feature distribution drift, prediction distribution drift, serving SLOs,
  per-model custom metrics; alerting and (bounded) automated responses.

Out of scope v1 (state them): AutoML/hyperparameter search services (teams bring their own
training code), GPU *hardware* scheduling internals (we treat the cluster as a given),
online-learning models updating weights in real time (batch/incremental retrain only), A/B
experiment assignment (assume the experiment service exists; we consume flags), data
lineage governance beyond an audit log on the registry, on-device model deployment.

## Non-functional

| Property | Target (estimate) | Consequence |
|---|---|---|
| Prediction latency | p99 **< 50 ms** end-to-end (feature fetch + score), inside the product page's ~200 ms budget | features must come from a KV store, not batch tables; model code in-process or co-located, no extra network hop |
| Serving availability | 99.9% of predict requests (degrade: fall back to simpler model / cached scores / prior) | serving never hard-depends on the *training* side being up |
| Feature freshness (online) | streaming features < ~5 min stale; batch features updated daily — but a value is never silently *wrong-vintage* (each value carries its compute time) | every feature row stores `computed_at`; "freshness" is metadata, not a guarantee you can cheat |
| Feature freshness (training) | labels complete within ~24 h (label window closes nightly) | training always trains on *closed*, labeled windows; never partial labels |
| Training-set build | full retrain of the ~30-day window in **~6 days** on the burst (10×) pool; nightly incremental in **~5 h** on the same burst pool (at the standing pool's ~3 GB/s it would be ≈ 2 days, too slow for a daily cadence) | backfills must fit alongside; diffing against the last full rebuild keeps the nightly cheap |
| Feature skew | **zero tolerance** for train/serve value mismatch on logged features — enforceable by construction, not by review | the design decision everything else hangs off (deep dive 1) |
| Promotion cadence | top models retrain daily; ship ~2–3 promoted versions/week per hot model | workflow must be safe at this frequency — no human-only gates |
| Rollback time | any serving rollback effective fleet-wide in **< 10 min** | versions are immutable artifacts; rollback is a pointer flip, not a rebuild |
| Cost | feature computation once per definition, not once per consumer | the registry-wide feature reuse is the platform's economic reason to exist |

## Scope decisions that shape everything

1. **Train on what was served, not what would have been served.** The logged feature
   values from production are the *primary* source for training data; batch recomputation
   is only the fallback for features too new to have been logged. This is the cheapest
   skew-elimination decision available (deep dive 1).
2. **The registry is a directed workflow, not just a directory.** A model that reached
   `production` carries its eval receipts; the serving stack refuses to load anything the
   workflow hasn't touched. This converts a policy question into a machine-checkable one.
3. **Serving degrades to a simpler model, not to zero.** Every surface has a hand-set
   fallback (popularity prior / tiny model); the platform's availability engineering is
   choosing and exercising that path, since a 503 in ranking is invisible until you look.
