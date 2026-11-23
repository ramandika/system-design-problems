# Case 49 — Prometheus / time-series monitoring

**The system**: pull-based metrics monitoring. Instrumented targets expose numeric
time series over HTTP; a Prometheus server *scrapes* them on a fixed cadence, stores the
samples in a purpose-built time-series database (in-memory head + write-ahead log +
compactable blocks), answers PromQL range/instant queries, evaluates alert rules on the
same data, and hands firing alerts to Alertmanager for routing and dedup. The product is
not "store all the numbers" — it is *answer "what was X between T1 and T2, aggregated
across these labels" in under a second*, continuously, for dozens of dashboards and
thousands of alert rules.

The pipeline in one line:

```
exposition (/metrics endpoint) → scrape (pull, fixed interval) → TSDB (head/WAL/blocks)
→ PromQL engine (range + aggregation queries) → dashboards & alert rule evaluation
→ Alertmanager
```

**Why it's hard**:

- **The data model is the workload, and it is combinatorial.** A sample is not "metric
  foo = 5"; it is "metric foo with label set {job, instance, handler, method, code, …} =
  5". Every distinct label value combination is a separate series you must name, index,
  and keep fresh. Ten label values each on four labels is ~10^4 series from one metric —
  cardinality explodes multiplicatively, and it explodes from *ordinary deployments*
  (a per-customer or per-endpoint label), not from traffic spikes. Query-time, every
  aggregation (`sum by (job)`) re-combines those series back together, so the read path
  pays for the fan-out the write path created.
- **Scrape fanout is a many-tiny-requests workload.** One server pulls from thousands of
  targets every 15 s — thousands of small HTTP GETs per interval, each returning dozens
  to thousands of lines. Nothing is batchable at the transport (targets are independent
  endpoints); the system lives on per-target concurrency control and predictable cadence.
- **Write path is trivially cheap; it is designed to be.** Samples arrive as an
  append-only, in-order, small-value stream — an LSM-like head block plus WAL, compacted
  into immutable chunks. If the write path gets expensive or random-access, the whole
  design has lost (compare [Kafka](../13.kafka/README.md)'s log and
  [Bigtable](../12.bigtable/README.md)'s LSM: cheap appends, pay at read/compaction).
- **The hard decisions are query-side and semantics-side**: which label index supports
  "these 40,000 series out of 4M, now"; what a `rate()` means when the counter reset or
  the scrape gap; and how to scale past one server (Prometheus's answer is deliberately
  awkward — and the Thanos/Cortex/Mimir layer exists to paper over that choice).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: pull over push for self-service health (the monitor discovers
targets, targets stay dumb), append-only + compaction as the storage engine answer for
immutable time-ordered data (same family as Kafka's segments and Bigtable's SSTables;
§2), inverted index + postings to buy query speed from a write-amplified index (§3),
vertical sharding chosen over horizontal despite the pain (§4), a cost model as the
mitigation for combinatorial data (§5), and windowed semantics on lossy input data (§6).

**Try it first**: before reading 03, answer: (1) the monitor must watch the cluster as a
whole — push and pull both deliver samples. What does *pull* buy that push doesn't, that
matters when the monitored system itself is sick? (2) A query is `rate(errors[5m])` and
the scrape was down for 8 minutes. What should the graph show — a hole? a zero? Why did
the designers pick what they picked? (3) Someone adds one label `customer_id` with 50,000
values to a hot metric. Which parts of the system break first, and in what order?
