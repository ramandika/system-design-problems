# 02 — Scale estimates

Back-of-envelope for one backbone cluster. All figures rough estimates; adjust the
assumptions and redo the arithmetic. Real Kafka deployments publicly report clusters in
the hundreds of thousands of msgs/s per broker and multi-million msgs/s per cluster
(e.g. LinkedIn's original 2009–2011 writeups and later conference talks) — numbers below
stay an order of magnitude inside that envelope.

## Assumptions (state them, everything else follows)

- Aggregate ingest: **1M msgs/s average**, peak 2M/s (daily-traffic shape of a large
  tech company: bursts at business hours, batch backfills).
- Average record size: **1 KB** (payload + key + headers + framing).
- 500 topics; hottest topic carries ~10% of traffic; it has **120 partitions**.
- Replication factor **R = 3**; production topics `min.insync.replicas = 2`.
- **Producers**: ~200 services, ~50K producer client instances (sidecars, pods).
- Consumers: 3 consumer groups average per topic (2 for most, heavy fan-out on the
  clickstream topic with ~10 groups). Partition-assignments: only the hottest topics get
  120 partitions; even the ceiling of 500 topics × 120 × 3 groups ≈ **180K assignments**
  (realistic count lower — an estimate) — simplify: read amplification **×3**.

## Write path

**Message rate → bytes on the wire (uncompressed)**:
1M msgs/s × 1 KB = **1 GB/s** raw ingest.

**Compression**: producers compress batches (lz4/zstd; realistic ratio for JSON-ish
telemetry ~3–4×, conservative: 3×):
1 GB/s ÷ 3 ≈ **~330 MB/s compressed ingest** (cluster-wide, per replica stream before R).

**Replicated write traffic**: ×R = 3 → **~1 GB/s** of broker-side write bandwidth.

**Produce request rate**: 1M msgs/s, but producers batch — a batch of ~200 records per
request is a normal target at 4–16 KB+ of data. Request rate:
1M ÷ 200 = **~5K produce requests/s** cluster-wide before replication (followers also
fetch, doubling connection traffic roughly). This is why request COUNT is not the
problem — the problem is bytes and fsync/flush policy, both handled by the log design
(deep dives §2, §6).

**Per broker**: 40 brokers (storage math below) → ~25 MB/s ingest per broker compressed
(×R locally, but partition leaders spread across brokers). Peak 2× → 50 MB/s. Comfortably
inside what a broker with a 10 GbE NIC and page-cache-friendly disks handles; plenty of
headroom by design, because rebalancing and leader elections make traffic lumpy.

## Read path

**Consume volume**: ingest 1M msgs/s × 3 groups ≈ **3M consume msgs/s** ≈ 3 GB/s raw,
~1 GB/s compressed read traffic from partition leaders.

**Zero-copy**: consumer fetches leave the page cache via `sendfile` — no app-layer copy,
no JVM heap. Reads therefore do NOT compete with produce for heap or GC; the practical
capacity constraint is NIC + page-cache hit rate (deep dives §6).

**Replay/backfill**: one warehouse backfill reading 24 h of one mid-size topic:
topic share ~2% of ~29 TB/day stored → 0.6 TB/day ≈ ~7 MB/s extra — noise, BUT a
backfill of the hottest topic's full 7-day window reads stored (compressed) bytes off
disk, not raw: 7 × (0.1 × 29 TB) ≈ **20 TB** ≈ 20e12/86,400 s ≈ **~230 MB/s sustained**
for ~1 day on the readers — ~3× less than the raw-ingest figure would suggest, but still
the "read amplification surprise" to budget for; rate-limit backfills or they look like
the cluster is dying.

## Storage

**Raw data volume**: 1 GB/s × 86,400 s = **86 TB/day raw**; after 3× producer
compression ≈ **~29 TB/day stored per replica copy**.  × R=3 → **~86 TB/day of disk**.

**Retention 7 days**: 86 × 7 ≈ **~600 TB disk** for the hot retention window.

**Brokers**: 600 TB ÷ 16 TB usable per broker ≈ 38 → **40 brokers** (plus a few for
controller/quorum roles; quorum needs only 3–5 small nodes).

**Per-broker disk bandwidth**: 86 TB/day ÷ 40 ≈ 2.2 TB/day/broker ≈ **25 MB/s
sustained writes** — trivial. Disk is not the bottleneck; the log design makes disk
random-access (which IS the bottleneck in classic MQ designs) disappear.

**Small-message pathology (the anti-check)**: if average message size drops to 100 B
(thousands of topics of tiny events) with the same 1M msgs/s, bytes drop 10× but
record-rate doesn't — the 5K requests/s becomes 5K/s of metadata-heavy work and batch
compression barely compresses (headers dominate). The same cluster then needs batching
aggressiveness (linger ms, batch size) tuned UP, and max.request.size / fetch sizes
tuned DOWN. Message-size distribution, not msg/s, decides tuning.

## Consistency check (arithmetic must close)

- Ingest bytes: 1M/s × 1 KB = 1 GB/s; × 86,400 s ≈ 86 TB/day — matches raw storage line.
- Compressed stored: 86 ÷ 3 ≈ 29 TB/day; ×3 replicas = 86 TB/day; ×7 days = 605 TB —
  the 600 TB disk figure stands.
- Consumer amplification: 3M msgs/s consumed ÷ 1M msgs/s produced = 3× — matches the
  "3 groups" assumption.
- 40 brokers × 16 TB = 640 TB ≥ 600 TB with ~6% headroom — thin; realistically 45+
  brokers for skew and rebuild headroom (a single broker failure forces R-1 = 2 replicas
  to re-replicate 600/40 ≈ 15 TB of data; at 1 GB/s per-follower fetch cap that is
  ~4.2 hours of re-replication pressure — budget for it, or use shorter retention on
  low-value topics).

## Where the design bends

- Bump avg message to 4 KB (Avro/Parquet-adjacent) at the same 1M msgs/s and improve
  compression to 4×: raw ingest quadruples to 4 GB/s → 4 × 86.4 = 345.6 TB/day raw,
  ÷ 4 compression = ~86 TB/day stored per replica copy, ×3 = ~259 TB/day → ~1.8 PB
  of 7-day disk, ~3× the 600 TB baseline — a 4× compression win cannot offset a 4×
  message-size growth. Size
  discipline (schema-registry-managed, columnar-ish payloads) is the cheapest
  infrastructure win available; message-size growth is its most expensive regression.
- Raise R to 5 for the critical CDC topic only: +66% disk for that topic, worth it —
  but only on the topic where losing data means losing money.
