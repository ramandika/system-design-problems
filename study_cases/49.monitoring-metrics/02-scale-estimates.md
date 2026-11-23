# 02 — Scale estimates

Back-of-envelope for one monitoring cell (one HA pair of Prometheus servers). All figures
rough estimates; adjust the assumptions and redo the arithmetic. Public reference points:
Prometheus's own docs give ~1.8 bytes/sample on disk as the typical compressed sample
size; operator guidance treats a *single* Prometheus server as comfortable up to millions
of active series and degrading in the tens of millions; community cardinality writeups
(publicly reported) show real fleets with hundreds of thousands to a few million active
series. Numbers below stay inside that envelope.

## Assumptions (state them, everything else follows)

- Fleet: **3,000 hosts** (VMs / bare metal / K8s nodes), ~250 apps.
- **10,000 scrape targets**: 6,000 infra exporters (node/cAdvisor-style: ~500 series
  each) + 4,000 app/service exporters (~250 series each, mix of counters, gauges,
  histogram buckets).
- Active series: 6,000 × 500 + 4,000 × 250 = **4M active series**.
- Scrape interval **15 s** standard, 60 s for a tail of low-value exporters (the tail
  lowers the totals slightly; the arithmetic below assumes 15 s everywhere — the tail is
  a second-order correction).
- Retention: **30 days local**, 13 months in the remote layer (downsampled).
- Dashboards: **200 Grafana dashboards × ~25 panels**, auto-refresh 30 s.
- Alert rules **30,000**, evaluated every **30 s**; recording rules 5,000, every 30 s.
- Sample identity: series + timestamp; value float64. Timestamp and value compress to
  ~1.8 B combined (double-delta on timestamps, XOR on values — publicly reported figure).

## Scrape path (write direction)

**Scrape rate**: 10,000 targets ÷ 15 s ≈ **667 scrapes/s** (steady; staggered by
scrape-offset scheduling, not thundering on the second).

**Per-target payload**: average 400 series × ~120 B of text exposition ≈ 48 KB/scrape
uncompressed; HTTP gzip ≈ 6–8 KB. Upstream targets collectively serve
667/s × 8 KB ≈ **5 MB/s** compressed (~32 MB/s uncompressed at the exporters' write-out;
exporters are the *cheap* side, they just format text).

**Sample ingest**: 4M series ÷ 15 s = **~267K samples/s** sustained, ~2× (500K/s) at
daily peak. After a mass restart (rolling deploy of 500 pods), transient +10% churn —
the ingest QPS barely moves; the *series* count is what moves (§ cardinality below).

**WAL**: samples hit the WAL before they are queryable. A varint-encoded WAL record for
(ts, value, stream id) ≈ ~5 B/sample (estimate):
267K/s × 86,400 s ≈ 23G samples/day × 5 B ≈ **~115 GB/day of WAL writes** — WAL segments
cover the 2 h head window only (~10 GB live) and are truncated as blocks compact, so the
disk bill (below) counts them as *transient* I/O, not stored data. This WAL is why a
crash preserves the head block: same "log before you ack" family as Kafka's and
Bigtable's commit logs ([13.kafka](../13.kafka/03-architecture.md),
[12.bigtable](../12.bigtable/04-deep-dives.md) §2).

## Storage

**Sample data**: 23G samples/day × 1.8 B ≈ **41.5 GB/day** of compressed sample bytes.

**Index + metadata + block overhead**: postings, label-value dictionaries, tombstones,
block meta — add ~1.5× → **~60 GB/day stored**.

**30-day local retention**: 60 × 30 ≈ **1.8 TB** → **2 TB disk** per Prometheus server;
**4 TB for the HA pair**. Monitoring storage is *cheap by construction* — the same 4M
series at 1 raw sample/s would be 15× more (86,400 samples/series/day ÷ 5,760 at 15 s);
the 15 s cadence is the entire budget.

**RAM (head block + index)**: per active series, chunk cache + series metadata + index
entries ≈ 2–4 KB (estimate; matches operator-reported heap behavior):
4M × 3 KB ≈ **12 GB heap** for the head → 32–64 GB RAM servers with query headroom.

**Remote long-term store**: 13 months needs ~13 × 30 × 60 GB ≈ **23 TB raw-rate**; with
downsampling (raw 15 s for 30 d → 1 min for 90 d → 5 min after) the retained set is
~30% of raw-rate ≈ **7–8 TB** on object storage.

## Series churn and cardinality (the number that actually hurts)

Ingest QPS is nearly constant; *distinct series names* churn because pods come and go.

- Churn rate: ~10% of active series replaced per hour during rolling deploys (estimate:
  ~500 pod restarts/h × ~800 series each) → **~400K new series name/hour**; sustained
  round-the-clock that is ~9.6M distinct series/day, but deploys concentrate in working
  hours — call it **~8M distinct series/day**, ~2× active.
- Index must hold postings for all distinct names in the retention window's head +
  recent blocks; the *label-value dictionary and postings lists* grow with total
  distinct series, not active — this is exactly the divergence that makes cardinality
  limits bite (deep dive §5).
- **The cardinality bomb example**: one histogram
  `http_request_duration_seconds{job, instance, method, code, handler}` with label
  domains job=200, instance=10,000, method=5, code=10, handler=50 →
  200 × 10,000 × 5 × 10 × 50 = **5B combinations** (plus ~11 buckets each — histogram
  metrics multiply by bucket count, so ~55B with them). One ordinary-looking metric.
  Per-metric limits and
  per-series accounting exist because no human can intuit this arithmetic (§5).

## Query path

**Baseline QPS**: 200 dashboards × 25 panels ÷ 30 s refresh ≈ **167 QPS** of range
queries, plus interactive ad-hoc (~30 QPS) + rule evaluation 30K ÷ 30 s ≈ **1,000
evals/s** + recording rules (**167/s**). Sustained query load ~200 QPS (dashboard
refresh + ad-hoc); 5× during an incident when every on-call opens more dashboards.

**The right unit is points touched, not QPS.** A range query of 1,000 steps × selector
matching ~500 series × `rate(5m)` recomputing over lookback ~20 samples/step ≈
500 × 1,000 × 20 = **10M point touches** per panel query. Aggregate:
~200 QPS × avg (order 10M) ≈ **order 10^9 point touches/s** — all against in-memory
head + page cache; this is the real load and why selectors and lookback windows (§6)
are what you tune, not query count. Recording rules pre-aggregate the top offenders:
a 5,000-series histogram percentile computed once per 30 s and reused by 50 panels
turns 50 × 10M points into ~5K × 1 point.

**Alert evaluation budget**: 1,000 evals/s ÷ 32 cores ≈ 30/s/core — small instant
queries (selector < 1K series); the budget is a few seconds *per batch* because an eval
batch that overruns its period delays every rule behind it (monotone degradation: alerts
arrive late, nothing corrupts).

**Alertmanager volume**: firing rate ~1% of rules sustained + incident bursts →
~300–3,000 active alerts to dedup/group/silence; routing load is QPS-trivial —
Alertmanager's hard problems are semantics (grouping, inhibition, dedup), not throughput.

## Consistency check (arithmetic must close)

- Ingest: 4M ÷ 15 s = 267K/s; × 86,400 s = 23G/day; × 1.8 B = 41.5 GB/day — matches the
  storage line.
- Scrape fanout: 10,000 ÷ 15 s = 667/s; × 8 KB gz = 5.3 MB/s — matches scrape line.
- 30-day disk at 60 GB/day = 1.8 TB ≈ the quoted 2 TB per server; HA pair ×2 = 4 TB.
- Head RAM 4M × 3 KB = 12 GB — fits the 32 GB figure with query headroom, and explains
  why *active* (not historical) series governs RAM while *distinct* series governs disk.

## Where the design bends

- Cut the interval to 5 s fleet-wide: samples 3× → 800K/s ingest, ~180 GB/day, 5.4 TB
  per server per 30 d, head RAM unchanged. Cost is disk and WAL I/O, not RAM — a
  targeted 5 s only on the 500 latency-sensitive targets is the rational version
  (500 × 400 series × (1/5 − 1/15) = ~27K/s extra samples — ~10% of ingest, trivial
  against the 3× disk effect).
- Fleet 10× (30,000 hosts, 40M series): exceeds what one server serves (public
  operator-reported degradation at tens of millions) — vertical federation and shard
  boundaries become mandatory, and the remote store stops being optional (§4). This is
  the deliberate pain Prometheus accepted, not an accident.
- Add 1 s resolution for synthetic probes: ×15 sample volume for that class; at ~1,000
  probe targets × 50 series that is only +50K/s — the lesson is that cadence policy per
  target class, not a global knob, is the cheapest lever.
