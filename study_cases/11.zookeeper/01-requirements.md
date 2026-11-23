# 01 — Requirements

## What ZooKeeper actually is

A tiny replicated in-memory key-value tree with strong ordering guarantees and
session-scoped state, used by *other* distributed systems as their brain: leader election,
group membership, distributed locks, configuration, and metadata storage. Everything else
about the design follows from choosing "small data, high trust" — this is the opposite brief
from the big-storage cases in this corpus.

## Functional requirements

**F1 — Hierarchical znode tree.** Clients create/delete/read nodes at paths
(`/app/config`, `/election/n_0000000417`). Data is read/written as a unit; a node carries a
`stat` (version, zxid of last change, children count). Znodes are the whole data model — no
queries, no ranges, no partial reads.

**F2 — Writes are totally ordered.** Every state-changing request is serialized into one
global order, stamped with a monotonically increasing **zxid** that clients can see and
compare. If two clients learn a write through ZooKeeper, they learn the same order.

**F3 — Versioned CAS updates.** `setData(path, data, version)` succeeds only if the
current version matches. This is the fence the recipes build on (see deep dive 6).

**F4 — Ephemeral nodes.** A node can be created *with the client's session as its
lifetime*; when the session dies (disconnect, client crash, GC pause past timeout, eviction),
the server deletes the node. This is the system's "liveness signal" primitive.

**F5 — Sequential nodes.** A create can request a monotonically increasing
per-parent suffix (`/lock/n_0000000012`). Combined with ephemeral, this gives a fair queue
of candidates without all of them hammering one node.

**F6 — Watches.** A read can arm a **one-shot** notification ("tell me when `exists` /
`setData` / `delete` / children changes next fires for this znode"). Ordering guarantee:
the watch event is delivered to the same session *before* it can see the new data. There is
no subscription — every delivery invalidates itself.

**F7 — Sessions.** A client holds one TCP connection to one server, representing a
session with a negotiated timeout (bounded by `minSession = 2 × tick`, `max = 20 × tick`
server config; tick is typically ~2 s, so timeouts run ~4–40 s). The client heartbeats;
the server expires silent sessions and reaps their ephemerals. A client reconnects to any
ensemble member and its session continues if it hasn't expired.

**F8 — Recipes, client-side.** The server ships *no* lock/election concept. Leader
election, locks, barriers, and group membership are recipes over F3–F6 in client libraries
(Curator et al.). This scope decision is important and is repeated in the out-of-scope list.

## Non-functional requirements

| Property | Target | Notes |
|---|---|---|
| Availability | survive failure of `f` of `2f+1` servers; ~3–5 nodes typical | Writes stop below quorum; minority-side servers still serve reads of their committed prefix |
| Consistency | linearizable writes; reads may be slightly stale; `sync()` forces a fresh read | see deep dive 2 — the staleness is a design choice, not a bug |
| Durability | every committed write fsync'd to a txn log on a quorum; periodic snapshots | snapshots make restart fast, logs make it correct |
| Read latency | ~few ms p99 — memory tree + one RPC, no disk IO on the read path | the read path must not touch fsync |
| Write latency | bounded by the slowest quorum-member fsync (~2–20 ms typical, estimate) | group commit batches concurrent proposals per fsync |
| Throughput | reads ~10–100× writes in real deployments; paper-era ensembles did tens of 1000s of reads/s, writes several× lower (ATC'10, publicly reported) | writes cost a quorum fsync; reads cost memory bandwidth |
| Operationally simple | fixed ensemble, no sharding, no schema, no tuning per workload | if you need to scale it, you're using it wrong (deep dive 5) |

## Consistency targets clarified

- **Writes/updates**: linearizable with respect to each other — one global zxid order,
  a majority-quorum disk barrier per transaction.
- **Reads**: served from each server's locally committed snapshot — a *prefix* of the true
  order, never a value that doesn't exist in it, but possibly a few transactions behind.
- **Client's own writes**: a client sees its own write take effect on its next read —
  the write is acked after quorum commit, and the serving connection's server has that
  transaction by then; the `sync()` primitive exists when the client reads through a
  *different* server than it wrote to.
- **Session semantics under partition**: the minority side keeps *serving* (stale reads)
  but cannot commit; the majority side elects and carries the system forward. Session
  expiration is the tiebreaker for liveness claims — see deep dive 3.

## Scale decisions

This is the one case in the corpus that does **not** grow with user traffic. The estimates
in [02](02-scale-estimates.md) assume a *fleet* of ensembles (one or a few per cluster of
services), each serving its owning system. Everything is tuned so an ensemble's correctness
surface stays small: memory-resident tree, single-order log, fixed quorum.

## Explicitly OUT of scope

- **Bulk data.** Znode data is capped small by convention/default (~1 MB) and should be
  KB-scale. No blob storage, no media — that is the job of the systems ZooKeeper coordinates
  (see [Bigtable's metadata split](../12.bigtable/04-deep-dives.md): metadata vs data is a
  hard boundary).
- **Query/range/search.** Path-prefix lookups only. No secondary index, no scans, no SQL.
  Anything needing rich lookup belongs in the coordinated system, not in this service.
- **Server-side locks or automatic recipes.** The server never understands "I am holding
  a lock." Locks are client conventions; this is deliberate (see deep dive 6) — keeping the
  service simple is why it can be correct.
- **Exactly-once or queued notifications.** Watches are one-shot and can miss events in the
  re-arm window; there is no delivery log, no retry of a watch event. Clients are expected
  to tolerate this and re-read.
- **Geo-replication / multi-region.** A single LAN quorum. Spanning a WAN quorum destroys
  write latency for no benefit — the disaster story is fleet-level (many independent
  ensembles), not replication.
- **Sharding/partitioning for scale.** One ordered log, one memory tree. Scale out by
  fleet-splitting per system, plus observers for read scaling (deep dive 5) — never by
  sharding the consensus domain.
- **General-purpose coordination as a database substitute.** If a workload writes
  continuously at a rate comparable to its data service, ZooKeeper is the wrong tool for
  that part (this is exactly Kafka's later lesson — see [13.kafka](../13.kafka/04-deep-dives.md)).
