# Kafka across the case catalog

Where Kafka appears in the 50 study cases, what role it plays, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md) (event-backbone union, Tier 2: "distributed append-only
log / event bus, Kafka-style", 23/43 cases).

Headline: **Kafka is not really a technology in this catalog — it is a shape.** In almost every
case it appears as *a partitioned append-only log, partitioned by a domain key, consumed by
several independent consumers that own their own cursors*. Only one case studies Kafka itself;
three use it as an analogy or a counter-example; one rejects it with an explicit trade-off.

- Cases that mention Kafka: **24 / 50**
- Kafka (or a "Kafka-class" log) is a designed architectural component: **17**
- Brief mention / sizing shorthand only: **4** (04, 06, 45, 46)
- Analogy only (no Kafka component): **2** (48 partially, 49)
- Studied or rejected explicitly: **2** (13 = the case itself, 50 = rejected), plus 11 as counter-example

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [11.zookeeper](11.zookeeper/README.md) | Counter-example: what Kafka used ZK for, why KRaft replaced it | Deep (not a component) |
| [13.kafka](13.kafka/README.md) | The case itself: the log backbone, modeled end-to-end | Full case |
| [04.deployment-system](04.deployment-system/README.md) | Event-stream sizing shorthand for run/job lifecycle events | Brief |
| [06.apple-app-store](06.apple-app-store/README.md) | Small stream (~35K ev/s) feeding hourly chart aggregates | Brief |
| [14.ml-production-system](14.ml-production-system/README.md) | Single spine: ingestion → training → offline store → monitoring | Core component |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Durable ingest bus for flaky devices, partitioned by charger/site | Core component |
| [28.uber](28.uber/README.md) | Event log partitioned by city/driver; lossy + must-not-lose signals on separate topics | Core component |
| [29.uber-eats](29.uber-eats/README.md) | Backbone splitting courier telemetry from the order store | Core component |
| [33.instagram](33.instagram/README.md) | Post/engage/view events; source of fanout into candidate pools | Brief in text, core in design |
| [34.reddit](34.reddit/README.md) | Vote/impression/mod events for ingest, rank-cache refill, abuse | Core component |
| [35.tiktok](35.tiktok/README.md) | Observe-plane log partitioned by user_id feeding 3 consumer families | Core component |
| [37.ticketmaster](37.ticketmaster/README.md) | Log as source of truth for the seat/order state machine | Core component |
| [38.leetcode](38.leetcode/README.md) | Verdict log (append-only stream); log = truth, projections elsewhere | Core component |
| [39.e-commerce](39.e-commerce/README.md) | Event spine (~15B ev/day): transactional plane → read models | Core component |
| [40.tinyurl](40.tinyurl/README.md) | Click fire-hose; explicit fire-and-forget trade-off | Moderate |
| [41.web-crawler](41.web-crawler/README.md) | Post-dedup crawl handoff log partitioned by canonical URL | Core component |
| [42.linkedin](42.linkedin/README.md) | Named EVENT BUS fanning out edge/profile/feed events + notifications | Core component |
| [44.payment-system](44.payment-system/README.md) | Payment event bus; consumer-lag trade-off for fraud counters | Brief–moderate |
| [45.typeahead](45.typeahead/README.md) | Query-event bus for the trending layer (~9K ev/s) | Brief |
| [46.google-docs](46.google-docs/README.md) | Non-collaborative change→notify only (comments, sharing) | Brief |
| [47.rate-limiter](47.rate-limiter/README.md) | Decision-log fire-hose; sampled/dropped under backpressure | Moderate |
| [48.google-search](48.google-search/README.md) | Crawler→index stream semantics + segment-compaction analogy | By-reference |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Pure analogy: TSDB head/WAL/blocks ≈ Kafka's log/segments | Analogy only |
| [50.stock-exchange](50.stock-exchange/README.md) | Same log skeleton; Kafka-style replication explicitly evaluated and rejected | Deep (rejected) |

## Per-case notes

### The case itself

**[13.kafka](13.kafka/README.md)** — Modeled as a partitioned, replicated append-only commit
log ([03-architecture.md](13.kafka/03-architecture.md)): 40 brokers; topics `orders`, `clickstream`
at 120 partitions each; `__consumer_offsets` as a compacted internal topic reusing the log's own
compaction; brokers deliberately dumb, all smarts in producers/consumers.
Key decisions in [04-deep-dives.md](13.kafka/04-deep-dives.md): ordering is per-partition only and
most incidents are key→partition mistakes or changing partition count mid-life; idempotent producer
(PID + per-partition sequence); cooperative-sticky rebalancing, one-partition-one-consumer;
retention for event topics vs compaction for state topics; zero-copy sendfile is why compaction can
be a background job; loss-vs-availability (`acks`, `min.insync.replicas`, unclean leader election)
framed as a deliberate product decision.

### Counter-example

**[11.zookeeper](11.zookeeper/README.md)** — Kafka is the case's flagship "where external
coordination failed" story ([04-deep-dives.md](11.zookeeper/04-deep-dives.md)): Kafka ran on
ZooKeeper 2011–~2022 for broker registration, ISR tracking, controller election, ACLs, quotas; two
consistency domains (ZAB vs Kafka replication), minutes-long controller failover reloading full
cluster metadata, every metadata write paying a consensus round-trip — motivating KIP-500/KRaft.
Lesson reused repo-wide: coordination is for slow, rarely-decided decisions; once decisions become
a high-rate ordered stream, move them into your own log.

### Core component: log-first ingest and decoupling

These cases deploy a Kafka-class log as the pipe between a transactional plane and everything else.
The recurring contract: writes land on the log first; each downstream subsystem (features, caches,
index, analytics, recompute) replays it independently; idempotency absorbs at-least-once delivery.

- **[14.ml-production-system](14.ml-production-system/README.md)** — one spine for everything:
  action events + prediction rows → offline store via compaction; the training path consumes the
  same stream for labeled windows; drift monitors read the same stream. Sized at ~17 GB/s raw
  (~6 GB/s compressed) — "a serious ingest tier, not a sidecar".
  Named failure mode: Kafka retention shorter than the training window.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — the bus exists for a durability
  requirement ("no message lost, ever, across gateway crashes"), not for volume; topics
  tx-events/meter-values/status/auth/commands-ack, partitioned by charger so per-key ordering holds;
  at-least-once + idempotent consumers.
- **[28.uber](28.uber/README.md)** — position samples (huge, lossy OK) and trip lifecycle events
  (few, must-not-lose) on separate topics with different guarantees; partition by city/driver.
- **[29.uber-eats](29.uber-eats/README.md)** — courier pings (~hundreds of K/s, dominant writer by
  2 orders of magnitude) go to the log with downsampling at ingest, never into the order DB;
  consumers: dispatch, tracking, risk.
- **[33.instagram](33.instagram/README.md)** — post/engage/view events; the bus is the source of
  fanout into per-user candidate-pool caches.
- **[34.reddit](34.reddit/README.md)** — 20–45K QPS vote spikes reframed as "a message-throughput
  problem, not a transaction problem"; one event log feeds vote ingest (dedupe per
  user×item), rank-cache refill, and spam classifiers. Tiny 50–100 B events → batching.
- **[35.tiktok](35.tiktok/README.md)** — observe-plane collects batched client events, then a log
  partitioned by user_id feeds online feature updates (≤5–30 s freshness budget), approximate
  counters, and the raw-event lake for training/replay.
- **[37.ticketmaster](37.ticketmaster/README.md)** — the log is the *source of truth* for the
  seat/order/hold/reap state machine; inventory shards, the hold-expiry reaper, and dispute audit
  are projections; snapshot + replay rebuilds a strip's partition in seconds. Contrasts itself with
  13: ticketing is the purest *hot-partition* problem — default key partitioning cannot spread it.
- **[38.leetcode](38.leetcode/README.md)** — verdict log decouples stateless judges from result
  delivery, leaderboard, anti-cheat; leaderboard topic partitioned by participant_id; Redis ZSET
  is a projection, rebuildable after re-runs flip verdicts.
- **[39.e-commerce](39.e-commerce/README.md)** — event spine at ~15B events/day partitioned by
  order_id/sku_id/customer_id; nothing writes a read model inline; search index, availability,
  personalization, analytics are all rebuildable projections (~3 TB/day, 7 days hot = 21 TB).
- **[41.web-crawler](41.web-crawler/README.md)** — post-dedup crawl handoff log, partitioned by
  canonical-URL hash so all events for one URL stay ordered on one partition; indexer, link-graph
  builder, and freshness scheduler are three cursor owners; consumers dedup by URL+content-hash.
- **[42.linkedin](42.linkedin/README.md)** — the named-Kafka case (fittingly; Kafka was born
  there): topics edge-changes/profile-edits/profile-views/feed-events feed Samza-type stream jobs
  (invalidations, PYMK, WVYP) and async notification fanout; the synchronous edge state machine
  stays authoritative, everything downstream is derived.
- **[44.payment-system](44.payment-system/README.md)** — event bus beside the payment gateway;
  the one trade-off spelled out: fraud/velocity counters go stale under consumer lag (model sees
  yesterday's frequencies — fine for minutes, alarmed beyond).

### Designed telemetry pipelines: fire-and-forget by choice

**[40.tinyurl](40.tinyurl/README.md)** and **[47.rate-limiter](47.rate-limiter/README.md)** pick
the opposite contract on the same log shape: the redirect path (or admission decision) does one
non-blocking emit to a buffered producer; under backpressure events are *dropped*, not queued.
Chosen because the interactive path must never slow down; the cost is stated — counts lag by
seconds-to-minutes, drops are real, raw log and rollups can disagree. 47 sizes it: ~0.6 TB/day of
~100 B decision records.

### Brief mentions

- **[04.deployment-system](04.deployment-system/README.md)** — sizing shorthand: 130M run/job
  lifecycle events/day + ~200K log-stream appends/s on a "Kafka-shaped" bus.
- **[06.apple-app-store](06.apple-app-store/README.md)** — ~3B events/day (~35K ev/s) as "a small
  Kafka-tier stream" feeding hourly chart aggregates; charts explicitly not real-time.
- **[45.typeahead](45.typeahead/README.md)** — ~800M queries/day (~9K ev/s) into the trending
  layer; "modest topic; the correctness question is freshness windows, not throughput".
- **[46.google-docs](46.google-docs/README.md)** — event bus only for comments/sharing
  notifications; the collaborative editing path deliberately uses a TTL'd presence hub, not a
  durable log.

### Analogy / by-reference

- **[48.google-search](48.google-search/README.md)** — the crawler→index stream borrows 13's
  delivery semantics (at-least-once, idempotent by doc ID; replay to rebuild the fresh tier), and
  fresh/base index tiering is compared to Kafka segment compaction — named by reference, not as a
  component.
- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — pure analogy: TSDB
  head-block/WAL/immutable-block storage engine described via Kafka's log-and-segments vocabulary
  ("log before you ack", single-writer appends are cheap). No Kafka in the architecture.

### Explicitly rejected

**[50.stock-exchange](50.stock-exchange/README.md)** — the matching engine is the same skeleton
as 13 (deterministic state machine consuming a totally-ordered log), and [04-deep-dives.md]
(50.stock-exchange/04-deep-dives.md) weighs Kafka-style 3-way-replicated inbound logs as Option A:
wins inherited durability and failover, loses on ~ms-class per-message quorum commits + fsync
against a ~20 µs matching budget. Chosen: bespoke sequencer + standby replay — "replay beats
sync-replicate whenever the reducer is deterministic". The sharpest boundary in the catalog:
Kafka's shape holds, its replication latency contract does not.

## The portable pattern

Every substantive use reduces to one contract, cross-linked to
[13.kafka](13.kafka/README.md) as the reference implementation:

1. **Write to the log first.** The transactional plane stays small and consistent; the log carries
   everything derived (39, 37, 38, 42).
2. **Partition by the domain key** whose ordering matters: order_id, charger, driver, user_id,
   participant_id, canonical URL (39, 27, 28, 35, 38, 41).
3. **N independent consumers own their cursors** on the same stream; new consumers replay from
   retention, not from backfill jobs (14, 41, 37, 48).
4. **At-least-once delivery + idempotent consumers** absorb redelivery (27, 34, 41, 48); the
   union concept in [CONCEPTS.md](CONCEPTS.md) names its most portable piece: client-supplied ID
   + dedup window.
5. **Guarantee level is a per-topic decision**: must-not-lose trip/payment/seat events vs lossy
   telemetry that may be dropped or downsampled (28, 29, 40, 47).
6. **Hot-partition and latency limits are where the shape ends**: ticketing's hot events converge
   on one shard (37), and microsecond matchers reject quorum-committed logs (50).
