# 01 — Requirements

## Functional

Core v1 (the storage contract, not an app feature set):

- **Point read**: given (row key, column family:column, max versions), return the latest
  N versions of the cell. This is THE latency-sensitive path.
- **Range scan**: read a contiguous key range — possibly whole-table — typically consumed
  by MapReduce-style batch jobs, streamed at maximum bandwidth, not low-latency.
- **Write / cell update**: put a new version of a cell. Writes are put-or-delete with a
  timestamp; no multi-row transactions in v1.
- **Schema (thin)**: tables have column families (declared, unit of access control /
  retention / locality), which contain columns (created on the fly by clients, NOT
  declared). Versions kept per cell with configurable GC (keep last k, keep newest t seconds).
- **Table admin**: create/delete tables, per-family retention, locality groups (which
  families are grouped on disk), snapshotting left out of v1 core (real Bigtable grew it
  via the replication/SSTable story).

Out of scope v1, deliberately (and why):

- **Multi-row transactions** — cross-tablet transactions are a different machine
  (Spanner, per Google's own roadmap; the paper found most applications needed only
  single-row transactions, and the main requested use of cross-row ones was maintaining
  secondary indexes — for which a specialized mechanism beats general transactions).
- **Secondary indexes / query by column value** — the model is "everything you need is in
  the row key". A query layer (indexing service) is a separate system.
- **Ad-hoc analytic SQL** — scans exist for batch jobs; interactive analytics is a different
  engine (this corpus treats it in the warehouse sense).
- **Strong multi-cluster consistency** — v1 replicates clusters eventually (see deep dive 5).

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Point-read latency | p99 single-digit ms (local, healthy cluster) | drives bloom filters, locality groups, MemTable/SSTable layering |
| Write latency | ~ms, and **stays flat under load** | append-only path; latency never depends on table size |
| Scan bandwidth | maximize, bounded by aggregate disk/network | scans are bandwidth problems, not latency problems |
| Scale | billions of rows, PB-scale table, ~1,000s of machines | tablets + two-level metadata; no per-table global lock |
| Consistency model | strong within a row; last-write-wins or first-write-wins per cell version; eventual across replica clusters | row-level atomicity is cheap because one tablet owns the row |
| Availability | single-master control plane, but data plane keeps serving if master dies | master is not on the read/write path at all |
| Durability | no acknowledged write lost without operator action | committed writes are in a log on replicated storage (GFS-like) before ack |
| Throughput balance | writes:reads ≈ 1:4 (see estimates) | enough to justify the read-amplification side of the LSM choice |

## Scope decisions that shape everything

1. **Writes are made cheap by making reads pricier** (LSM, inverted vs B-tree). Given the
   write-heavy skewed workload below, Option LSM wins "write latency independent of data
   size", loses "point reads may touch k files" — we pay for the loss with bloom filters
   and compaction (deep dive 2).
2. **One tablet owns a row's data; rows are the unit of atomicity** — no row spans tablets,
   so per-row atomic read-modify-append is free (single machine). Losing: no cross-row
   transactions (see out-of-scope).
3. **Data plane is masterless; control plane is master-based** — tablet servers serve reads
   and writes directly off shared storage with no coordinator in the request path. The
   coordinator (Chubby) is touched at discovery time only. This single decision shapes
   deep dives 3 and 4.
