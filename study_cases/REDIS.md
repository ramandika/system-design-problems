# Redis across the case catalog

Where Redis (merged with memcached-class caches) appears in the 50 study cases, what role it
plays, and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) (caching union,
Tier 1: "caching in any form + invalidation/coherence + hot-key & thundering-herd defense",
36/50 cases). The union is much wider than this doc: Redis is only where the catalog names an
in-memory, key-addressed store specifically.

Headline: **Redis in this catalog is a role, not a product.** Almost every case writes
"Redis-class" or "Redis-shaped" — the contract that matters is *in-memory, key-addressed,
single-key atomic, expirable*. Three distinct jobs recur behind that contract: a **side cache**
in front of an origin (correctness optional), a **hot store** where the serving copy *lives* in
the tier (availability and durability mandatory), and a **session/coordination plane** keyed by
an entity ID (user→gateway, device shadow, rate-limit counters). Only one case studies the
technology itself; two brief mentions use it as a component without sizing it.

- Cases that mention Redis / memcached-class: **15 / 50**
- Designed in the architecture (diagram, sizing, or state lives in it): **11** (02, 27, 29, 34, 37, 38, 39, 40, 42, 45, 47)
- Brief single-line mentions: **2** (03, 01)
- Mention-only, not used as a cache: **1** (09 — memcached appears as a DDoS amplification vector)
- Studied explicitly: **1** (10 = the case itself, covering both Redis Cluster and memcached-style)
- Sweep hygiene: "rediscovers"/"redistribution" matched the grep in 11, 12, 31 — excluded

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [10.distributed-cache](10.distributed-cache/README.md) | The case itself: shared elastic tier modeled end-to-end (Redis Cluster / memcached-style) | Full case |
| [02.twitter](02.twitter/README.md) | Timeline cache is the *fanout target*, not just a read cache; tweet cache for hydration | Core component |
| [42.linkedin](42.linkedin/README.md) | Profile cache, cache-aside with edit-driven invalidation, ~95% hit | Core component |
| [34.reddit](34.reddit/README.md) | Hot caches (comment trees, hot lists, rank inputs) + HINCRBY counters + ban-list cache | Core component |
| [45.typeahead](45.typeahead/README.md) | Per-prefix cache with request coalescing; the designated hot-key absorber | Core component |
| [40.tinyurl](40.tinyurl/README.md) | Layer-2 shared redirect cache, consistent-hash sharded by code; holds `status` too | Moderate |
| [01.youtube](01.youtube/README.md) | Counter store for the hot recent window (~10 min staleness tolerance) | Brief |
| [37.ticketmaster](37.ticketmaster/README.md) | Waiting-room admission state as a Redis ZSET; "just a Redis" seat-state warning | Core component |
| [38.leetcode](38.leetcode/README.md) | Leaderboard as one ZSET per contest (projection, rebuildable); Redis token-bucket limiter | Core component |
| [29.uber-eats](29.uber-eats/README.md) | Socket session state, geo-sharded live supply store mirror, per-market stock-out overlay | Core component |
| [39.e-commerce](39.e-commerce/README.md) | Cart store: DynamoDB/Redis-class KV evaluated and *chosen* for Option C | Designed option |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Device shadow in Redis/DB beside stateless OCPP gateways; token/partner-lookup cache | Moderate |
| [03.whatsapp](03.whatsapp/README.md) | "Who is connected where" registry mapping user_id → gateway instance | Brief |
| [47.rate-limiter](47.rate-limiter/README.md) | Quota authority + counter store; the "≤1 ms of budget" latency arithmetic | Deep (sized) |
| [09.cdn](09.cdn/README.md) | Memcached named only as an amplification/reflection DDoS vector | Mention only |

## Per-case notes

### The case itself

**[10.distributed-cache](10.distributed-cache/README.md)** — A shared elastic tier
(Redis Cluster / memcached-style treated as one class) serving three deliberately separated job
classes. Its central split, made explicit in [03-architecture.md](10.distributed-cache/03-architecture.md):
**object-class** keys accept eviction and stale reads (synchronously cheap, async invalidation,
replication only for failover) versus **session-class** keys that have *no origin to rebuild
from* — sessions are never evicted while live, writes sync to one replica before the primary
acks, and a durable fallback store catches what the tier loses. Everything else in the case
flows from the "consistency layer pretending to be a performance layer" framing: the
cache-aside race that pins a stale value for a full TTL, request coalescing, hot-key
mitigation, LRU/LFU eviction's memory-vs-hit-rate curve. Every Redis-class use elsewhere in
this catalog is an instance of one of its job classes.

### Core component: fanout / write-path materialization

**[02.twitter](02.twitter/README.md)** — The strongest single use in the catalog: the Redis
timeline cache exists because of the *write* path. Fanout workers `RPUSH` tweet IDs into
~800-entry per-user lists ([04-deep-dives.md](02.twitter/04-deep-dives.md): the 2012-era
Fanout component pipelined 4,000 recipients per batch into a replicated Redis timeline cache);
one timeline read is then a single Redis multi-get instead of a merge over `following_count`.
Sized in [02-scale-estimates.md](02.twitter/02-scale-estimates.md): ~10 KB/user hot × 200M DAU
≈ ~2 TB, ~4 TB with replicas across a few hundred nodes, consistent-hash by user_id (estimate).
Two consequences the case works through: tweet-body hydration separately sits behind a tweet
cache at ~95% hit ratio needing 200-key batched multi-gets, and tweet *deletion* becomes
"rewrite millions of timelines or tombstone at hydrate". The named trade-off: fanout-on-write
buys O(1) reads at the price of the 180B/day fanout-insert rate; that rate is *why* the Redis
tier exists at all.

### Core component: side cache (cache-aside, origin-repairable)

These cases use the tier as a discardable accelerator. The consistent contract: a miss must be
cheap to repair, and invalidation must be explicit where staleness would be user-visible —
TTL-only is reserved for content with a stated minutes-wide freshness tolerance.

- **[42.linkedin](42.linkedin/README.md)** — hot profiles in Redis, cache-aside with
  edit-driven invalidation, ~95% hit ratio ([03-architecture.md](42.linkedin/03-architecture.md)).
  Canonical profile stored viewer-independent; ACL applied *after* fetch — the cache key does
  not have to understand permissions.
- **[34.reddit](34.reddit/README.md)** — Redis caches comment tree pages keyed
  `(post_id, sort, prefix_page, version)` and community hot lists keyed
  `(subreddit_id, sort, window)`; invalidation is **explicit and versioned, not TTL-only**
  ([03-architecture.md](34.reddit/03-architecture.md)) because thread reads are extreme — a
  score-only delta does not bump the tree version, only the counts. Also two auxiliary uses:
  per-item counters as Redis `HINCRBY`-class eventually-per-second aggregations
  ([04-deep-dives.md](34.reddit/04-deep-dives.md), Option B's stated cost: seconds of vote
  lag), and a ban-list cache keyed `(subreddit, user)` with explicit invalidation on events —
  deliberately *not* a synchronous mod-service read per content write.
- **[45.typeahead](45.typeahead/README.md)** — the whole serving system is a cache architecture
  against a Zipfian key distribution; Redis is the per-prefix tier with **request coalescing**
  ([04-deep-dives.md](45.typeahead/04-deep-dives.md)): one in-flight fetch per prefix per
  replica, duplicates await the leader, TTL staggered by layer. Failure mode named and costed:
  Redis outage → RAM-index origin serves and p99 rises — coalescing caps origin load at
  *unique prefixes per interval*, not raw QPS; stale-on-error beats erroring mid-keystroke.
- **[40.tinyurl](40.tinyurl/README.md)** — three-layer lookup (in-process LRU → Redis-class
  shared cache consistently hashed by code → KV store; the second layer also holds `status`
  so abuse takedowns propagate without rewriting URLs). The case explicitly cross-references
  [10.distributed-cache](10.distributed-cache/README.md) for the sharding machinery.
- **[01.youtube](01.youtube/README.md)** — brief but contract-complete: playback counts go to
  "Redis for the hot recent window + DB for authoritative totals, periodic reconciliation"
  ([04-deep-dives.md](01.youtube/04-deep-dives.md)), with ~10 min staleness tolerance and
  approximate bot-dedup at the edge. No per-view rows anywhere.

### Core component: hot store (state lives in the tier)

The opposite contract from the side cache: the cache is not discardable in the design's mind —
some state's *serving copy* is here, so the tier's failure modes get designed mitigation.

- **[37.ticketmaster](37.ticketmaster/README.md)** — the waiting room holds its per-on-sale
  admission order as a **Redis sorted set + rotating issue epochs** so late joiners cannot
  retro-inject ahead; positions assigned FCFS with a documented randomized tie-break
  ([03-architecture.md](37.ticketmaster/03-architecture.md)). Deep dive names the failure
  mode ("waiting-room Redis dies mid-sale → positions lost, fairness gone") and the fix:
  the state is small (2M-entry ZSET + tokens, ~100–200 MB, estimate), so replicate with
  automatic failover *and* log every position assignment to the event log for rebuild —
  "snapshot + replay… often forgotten for 'just a Redis' seat state."
- **[38.leetcode](38.leetcode/README.md)** — Option B for the leaderboard: one Redis-shaped
  **ZSET per contest** keyed `(score, penalty, id)` ([04-deep-dives.md](38.leetcode/04-deep-dives.md)):
  atomic `ZADD`/`ZRANK` at O(log N), top-100 via `ZRANGE`, ~30 MB per contest fits a shard.
  Loses named: Redis's single-threaded ZADD path under contest burst (real but manageable),
  and cross-shard "global rank" needs a shard map (rank within shard + sum of shard sizes
  above). The load-bearing line: **the verdict log is the truth, the ZSET is only a projection**
  — re-runs that flip verdicts after the contest are survivable *because* the deltas were not
  kept only in Redis. A second use: the submit-rate limiter is a Redis token bucket that
  **fails open to a local per-instance bucket** — brief under-limiting beats blocking all
  submissions.
- **[29.uber-eats](29.uber-eats/README.md)** — three uses at three tolerances
  ([02-scale-estimates.md](29.uber-eats/02-scale-estimates.md),
  [04-deep-dives.md](29.uber-eats/04-deep-dives.md)): tracking/push session state (~4M
  sockets × ~2 KB ≈ 8–10 GB live RAM, ephemeral), a geo-sharded **live supply store**
  (in-mem + Redis mirror) fed by downsampled courier pings for dispatch reads, and a
  **per-market hot overlay Redis** for stock-out / restaurant-closed flags — a *fast lane*
  that bypasses the versioned catalog lane because those flags are *state*, with a
  seconds-to-minutes tolerance, checked at cart-add and order-accept.
- **[39.e-commerce](39.e-commerce/README.md)** — the one case that makes Redis-class storage a
  **chosen, evaluated option** for authoritative-ish data: carts in a
  DynamoDB/Redis-class KV store, TTL'd by access, with a write-behind event stream
  ([04-deep-dives.md](39.e-commerce/04-deep-dives.md)). The spelled-out trade-off: wins
  memory-latency at cart QPS, TTL handles abandonment for free, the store fails *small*; loses
  durability semantics — acceptable for carts, and the case enforces it ("checkout never
  trusts the cart store alone"; account carts additionally mirrored to order-core). Cross-references
  case 10 on ephemeral-tier placement.

### Session / routing / coordination plane

- **[27.ev-charging-network](27.ev-charging-network/README.md)** — OCPP gateway acceptors are
  stateless because the **device shadow** (config, pending commands, session context) sits in
  Redis/DB; any node serves any charger after reconnect
  ([03-architecture.md](27.ev-charging-network/03-architecture.md)). Explicit trade-off
  ([04-deep-dives.md](27.ev-charging-network/04-deep-dives.md)): one extra store lookup per
  connection vs sticky gateways — which would turn every gateway deploy into a self-inflicted
  reconnect storm. Auth holds a token cache + partner lookups on the same tier.
- **[03.whatsapp](03.whatsapp/README.md)** — brief: a "who is connected where" registry (Redis
  or in-memory cluster) maps user_id → gateway instance, updated on connect/disconnect
  ([04-deep-dives.md](03.whatsapp/04-deep-dives.md)); message routing looks up and pushes.
  Same shape as 27's shadow, stated in one line rather than built.
- **[47.rate-limiter](47.rate-limiter/README.md)** — the deepest treatment of what a
  Redis-class tier can and cannot be. Two clusters ([03-architecture.md](47.rate-limiter/03-architecture.md)):
  a **quota authority** (lease grants for local buckets, batched ~500 ms, plus atomic
  decide-and-update scripts for central decisions) and an ~10-shard **counter store** for the
  ~5% hard-quota tier, decisions as single atomic Lua-class scripts (increment + compare +
  expire), chunked counters for hot keys. The arithmetic that shapes everything
  ([02-scale-estimates.md](47.rate-limiter/02-scale-estimates.md)): a Redis-class shard
  sustains ~100K simple ops/s, ~130 B/entry (hash-table overhead dominates), and "this one
  number is why 'just rate limit in Redis' is not an architecture" — same-DC ~0.3–0.8 ms fits,
  edge→central at 10–100 ms fails by 10–100× against a ≤1 ms budget. The fail-open vs
  fail-closed trade-off is a full deep dive: fail closed makes counter-store availability
  bound the whole platform enforcement tier.

### Mention only

- **[09.cdn](09.cdn/README.md)** — memcached appears in the DDoS deep dive as one of the
  amplified/reflection UDP vectors (alongside DNS and NTP), absorbed by egress-rate limits and
  BCP38-style source validation ([04-deep-dives.md](09.cdn/04-deep-dives.md)). The CDN's own
  edge cache tier is HTTP-object caching, not a Redis/memcached-class store; no cache-tier
  component here.

## The portable pattern

Every substantive use reduces to one contract, cross-linked to
[10.distributed-cache](10.distributed-cache/README.md) as the reference implementation:

1. **Pick the role before the product.** Side cache (correctness optional), hot store
   (state lives here), or session/coordination plane (no origin, availability mandatory). Case
   10's object-vs-session split is the decision; 39 and 47 depend on getting it right.
2. **Cache-aside with explicit invalidation where staleness is user-visible; TTL where the
   freshness budget is minutes.** 42 and 34 invalidate on edit/version-bump; 01 and 29's
   overlay are happy with reconciliation and a fast lane. 10's cache-aside race (stale value
   pinned for a full TTL) is the reason TTL-only is never the default answer.
3. **Single-key atomicity is the real API.** `HINCRBY` counters (34), `ZADD`/`ZRANK` per
   contest (38), admission order in one ZSET (37), atomic decide-and-update scripts (47).
   When the operation spans keys, design a shard map instead (38's global-rank) or a
   per-domain-shard layout (29's geo-sharded supply store).
4. **Consistent hashing sets placement; it does not fix load.** Hot keys are a named failure
   mode in 10, 45, and 47 — mitigated by request coalescing (45), chunked counters + leases
   (47), or keeping the whole working set small enough to replicate and rebuild (37):
   the counter is "small for what it buys" (~100–200 MB).
5. **Always design what the origin does on tier failure.** The catalog never leaves this
   unstated: fail open to a local fallback (38, 47), stale-on-error (45), degraded guests with
   a re-add cost (39), sized-for-miss-storm origins (10, 45's coalescing cap).
6. **Rebuildability comes from outside the tier.** ZSETs and shadows are projections of a
   source of truth elsewhere — the verdict log (38), the event log of position assignments
   (37), the auth plane's durable copy (27, 10's session fallback). "Just a Redis" is safe
   exactly when someone wrote down how it gets rebuilt.
7. **Latency arithmetic decides placement.** 47's ≤1 ms budget, 02's batched multi-get
   hydration, 27's connection-time lookup — all assume co-located same-DC tier. Across a metro
   link, the tier is disqualified, not made bigger.
