# 01 — Requirements

Scope: one event-log cluster serving as a company's central data backbone — clickstream,
order events, change-data-capture, metrics — feeding both streaming consumers (services
reacting in real time) and batch consumers (data warehouse loads).

## Functional

- **Topics**: named categories of events; a topic is split into N partitions; each
  partition is an ordered, immutable, append-only sequence of records, each with a
  numeric offset (its position, assigned by the broker).
- **Produce**: append a record to a topic; the producer picks the partition, either
  explicitly, by round-robin, or by hashing a key (same key → same partition → ordered
  per key). Records carry a key, a value, optional headers, and an application-supplied
  event timestamp.
- **Consume**: read a partition by offset range; consumers track their own position
  (offset) per partition; a consumer can rewind to any offset still within retention —
  replay is a first-class feature, not an exception.
- **Consumer groups**: a set of consumers that collectively read a topic — each
  partition is assigned to exactly ONE member of a group at a time; membership changes
  trigger a rebalance. Independent groups read independently (multiple subscribers to
  the same data).
- **Replication**: each partition replicated across R brokers; one replica is the leader,
  the rest followers; producers and consumers talk to the leader.
- **Retention**: keep data for a time-based window (days) and/or size-based cap;
  optionally compact per key instead of deleting (keep the LATEST record per key).
- **Delivery semantics, at the application level**: at-least-once (consumer processes a
  record at least once, may duplicate on retry); exactly-once requires either
  idempotent consumers (transactional offset-commit + processing) or an idempotent
  sink. The log gives at-least-once for free and dedup/ordering to whoever needs it.

## Non-functional (rough targets, estimates)

- **Throughput**: aggregate ingest ~1M msgs/s average, ~2M/s sustained peak (see
  [02-scale-estimates](02-scale-estimates.md) for arithmetic). A single broker must
  sustain several hundred MB/s of produce+consume traffic without p99 collapse.
- **Latency**: acked produce p99 within tens of ms for the default config (batching adds
  deliberate queueing ms); consume end-to-end (produce → consumer poll) typically <100 ms
  when consumers keep up. These are design targets, not measurements.
- **Durability**: a broker's death must not lose acked data, subject to configured
  quorum (`acks=all`, `min.insync.replicas >= 2` of RF 3); the trade-off knobs for
  weaker promises are explicit (deep dives §3).
- **Ordering**: strict per-partition (stronger practical promise: per-key, since a key
  maps 1:1 to a partition for a fixed partition count). NO ordering guarantee across
  partitions or topics — ever. Consumers needing global order must design for it.
- **Availability**: serve reads and writes through single-broker failure; a short
  read-only window during leader election is acceptable; a hard choice on unclean
  election is required (deep dives §3).
- **Bounded storage**: retention window enforced by deletion, not by a compaction
  contract that "somebody will read it in time".

## Scope decisions

**In scope**: the log itself, producer batching/compression, consumer groups and the
rebalance protocol, offset storage as a compacted log, retention and log compaction,
the replication protocol.

**Explicitly OUT of scope**:

- **A full stream-processing framework** (windowed joins, exactly-once pipelines with
  state stores) — a consumer library topic of its own; we assume consumers are ordinary
  services.
- **Schema management** (schema registry, evolution, compatibility rules) — the log
  carries bytes; schema governance is a separate service.
- **Cross-cluster replication / geo-failover** (mirroring) — mentioned in architecture as
  a boundary only.
- **Tiered storage, cold archives to object storage** — a retention-cost optimization;
  the core design works without it.
- **Push-based delivery to mobile/web clients** — the log serves server-side consumers
  with managed connections; phone fanout is a delivery layer on top (pattern: see
  [Twitter](../02.twitter/03-architecture.md) — log feeds the fanout services, it is not
  the fanout itself).
- **Security (ACLs, encryption, quotas)** — orthogonal, assumed present.
- **Transactions spanning produce + consume + processing** — we cover exactly-once at
  the level of "idempotent consumer or atomic offset-commit within a transactional
  producer", not full streaming transactions.
