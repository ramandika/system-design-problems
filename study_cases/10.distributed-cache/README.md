# Case 10 — Distributed Cache

**The system**: a shared elastic cache tier (Redis Cluster / memcached-style) sitting between
application services and the durable stores. It serves three distinct jobs — read scaling for
hot objects, session/state storage, and fast lookup for hot datasets (features, rate-limit
counters, timelines) — with sharding, replication, eviction and invalidation.

**Why it's hard**:

- A cache is the only major component whose **correctness is optional but whose failure is
  not**: any value can be evicted or served stale, but if the tier falls over, the full read
  load lands on databases that were sized assuming ~90% of it never arrives.
- Two hard problems hide behind one word "sharding": keeping data evenly distributed as
  nodes join and leave (consistent hashing / hash slots), and doing it **without blowing the
  hit rate** — every key that moves to a new node is a miss.
- The cache is a *consistency* layer pretending to be a *performance* layer. Cache-aside has
  a documented race that can pin a stale value for a full TTL; invalidation ordering between
  app, cache and database is a distributed-systems problem, not a config flag.
- Load is adversarial: one hot key (a celebrity, a flash-sale SKU) can saturate a single
  shard while the other 99 sit idle, and a cold cache restart triggers a thundering herd
  that can take the origin down.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: consistent hashing with virtual nodes vs hash slots
(Redis Cluster's 16,384 slots), primary–replica sharded replication and failover,
cache-aside vs write-through and their races, request coalescing, hot-key mitigation,
LRU vs LFU eviction and the memory-vs-hit-rate curve.

**Try it first**: before reading 03, answer: (1) session data has no origin of truth to
rebuild from — how does that change what the cache tier must guarantee for it? (2) a key is
deleted from the cache; the app then writes a new value to the DB; a concurrent reader who
missed the cache a moment earlier has just fetched the *old* DB row and is about to SET it —
walk the interleaving that pins stale data. (3) 100K QPS lands on one key that sits on one
shard — name three different places you can spread that load.
