# 01 — Requirements

Scope: a self-hosted Prometheus-style pull-based monitoring stack for a fleet of
~10,000 scrape targets, with HA scraping, long-term storage via a remote/federated layer,
PromQL querying, and rule-based alerting. Logs and tracing are explicitly out of scope
(§ scope decisions).

## Functional

Core v1:

- **Pull-based collection**: the monitor discovers targets (static config + service
  discovery against the deployment system / DNS), and scrapes each target's `/metrics`
  HTTP endpoint at a fixed interval (default 15 s). Target exposes samples as
  `metric_name{labels} value` lines. Failed or malformed scrapes are recorded as
  observations, not silently skipped.
- **Label-based data model**: every sample belongs to a *series* identified by
  (`metric name`, set of label key/value pairs). Metric names are labels underneath
  (`__name__`), so name matching and label matching are the same mechanism. A sample's
  identity is: series + timestamp; values are float64 (+ one staleness marker value).
- **Push gateway exemption**: short-lived batch jobs that cannot be scraped on a cadence
  push to an intermediary that *is* scrapeable. Push exists to solve the job-lifetime
  problem, not as the primary direction (deep dive §1).
- **Instant queries**: evaluate an expression at one point in time (dashboard stat tiles,
  alert evaluation, `topk`, current values).
- **Range queries**: evaluate an expression over a step-cadence across a time window
  (the graph path: start/end/range + resolution; e.g. 1,000 × rate over 24 h + 5-min
  lookback windows).
- **Aggregation language (PromQL)**: selectors with label matchers (`=` `!=` `=~`),
  functions (rate/deriv/histogram_quantile), aggregation operators (`sum/avg/by/without`),
  and *limited* joins: vector matching on identical or listed labels, and one-to-many
  across a small label set (group left) — not arbitrary relational joins (deep dive §3).
- **Alert rules**: boolean/threshold expressions evaluated periodically (e.g. every
  30 s) with `for`-duration hysteresis; firing alerts carry labels and annotations and
  are sent to Alertmanager, which does dedup, silencing, grouping, inhibition and routing.
- **Recording rules**: pre-compute expensive expressions into new series for reuse
  (dashboard and cardinality pressure valve).
- **Retention & compaction**: local blocks (e.g. 2 h each) with down-sampling and
  block deletion per retention policy; federation of data older than the local window to
  a remote store.
- **Self-monitoring**: the system instruments itself (scrape durations, head series
  count, memory, rule evaluation time) — a monitoring system with blind spots cannot be
  operated.

Out of scope v1, deliberately (and why):

- **Logs** — high-volume, unindexed-until-indexed text is a different storage shape and a
  different query language; a separate system (Loki/ELK-style). Metrics deliberately keep
  only the numeric extract, which is why metrics storage is cheap where log storage is
  not.
- **Tracing / spans** — request-scoped causal data with tree structure; orthogonal to
  aggregated counters and gauges. Only "metrics about tracing" (RED rates) live here.
- **Push as primary ingestion** — see deep dive §1; without cadence you cannot compute
  rate, and the monitor can no longer discover what exists.
- **Arbitrary SQL / joins across arbitrary label sets / full-text search** — the label
  index answers "which series match this selector", not "join series set A against series
  set B on any predicate" (deep dive §3).
- **Exactly-once or per-observation queryable data** — one sample per series per scrape,
  with lossy network semantics. This is monitoring, not event sourcing; see deep dive §6.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Scrape interval | 15 s default; 60 s for low-value targets (arithmetic in [02](02-scale-estimates.md)) | fixes sample rate; drives ingest QPS, storage, and the lookback semantics of `rate` and staleness |
| Ingest capacity | ~270K samples/s sustained, 2× peak per server (estimate) | head block + WAL must absorb bursts without backpressuring scrapes for minutes |
| Query latency | dashboard panel p95 ~1 s at 10K-series selector scale; alert eval budget ~5 s per batch | forces the label index (postings) and per-selector worst-case accounting (deep dives §3, §5) |
| Query semantics | *eventual and approximate* — data may be minutes stale; results approximate | every read path must decide how staleness/missing samples render (deep dive §6) |
| Retention | local 15–30 days (1–2 TB, arithmetic in [02](02-scale-estimates.md)); longer horizons via remote store | local storage stays disk-cheap; the remote layer pays for 13 months |
| Availability | HA via two identical, redundant *non-consensus* servers (both scrape everything; upstream dedups) | no quorum in the data path; losing one server loses nothing but the redundancy margin (deep dives §1, §4) |
| Durability | WAL flush before samples are queryable; block upload on boundary; losing ≤ 2 h local history on total node loss is acceptable | monitoring data has no per-sample value — losing a slice of one series is survivable, missing the alert was not |
| Write amplification tolerance | compaction may rewrite blocks several times; writes are still appends | same family as [Bigtable](../12.bigtable/01-requirements.md)'s LSM trade — cheap writes, compaction pays |
| Cardinality budget | bounded active series per server (millions, enforcement via limits) | the whole system's failure floor is "too many label combinations" (deep dive §5) |
| No coordination in the data path | monitors talk to targets directly; no consensus service between alert evaluation and scrape | same pattern as [ZooKeeper](../11.zookeeper/01-requirements.md)'s "off the hot path" — availability is per-target, not global |

## Scope decisions that shape everything

1. **Pull, and make targets dumb.** Given the monitor must survive the monitored fleet
   being sick, Option pull wins "discover targets, detect dead targets, smooth cadence"
   and loses "firewall/NAT traversal, instant event capture" — we pay for the loss with
   service discovery + the push gateway (deep dive §1).
2. **Labels as the whole data model, not pre-declared metrics.** No schema/registry:
   wins operator self-service at the target, loses any global constraint on cardinality
   short of runtime limits — the bill is paid in §5.
3. **One sample per series per scrape; staleness markers instead of silence.** Wins
   uniform 15-s cadence (uniform cadence makes `rate` well-defined and index scans
   predictably sized), loses event-y fidelity (a counter that changes between scrapes is
   collapsed into one delta) — §6.
4. **Local-first storage, remote as a bolt-on.** Wins a working system with no
   distributed-DB dependencies; loses clean long-term scaling (§4, and the reason
   Thanos/Cortex/Mimir exist).
