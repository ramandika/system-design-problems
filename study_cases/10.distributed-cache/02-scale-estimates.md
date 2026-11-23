# 10.distributed-cache — 02 Scale Estimates

All numbers are rough estimates from stated assumptions; arithmetic is shown. One region,
one cache tier; the fleet is N regions of roughly this shape — a deliberately generic
"large consumer service" load (comparable in shape to cases 01 and 02, not equal to them).

## Traffic assumptions

- **DAU: 50M** users in the primary region's catchment (estimate).
- Requests per user per day: ~200 app requests × ~8 cache reads each = **1,600 cache GETs
  per active user per day** (plus ~1 user-data SET; the L2 write volume below is dominated
  by counter INCRs and post-commit DELs, not user SETs). Estimate.
- Daily get volume: 50M × 1,600 = **80B GETs/day**.
- Average QPS if every lookup hit this tier: 80B / 86,400 s ≈ **926K QPS**, ~2.8M QPS peak
  (factor 3 over mean). But not every app request reaches the *shared* tier — L1s exist.
  Layer the caches:

- **L1 (per-service in-process + node-local)** catches ~40% of lookups (repeat reads within
  one request handler or one instance's recent keys) — never leaves the service host.
- **L2 (this shared tier)** is asked for the remaining ~60%: 80B × 0.6 ≈ **48B GETs/day** ≈
  556K QPS average, **~1.7M QPS peak** (estimate, factor 3).
- Of L2 traffic, **~90% is reads, ~10% writes** (SET/DEL/INCR): peak ≈ 1.5M GET/s +
  170K writes/s. Round: **1.5M read QPS peak per region**.

## Key and value profiles (estimates)

| Class | Share of keys | Avg value size | TTL |
|---|---|---|---|
| Hot objects (profiles, product/entity blobs, hydrations) | ~60% | ~1 KB | 5 min |
| Sessions | ~25% | ~2 KB | 14 d sliding |
| Counters / rate limits | ~10% | ~16 B | 1 min–1 d |
| Hot datasets (feature vectors, timeline fragments) | ~5% | ~10 KB | 1 h |

Weighted average value ≈ 0.6·1 KB + 0.25·2 KB + 0.1·16 B + 0.05·10 KB = 600 B + 500 B
+ 1.6 B + 500 B ≈ **1.6 KB**, call it
**~2 KB** including key, overhead and per-entry metadata (estimate).

## Storage math

- Requested keys/day (L2 tier, reads + write refreshes): ~48B distinct-ish lookups; but the
  *stored* population is what matters. Assume **500M actively-touched keys** in the tier at
  any moment (distinct hot keys, estimate).
- Steady-state memory: 500M × 2 KB ≈ **1 TB of values per region**.
- Target memory utilization 70% (eviction headroom, hash-table load factor, fragmentation —
  LRU's admit-window needs slack): needed capacity ≈ 1 TB / 0.7 ≈ **1.4 TB**.
- Replication: 1 primary + 2 replicas = 3 total copies → 1.43 TB × 3 ≈ **~4.3 TB RAM per
  region**. Say **~4.5 TB/region** (estimate).
- Shard sizing: 64 GB nodes → 1.4 TB primary footprint / 64 GB ≈ **22 shards**, round to
  **30 shard groups** (memory headroom + future growth), each 1 primary + 2 replicas =
  **90 nodes per region**. Estimate.
- Sessions: 25% of 500M = 125M sessions × 2 KB = 250 GB; if DAU 50M and most users hold
  1–2 active sessions, 125M is consistent. Show check: 50M DAU × 2 sessions × 2.5 KB
  (session + overhead) ≈ 250 GB. Consistent.

## Traffic → per-node load

- Peak reads 1.5M QPS over 30 primaries ≈ **50K ops/s per primary** (memcached-class nodes
  handle 100K+ ops/s; Redis-class single-threaded ~80–100K small-op/s when pipelined —
  publicly reported ballpark, so we stay at ~50% utilization).
- Network per primary at peak: 50K ops/s × 2 KB × ~1.05 (protocol overhead) ≈ **105 MB/s**
  ≈ **840 Mbps** — fits a 10 Gbps NIC with replicas each receiving the write half; the
  *replication fan-out* is 1 write → 2 replicas ≈ 2 × write-bw extra on the primary's
  egress. Write peak 170K ops/s / 30 ≈ 5.7K writes/s/shard × ~1 KB avg write value ≈
  6 MB/s replication per shard — negligible vs reads. Consistent.
- Connections: a few hundred thousand client connections per node if clients keep pools;
  cap client pools (e.g., ≤ 200/node/service) or this is the first thing to break.

## Bandwidth to/from origin

- Miss rate 10% of 1.5M peak reads = **150K origin reads/s at peak**. Provisioning the DB
  anywhere near the residual fails fast: sized at ~2× the residual (300K QPS), a *fully
  cold* tier (deploy/restart/eviction catastrophe) asks the DB for ~1.5M QPS — **5× a
  300K-QPS provision, 10× the residual**. That asymmetry is why provisioning is set at
  ~10× the steady-state residual (see the consistency check below) and why request
  coalescing and staged warmup are load-bearing, not niceties (see deep dives 4 and 6).
- Session class: no origin read on the request path at all, so the cold-DB cliff is driven
  by the object class only.

## Consistency check on the numbers

- 80B client reads/day ÷ (50M × 200 app requests) = 8 cache reads per app request —
  plausible for hydration-style reads (estimate).
- 90% hit at L2 + 40% at L1 → overall ~94% of lookups never reach origin; DB provisioning
  must cover the remaining 6% *plus* coalescing collapses — sized at ~10× the steady-state
  residual to survive a cold tier, which is the conservative part (estimate).
- 4.5 TB RAM/region is cost-real: ~18 commodity 256 GB-class hosts (4.5 TB = 4,500 GB /
  256 GB ≈ 18), or the ~90-node 64 GB fleet above — the tier is a mid-single-digit-% cost
  line next to region databases. Estimate.
