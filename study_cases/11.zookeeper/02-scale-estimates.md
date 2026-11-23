## Primary datapoints (historical, 2010 paper era)

From the USENIX ATC 2010 ZooKeeper paper (publicly reported): the evaluation machines were
2008-era dual-core servers; read-only workloads reached **tens of thousands of ops/s** on a
3-server ensemble, aggregate. Write throughput was several times lower, and *fell* as
servers were added — each write costs a majority fsync, and reads cost nothing comparable.
The load in production deployments is read-dominated, with writes concentrated in bursts
(deploys, failovers, membership churn). All figures below are estimates anchored to that
paper's shape, re-derived for a modern fleet.

## Fleet setup (the unit of scale is the ensemble, not the user)

ZooKeeper is deployed as a small number of *independent ensembles*, one (or a few) per
production cluster of services — not one global service. Estimate for a big company:

- ~200 production ensembles (one per cluster/region of services)
- each ensemble: 5 voting servers (3 is the budget floor, 5 the norm), plus 0–2 observers
  on read-heavy clusters (see deep dive 5)
- per fleet: 200 × 5 = **~1,000 ZooKeeper servers** total — trivial hardware footprint for
  how much it is trusted.

## Per-ensemble workload (the interesting numbers)

Assumptions (estimates — adjust and redo arithmetic if these change):

- ~1,000 client hosts + services on a big ensemble, each maintaining 1–2 sessions →
  **~2,000 concurrent sessions** per ensemble, ~50K watches and ephemerals combined in
  aggregate across those sessions (estimate).
- Ops mix: **~50,000 ops/s average, 150,000 peak**; ~95% reads, ~5% writes
  (read-dominated is the norm — the ATC'10 paper's benchmark traffic was also read-heavy).
- Average znode: ~200 bytes (config strings, membership JSON, lock sequences).

**Reads served locally.** 95% of 50K = 47,500 reads/s average, ~142,500 peak. With 5
servers all serving reads from memory (deep dive 2 explains why this is safe), that is
~9,500 reads/s/server average, ~28,500 at peak — comfortably inside the
tens-of-1000s/s envelope the 2010 paper demonstrated on two cores; modern hardware is far
beyond it (estimate). The read path is one memory lookup + one RPC — no disk, no lock.

**Writes hit disk on a quorum.** 5% of 50K = 2,500 writes/s average, 7,500 peak.

Arithmetic for worst-case peak: each write is one proposal broadcast to 4 followers plus an
fsync barrier per follower (leader's own log adds a 5th fsync). 7,500 writes/s × 5 fsyncs =
37,500 fsyncs/s requested. Real systems do **group commit**: ZAB pipelines concurrent
proposals and a single fsync drains everything queued — assume ~50 proposals drained per
fsync at peak → 7,500 fsyncs/s requested per server / 50 ≈ **150 fsyncs/s per disk** at
peak (every one of the 5 voting servers fsyncs *every* proposal, so the load does not
divide across them), ~2,500/50 = 50/s per disk on average. That is the whole write story:
the ceiling is fsync, not CPU, not network.

- HDD (~10 ms fsync): caps at ~50–100 fsync/s *without* any batching headroom — the ~50/s
  per-disk average already sits at the cap and the ~150/s peak exceeds it. NVMe
  (~0.1–1 ms): hundreds to thousands of fsyncs/s headroom. **Pick NVMe for
  the txn log** (separate disk from the OS/snapshot disk) — this is a design requirement,
  not an optimization, and it is why the estimate says "the fsync is the system."

```text
      write requests (2,500/s avg, 7,500/s peak)
        ↓
   leader fsyncs own log ──┐
        ↓                  │  ZAB PROPOSE, pipelined
   follower fsync (quorum barrier)   ← the real ceiling
        ↓                  │
   quorum ACKs → COMMIT → apply to in-memory tree
```

**Bandwidth: the network is irrelevant here.** Estimate ~300 bytes/proposal envelope:
7,500 writes/s × 4 follower PROPOSEs × 300 B + ACK + COMMIT traffic ≈ **~15 MB/s peak** inside the
ensemble — noise on any LAN. This inverts the cost model of every streaming case
([WhatsApp's media path](../03.whatsapp/02-scale-estimates.md) is PB/day; ZooKeeper is
GB/day). Compare with [Kafka's bandwidth math](../13.kafka/02-scale-estimates.md), where
bandwidth *is* the design driver.

## Storage

Per ensemble (estimates):

- in-memory tree: 500K znodes × 200 B avg + stat/children overhead ≈
  ~150 MB live data ×~3 for object overhead ≈ **~500 MB heap** — fits in a small host; a 1 MB
  data cap per znode (default) exists precisely so one client cannot blow this up.
- txn log: ~300 bytes/txn × 2,500 writes/s = 750 KB/s × 86,400 s ≈ **~65 GB/day** (est.),
  dominated by churn (ephemeral create/delete and lock retries), not by config writes.
  Retention: snapshot every ~100K txns — every ~40 s at this write rate — and pruning keeps
  a few days of logs → **~65 GB/day, ~130–200 GB of disk for 2–3 retained days** (small next
  to the data systems it coordinates, but not the MB-scale story the heap size suggests).
- snapshot: serialization of the in-memory tree ~200 MB (tree + per-znode stat data,
  without the heap's object overhead — the 500 MB heap number above) plus the session
  table, written to a second disk so it
  does not compete with the txn log's fsync.

Storage is a non-story by design (01-requirements, out-of-scope list). The one real risk is
unbounded *churn*, not bytes: a misbehaving client that recreates an ephemeral-sequential
node in a tight loop can generate tens of txns/s doing nothing useful (see the failure
modes in deep dive 6).

## Sessions, ephemerals, watches (the churn numbers)

- **Heartbeats**: session ping interval ~ timeout/3 (e.g. ~10 s at a 30 s timeout).
  2,000 sessions → ~200 pings/s/ensemble — trivial CPU-wise; the real cost is connection
  liveness bookkeeping on every server, not throughput.
- **Reconnect blips**: assume ~5% of sessions reconnect per hour from network blips →
  ~2,000 × 0.05 / 3,600 ≈ **0.03 reconnects/s average**, spiking to thousands/s during a
  deployment or a network event when many hosts come back at once. Each reconnect is a new
  zookeeper connection + session re-validate — cheap individually, a stampede when batched
  (mitigate client-side with backoff, same lesson as WhatsApp's [reconnect storms](../03.whatsapp/04-deep-dives.md)).
- **Ephemeral nodes**: ~1 per session → ~2,000 per ensemble. When the *leader* fails,
  the incoming leader reaps expired sessions; if a network blip expired a chunk of them
  simultaneously, that is ~1,000+ delete transactions in a burst — a few seconds of
  elevated write load on the fsync-bound path. This is the load signature every
  ZooKeeper operator recognizes (and why session timeouts are tuned, deep dive 3).
- **Watches**: a hot config znode might have ~1,000 watchers; a big election path has
  ~candidates-worth (say 10–100). One setData of a hot znode fires **~1,000 notifications
  in a burst** — trivial bytes, but every notified client immediately re-reads and re-arms,
  so a single write *causes* ~1,000 reads in the next instant. Quantified thundering herd;
  see deep dive 4 for the fix.

## Cross-ensemble consistency check

- 200 ensembles × 5 servers = 1,000 hosts total for the coordination plane — < 1% of
  server count, ~100% of correctness weight. Every compute-heavy case in this corpus
  (Twitter, WhatsApp, Kafka's consumer groups, Bigtable's tablet assignment) has a
  coordination dependency with exactly this shape.
- Reads per fleet: 200 × 47,500 ≈ **~10M reads/s**, all memory-only, no cross-ensemble
  traffic. Writes: 200 × 2,500 ≈ **~500K writes/s fleet-wide**, all fsync-quorum-bound
  within their own ensemble. Nothing global is ordered — each ensemble is its own
  consensus domain; that isolation is a feature.
