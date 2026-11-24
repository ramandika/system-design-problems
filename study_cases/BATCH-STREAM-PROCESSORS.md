# Batch / stream processors across the case catalog

Where Spark-class / Samza-class / MapReduce-class distributed compute appears in the 50 study
cases — named or not — what role it plays, and how deep each case goes. Companion to
[CONCEPTS.md](CONCEPTS.md), which has no row for batch/stream processors; its nearest row
("Precompute-at-write vs derive-at-read", 13/50) is a tier-trade-off pattern, not a compute
tier. This doc is a first pass at the compute-side concept. Boundary: [KAFKA.md](KAFKA.md)
owns the event log — this doc owns the compute that reads from it.

Headline: **in this catalog the processors are almost never the subject — they are the
derived-data consumers.** Only 5 / 50 cases name an engine; 10 more design a batch or stream
compute tier without naming one. Wherever compute appears it is never the source of truth
(the transactional plane or the log is), its freshness bound is set by a job schedule
(hourly, nightly, day-2-complete), and the sharpest named trade-off is the same in every
case: the same transformation implemented twice (batch engine + online runtime) is a skew
bug waiting to happen.

- Cases with a batch/stream compute role: **15 / 50**
- Named engine (spark / samza / MapReduce / hadoop / flink): **5** (08, 11, 12, 14, 42); all
  5 are counted in the table above
- Unnamed but designed compute tier (batch tier, sweep/rebuild/aggregate job): **10** (06, 10, 15, 17, 31, 36, 38, 39, 45, 48)
- Of the 5 named: designed workload 2 (14, 42), named as the required consumer class 1 (12),
  rejected alternative 1 (08), analogy 1 (11)
- Depth here is mention-based and **not comparable** to the CONCEPTS.md union counts (that
  count counts cases per concept-row; this counts hits of one tech family)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [08.chatgpt](08.chatgpt/README.md) | Map-reduce summarization named as a rejected alternative for long-context serving | Brief |
| [06.apple-app-store](06.apple-app-store/README.md) | Hourly chart-aggregate jobs over the transaction event stream (not real-time by requirement) | Brief |
| [10.distributed-cache](10.distributed-cache/README.md) | Nightly one-time-scan job as a workload the eviction policy must survive | Brief |
| [11.zookeeper](11.zookeeper/README.md) | Spark driver / HDFS NameNode as fleet-leader analogy in the 40 s GC story | Analogy |
| [12.bigtable](12.bigtable/README.md) | MapReduce-style batch scans as the workload the engine is shaped for | Core component |
| [14.ml-production-system](14.ml-production-system/README.md) | Petabyte-class Spark training pipeline; batch-vs-serving skew is THE dive | Full case |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Realtime (streaming counters) + batch (daily/hourly precompute) feature tiers, off one event log | Core component |
| [17.elevenlabs](17.elevenlabs/README.md) | Batch render jobs vs interactive streams on shared GPUs, with preemption asymmetry | Core component |
| [31.netflix](31.netflix/README.md) | Nightly forecast job drives the off-peak CDN cache fill | Brief |
| [36.airbnb](36.airbnb/README.md) | Nightly + on-demand rebuild-as-audit sweep job over availability bitmaps | Brief |
| [38.leetcode](38.leetcode/README.md) | Anti-cheat sweep as a scheduled batch job producing review evidence, not verdicts | Brief |
| [39.e-commerce](39.e-commerce/README.md) | All feature computation (ranking, personalization) is offline/stream over the spine | Brief |
| [42.linkedin](42.linkedin/README.md) | Samza-type stream jobs (invalidations, PYMK, WVYP) + Hadoop/Spark-era offline/ML jobs | Core component |
| [45.typeahead](45.typeahead/README.md) | Top-k index rebuilt batch-side hourly; hot layer updated per-event | Brief |
| [48.google-search](48.google-search/README.md) | Indexer as the streaming transformer from crawler stream to immutable segments | Moderate |

(Not counted, see false-positive dictionary: 37.hourly reconciliation, 41 batch-of-URLs work
units, 49 pushgateway batch jobs, GPU request batching throughout 08.)

## Per-case notes

### The flagship: compute as a correctness boundary

**[14.ml-production-system](14.ml-production-system/README.md)** — the only case where the
processor is a first-class subject. Two things carry the case:

1. **The skew dive** ([04-deep-dives.md](14.ml-production-system/04-deep-dives.md)) names
   the dual-computation problem precisely: "**double computation**: the same feature
   implemented twice (Spark for training, Java for serving) — floating point, timezone,
   join-order, and filter-order differences leak in." Three options: A = two
   implementations, kept in sync by review ("review is not a consistency model"); B = one
   declaration compiled to both runtimes; C = train strictly on the served log and recompute
   offline only for gaps. **Chosen: C as the backbone, B underneath, A banned** — C wins
   "training/serving fidelity and vintage safety", loses the freedom to train on
   never-served features and adds a shadow-logging window for every new feature.
2. **The petabyte-scale pipeline** ([02-scale-estimates.md](14.ml-production-system/02-scale-estimates.md)):
   30-day window ≈ 750B rows/day × 30 = 22.5T rows × ~2 KB ≈ ~45 PB raw → ~15 PB compressed
   read per cycle; at a ~500-node standing pool's ~3 GB/s that is ~1,400 h (~2 months), so
   rebuilds burst to a ~10× pool (~30 GB/s): 15 PB ÷ 30 GB/s ≈ ~6 days, while the nightly
   incremental (500 TB ÷ 30 GB/s ≈ ~5 h) makes the daily cadence possible. The deep dive
   picks **incremental append + late-data repair, with the full nightly rebuild kept as an
   audited fallback** — incremental wins ~30× nightly cost, loses semantic simplicity and
   adds a per-day "closed" flag plus a full-rebuild diff harness. A named compute failure
   mode: **skew in Spark for high-cardinality keys** — popular items explode one join
   partition; salting / replicate-join for hot keys. Named constraint in README: the
   model/serving invariant "crosses a boundary Spark controls (batch, minutes-to-hours) and
   a KV store serves (online, milliseconds)". Boundaries: the offline-store embedding angle
   is covered by reference in [VECTOR-SEARCH.md](VECTOR-SEARCH.md) and the serving/storage
   side is other docs' territory; this doc owns only the training/compute path.

### Named processors as designed workloads

- **[42.linkedin](42.linkedin/README.md)** — the Samza case by lineage. Components table
  ([03-architecture.md](42.linkedin/03-architecture.md)): Stream jobs = "invalidations, PYMK
  recompute, WVYP aggregation / Samza-type consumers"; Offline/ML = "feature pipelines,
  ranker training, PYMK ranking / Hadoop/Spark-era jobs". The deep dive's storage Option B
  puts it the same way — dist-2 is "aggregated offline/streaming where cost is amortized"
  — and records where LinkedIn's own stack went: "Kafka/Samza for the rebuild stream".
  Boundary: the event bus those jobs consume is [KAFKA.md](KAFKA.md)'s; the jobs are this
  doc's.
- **[12.bigtable](12.bigtable/README.md)** — the engine is a storage case; the processor is
  why the engine is shaped so. Requirements: range scans are "typically consumed by
  MapReduce-style batch jobs, streamed at maximum bandwidth, not low-latency" (scans sized
  ~200 table-equivalent scans/day, at most 1 TB of hot data each after locality pruning).
  Architecture: "**Batch scan**: MapReduce-style job → client library enumerates tablets
  from METADATA → workers stream their tablets directly" — the tablet map *is* the
  partitioning plan, so no central fanout service exists. Failure mode on the compute side:
  a batch job writing a sequential key range lands on one tablet/server → compaction queue
  backs up; mitigation is pre-splitting on table creation. Storage engine internals are
  [BIGTABLE.md](BIGTABLE.md)'s; its batch-scan consumer is also already described by
  reference in [RPC-SERIALIZATION.md](RPC-SERIALIZATION.md).

### Designed compute tiers without a named engine

These are the cases that make the catalog's argument: the design decisions are
engine-independent. They are Spark-eligible workloads that never needed to say "Spark".

- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** — the two-tier split stated as
  a trade-off ([04-deep-dives.md](15.ml-ranking-system/04-deep-dives.md)): realtime tier
  streams recent-window counters off the event log (seconds stale, small, hot); batch tier
  daily/hourly-precomputes long-horizon stats and profile rows (hours-to-day stale, big:
  ~50 TB user tier). Trainer retrains daily/hourly from a label joiner that waits out the
  label delay.
- **[17.elevenlabs](17.elevenlabs/README.md)** — batch *scheduling*, not batch data: batch
  render jobs and interactive streams want the same GPUs. Chosen option: separate
  schedulers, shared hardware, **preemption asymmetry** — batch is preemptible only at
  segment boundaries, each with a persisted checkpoint, so preemption costs at most one
  segment re-render; stated trade-off: give up utilization to make "batch can never make an
  interactive stream audible-degradeable" provable.
- **[48.google-search](48.google-search/README.md)** — the indexer is clean stream
  processing: the crawler stream (at-least-once, idempotent by canonical-URL hash) → parse →
  tokenize → shard-local doc-ID → postings → compression → immutable fresh-tier segment;
  on outage the freshness SLA slips, but re-delivery never corrupts.
- **[39.e-commerce](39.e-commerce/README.md)** — one rule carries the compute tier: "No
  inline analytics on the write path. All feature computation (ranking, personalization) is
  offline/stream over the spine" ([03-architecture.md](39.e-commerce/03-architecture.md));
  the Analytics/ML consumer family is projections of the same event spine, rebuildable by
  design.

### Brief mentions

- **[06.apple-app-store](06.apple-app-store/README.md)** — charts are hourly aggregates over
  the transaction event stream, produced by aggregators feeding store-chart tables;
  explicitly *not* real-time, so the job schedule is the freshness bound.
- **[36.airbnb](36.airbnb/README.md)** — per-event deltas for freshness + "a nightly (and
  on-demand per-listing) rebuild-as-audit that recomputes bitmaps from the booking database
  and diffs" — drift found is an incident, not the repair path. Same shape as 14's
  full-rebuild fallback.
- **[38.leetcode](38.leetcode/README.md)** — anti-cheat similarity pipeline runs as "a
  scheduled batch job that produces review evidence, not auto-bans"; per-submission path
  must stay synchronous, so the sweep travels as a flag with the verdict.
- **[45.typeahead](45.typeahead/README.md)** — the stable top-k index "is rebuilt batch-side
  hourly" while the hot layer updates per-event; ~2.4B reads/day ride rebuilds, so build
  cost amortizes over minutes against 28K QPS of query-time work.
- **[31.netflix](31.netflix/README.md)** — the nightly forecast job is the brain of the
  off-peak fill: forecast-popular content is prepositioned into ISP caches while compute and
  bandwidth are cheap.
- **[10.distributed-cache](10.distributed-cache/README.md)** — a "nightly job reading every
  user once" is the workload that a recency-bound cache must absorb without letting
  one-time scans pollute eviction decisions.

### Analogy

- **[11.zookeeper](11.zookeeper/README.md)** — no processor in the architecture; the deep
  dive's 40 s GC story borrows the fleet-leader role: "a Kafka broker (or HDFS NameNode, or
  Spark driver — same everywhere) that was elected leader via an ephemeral node hits a
  garbage-collection pause for 40 s with a 30 s session timeout". The point is that Spark
  (and Kafka) are JVM long-lived leader processes whose liveness rides session
  heartbeats.

### Rejected alternative

- **[08.chatgpt](08.chatgpt/README.md)** — "map-reduce summarization" appears only as a
  rejected option for long-context serving: it "add[s] a second model pass on the serving
  path" ([04-deep-dives.md](08.chatgpt/04-deep-dives.md)). MapReduce the *pattern*, not the
  engine; the case says no because compute belongs off the interactive path — the same
  boundary 17 argues, and the reason batch tiers keep serving latency clean everywhere else.

## False-positive dictionary

Hits excluded from the counts, with reasons:

- **"batch(ing)" as GPU/inference request batching** (08.chatgpt memory arithmetic, 17
  audio-chunk batching, 48 heavy ranker batching "a re-scored top ~200"): request-batching
  inside one process or query, not a distributed batch processor.
- **08's "map-reduce summarization"**: a chunk-map/summarize-reduce pattern for LLM context,
  not Hadoop MapReduce (counted as a Brief row for the named phrase, flagged here anyway).
- **41.web-crawler "work unit (batch of URLs for one host)"**: politeness batching of crawl
  work items, not batch compute.
- **37.ticketmaster hourly reconciliation, 49.monitoring-metrics pushgateway "short-lived
  batch jobs"**: mentions of reconciliation discipline and of monitoring a batch job, not of
  running one.
- **15's "Batch tier"** is a freshness tier name, not an engine name (counted under its
  designed tier, not under named engines).
- No "spark" false positives (no ignition/energy-sense uses found in any case).

## The portable pattern

1. **Compute is a derived-data consumer, never the source of truth.** It reads the event
   spine (42, 15, 39, 06) or the table/scan surface (12) and writes projections back.
2. **The freshness bound is the job schedule, and it is a product decision**: hourly charts
   (06), hourly top-k rebuild (45), nightly fills (31), day-2-complete training partitions
   labeled "never partial" (14).
3. **Dual computation is the recurring named enemy**: Spark-for-training + Java-for-serving
   is a skew generator (14, banned there); the surviving defenses are log-and-replay (14),
   one shared declaration compiled to both runtimes (14 Option B), and stream/batch tiers
   reading the same event log so only the window differs (15).
4. **Incremental + audited full rebuild** beats full-rebuild-everything on cost (14: ~5 h vs
   ~6 days) and the full path is kept, rare and audited, as ground truth (14, 36, 39).
5. **Stream-batch on shared hardware survives only with cheap-recovery preemption**: 17's
   segment-boundary checkpoints and preempt-only-where-recovery-is-cheap rule generalize to
   any co-located batch workload.
6. **Where the pattern is the wrong tool**: interactive serving paths (08 rejects a compute
   pass between request and token), microsecond budgets (the same asymmetry
   [50.stock-exchange](50.stock-exchange/README.md) argues for its matcher — deterministic
   replay, not scheduled compute), and any workload whose correctness depends on
   scan-side latency (12 pays for scan bandwidth by keeping the interactive point-read path
   a different contract).
