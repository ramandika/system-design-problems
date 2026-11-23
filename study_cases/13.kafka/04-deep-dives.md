# 04 — Deep dives

## 1. The partitioned append-only log, and what ordering really buys you

A partition is an append-only sequence of records, each stamped with its offset. Every
stronger promise — ordering, dedup, replay, exactly-once — is a consequence of "one
append-only file, one cursor per consumer". The trap: the promise is **per-partition
only**.

Design decisions inside this:

- **Key-based partitioning**: producer hashes the key → partition. Same key → same
  partition → ordered per key. This is the only ordering most applications need
  (per-order, per-user, per-account), and it's the cheapest one to get.
- **No cross-partition ordering, ever**: two events about the same order land in
  different partitions (shifting keys, retries after a rebalance, a repartitioning
  stream, manually chosen partition) → any consumer reading them together sees
  arbitrary order. Either the application tolerates it, or the application must buffer
  and re-sort (a stream join/windowing job of its own — out of scope here) — see the
  consumer-joins section below for the read-side equivalent of Twitter's merge problem.

**Trade-off**: more partitions = more parallelism and per-broker spreading, but also
more open file handles, more replication traffic (each partition replicated
independently), more rebalance-assignment work in consumer groups, and — the killer —
**changing partition count mid-life breaks key→partition mapping**, so per-key ordering
silently dies. A topic should be sized for its foreseeable peak and then stopped
growing; adding partitions is a migration, not an ops command.

**Failure modes**:

- Out-of-order events after a partition-count change → consumers apply stale/late
  state before fresh state; fix = double-write both old and new partitions during
  migration, or use a key→partition mapping service instead of hash-of-key.
- Hot partition (one user/order with 10M events/day skews the hash) → partitions are
  even in count but wildly uneven in load; fix = finer key (add sub-key) or accept it
  and slow-drain (bounded skew, not unbounded).
- Per-key "last event wins" logic in consumers will corrupt state if the ordering
  guarantee is silently violated; pair every consumer with an explicit statement of
  WHAT ordering it assumes.

## 2. Producer batching and compression vs latency

Producers accumulate records into a per-partition buffer and send **batches**, not
records. Two knobs: `linger.ms` (wait up to N ms for more records) and `batch.size`
(send when batch is full). Compression (lz4/zstd/snappy) happens per batch.

**Batch of ~200 records** (from estimates): 5K produce requests/s instead of 1M — 200×
fewer requests, and the ~3× compression ratio is only reachable over a batch
(headers/keyframes amortize; per-record compression of 1 KB records barely gains).
This is the same move as Twitter pipelining 4,000
recipients per fanout batch (see [02.twitter](../02.twitter/04-deep-dives.md) §1):
the win comes from amortizing per-request cost over many records.

**Trade-off**:
- Larger batches / higher linger: better throughput (fewer requests, better
  compression, fewer fsyncs amortized per byte) — loses latency (each record waits up
  to linger.ms even when the cluster is idle) and loses granularity (a slow consumer
  fetches a bigger chunk, a retried record drags a whole batch with it).
- Small batches / linger 0: lower per-record latency — loses throughput (request
  overhead per record is what actually kills brokers at scale; the 1M msgs/s target is
  unreachable request-per-record).

**Chosen: linger.ms 5–20 ms, batch.size ~16–64 KB, compression producer-side zstd/lz4.**
Wins the 200× request reduction and compression, at a cost of ≤20 ms added latency —
invisible against end-to-end produce→consume (~100 ms target) while throughput goes
from impossible to comfortable.

**Failure modes**:

- Batch too big (> broker max.request.size) → producer error whole batch; fix = clamp
  batch.size to a fraction of max message size.
- Compression on tiny batches ≈ no compression (headers dominate) — the estimates'
  "small-message pathology": tune batching UP when messages shrink.
- Producer retry of a batch after a leader change can **reorder** (failed batch lands
  after a later batch) → set `max.in.flight <= 5` with idempotence enabled, or accept
  reordering; idempotent producer (PID per producer, per-partition sequence numbers)
  makes retries dedup-safe and order-preserving.

## 3. Replication with ISR: durability vs availability as a config, then as a policy

Each partition: leader + R-1 followers. Followers pull in sync; a replica inside
`replica.lag.time.max` of the leader is **in-sync (ISR)**. Producers choose `acks`:
0 (fire-and-forget), 1 (leader only), all (all current ISR members). `min.insync.replicas`
sets the floor below which `acks=all` producers fail rather than write.

**Unclean leader election**: when the leader dies and NO in-sync replica remains,
choose between (a) fail the partition (stay unavailable, keep the durability promise),
(b) elect an out-of-sync replica (unclean) and accept losing the records between the
old leader's last acked offset and the new leader's log head.

**Trade-off (pick explicitly, it's a product decision)**:
- (a) wins correctness — no acked data vanishes; loses availability — the partition is
  read-only/down until a replica reconciles; and it loses on a real failure mode:
  losing ALL in-sync replicas at once (rack/region failure, correlated crash, or
  sustained under-replication during a rebalance) is rare but not hypothetical.
- (b) wins availability — the partition keeps flowing; loses durability — committed
  events silently disappear, and downstream consumers that already read them (offsets
  committed before the unclean election) have now-phantom state.
- Pick (a) for money-path data (payments, CDC, order events) and (b) for high-volume,
  low-value telemetry where a gap beats an outage. This mirrors the WhatsApp call in
  [03.whatsapp](../03.whatsapp/04-deep-dives.md) §6: messages MUST-NOT-LOSE vs presence
  MUST-NOT-COST-ANYTHING — decide per data class, not once per cluster.
- Concrete config per topic: `acks=all`, `min.insync.replicas=2`, `unclean.leader.election.enable=false`
  for money-path; `acks=1`, unclean=true (or even acks=0) for clickstream.

**Failure modes**:

- `acks=all` with `min.insync.replicas=1` is NOT durable (leader-only ack) — a real
  config landmine; the "all" is against the ISR, which can shrink to 1.
- ISR shrinks during a broker restart (lag exceeds replica.lag time) → with RF 3 and
  one broker out, ISR = 2 = min.insync.replicas, so `acks=all` writes still SUCCEED —
  they fail only when ISR < min.insync, i.e. ISR = 1 here, which takes TWO brokers out.
  But a single restart leaves the topic sitting exactly ON the floor with no margin:
  any second failure becomes a produce outage, not degradation; plan rolling restarts
  for max ISR headroom, not just uptime.
- Unclean election in a cluster where some topics allowed it → the lost records are
  INVISIBLE to consumers (no gap marker in the log); the offset is continuous but the
  data isn't. Mitigate with upstream dedup/reconciliation or per-topic unclean=off on
  anything that matters.

## 4. Consumer groups, rebalance storms, and offset storage

A consumer group distributes a topic's partitions among its members; one partition, one
group-owner. A group coordinator (broker) tracks membership; a member change triggers a
**rebalance**: everyone stops, hands assignments back, and re-renders them.

Design decisions:

- **Eager vs cooperative (incremental) rebalancing**. Eager: whole group stops for the
  duration (revoke all → reassign). Cooperative: only partitions that actually move
  between members are revoked, others keep consuming.
- **Trade-off**: eager wins simplicity (one protocol, easy to reason about, easy to
  test) — loses availability because EVERY rebalance pauses ALL consumption in the
  group. Cooperative wins availability (only moving partitions pause briefly, at most
  2 phases instead of 1) — loses on complexity: the protocol has more states, and
  buggy implementations cause partial-assignment hangs that are harder to reason about
  than a whole-group stop. Chosen: **cooperative-sticky** for consumer groups, because
  rebalances happen on every deploy/scale event (at a 200-service company, daily), and
  "pause everything every deploy" is a p99 disaster; eager remains acceptable for small
  groups where pauses are rare and debuggability matters.
- **Rebalance storms**: the classic failure — a consumer takes too long processing
  (`max.poll.interval.ms` exceeded) → kicked out → rejoins → triggers rebalance →
  other consumers' processing slows during the rebalance → they get kicked → cascade.
  Mitigations: heartbeat separate from poll (session.timeout.ms handles liveness,
  max.poll.interval.ms handles processing time), max.poll.records tuned DOWN so
  a batch can't overrun the interval, backoff on join (randomized, not instant),
  and — the real fix — consumers must not be doing per-record slow work inline.
- **Offset storage**: `__consumer_offsets`, an internal log COMPACTED by key
  (group, topic, partition) → offset. Group coordinators treat it like any topic.
  Consumers commit offsets periodically (async, may lose recent commits on crash →
  at-least-once replay) or exactly-once (in a transaction with the side-effects, more
  machinery). This design reuses the log's own compaction primitive to store the log's
  own cursor — no external DB, no extra service to keep consistent.

**Trade-off (offsets)**: **async commit wins throughput and simplicity — loses at-least-once
exactness in one direction (crash between process and commit → reprocess, safe) and in
the other (commit before process → potentially SKIP a record, a real loss)**. Rule:
commit AFTER processing, not before; if you must commit-early for rate reasons, save
the offsets to a local checkpoint in between.

**Failure modes**:

- Session timeout too aggressive + GC pause / network blip → spurious rebalance; too
  lax → dead member holds partitions for minutes. Set from MEASURED worst GC/pause on
  your runtime, not defaults.
- Rebalance during a leader election (broker down) → consumers get a double pause and
  stale metadata; retry logic must assume both happen at once.
- Large partition count × many groups → assignment metadata and per-group coordinator
  state adds up (up to 500 topics × 120 partitions × 3 groups ≈ 180K assignments, an
  estimate ceiling — see below the estimates file); if groups > ~10 on
  hot topics, watch the coordinator broker, it becomes the hotspot.

## 5. Retention vs log compaction

Two different answers to "how much history does the log keep":

- **Retention (delete old)**: keep N days / M bytes, then delete whole segments.
  For event streams where history is a bounded analytic window.
- **Compaction (keep latest per key)**: never delete the LATEST record for each key,
  delete superseded ones. The log then always holds the "current state" for every key
  — usable as a table, rebuilt by replaying from offset 0.

They're not alternatives, they're orthogonal — a topic has one of them, per topic.

**Trade-off**:
- **Time-retention**: wins simplicity and bounded, predictable disk (600 TB from
  estimates) — loses the ability to rebuild from offset 0 forever (offsets past the
  window are gone), and loses records a slow consumer might still need. Sizing the
  window is a bet on "how long might a consumer need to catch up / a backfill need to
  re-read".
- **Compaction**: wins "log as database" semantics (replay from 0 = rebuild current
  state; used for __consumer_offsets, KTable-like changelogs, CDC) — loses bounded
  disk (disk scales with #keys, not time: a 100M-key changelog is a 100M-entry log
  forever, even with aggressive compaction), and compaction is a background job whose
  dirty/clean ratio must be tuned or disk balloons between runs.
- **Chosen: retention for event topics (order-events, clickstream), compaction for
  state topics (entity snapshots, offsets tables, CDC changelogs)** — because their
  consumers' needs differ (bounded re-read vs replay-to-present), and forcing both
  into one policy breaks one of them.

**Failure modes**:

- Retention too short for a slow consumer → "offset out of range" — consumer can't
  resume (must reset to earliest or skip); alert on consumer lag against BOTH
  throughput and retention window, not just lag.
- Compaction can DELETE a record you wanted (compaction keeps last-per-key — a
  tombstone marker must be used to signal deletion, with a bounded tombstone retention);
  misusing "absence of key" as "never existed" vs "deleted" is a modeling bug.
- Compacted topic as a source of truth but consumers never re-snapshot → state
  reconstruction = reading the whole compacted log; size it and drill for it, or
  it becomes an outage-length job.

## 6. Zero-copy transfer and throughput math

Why does a dumb log beat a smart broker? Because the fast path is:

1. Producer appends: network → page cache → **sequential append to the log file**
   (sequential disk writes are ~2–3 orders of magnitude faster than random writes —
   a publicly-reported, well-established disk characteristic, not a Kafka invention).
2. Consumer reads: partition log is in **page cache** (recently written, still hot),
   served directly from cache to NIC via **sendfile** — the kernel copies from page
   cache to socket buffer without copying into userspace, no per-record parsing, no
   application-object allocation.

**Math (rough, per broker)**:
- Produce path: 25 MB/s per broker sustained (estimates) → write-into-page-cache is
  ~10× headroom below even a slow NVMe's sequential write rate; page-cache hit rate on
  consumer reads is high because consumers read what was just written (lag is seconds
  to minutes << retention days).
- Consume path (zero-copy, from page cache): a single NIC at 10 GbE ≈ 1.25 GB/s;
  cluster-wide consume reads ≈ 1 GB/s compressed (estimates §2: 3 GB/s raw ÷ 3
  compression), ≈ 25 MB/s per broker across 40 brokers — the bottleneck is the
  NIC, NOT the disk and NOT the JVM heap. No per-record copies → no GC pressure on
  the read path → throughput scales with hardware, not with GC tuning.
- The cost of this design: consumers fetch **byte ranges**, not parsed records —
  deserialization moves to the consumer. The broker being dumb is what makes the log
  fast; the price is paid by every consumer (and why consumer-side schema/serde
  discipline matters).

**Trade-off (where zero-copy loses)**:

- **TLS**: sendfile can't be used with encrypted transport (the kernel can't hand
  encrypted bytes from page cache without a copy); encrypted clusters lose some of the
  zero-copy benefit OR pay for TLS offload/`SSLContext`-level optimizations.
- **Transformation at the broker** (re-partitioning, filtering, compaction on read)
  would break the zero-copy path — that's WHY compaction runs as a background job on
  segment files, not as a read-time filter.
- **Tiered storage / cold reads**: reading from object storage abandons page cache;
  tiered storage is fine for retention-cost, bad for replay-latency.

Compare with the same physics in [02.twitter](../02.twitter/04-deep-dives.md) §2
(pre-materialized timelines beat per-read fanout queries — same reasoning: pay at
write time with a simple append, read at cache speed with no per-read computation),
and the client-owns-state move shared with
[03.whatsapp](../03.whatsapp/04-deep-dives.md) §1 — the log is a store-and-forward
mailbox with client-side cursors.

**Failure modes**:

- Page cache thrash: a consumer reading OLD data (backfill of a 7-day topic, from
  estimates: ~230 MB/s sustained, ~20 TB total over a day) evicts fresh pages → fresh
  consumers now miss cache →
  reads hit DISK → broker p99 collapses; rate-limit backfills or run them on dedicated
  follower reads.
- File handles / open segments: 1000s of partitions × R × segment count →
  `ulimit`/fd exhaustion; watch it before it's the outage.
- fsync policy: the log relies on replication, not on fsync-per-batch, for durability
  (fsync every batch would collapse throughput). Power-loss of ONLY the leader without
  any follower sync (ISR=1) is where the durability guarantee actually weakens — which
  is why min.insync.replicas=2 is the load-bearing config, not the filesystem.
