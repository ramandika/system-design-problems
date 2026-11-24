# Cross-Case Concepts

Which properties, features and concepts are most shareable across the 50 study cases.
Derived from a per-case sweep (2026-11-24) that extracted the primitives, patterns and
techniques each case actually uses in its design, then normalized near-synonyms
("key-value store"/"in-memory cache tier" → one row; "blob store"/"object store"/
"content-addressed artifact store" → one row) and counted frequency.

Counting basis: backend n=43 (01–18, 26–50), client n=7 (19–25). Each case counts once
per concept, even if it names several synonyms of it. ★ marks cross-domain concepts —
ones that appear in both backend and client cases.

## Tier 1 — used in ≥60% of relevant cases

| Concept | Count | Share | Categories | Why it recurs |
|---|---|---|---|---|
| ★ Caching in any form (cache/KV tiers + invalidation/coherence + hot-key & thundering-herd defense) | 36/50 (32B/4C) | 72% | infra + storage + technique | Read-heavy hot sets in front of nearly every store: every miss storm, invalidation bug and TTL discipline is the same problem ([10.distributed-cache](10.distributed-cache/README.md), [09.cdn](09.cdn/README.md)) |
| ★ Async event backbone (append-only log/event bus + outbox + idempotency + at-least-once) | 36/50 (31B/5C) | 72% | infra + pattern + technique | Writes outrun synchronous processing everywhere; log-first decoupling with dedup is the universal answer ([13.kafka](13.kafka/README.md)) |
| Cursor & sequence sync — client side | 6/7 | 86% of client | technique + pattern | Reconnect, offline replay and feed paging all reduce to "last sequence I saw, catch me up" ([21.frontend-calendar](21.frontend-calendar/README.md), [24.mobile-chat-app](24.mobile-chat-app/README.md)) |

Honesty note: at **single-primitive** granularity nothing clears 60% — the highest
single concept is 53% (log/event bus, sharding). Tier 1 exists only because these
unions are treated as one concept each; at finer granularity Tier 1 is empty and the
rows below are the ceiling.

## Tier 2 — 25–60% of relevant cases

| Concept | Count | Share | Categories | Why it recurs |
|---|---|---|---|---|
| Distributed append-only log / event bus (Kafka-style) | 23/43 | 53% B | infra | Feeds, payments, ride-hailing, ML, crawler all replay events to multiple consumers |
| Sharding / partitioning (hash, range, geo, single-writer) | 23/43 | 53% B | pattern + storage + technique | Scale always outruns one box; the split decision recurs per domain ([12.bigtable](12.bigtable/README.md)) |
| ★ CDN / edge delivery | 24/50 (20B/4C) | 48% | infra | Any immutable bytes (video, audio, tiles, artifacts, packages) move to the edge ([09.cdn](09.cdn/README.md)) |
| ★ Idempotency keys / at-least-once delivery | 23/50 (19B/4C) | 46% | technique + pattern | Networks duplicate; retries need client-generated dedup IDs, from payments to sync ([44.payment-system](44.payment-system/README.md)) |
| ★ In-memory cache / KV tier | 23/50 (21B/2C) | 46% | infra + storage | Microsecond reads for hot keys in front of the transactional store |
| ★ Blob / object store (incl. content-addressed) | 20/50 (19B/1C) | 40% | storage | Media, artifacts and binaries are immutable and read-heavy everywhere ([07.blob-store](07.blob-store/README.md)) |
| Relational database (incl. sharded) | 18/43 | 42% B | storage | Money, orders and metadata want transactions, joins, strong consistency — SQL is not in every case, but it is the default for transactional truth |
| ★ Cursor & sequence sync (overall) | 15/50 | 30% | technique + pattern | Gap-filling reconnect and pagination without distributed clocks |
| ★ Snapshot + log-replay recovery | 14/50 (12B/2C) | 28% | technique + pattern | Rebuild state fast instead of replaying unbounded journals |
| Rate limiting / admission control | 12/43 | 28% B | technique | Every quota-bearing or abuse-prone surface meters entry ([47.rate-limiter](47.rate-limiter/README.md)) |
| Fanout (fanout-on-write / on-read / hybrid) | 11/43 | 26% B | pattern | Feeds, rooms, notifications; celebrity skew forces the same choice ([02.twitter](02.twitter/README.md)) |
| ★ Precompute-at-write vs derive-at-read | 13/50 (12B/1C) | 26% | pattern + technique | The freshness-vs-read-cost trade-off recurs in feeds, maps, profiles |

## Tier 3 — niche but important

| Concept | Count | Share | Categories | Why it recurs |
|---|---|---|---|---|
| Replication / quorum / consensus (failover, anti-entropy) | 10/43 | 23% B | infra + technique | Durability above single machine ([11.zookeeper](11.zookeeper/README.md)) |
| Moderation / anti-abuse pipeline | 10/43 | 23% B | technique + pattern | Any user input surface pays the same tax |
| Durable work queues (incl. saga/outbox) | 11/50 | 22% | infra + pattern | Slow/fragile side effects need durable handoff ([04.deployment-system](04.deployment-system/README.md)) |
| Lease / lock / fencing tokens | 9/43 | 21% B | technique | Exclusive writers and safe reclaim |
| Primary-connection gateway (WebSocket/long-lived) | 8/43 | 19% B | infra | Realtime surfaces need stateful edge connection fleets ([03.whatsapp](03.whatsapp/README.md)) |
| Search / inverted index | 8/43 | 19% B | storage + technique | Text and tag queries want postings |
| Retrieval-then-rank funnel | 7/43 | 16% B | pattern | Recall cheaply, rank expensively ([15.ml-ranking-system](15.ml-ranking-system/README.md)) |
| Reservation / CAS "hold-then-confirm" | 7/43 | 16% B | technique + pattern | Bookings, tickets, payments, quota — all need expirability ([37.ticketmaster](37.ticketmaster/README.md)) |
| Client trio: optimistic UI / offline-first store-and-forward / virtualized list | 3/7 each | 43% C | technique | Defines the client-side vocabulary (with ~18 further client concepts at ≤6 cases each) |

## Cross-case glue — patterns worth standardizing

1. **Idempotency keys + at-least-once dedup** — YouTube (01), WhatsApp (03), payments
   (44), EV charging (27), ride-hailing (28), stock exchange (50), plus 4 client cases:
   the single most portable contract (client-supplied ID + dedup window).
2. **Append-only distributed log/event bus** — 23 unrelated domains (chat, ride-hailing,
   payments, crawler, ML pipeline, metrics) consume the same "log as backbone" shape.
3. **Cursor/sequence-number sync with gap-fill** — chat (03, 24), feeds (20, 25), Maps
   (26), Kafka consumers (13), stock exchange (50): one vocabulary of cursors, sequences
   and gaps.
4. **Rate limiting / admission control** — from the dedicated rate-limiter case (47) to
   ChatGPT (08), Ticketmaster (37), TinyURL (40) and the crawler (41): token-bucket
   semantics and fail-open/closed ladders.
5. **Snapshot + log-replay recovery** — ZooKeeper (11), stores, Uber (28), Docs (46)
   all recover state the same way.
6. **Fanout choice (write/read/hybrid)** — Twitter (02), Instagram (33), Reddit (34) vs
   chat (03, 30) vs market data (50): one decision with domain-tuned skew.
7. **Reservation/CAS hold-then-confirm with expiry** — Airbnb (36), Ticketmaster (37),
   Uber (28), e-commerce (39), Maps (26) all need the same expirable hold.
8. **Precompute-at-write vs derive-at-read** — feeds (02, 20), ranking (15), typeahead
   (45), metrics (49): one freshness-vs-latency trade-off expressed everywhere.

## Takeaway

A reuse library built from this corpus should start with ~12 canonical concepts, not 200
primitives. First: the caching stack (cache/KV tier, invalidation coherence, herd defense
— one chapter, since it appears in 72% of cases). Second: the event backbone
(append-only log/bus + outbox + idempotency keys + at-least-once dedup as one integrated
pattern, also 72%). Third: the four storage pillars (blob store, relational DB, KV cache
tier, wide-column store) framed as the "system-of-record choice", plus
partitioning/sharding and cursor-based sync as the two mechanisms that ride on top of any
of them. The most defensible reusable *contracts* — concepts that recur verbatim across
unrelated domains — are the idempotency/at-least-once contract, the cursor+sequence sync
contract, and the hold-then-confirm reservation contract: all three can be specified once
and reused by nearly every case in the corpus.
