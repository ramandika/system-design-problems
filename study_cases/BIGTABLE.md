# BigTable (wide-column stores & the LSM engine) across the case catalog

Where a Bigtable-class wide-column store or the LSM storage engine appears in the 50 study
cases, what role it plays, and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md),
which names the wide-column store as one of the four storage pillars in the "system-of-record
choice" union.

Headline: **Bigtable appears mostly as a borrowed engine, not a borrowed product.** Only one
case studies Bigtable itself. Everywhere else the case borrows one of two pieces: the
*wide-column data model* (one sorted row key, sparse wide rows, point reads + prefix scans,
one partition per domain key) or the *LSM write path* (append to WAL + memtable, flush to
immutable sorted files, reclaim later by background compaction). Several cases invoke the LSM
shape by analogy without designing the engine at all.

- Cases that mention Bigtable-class stores or the LSM engine: **12 / 50**
- Wide-column store is a designed architectural component: **5** (05, 34, 41, 28, and 12 = the case itself)
- Validated real-world store anchor (load-bearing history, not a component): **2** (01, 30)
- LSM explicitly evaluated and rejected: **1** (49)
- Analogy / by-reference only: **3** (11, 03, 48)
- Brief mention only: **1** (27)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [12.bigtable](12.bigtable/README.md) | The case itself: wide-column store + LSM engine modeled end-to-end | Full case |
| [05.discord](05.discord/README.md) | Wide-column message store, partition = channel + time bucket | Core component |
| [34.reddit](34.reddit/README.md) | Cassandra side-store: comment tree by materialized path | Core component |
| [41.web-crawler](41.web-crawler/README.md) | bigtable-style URL/content table (a KV workload with range scans) | Moderate (by-reference) |
| [28.uber](28.uber/README.md) | Trip DB: "Case-12-style wide-column or plain relational" | Brief |
| [01.youtube](01.youtube/README.md) | Real YouTube metadata: Bigtable warehouse vs Vitess operational store | Anchor (deep in dive) |
| [30.facebook-messenger](30.facebook-messenger/README.md) | HBase → MyRocks migration as a write-amplification anchor | Anchor (historical) |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | General LSM (Option A) evaluated and rejected for the TSDB engine | Deep (rejected) |
| [11.zookeeper](11.zookeeper/README.md) | Metadata-split and consensus/serving-line comparisons; HBase named user | By-reference |
| [03.whatsapp](03.whatsapp/README.md) | Discord's wide-column store quoted as the relay-side comparable | By-reference |
| [48.google-search](48.google-search/README.md) | "Updates are the enemy of a read-optimized index" ≈ LSM compaction | Analogy only |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Related-case row only: a store it buys, not one it teaches | Brief |

## Per-case notes

### The case itself

**[12.bigtable](12.bigtable/README.md)** — Distributed wide-column store: row key → column
families → timestamped versions; tablets (~50 per server) as the split/merge/move unit; Chubby
for consensus and tablet location; SSTables in a replicated shared store. The engine is a live
tension, not a choice: write path is write-ahead log → memtable → ack, frozen memtables flush
into immutable sorted SSTables, compaction reconciles later.
Key decisions in [04-deep-dives.md](12.bigtable/04-deep-dives.md): Option B-LSM wins write
latency independent of data size, loses read amplification (point read = memtable + up to
~10–15 SSTables, cut to ~2 expected via a per-SSTable bloom filter, ~10 bits/key ≈ 1% false
positives); compaction tiers (minor/merging/major) with the failure modes of compaction
pileup (backpressure past 2 frozen memtables) and log-recovery overhead; size-tiered-leaning
vs leveled; locality groups split hot from cold columns into separate SSTable files; and row
keys: monotonically increasing keys hot-spot onto one server, fixed only in the schema
(reverse-timestamp rows). Explicit trade-off in the [README](12.bigtable/README.md): the LSM's
flat write latency is bought by refusing point reads' cheapness — that refusal is the case.

### Wide-column store as the system of record

These cases deploy a Bigtable-class store or the engine's shape as a designed component. The
recurring contract: one partition per domain key, point reads and bounded prefix scans on the
write-heavy plane, with something else (Postgres, a warehouse, object storage) holding truth
or cold data.

- **[05.discord](05.discord/README.md)** — The widest use: a wide-column message store sized
  at trillions of rows. [Deep dive](05.discord/04-deep-dives.md) picks partition key =
  `channel_id` + static time bucket over bare `channel_id` (Option A loses: a ten-year-old
  public guild funnels billions of rows into one partition until its compaction/repair degrades
  the whole cluster; Option B wins bounded buckets, loses a routing layer and occasional
  cross-bucket reads, both absorbed by the data-services shim). Publicly anchored to Discord's
  Cassandra→ScyllaDB store (~2022, p99 inserts ~5 ms, historical reads ~15 ms, caveat-labeled);
  snowflake IDs supply per-channel ordering, retention is infinite-with-tiering keyed by bucket.
- **[34.reddit](34.reddit/README.md)** — Cassandra as a derived side-store, not the truth:
  Postgres adjacency rows stay the integrity source; the Cassandra tree holds comment rows
  keyed by materialized path (`path LIKE 't3_abc,t1_x,%'` = one prefix scan per render), written
  first to Postgres and reconciled asynchronously. Chosen because hot threads read ~100:1 vs
  write and the path layout buys the dominant read as a single scan — the same trade the case's
  real-world counterpart made when Reddit moved comments into Cassandra, publicly (cited and
  caveat-labeled).
- **[41.web-crawler](41.web-crawler/README.md)** — Explicit by-reference: "a key-value workload
  with range scans by host — the [12.bigtable access pattern](12.bigtable/03-architecture.md)".
  One row per canonical URL in a bigtable-style table (status, next attempt, priority tier,
  signature pointers), written by fetcher/extractor, read by the scheduler's point lookups; the
  case deliberately does not re-derive the design, it cross-links it.
- **[28.uber](28.uber/README.md)** — Sizing shorthand for the Trip DB: 20 GB/day of trips,
  quorum-replicated "[Case-12-style wide-column or plain relational](28.uber/03-architecture.md)"
  by city+time, system of record for "reconstruct trip" — the alternative is stated in the same
  breath, so it is a component with an explicit cousin, not a committed engine.

### Validated real-world anchors (history in place of a build)

- **[01.youtube](01.youtube/README.md)** — Uses Bigtable to settle a design question rather
  than as a component: [deep dive](01.youtube/04-deep-dives.md) checks YouTube's two separate
  planes against published sources — the request-time watch page on Vitess (sharded MySQL),
  the dimensional metadata warehouse on Bigtable (1:1 entity-to-table, one protobuf row per
  entity; Google Cloud blog, Aug 2023, by the system's engineers). Lesson quoted as the case's
  own trade-off: don't design one store to do both, and Bigtable lands on the *scan/aggregation*
  plane, not the transactional one.
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — Anchor for the engine-choice
  question itself: the Messenger inbox's publicly reported trajectory, HBase (~2013–2016 era)
  → MySQL with the RocksDB-derived MyRocks engine (~2018), is framed in
  [02-scale-estimates.md](30.facebook-messenger/02-scale-estimates.md) and
  [04-deep-dives.md](30.facebook-messenger/04-deep-dives.md) as a write-amplification fight on
  a ~70 PB/year small-row corpus — compaction characteristics of the storage engine decide what
  maintaining per-user mailboxes and counters costs, not the schema. Cross-linked to Discord's
  Cassandra→ScyllaDB migration as the same lesson at a different scale. No LSM engine is
  designed for this case; the case is MySQL-and-log-first, and buys its history.

### Explicitly rejected

**[49.monitoring-metrics](49.monitoring-metrics/README.md)** — The TSDB case's Option A is a
general LSM engine, named as such: "Bigtable-style MemTable/SSTable, tombstones, merge-on-read"
([04-deep-dives.md](49.monitoring-metrics/04-deep-dives.md)). It loses because the scarce budget
is the read path (every selector query already pays posting intersections; k-file merges per
chunk make p95 panel latency unownable), and because tombstone machinery is dead weight when
retention deletes whole blocks by time range. The chosen purpose-built engine is framed as
*Bigtable's SSTable layering with the tombstone machinery deleted*: immutable sorted chunks,
but "compaction" is a time-ordered block merge, not key reconciliation — legal only because
pull imposes the arrival clock. [01-requirements.md](49.monitoring-metrics/01-requirements.md)
claims the same family explicitly ("same family as Bigtable's LSM trade — cheap writes,
compaction pays"). The sharpest boundary in the catalog on this side too: when the write stream
is monotone by construction, the LSM's general-purpose read-merge machinery becomes cost without
benefit.

### Analogy / by-reference

- **[11.zookeeper](11.zookeeper/README.md)** — Bigtable is the recurring comparator, never a
  component: the consensus-vs-serving split (tablet servers are non-consensus; Chubby-class
  systems hold metadata), the metadata/data boundary as the reason Znodes stay KB-scale, and
  the metadata-location move ("kept Chubby-tier coordination but pushed hot tablet locations
  into its own replicated table because Chubby QPS could not survive hot-path traffic") — the
  same lesson KIP-500/KRaft drew from the other side. HBase appears only in the lineage list of
  systems ZooKeeper actually coordinated. The point of each comparison: separate what needs
  consensus from what needs throughput.
- **[03.whatsapp](03.whatsapp/README.md)** — The relay case does not choose a store; it cites
  Discord's wide-column store ([04-deep-dives.md](03.whatsapp/04-deep-dives.md) §3, Cassandra
  177 nodes → ScyllaDB 72, May 2022) as the best-documented comparable for per-conversation
  partition design and quorum reads/writes, with the explicit caveat that the latency numbers
  are Discord's own, not an independent benchmark. By-reference store design, borrowed figures.
- **[48.google-search](48.google-search/README.md)** — The one-line LSM analogy: rejecting
  in-place updates to the base index is "the same shape as LSM compaction in
  [12.bigtable](12.bigtable/03-architecture.md)" — updates are the enemy of a read-optimized
  index, so the index stays immutable and a small fresh tier merges at query time. No LSM
  engine, only the shared argument (immut buys compression and merge economics, reads pay).
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — One row in
  [05-related-cases.md](27.ev-charging-network/05-related-cases.md): borrowed append-before-ack
  ingest and staleness-bound reads; "a storage engine this case buys, not one it teaches".

## The portable pattern

Every substantive use reduces to one contract, cross-linked to
[12.bigtable](12.bigtable/README.md) as the reference implementation:

1. **Write-optimized append, background reclamation.** Writes land at flat latency (WAL →
   memtable → ack); immutable sorted files accumulate (12, 49, 05); compaction pays the
   deferred bill. The one rule never to break in the corpus: never let reclamation run on the
   interactive path.
2. **Point lookups vs range scans decide the data model as much as the engine.** One row per
   canonical URL read by point lookups (41), comment trees read by prefix scan (34), messages
   read as "everything after X" per channel (05), warehouse rows read by aggregation (01) —
   the row key encodes the query, and a monotone key hot-spots (12, 05's bucket fix, 49's
   time-partitioned blocks is the same fix in engine form).
3. **Compaction (and its cost dial) is where the engine choice is really made.** Two cases
   turn the dial and they turn it in opposite directions: 12 keeps general-purpose key
   reconciliation because the workload mixes keys; 49 deletes it because retention deletes
   whole blocks by time, and Discord/Reddit cap the dial's blast radius by bounding partition
   size (time buckets, per-post trees). The dial's cost is read amplification; its control
   is partition shape.
4. **Bounded beats unbounded every time.** Every component use re-solves 12's tablet split the
   same way: channel + time bucket (05), per-post (34), host-keyed rows (41) — a partition that
   grows without bound eventually makes compaction a whole-cluster event (05's Option A
   failure mode), and at trillion-row scale the bound is not optional.
5. **The wide-column store is the write-heavy plane; truth and cold data live elsewhere.**
   Postgres adjacency for integrity (34), Vitess for transactions on YouTube (01), object
   storage for cold buckets (05) — the store answers the scan, not the transaction.
