# Time-series databases & metrics pipelines across the case catalog

Where TSDB and metrics-pipeline patterns appear in the 50 study cases, what role they play,
and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md), which lists the
metrics pipeline among the consumers of the "log as backbone" union and names
"precompute-at-write vs derive-at-read" as the freshness-vs-latency trade-off that
metrics (49) shares with feeds, ranking, and typeahead (glue item 8).

Headline: **the catalog builds almost no metrics infrastructure — it consumes metrics.**
One case studies the TSDB and its scrape-to-PromQL pipeline itself; another four make a
metrics feed a designed architectural dependency (deploy gates, drift monitors,
rank-feature freshness SLOs, canary serving metrics, QoE telemetry); five run
telemetry-*shaped* pipelines (log fire-hose → downsample → rollup → dashboards) with no
TSDB and no alert engine anywhere; and half of the remaining vocabulary hits are false
positives — bitmap downsampling, cache-key cardinality, product dashboards.

Counts below are **mention-based from this sweep's greps, not from CONCEPTS.md**:

- Narrow TSDB vocabulary (`tsdb | time-series | prometheus | rollup | downsample |
  histogram | quantile`): **14 / 50** cases
- Wider observability vocabulary (adds alerts / dashboards / telemetry / monitoring):
  **22 / 50** — this is the inflated figure; alerting one's own pipeline is generic prose
- Substantive role (component, designed consumer, or designed telemetry pipeline): **11**
- Brief metrics-flavored mention only: **5**
- Filtered false positives: **5** (listed; they carry no telemetry semantics)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | The case itself: scrape → TSDB (head/WAL/blocks) → PromQL → alerts | Full case |
| [04.deployment-system](04.deployment-system/README.md) | Health evaluator queries an existing TSDB for deploy gates | Deep (consumer) |
| [08.chatgpt](08.chatgpt/README.md) | Canary gates on token-rate/TTFT histograms, rollback on metric deltas | Core consumer |
| [14.ml-production-system](14.ml-production-system/README.md) | Drift monitors + tiered alerting (page vs auto-retrain) | Core consumer |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | recall@K dashboards, counter-staleness SLO alerting | Core consumer |
| [31.netflix](31.netflix/README.md) | Client QoE telemetry → dashboards, monitoring path kept independent | Core consumer |
| [12.bigtable](12.bigtable/README.md) | Replication lag as a first-class metric with alerting | Brief |
| [23.frontend-video-streaming](23.frontend-video-streaming/README.md) | Playback-quality telemetry emitter; collector out of scope | Brief |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Fleet health monitoring; device metrics as firmware-gate inputs | Brief |
| [38.leetcode](38.leetcode/README.md) | Verdict-histogram anomaly detection for cheating | Brief |
| [09.cdn](09.cdn/README.md) | Edge log fire-hose; hot keys downsampled for analytics | Adjacent |
| [28.uber](28.uber/README.md) | Location/trace telemetry downsampled to 15-min rollups; "rollups, traces, ML" consumers | Adjacent |
| [29.uber-eats](29.uber-eats/README.md) | Courier pings downsampled at ingest, never into the order DB | Adjacent |
| [40.tinyurl](40.tinyurl/README.md) | Raw click log vs precomputed rollup counters as two truth sources | Adjacent |
| [47.rate-limiter](47.rate-limiter/README.md) | Decision-log fire-hose feeding dashboards + billing rollups, sampled | Adjacent |
| [06.apple-app-store](06.apple-app-store/README.md) | Hourly chart rollups (analytics, not metrics) + an alert on fallback rate | Adjacent |
| [39.e-commerce](39.e-commerce/README.md) | Generic: alerts on own index freshness — self-monitoring prose | Mention |
| [10.distributed-cache](10.distributed-cache/README.md) | False positive: Redis hash-tag "cardinality", not label cardinality | Filtered |
| [25.mobile-newsfeed](25.mobile-newsfeed/README.md) | False positive: image bitmap "downsampling" | Filtered |
| [26.google-maps](26.google-maps/README.md) | False positive: cache-key cardinality | Filtered |
| [42.linkedin](42.linkedin/README.md) | False positive: social-graph key "cardinality" forks | Filtered |
| [44.payment-system](44.payment-system/README.md) | False positive: merchant product dashboards + ledger balance rollups | Filtered |

## Per-case notes

### The case itself

**[49.monitoring-metrics](49.monitoring-metrics/README.md)** — a pull-based monitoring
stack modeled end-to-end ([03-architecture.md](49.monitoring-metrics/03-architecture.md)):
~10,000 scrape targets → one HA Prometheus pair → TSDB (in-memory head + WAL + compacted
immutable 2 h blocks) → PromQL (range queries + 30,000 alert rules + 5,000 recording
rules) → 200 dashboards and Alertmanager. Sizing anchor
([02-scale-estimates.md](49.monitoring-metrics/02-scale-estimates.md)): 4M active series
→ ~267K samples/s ingest, ~41.5 GB/day compressed (~60 GB/day with index), 2 TB disk per
server at 30-day retention, ~12 GB head RAM (3 KB/series), ~10^9 point touches/s is the
real query load ("QPS is the wrong unit"), and the remote 13-month downsampled tier is
7–8 TB on object storage. Key decisions in [04-deep-dives.md](49.monitoring-metrics/04-deep-dives.md):
pull beats push because the monitor must distinguish *dead from idle* and keep a uniform
cadence when the fleet is sick (`up` is a first-class series; HA is a no-consensus dumb
pair with downstream dedup); a general LSM loses to a purpose-built time-partitioned
engine because retention deletes whole blocks and read amp is the scarce budget; the
label index is an inverted postings index per block; scaling past one server is vertical
federation with the pain kept on purpose, and Thanos/Cortex/Mimir add the object-storage
long-term tier after the fact; series limits are enforced at ingress because a monitoring
stack that dies from cardinality dies exactly when it is most needed; and every windowed
semantic (`rate` reset handling, 5-minute staleness lookback, gap averaging) is chosen to
make alerting expressions computable on lossy input.

### Metrics consumers: the TSDB as someone else's dependency

These cases do not design the pipeline — none of them even names a TSDB vendor. Their
substance is *consumer semantics*: what the freshness, lag, and gap properties of a
metrics feed do to a system that acts on it.

- **[04.deployment-system](04.deployment-system/README.md)** — the sharpest consumer in
  the catalog. Deploy gates are advisory: a health evaluator issues verdicts from
  *queries* against an existing TSDB, explicitly scoped out as a component
  ([01-requirements.md](04.deployment-system/01-requirements.md)). The deep dive
  ([04-deep-dives.md](04.deployment-system/04-deep-dives.md)) reframes rollout as a race:
  TSDB data is 30–60 s stale, so the last window is mostly pre-promotion traffic, and a
  naive "hold ≥10 min" gate still judges on minute-old data after an ingest stall. Chosen:
  data-freshness bounding — the evaluator refuses to return a verdict until the TSDB's own
  ingestion watermark proves the window covers the promotion; and *insufficient data* is a
  first-class verdict (a 1% canary of a low-traffic service correctly holds instead of
  promoting). Costs a TSDB outage: all gates hold, deploys stop — fail-safe by design.
- **[08.chatgpt](08.chatgpt/README.md)** — canary serving metrics as the release
  mechanism ([04-deep-dives.md](08.chatgpt/04-deep-dives.md)): the candidate version's own
  token-rate/TTFT histograms and safety-refusal deltas drive automatic rollback, layered
  with quality via side-by-side preference sampling. The trade-off is stated: automated
  gates are cheap but *canary metrics lag product regressions*, so sampling complements
  them; days-long canaries are the accepted cost.
- **[14.ml-production-system](14.ml-production-system/README.md)** — monitoring as a
  triage discipline ([04-deep-dives.md](14.ml-production-system/04-deep-dives.md)):
  watermark breaches, log-coverage holes, and feature-histogram skew are alarms, and the
  response is tiered — Tier 1 (data-plane breakage) auto-acts immediately, Tier 2 (broad
  slow signal) pages a human who can tell "seasonal drift" from "upstream feature broke"
  from "world changed", Tier 3 (confirmed business-metric decay) is the only tier that
  retrains. Trade-off: Option A auto-retrain reacts fast but the signal is unconfirmed;
  Option B pages accumulate alert fatigue across dozens of models — the tiering splits
  the difference.
- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** — freshness alerting that
  targets a *silent* failure
  ([04-deep-dives.md](15.ml-ranking-system/04-deep-dives.md)): a stalled feature counter
  store looks exactly like "model got worse", so the guard is an SLO alert on stream lag
  (p99 event-to-feature < 60 s), with batch-only degradation as the fallback; and
  recall@K dashboards per retrieval source, alerting on drop, because score distributions
  alone rank garbage with high confidence.
- **[31.netflix](31.netflix/README.md)** — the client-telemetry loop
  ([03-architecture.md](31.netflix/03-architecture.md)): the same session events that
  feed playback feed QoE dashboards, asynchronously, because server-side QoE collection
  at 50M concurrent sessions would need per-session state. Two stated rules that 49
  would endorse: telemetry loss must not look identical to "nobody watched", so the
  monitoring path is kept independent of the data path; and a ranker measured on QoE must
  itself be guarded by guardrail metrics, or it learns to game them.

### Brief mentions

- **[12.bigtable](12.bigtable/README.md)** — one clause inside the deep dives
  ([04-deep-dives.md](12.bigtable/04-deep-dives.md)): replication lag invisibility is
  countered by "lag as a first-class metric with alerting" plus per-request
  `max_staleness` — the same make-staleness-observable rule 49's `up` semantics encode.
- **[23.frontend-video-streaming](23.frontend-video-streaming/README.md)** — the client
  half of telemetry: telemetry emission is in the player's scope, the analytics backend
  is explicitly out of scope — "we emit events; we don't build the collector"
  ([01-requirements.md](23.frontend-video-streaming/01-requirements.md)); the same split
  appears server-side in 31 below.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — fleet operations is a
  product surface; firmware rollout gates read *business* metrics alongside device
  metrics, because device metrics confirmed healthy is exactly the false signal that ends
  in a bricked fleet ([04-deep-dives.md](27.ev-charging-network/04-deep-dives.md)).
- **[38.leetcode](38.leetcode/README.md)** — anti-cheat quarantines participants whose
  verdict histogram deviates from the fleet
  ([04-deep-dives.md](38.leetcode/04-deep-dives.md)): distribution-shape monitoring, one
  sentence, no pipeline.
- **Generic self-monitoring prose** — [39.e-commerce](39.e-commerce/README.md),
  [13.kafka](13.kafka/README.md), and others alert on their *own* pipelines ("now −
  latest event" staleness, consumer lag). This is telemetry vocabulary, not a metrics
  pipeline; it counts toward the 22/50 figure and nothing more.

### Adjacent telemetry pipelines: the shape without the TSDB

These five cases run log fire-hose → downsample → time-bucketed rollup → dashboards —
the physical shape of a metrics pipeline, applied to *source records* (clicks, decisions,
positions, edge lines) rather than to scraped samples. None deploys a TSDB or an alert
engine; the consuming product is billing, analytics, or abuse feedback.

- **[09.cdn](09.cdn/README.md)** — edge access logs (~15 GB/s) feed per-customer
  aggregation; the hot key is protected by partitioning logs by (customer, time-bucket)
  and *downsampling* hot keys for analytics while billing keeps full fidelity
  ([04-deep-dives.md](09.cdn/04-deep-dives.md)). Trade-off: analytics loses accuracy on
  hot traffic, the customer's actual bill never does.
- **[28.uber](28.uber/README.md)** — position samples (huge, lossy OK) go to the event
  log and are downsampled to per-trip traces (~800 GB/day) plus 15-minute rollups, most
  of it deleted quickly by policy because it exists for replay, not serving; "rollups,
  traces, ML" is an explicit consumer family of the log
  ([02-scale-estimates.md](28.uber/02-scale-estimates.md),
  [03-architecture.md](28.uber/03-architecture.md)). This is also the only place in the
  catalog besides 49 that mentions distributed traces at all.
- **[29.uber-eats](29.uber-eats/README.md)** — courier pings (the dominant writer by two
  orders of magnitude) are downsampled *at ingest*, immediately, and never touch the
  order DB; tracking and analytics consume the reduced stream
  ([03-architecture.md](29.uber-eats/03-architecture.md)).
- **[40.tinyurl](40.tinyurl/README.md)** — the clearest statement of the two-truths
  problem this pattern creates: raw click log and precomputed per-link hourly/day rollup
  counters are *two sources of truth that can disagree*, and a stalled aggregation job
  leaves dashboards stale-but-consistent-on-catch-up while redirects never notice
  ([04-deep-dives.md](40.tinyurl/04-deep-dives.md)).
- **[47.rate-limiter](47.rate-limiter/README.md)** — the decision log is a Kafka-class
  fire-hose feeding usage dashboards, billing rollups, and the abuse feedback loop,
  fire-and-forget, sampled or *dropped* under backpressure, never on the enforcement
  path; the stated acceptance ([04-deep-dives.md](47.rate-limiter/04-deep-dives.md)) is
  that when the counter store fails, dashboards built on it go dark at the same time —
  dashboards are not enforcement.
- **[06.apple-app-store](06.apple-app-store/README.md)** — classified here with a caveat:
  its hourly rollups feed *chart rows* (analytics aggregates, not metrics), but the deep
  dives do alert on a rate (download-fallback percentage), the one genuine metrics
  behavior in the case. Counted once, in the adjacent bucket.

### Filtered false positives

These hit the vocabulary grep and carry no telemetry semantics; counted nowhere:

- **[10.distributed-cache](10.distributed-cache/README.md)** — "enforce tag cardinality
  in review" is about Redis hash tags creating hot shards, not label cardinality.
- **[25.mobile-newsfeed](25.mobile-newsfeed/README.md)** — "downsampling" is image
  bitmap scaling.
- **[26.google-maps](26.google-maps/README.md)** — cache-key cardinality.
- **[42.linkedin](42.linkedin/README.md)** — "key cardinality forks" of the social
  graph (degree × privacy × blocked-list membership). The irony: this is a *real*
  cardinality-explosion discussion in spirit, but about index design, not metrics.
- **[44.payment-system](44.payment-system/README.md)** — "dashboard" is the merchant's
  product UI and "balance rollups" are derived ledger state; no metrics pipeline.

## The portable pattern

One consumer contract, then three boundaries. The reference implementation
([49.monitoring-metrics](49.monitoring-metrics/README.md)) is linked, not restated — its
storage-engine analogy to Kafka's log-and-segments is already recorded in
[KAFKA.md](KAFKA.md), and its LSM rejection is already recorded in
[BIGTABLE.md](BIGTABLE.md) ("the workload deletes the tombstone machinery").

1. **A metrics feed is a dependency with a freshness contract, and consumers must read
   its watermark, not assume its lag.** 04's deploy gates refuse to judge a window the
   TSDB hasn't fully ingested; 15 alerts on a staleness SLO because a stalled feed
   masquerades as a regression; 31 keeps the monitor independent of the data path so
   telemetry loss is diagnosable. The general form: every consumer of a lossy,
   eventually-current stream states what it does when the feed is stale, thin, or gone
   — hold (04), degrade to batch (15), fail-safe stop (04's TSDB outage), never
   silently proceed.
2. **Precompute-at-write vs derive-at-read is the shared trade-off** (the
   [CONCEPTS.md](CONCEPTS.md) glue item): 49's recording rules pre-aggregate hot series
   once per interval for many panels (50 × 10M point touches → one), exactly as 40 and 47
   precompute rollup counters for dashboards/billing, and they inherit the same cost —
   recording-rule cardinality is not free (49), rollups are a second truth that can
   disagree with the raw log (40), and precomputed dashboards are not enforcement (47).
3. **Downsample at ingest, gate by fidelity class** — telemetry tiers (29: downsample
   immediately, never into the order DB; 28: raw for replay, traces and rollups for the
   rest, deleted by policy; 09: downsample analytics, never billing). The decision to
   make explicit is *which fidelity class the money path uses*; everything else can
   degrade.

### Where the shape ends

- **Cardinality is the whole cost model.** 49's arithmetic (label-value domains
  multiply; ~3 KB heap per active series; distinct series governs disk while active
  series governs RAM; one histogram with five ordinary labels is ~5B combinations) is
  the reason TSDBs enforce series limits at ingress instead of trusting conventions —
  the same lesson as Bigtable's row-key design (see [BIGTABLE.md](BIGTABLE.md)), with
  the twist that a monitoring stack which dies from its own cardinality dies at exactly
  the moment it is needed. Approximate counters are the mitigation for the *ingest
  volume* side of the family, covered in
  [PROBABILISTIC-STRUCTURES.md](PROBABILISTIC-STRUCTURES.md).
- **Scrape model vs push is a semantics choice, not a transport choice.** Pull buys one
  enforced cadence, positive observations of target death (`up`), and cheap no-consensus
  HA — at the cost of losing events between scrapes. The catalog's push-shaped cases
  (40, 47) accept per-event loss instead, chosen because their interactive path must
  never queue; 47's "sampled or dropped under backpressure" is the push contract, 49's
  gap-averaging and staleness marking is the pull contract, and they are not equivalent
  anywhere it matters (detectability of the failure itself).
- **A TSDB is the wrong tool for everything non-windowed.** 49 stops PromQL short of
  arbitrary joins on purpose, and pushes relational work to logs/traces/warehouse. The
  catalog agrees in one direction and not the other: no case designs a tracing
  infrastructure (28 has the word, no system), while *log* pipelines are everywhere —
  and the split stays clean because the products differ: logs answer "what happened for
  this one request" with full fidelity and high cost, metrics answer "what was the rate
  across the fleet" with sub-second stability and no per-event shape. When a case needs
  both, they stay separate consumer families on the same log (28) rather than one
  store (see [KAFKA.md](KAFKA.md) for that log).
- **Alert frameworks treat "no data" as not firing** — so long-outage detection is a
  multi-rule patch (`up == 0`, `absent()` around the rate rule), the same shape as the
  typeahead fallback stack [45.typeahead](45.typeahead/README.md) uses for degraded
  input. Detection of the absence of evidence is always a separate mechanism.
