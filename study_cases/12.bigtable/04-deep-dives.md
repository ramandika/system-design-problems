# 04 — Deep dives

## 1. Tablet sharding & row-key design: hot-spotting on monotonically increasing keys

The table is split into tablets by row-key range (not hash buckets). That buys **range
scans are sequential** and locality-aware scans; it costs you the classic failure: keys
that grow monotonically (auto-increment IDs, timestamps, "user_id = next sequential")
put 100% of new writes on the single tablet that owns the last shard — one server eats
the entire write rate (290K/s peak, per [02](02-scale-estimates.md)) while its neighbors
idle and split/balance machinery can't relieve it because the *insert point never moves*.

**Option A — hash-shard row keys** (consistent-hash or modulo):
- wins: writes spread perfectly, hot-spotting impossible.
- loses: range scans die (a scan over "yesterday's keys" = fan out to all tablets + merge),
  locality is gone, sorted order is destroyed — and sorted order IS the product here
  (the row key is the query).

**Option B — keep sorted ranges, redesign the key** (Bigtable's actual answer, and it's a
schema responsibility, not engine code):
- *Reverse the natural sort*: webtable stores URLs as `com.cnn.www/index.html` — pages of
  one host become contiguous (locality for host-level scans) and no global "now" edge
  accumulates all writes.
- *Reverse timestamps inside the key* (`<key>#<max_ts − ts>`): newest version sorts first,
  so "latest" reads/scans touch one small locality, and old data lands *behind* (in the
  direction already written), not on a new edge.
- *Salt the prefix* (`<hash(user_id) 2 bits>#<user_id>`): spreads writes but makes any
  scan over a full user base a 4-way merge with worst-case skew — bounded cost, chose a
  small salt space deliberately.

**Chosen: B.** Option A wins even-write distribution but loses the access pattern the
schema exists for; Bigtable's answer is that the *application's key design* carries the
load-spreading job. The engine does not silently re-shard your hot key — it splits the
tablet (deep dive 3), which relieves size but not the insert-point problem, since the
right edge still absorbs every new row and immediately re-bloats one child.

- Trade-off made explicit: we accept that key design is a per-application obligation with
  no engine-level cure, in exchange for keeping sorted-range locality and O(1)-tablet
  scans. A system that wanted engine-enforced balancing would pay with range scans
  (that's roughly the LSM-with-hash-partitioning family, e.g. Dynamo-style stores).

Failure modes: (i) a batch job writes a sequential key range → one tablet, one server,
compaction queue backs up there (mitigation: pre-split the range on create using the
known key domain — e.g., split dates into N ranges at table creation); (ii) celebrity-key
analog: a handful of very hot row keys (one user, one product) — same physics as Twitter's
celebrity follower skew, where a tiny fraction of keys carries most of the load — see
[Twitter deep dives](../02.twitter/04-deep-dives.md) §1 for how a *service* layer
bypasses materialization for hot authors; an analogous Bigtable-side tool is a row
cache or read-replica for those keys, not re-sharding.

## 2. LSM-tree writes: MemTable, SSTable, compaction and write amplification

Write path per cell: (1) append to commit log (WAL, on distributed replicated storage,
per-server not per-tablet) → (2) insert into MemTable → ack. When MemTable hits a size
threshold (100 MB here, a rough estimate — the paper specifies only "a size threshold"), it freezes; a background
thread writes it out as an SSTable (immutable, sorted, 64 KB blocks) while a fresh
MemTable takes over; each server has two MemTables so writes continue during flush.

**Option A — mutate in place (B-tree-style)**:
- wins: point reads touch one structure; read amplification ≈ 1.
- loses: a write becomes read-modify-write (page read, page write, index update), so write
  latency depends on data size and random-I/O; large sparse rows (wide-column! one row,
  thousands of columns) are pathological to keep balanced.

**Option B — LSM, append-only**:
- wins: writes are two cheap appends; write latency is independent of table size (the
  requirement the whole case leans on); immutable SSTables are cheap to replicate,
  checkpoint, and scan.
- loses: **read amplification** (a point read must merge MemTable + k SSTables, newest
  version wins) and **write amplification** (compaction re-reads and re-writes every byte,
  estimated ~10–15× total physical writes — see [02](02-scale-estimates.md)).

**Chosen: B** — write-heavy at 4:1 reads:writes, and write latency must be flat; the lost
property (read cost) is paid back with bloom filters (deep dive 6), a block cache, and
compaction layering.

Compaction design (and where the money actually goes):
- **Minor compactions**: each frozen MemTable becomes one SSTable (the paper's
  terminology) — frequent, cheap; it caps MemTable memory but, unchecked, neither bounds
  the SSTable count nor removes obsolete data.
- **Merging compactions**: rewrite a few recent SSTables + absorb the active MemTable
  (the paper's bound on file count); deletes ("mutation tombstones") and stale versions
  are dropped only at *major* compaction — hence the space amplification (2–4× transient)
  quantified in [02](02-scale-estimates.md).
- **Size-tiered vs leveled trade-off**: Bigtable's scheme is size-tiered-leaning (fewer,
  bigger merges). Leveled LSM (each level rewritten fully) buys near-O(1) read amp with
  much worse write amp — the opposite side of the same coin. We pay ~10–15 file reads
  worst-case per point read to avoid rewriting the whole table per level.

Failure modes: (i) **compaction pileup under write burst** — MemTable freezes arrive
faster than flushes finish → writes stall (the one place a Bigtable write is NOT flat);
mitigation: backpressure on write rate when >2 frozen MemTables; (ii) **log-recovery
cost with per-server logs**: a crash replays the whole server log to rebuild *all* its
tablets' MemTables — that's the price of one-log-per-server; (iii) major-compaction of a
hot tablet steals I/O and CPU from reads → schedule majors off-peak or per-locality-group
(hot family gets its own files and compaction schedule).

## 3. Cluster coordination: Chubby, and why the coordinator is not in the data path

Chubby is a Paxos-replicated (5 replicas, quorum 3) lock/file service. Bigtable uses it
for exactly four things: (a) ensure there is ONE master (master lease file); (b) store
the bootstrap location of the root tablet (the single row Bigtable's location is known);
(c) per-tablet-server liveness locks (a server serves only while it holds its lock);
(d) schema and ACLs' authoritative home.

**Data path is coordinator-free.** A client, after one-time discovery
(root → METADATA → target server, cached locally), talks to tablet servers directly.
Even the tablet-server→master gossip is advisory: a server keeps serving its tablets
while its Chubby lock lives, and the *master's* death stops rebalancing, not reads.

**Option A — coordinator in every request** (route/lock/lease per op):
- wins: strong global consistency of membership, trivial to reason about.
- loses: coordinator availability = database availability; coordinator QPS scales with
  data QPS — at ~1M peak reads/s the lock service becomes the whole system (compare the
  60K-QPS-into-a-service fan-out critique in Twitter deep dives §1 — the same "don't put
  the hot path through one shared component" lesson).

**Option B — coordinator for control plane only** (chosen):
- wins: Chubby load is negligible in steady state; data plane survives Chubby brownouts
  as long as leases renew; the master can die for minutes with zero write/read impact.
- loses: membership changes propagate at gossip/lease speed (a dead tablet server's
  tablets are stuck for the lease duration before the master reassigns them), and the
  system has a hard external single point of *config* failure.

**Consequence (publicly reported)**: Chubby is an availability single point of failure —
every tablet-server restart needs its lock and every cold client needs the root tablet
pointer, so "if Chubby becomes unavailable for an extended period of time, Bigtable
becomes unavailable" (paper, §5); the paper measures 0.0047% of Bigtable server-hours
lost to Chubby unavailability across 14 clusters (0.0326% for the worst). This pattern
(dedicated
coordination ensemble for membership/leases, off the data path) is what ZooKeeper
generalized as a standalone service; the same "off the hot path" critique also shows up
in Twitter's timeline design, where putting the hot read route through one shared
component is [Twitter deep dives §1](../02.twitter/04-deep-dives.md)'s central trade.

Failure modes: (i) lock expiry during a long GC pause on a tablet server → it stops
serving although it's alive; on lock loss its master reassigns tablets elsewhere — the
two-stop-serving rule (server stops on lock loss, master stops trusting on expiry) is
what prevents split-brain serving of one tablet; (ii) master election storms (frequent
Chubby flaps) → assign/unassign churn; mitigate with master hysteresis before
reassigning; (iii) METADATA tablet overload: every cold client boot walks root →
METADATA; keep the root tablet and first METADATA tablet's owning servers over-provisioned.

## 4. Master, tablet split/merge and load balancing

**Assignment**: master holds the live map tablet→server. A tablet is "assigned" exactly
once: unload on A, load (create MemTable, recover from log surface, read METADATA) on B,
write the METADATA row last. The master's only durable state is in Chubby + METADATA —
it rebuilds everything else by scanning METADATA at startup.

**Split** (only, in v1: no user-requested splits): when a tablet exceeds its size
threshold, server proposes a split point (middle of the size, not middle of the key
range — aim for even bytes); the owning tablet server commits the split by writing one
new METADATA row and notifies the master — if the notification is lost, the master
rediscovers the split the next time it asks for that tablet to be loaded (per the paper).
No data is moved: SSTables are files in shared storage, so a split is metadata + a new
server loading the files. Split is lazy — the children are "virtual" until one is
actually loaded/written.

**Merge**: unloaded, small, adjacent tablets get merged by deletion from METADATA and
leaving the range to a neighbor — an opportunistic, master-side GC of over-split history.

**Load balancing** is driven by per-server load gossip (tablet count, RPS, disk bytes,
Compaction debt) — not tablet count alone:
- **Option A — balance by tablet count**: simple, wrong: a hot 50 GB tablet and an idle
  1 GB tablet weigh the same; count-balancing would park both on the same server.
- **Option B — balance by measured load** (chosen): wins correctness of placement, loses
  simplicity of reasoning (need the gossip pipeline and hysteresis to avoid thrash),
  and hot-tablet relief is bounded by split: a single hot tablet is ONE unit of
  scheduling — it cannot be partially moved; only a split can relieve it, and a split
  only helps if the heat is spread over the key range (deep dive 1's insert-point case
  is NOT relievable by split — different failure).

- Trade-off made explicit: virtual splits and shared-storage files win "moves are O(1)
  bytes" (metadata-only) but lose "splits are free": every split adds tablet count that
  later costs gossip, METADATA rows, and compaction scheduling (hence merge exists).

Failure modes: (i) **split storm** — a tablet that bloats because of monotonic keys
splits, and its right child instantly re-bloats → repeated splits in minutes, tablet
count up, merge pressure up; mitigation: split hysteresis + pre-splitting at create
time; (ii) assignment backlog after a server death — up to ~50 tablets become one
server's-worth of hot load; the master throttles reassignment to keep log-recovery
(replay) from stampeding the distributed store; (iii) master death during move: METADATA
write is the commit point, so a crashed master leaves either "assigned-but-not-recorded"
or "recorded-but-unloaded" — the load step reconciles by checking Chubby locks:
unassigned tablets (no server holds them) are reassigned on the next master scan.

## 5. Replication between clusters and the eventual-consistency bill

v1 Bigtable ships one cluster per table instance; replication to other clusters is
asynchronous (the paper describes Personalized Search data replicated across several
Bigtable clusters with eventual consistency via a built-in replication subsystem) and
*last-write-wins by timestamp* on merge (estimate — the paper names no conflict-resolution
rule).

**Option A — synchronous multi-cluster replication (quorum writes)**:
- wins: losing one cluster does not lose acknowledged writes; automatic failover with
  RPO ≈ 0.
- loses: every write pays cross-region latency (writes are the hot path here — that's the
  whole LSM point), and availability now requires quorum interconnect health — the
  single-master-append design's "ack after local log" property dies.

**Option B — asynchronous, eventual replicas** (chosen v1):
- wins: write latency stays local and flat; failover is a client config change (replicas
  are independent Bigtable clusters, so a replica serves reads immediately).
- loses: RPO > 0 — an upstream cluster's last seconds of writes (acknowledged ones!) can
  be lost on failover; conflict resolution is last-timestamp-wins, so a clock-skewed
  writer can resurrect old data — hence the paper's own recommendation to prefer
  application-supplied versions (or rely on write-once keys) rather than server clocks.

- Trade-off made explicit: we buy the LSM write path's flat-latency property by refusing
  to make writes cross-region; in exchange we accept a failover story with holes.
  This is the same call Discord's wide-column store made in the opposite direction —
  quorum reads AND writes at p99 5ms writes (their numbers, reported in
  [WhatsApp deep dives §3](../03.whatsapp/04-deep-dives.md)) — proving which side of
  the trade you pick is a function of the money you'll spend on hardware and the RPO
  your product can absorb, not a universal best.

Failure modes: (i) **replication lag invisibility** — a lagging replica silently serves
older data until lag exceeds the client-visible threshold; mitigation: lag as a
first-class metric with alerting, and per-request `max_staleness` on reads if the
schema supports it (later-generation feature); (ii) **timestamp collision on merge** —
client-supplied timestamps collide across clusters and lose data; mitigation: monotonic
per-writer timestamps or app-encoded versions in the key itself; (iii) **failover
split-brain after upstream recovery** — stale upstream still accepts writes while
traffic moved; mitigation: operator-run fencing (upstream made read-only before
failover), not automatic.

## 6. Bloom filters and read amplification; locality groups

A point read must consult: MemTable + every SSTable of the tablet (newest wins, first
hit per cell). Without filtering, worst case is ~10–15 file probes per read — a read
amplification that the LSM choice of deep dive 2 put on the read path deliberately; the
engine must buy it back.

**Per-SSTable bloom filter** (probabilistic membership, in-file, in-memory): answers
"could this key be in this file?" — a "no" skips the file. Space ~10 bits/key with
~7 hash functions → ~1% false positives (textbook arithmetic; the paper publicly reports
that bloom filters "drastically reduce the number of disk seeks required for read
operations" — directional confirmation, no factor given). The trade: 10 bits/key = ~1.25
bytes per indexed key — an estimate of under ~0.2% extra SSTable bytes on disk next to
the ~1 KB median cell (and ~0.001% of a ~100 KB average row), ~1% chance of a wasted
probe. Tunable per table — for scan-heavy tables
blooms are wasted (scans read everything anyway).

**Locality groups** — the schema lever for read amplification: families can be grouped
so each group's columns live in their OWN set of SSTables. A read of "just the anchor
column family" no longer pays I/O for the (much larger) content family. Compression,
bloom filters, and in-memory-ness are set per locality group (the webtable reportedly
kept the small "anchor" family's locality group in RAM — publicly reported).

**Option A — one SSTable stream per tablet**: simpler compaction; every read pays for
every family's bytes.

**Option B — per-locality-group files** (chosen): wins separating hot/small families from
cold/large ones at read time (the wide-column row's own skew makes this huge — a row may
be 100 KB while the hot cell is 200 bytes), loses compaction- and file-count overhead
(~2× files per extra group) and cross-family scans now touch multiple files.

- Trade-off made explicit: the LSM gave cheap writes by fragmenting rows across k files;
  bloom filters and locality groups are the two levers that decide whether k is 2 or 15
  at read time. Bloom filters remove *files that can't have the key*; locality groups
  remove *files that can't have the column family*. They attack two different halves of
  the same read amplification.

Failure modes: (i) **bloom false positives under hot keys** — 1% FP on a very hot read
pattern still means ~1% of reads do a full file probe and miss; mitigation is not more
bits but the *block cache* (the probe's cost is mostly cached pages for hot files);
(ii) **stale bloom after major compaction** — a bloom built over data that was compacted
away says "maybe" longer than needed; harmless (extra probe) but it's why blooms are
rebuilt, not patched; (iii) **locality-group fragmentation with many groups** — each extra
group multiplies SSTable count and compaction debt; keep it to 2–3 groups per table,
and only split when the families' access really diverges.
