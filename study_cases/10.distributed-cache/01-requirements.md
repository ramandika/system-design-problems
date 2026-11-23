# 10.distributed-cache — 01 Requirements

## What the system is

Not an application — a **service**: an elastic, sharded, replicated in-memory key-value tier
that other services in the org treat as their cache and their session store. Concretely, the
behavior contract is close to Redis Cluster with memcached-style client sharding discipline.

## Functional requirements

- **F1. Key-value API.** Get/Set/Delete of string or binary values, plus atomic operations
  the app layer needs: INCR/DECR (counters, rate limits), small set/hash variants for
  structured objects. No queries by value, no secondary indexes, no scans in the hot path.
- **F2. Hash-based placement.** Every key maps to one shard by hash; the client (or a
  proxy) computes shard id from the key and routes there. Keys can be tagged so related keys
  `{user:42}:a`, `{user:42}:b` co-locate for multi-key ops.
- **F3. Sessions as a resident class.** Session tokens live here with sliding expiry and NO
  backing database of truth in the hot path (durable fallback exists but is not read on
  request). Losing a session forces a re-login: annoying but bounded.
- **F4. TTL and sliding expiry.** Every key carries a TTL from 30s (hot object fragments)
  to 14 days (sessions); access can extend the TTL for session-class keys.
- **F5. Invalidation hook.** Writers can DEL (or version-bust) keys after committing a
  database write; services rely on this for cache-aside correctness.
- **F6. Elastic resharding.** Adding/removing nodes migrates keys online with bounded
  client-visible error rate (<1% failed ops during a move, target), without stopping traffic.
- **F7. Multi-region awareness.** Reads hit the local region's tier; keys are region-local
  (sessions pinned to home region, hot objects replicated read-only per region).

## Non-functional requirements

- **N1. Latency**: p99 ≤ 1.5 ms per read operation intra-region, and for async-path writes
  (an in-memory hop with one network round trip); the tier is not allowed to be the tail.
  Session-class *writes* are the deliberate exception — they sync to one replica before the
  ack and carry a few-ms cross-zone hop instead (see N2 and deep dive 2).
- **N2. Availability**: the tier as a whole targets 99.99%; a single shard's primary failing
  promotes a replica in ≤ 15 s. For the *session* class this promotion loses no
  already-acked writes (session writes sync to a replica before the ack — 03 component 3,
  deep dive 2); the *object* class may lose writes acked in the last replication interval,
  which is acceptable because object values are reconstructible from origin (N6).
- **N3. Throughput**: peak aggregate ~1.7M ops/s per region (~1.5M reads + ~170K writes;
  see 02-scale-estimates),
  scaling linearly by adding shards.
- **N4. Hit rate**: for the object-read class, ≥ 90% of requests must be served from cache
  at steady state — this is a *capacity* requirement, because the database is sized for
  the residual.
- **N5. Memory**: sized so the tier holds ≥ 95% of the *hot* (frequently-accessed) working
  set; cold data may be evicted freely.
- **N6. No invented durability**: values are reconstructible from origin (except sessions);
  acknowledged session writes survive failover.

## What is explicitly OUT of scope

- **Search / range queries / secondary indexes** — this tier is hash-addressable only.
  Anything needing queries belongs in the origin databases.
- **Durable general-purpose storage.** The cache is not the system of record for anything
  except sessions. "It fits in memory so put the DB in Redis" is out of scope.
- **Transactions across keys or across shards.** Multi-key atomicity is limited to keys
  sharing a hash tag on one shard; no cross-shard transactions (that is a job for the origin
  stores or a coordination service).
- **Pub/sub and streams as a product feature.** Pub/sub may exist inside the implementation
  for config/failover signaling; it is not part of the offered API.
- **Cross-region synchronous replication.** Region-local writes; async read-only replicas
  per region at most. Cross-region consistency is the application's problem.
- **Vector/embedding search, disk-backed tiers, CDN.** Edge caching is a separate layer (see
  YouTube case, 01.youtube). This tier concerns datacenter-internal caching.

## Key scope decision

The single most consequential scope choice: **the tier is allowed to lie** (serve stale or
dropped values) for the *object* class, and **is not allowed to lie** for the *session* class.
Everything downstream — replication mode, failover behavior, invalidation effort — is set
per data class instead of one global policy. One global "strong consistency" setting would
cost latency and availability for data that does not need it; one global "best effort" would
break sessions. Class-based policy is the compromise.
