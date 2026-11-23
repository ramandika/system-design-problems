# 02 — Scale estimates

Design target: a web-index-like table at PB scale. Public anchor from the Bigtable paper
(OSDI 2006): Bigtable is designed for "petabytes of data across thousands of commodity
servers", and its largest reported production table is a crawl/web-index table of ~800 TB
(Table 2, ~10^12 cells). Our 1 PB target is the same order of magnitude, heavier per row
(estimate). All derived numbers are rough estimates shown with arithmetic; public anchors
are marked.

Assumptions:

- Table size: **~1 PB** compressed, **~10 billion rows** → ~100 KB average compressed row
  (content + anchors + a few versions per cell) — sanity check: 10^10 × 100 KB = 10^15 B
  = 1 PB. Consistent.
- Write load: 5B cell updates/day ≈ **58K writes/s average**, peak ≈ 3–5× → **175–290K/s**.
- Point reads: 20B/day ≈ **230K reads/s average**, peak 4× → **~925K/s**. Read:write ≈ 4:1.
- Scans: ~200 table-equivalent scans/day from batch jobs; a full scan reads at most 1 TB
  of *hot* data after locality pruning (most scans cover a key range, not the table).
- Value sizes: ~1 KB median cell, ~10 KB average *delta written* per update (an update
  touches a few cells, not a whole 100 KB row).

## Write path

**Logical writes**: 58K/s × 10 KB/update = ~580 MB/s of new data ≈ **~50 TB/day raw**,
~17 TB/day after ~3x compression (50/3 ≈ 17) — consistent with a long-lived PB-scale table where old
versions are GC'd (retention keeps 2–3 versions).

**Physical writes (write amplification)**: every logical byte is written once to the
commit log, once at MemTable flush, and rewritten at each compaction level it survives.
For a size-tiered cascade the multiplier is ~O(log of span) — estimate **~10–15×** →
17 TB/day × ~12 ≈ **~200 TB/day physical disk writes** ≈ 2.3 GB/s sustained, spread over
~1,000 tablet servers ≈ ~2.3 MB/s each: trivial in bandwidth, but the *CPU and read-path
interference* of compaction is the real cost (deep dive 2). Space amplification before
major compaction runs ≈ 2–4× the compressed table (~2–4 PB transient) — the trade for
never rewriting rows in place.

## Read path

**Point reads**: 230K/s average, ~925K/s peak. Each read must check MemTable + up to
~10–15 SSTables in the worst case. Per-SSTable bloom filter with ~10 bits/key
(textbook: m/n = 10, k ≈ 7 hashes → ~1% false positive) cuts expected SSTables touched
from ~8 to ~1–2. The paper publicly reports that bloom filters "drastically reduce the
number of disk seeks required for read operations" — directionally consistent with this
arithmetic, though the paper gives no factor (the 8→~2 cut is our estimate).

**Read amplification budget**: to hold ~1M QPS peak with ~2 SSTable files touched on
average → ~2M file-layer lookups/s fleet-wide, ~2K lookups/s per tablet server —
this is why per-server read serving MUST be masterless: any coordinator lookup in the
hot path would add its own load first (deep dive 4).

**Scans**: 1 TB over ~1,000 servers = ~1 GB/server/scan; at 100 MB/s per server that's
~10 s per scan job — scans are embarrassingly parallel and never contend with point
reads for latency SLO, only for disk I/O (mitigation: scans served with lower
SSTable-cache priority; locality groups keep hot families on separate files).

## Storage & tablets

- Tablet sizing choice: **~10–50 GB compressed per tablet** (the paper's *default* split
  size is 100–200 MB — for a PB table that would mean ~5–10M tablets; the paper reports
  production clusters with "somewhere between ten to a thousand tablets per tablet
  server", i.e. GB-scale tablets for the multi-hundred-TB tables). 1 PB ÷ 20 GB ≈
  **~50K tablets**.
- Tablet servers: ~1,000 (the paper's clusters run "from a handful to thousands of
  servers"; its scaling benchmark used up to 500) → ~50 tablets/server ≈ ~1 TB/server.
  With a split threshold of ~2× target size, one load-balancing move relocates ~2–5% of
  a server's data (50 tablets/server).
- Metadata: three-level location hierarchy (Chubby file → root tablet → METADATA tablets;
  each METADATA row ≈ ~1 KB in memory, per the paper) → 50K tablets × ~1 KB ≈ ~50 MB of
  METADATA → fits in memory. A cold client pays 3 round-trips including one Chubby read,
  1–2 hops after the cache warms.
- Commit log: 50 TB/day raw = 580 MB/s average into replicated distributed storage;
  one log per tablet server (not per tablet) amortizes seeks — at ~50 active tablets
  writing, a per-tablet log would thrash; per-server log costs log-surface reads during
  recovery (deep dive 2, failure modes).

## Consistency check

- 1 PB ÷ 10^10 rows = 100 KB/row; 10 KB average delta × 5B updates/day = 50 TB/day;
  a 2–3-version retention of that rate reaches ~PB scale in months of accumulation —
  internally consistent with the 1 PB steady state once old versions age out.
- Read:write = 20B:5B = 4:1, matches the requirement table's 1:4 (reads:writes).
- 50 tablets/server × ~1,000 servers = 50K tablets ≈ 1 PB ÷ 20 GB. Consistent.
- Peak point reads 925K/s × ~2 files touched = 1.85M/s ≈ 2M/s claimed above. Consistent.

## Calibration against public figures (Bigtable paper, OSDI 2006)

- Table size: the paper targets "petabytes of data across thousands of commodity
  servers"; its largest reported production table is the crawl/web-index table at ~800 TB
  with ~10^12 cells (Table 2). Our design target — 10^10 rows × ~100 KB compressed =
  1 PB — is the same order, heavier per row (estimate).
- Serving load: the paper reports 14 busy production clusters (8,069 tablet servers)
  sustaining more than 1.2M requests/s aggregate (§8) — our ~925K/s fleet peak ≈ ~1K
  reads/s per server, which the paper's *uncached disk* benchmark already ran per server
  (random reads of 1000-byte values: ~1,212 ops/s per tablet server, each read pulling a
  full 64 KB block from GFS at ~79 MB/s (= 1,212 × 64 KB); ~10,800 ops/s per server from memory). The gap
  between the ~1,200/s uncached number and a cached serving reality is exactly what bloom
  filters, block-cache sizing and hot-data locality have to buy (flagged as estimate, not
  measurement).
- Chubby: 5 replicas (Paxos quorum of 3); the paper's lessons section names Chubby a
  single point of failure — "if Chubby becomes unavailable for an extended period of
  time, Bigtable becomes unavailable" — and measures 0.0047% of Bigtable server-hours
  lost to Chubby unavailability across 14 clusters (0.0326% for the worst one). The
  reason deep dive 3 exists.
