# Case 12 — Bigtable

**The system**: a distributed, wide-column store over billions of rows: row key → column
families → columns → timestamped versions. Point reads at single-digit milliseconds,
range scans at terabytes per minute, horizontal scale to thousands of machines. The 2006
Google paper is the ancestor of Cassandra, HBase, ScyllaDB and the "wide-column" family
(the operational descendant in this corpus is described in
[WhatsApp's store discussion](../03.whatsapp/04-deep-dives.md), §3 — Discord's wide-column
store).

**Why it's hard**:

- The storage engine is a live tension, not a choice: writes must be fast (append-only LSM:
  log + MemTable + SSTables), which buys latency with read amplification (one point read
  may consult many SSTables) — every design point in this case is on that trade surface.
- Scale out must survive churn: the table is split into tablets that split, merge, and move
  between thousands of machines while clients are mid-scan. Coordination has to be
  available even when it's the weakest link in the chain.
- Row-key design is a product decision in disguise: monotonically increasing keys (timestamp-
  keyed writes, sequential IDs) funnel all traffic onto one server. Nobody can fix this in
  the storage engine; it must be fixed in the schema.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: LSM-tree write path (log → MemTable → SSTable → compaction),
write vs read amplification, bloom filters, tablet sharding and load balancing, external
coordination service (Chubby → the pattern behind ZooKeeper), reverse-timestamp row
design, locality groups.

**Try it first**: before reading 03, answer: (1) where does an incoming write actually
land, in how many devices? (2) why can a tablet server serve reads without asking anyone?
(3) your writes are timestamped — what happens to the tablet that owns "now", and name two
row-key fixes with their costs.
