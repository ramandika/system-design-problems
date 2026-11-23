# Case 13 — Kafka

**The system**: a distributed, partitioned, replicated append-only commit log. Producers
append events to topics; consumers read them at their own pace with their own cursor
(offset). Sounds like a message queue; it is actually a *distributed replicated log* —
the disk file IS the database, and the consumers, not the broker, own the delivery state.

**Why it's hard**:

- The whole design rests on one primitive — sequential disk appends and file reads — and
  every feature (ordering, replication, retention, compaction, delivery semantics) is a
  consequence of that primitive. Get the partitioning wrong and ordering or throughput
  breaks; there is no "fix it later with an index".
- Ordering is per-partition only. The moment two related events land in different
  partitions, any consumer that reads them together can see them out of order. Most
  Kafka incidents in practice are ordering accidents caused by partition-count changes
  or bad partition keys, not lost data.
- The failure mode you choose (data loss vs availability) is a small set of config:
  `acks`, `min.insync.replicas`, unclean leader election. It is a *product decision*
  masquerading as an ops setting.
- Consumer rebalancing is a distributed protocol that fires on every deploy, every scale
  event, every network blip — and a bad one can stop ALL consumption for a topic for
  minutes.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: append-only log as the source of truth, per-key ordering and
its limits, replicated log with ISR/quorum thinking, batching beats request-level
tuning (also seen as write coalescing in
[Twitter's fanout](../02.twitter/04-deep-dives.md)), pushing state to the client
(offsets ≈ consumer-side cursors, the same move as WhatsApp's client-side dedup in
[03.whatsapp](../03.whatsapp/04-deep-dives.md) §1), retention vs compaction as two
different answers to "how much history".

**Try it first**: before reading 03, answer: (1) why does "append to a file, keep the
file" beat a broker that deletes on delivery — for throughput AND for failure semantics?
(2) Two events about the same order land in partitions P1 and P2. What does a consumer
that needs to reconcile that order have to do? (3) A broker dies with data only on it.
What decides whether consumers see a gap, and who made that decision?
