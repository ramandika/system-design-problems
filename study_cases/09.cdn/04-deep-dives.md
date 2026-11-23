# 04 — Deep dives

## 1. Edge topology: anycast vs unicast routing

The request-routing decision: how does a client in Jakarta reach a POP that is fast, alive,
and not overloaded?

**Option A — anycast with a global routing policy** (announce the same /20ish prefixes
from every POP; BGP brings the client to the "network-nearest" POP):

- wins: zero per-request lookup cost, no DNS roundtrip on the hot path, instant failover
  semantics (withdraw the prefix and traffic re-converges), scales to attack absorption
  because every receiving POP shares the address (deep dive 6).
- loses: BGP optimizes for AS-path length and routing policy, not latency — a client can be
  routed to a POP that is 100 ms slower than the best one; you cannot load-balance by
  traffic because you don't choose the mapping; POP overload must be shed by withdrawing
  prefixes, which is coarse and slow (BGP reconvergence in tens of seconds to minutes).

**Option B — unicast with DNS/GSLB steering** (each POP has unique IPs; a DNS mapping
service answers each lookup with a POP chosen by load + latency, the Akamai-style map):

- wins: precise, per-client mapping balancing latency *and* load; changes take effect in
  TTL seconds, not BGP minutes; capacity can be added without touching routing.
- loses: every session pays a DNS lookup; the mapping service itself becomes a target and
  a single point of behavior; failover happens at DNS-TTL granularity (clients cache the
  answer); an attack aimed at one mapped IP is not spread across the fleet.

**Chosen: anycast as the base, with unicast/DNS steering as an escape hatch.** Anycast is
the only option whose happy path requires nothing per-request and whose failure path is
router-native — and the same property that makes failover fast is what makes DDoS
absorption work (deep dive 6). The known losses are mitigated explicitly:

- *Bad BGP mappings*: continuously measured latency per (client-region → POP) and, for the
  small share of clients that anycast serves badly, steer via DNS to a POP-specific
  hostname. This splits the fleet into "anycast for the 95%+, steering for the tail".
- *POP overload*: the POP advertises degradation by withdrawing a /24 *sub-portion* of its
  coverage (per-geo announcements from the same POP), so overload sheds the most distant
  traffic first — coarse, but it works with routing rather than against it.
- *State loss on re-homing*: a failover event moves a client to a POP with a cold cache
  for its session. TLS session resumption works fleet-wide (shared ticket keys, deep dive
  on keys below), so re-homing costs a round trip, not a renegotiation; per-client edge
  state (other than the cache itself) is deliberately kept near zero.

**As actually built (publicly described)**: Cloudflare and most cloud CDNs run the anycast
model for exactly this reason (vendor-documented); Akamai built the unicast/GSLB map
model, and the industry has converged on treating both as the two canonical designs. The
modern synthesis (anycast + targeted steering) is what we chose.

## 2. Cache key design and hit-rate economics (THE dive)

The CDN's P&L is hit rate: at the estimates' numbers (misses are 15% of *bytes*, i.e. 85%
byte hit rate), moving byte hit rate from 85% → 92.5% cuts origin-bound bytes by ~2×
((1−0.85)/(1−0.925) = 15/7.5 = 2) and is worth more than any hardware initiative. The cache
key defines what "the same object" means, and every correctness requirement tightens it.

The naive key is `host + raw URL`. Every real CDN key is a pipeline of decisions:

- **Query-string normalization**: `?utm_source=…&fbclid=…` are tracking params, not cache
  identity. Stripping a *customer-configured denylist* (or keeping an allowlist) turns a
  space of near-duplicate URLs into one key. Wins: large hit-rate gain on real-world
  traffic, where a large fraction of uncached misses are the same asset under different
  query strings (measured repeatedly across CDNs; treat as qualitative). Loses: an
  incorrectly stripped param that actually selects content returns wrong content. The
  denylist is per-customer, default-safe.
- **Vary/segmentation**: cookies, device class, auth state that alter the body must widen
  the key. `Vary: User-Agent` widened by every UA string destroys hit rates (each browser
  build becomes a different key) — CDNs key on a normalized *device class* (mobile/
  desktop/tablet) instead. Trade-off: slightly wrong rendering for devices between classes
  vs hit-rate collapse from per-UA keys; we pick classes and let customers opt into
  stricter keys.
- **Signed URLs / token-in-key**: for paid content, the auth token is in the query string.
  Keying on the full URI would mean no sharing; signed-URL schemes authenticate and then
  key *without* the token, so all authorized clients share one cached copy. Trade-off:
  any bug in token verification becomes a cache-poisoning vector — the verification must
  happen before the cache key is formed, and the signed portion must be exactly declared.
- **Negative caching**: 404/403/5xx responses get short TTLs keyed the same way. Wins: a
  broken link or a hammering crawler for `/nonexistent` becomes 1 origin request per TTL
  instead of millions. Loses: a resource that appears moments after a failed request stays
  missing up to the negative TTL — keep negative TTLs short (seconds) and purge-able.
- **Multi-tenancy coupling**: one POP caches for ~1M customer domains. A cache eviction is
  global across customers via a shared NVMe pool; a single tenant's traffic spike evicts
  other tenants' objects (shared-pool fairness problem). Mitigation: per-customer cache
  partitions (disk allocation/fair-share admission per tenant class) — trade-off: fewer
  cross-tenant evictions vs lower peak utilization of the same hardware; we partition by
  customer tier, not per customer (per-customer partitioning at ~1M tenants is unusable —
  pools per tier keep partition count in the dozens).

- **Trade-off made explicit**: every key-widening feature (cookie segments, strict Vary,
  auth headers in key) buys correctness and loses hit rate; every narrowing (query strips,
  device classes) buys hit rate and risks wrong-content bugs. The design answer is not a
  global choice: the *default* key is maximally shareable and correctness-sensitive
  widening is customer-controlled, because the customer — not the CDN — owns the blast
  radius of serving the wrong variant.

**Multi-tenant co-tenancy note**: popular tenants naturally displace long-tail tenants in a
shared cache; the shared tier's hit rate for a small customer can silently collapse during
another customer's launch. This is the cache-locality version of Twitter's celebrity
problem ([02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md)) — one outlier's fanout
(spike in traffic there, eviction pressure here) consuming a shared resource other tenants
depend on; the partitioning by tier is the same class of mitigation as hybrid fanout:
bound the outlier's blast radius.

## 3. Tiering and origin shielding

Serving 23M req/s from ~300 POPs directly to origins on every miss would generate ~1M+
req/s of origin fetches spread as *uncorrelated* singletons — origins would face a load
proportional to (misses per POP × POPs) rather than to actual cold objects. Tiering exists
to collapse those singletons.

Design: `POP (L1 DRAM + L2 POP-disk) → regional shield (L3) → origin`, with each POP
mapped to one nearby shield for each origin-geo. Misses at the edge that miss the shield
are coalesced at the shield (one origin request, N POPs served); hits at the shield
backfill the edge.

- **Trade-off**: one extra cache layer wins a large hit-rate multiplier (regional demand
  aggregation: the shield sees all edges' misses for a region, so its cache is far denser
  than any single edge's) and cuts origin QPS by the ratio of edges-per-shield; it loses
  one additional hop of latency on misses and adds capacity/stickiness complexity. We pay
  the hop because misses are dominated by byte-heavy objects, where the shield's
  dense-cache advantage dominates latency arithmetic; and dynamic (non-cacheable) traffic
  rides the same hop only for connection-reuse benefits.
- **Where the tier boundary sits is a load decision, not a constant**: a POP very close to
  users in an expensive metro has small disks (space is the constraint of the most
  expensive real estate) — the same inversion YouTube observed, where the tier nearest
  users stores the *least* ([01.youtube/04-deep-dives.md, deep dive 3](../01.youtube/04-deep-dives.md)).
  We mirror that: edge tier holds the hottest per-metro slice; shields hold the regional
  long tail.
- **Consistent mapping**: POP→shield assignment is a stable hashing of
  (customer-origin, edge-POP) into shields, changed only via the control plane — this is
  why coalescing at the shield works: all edges miss into the *same* shield, where their
  requests collapse.
- **Failure**: a shield's loss degrades that region's edges to direct-origin fetch (hit
  rate drops for cold content, availability does not). Trade-off: keeping a hot standby
  shield per region doubles shield hardware idle vs. minutes of degraded hit rate during
  failover — we run shared active/active shields (two per region, each covers all edges;
  the mapping just flips) and accept the doubling of regional shield capacity.

## 4. Invalidation at scale: propagation time vs correctness

A customer renames `logo.png`; 2×10^12 requests/day are in flight against caches holding
the old bytes. The contract "purge API makes the old version unreachable everywhere"
collides with physics: you cannot atomically un-send bytes from ~300 POPs.

Design: invalidation is a *log shipped per key*, not a synchronous global operation.

- **Hard purge**: the key is removed; next request is a full miss/re-fetch. Implemented as
  a stamped "purged-up-to generation G" watermark per (customer, cache-node): each cache
  node stores its version counter; a purge pushes the customer's watermark, and any key
  whose populate-version < watermark is treated as absent (the key is *logically* deleted
  without a physical scan — deleting from petabytes of L2 storage is not a thing you do
  synchronously).
- **Soft purge (default)**: the key is marked stale, not absent; the very next request
  revalidates (If-None-Match) against origin and serves stale to concurrent requests while
  revalidating (stale-while-revalidate as a purge semantic). Wins: no origin request
  stampede (deep dive 5 applies — a purge of a hot key with hard-purge semantics creates
  the exact thundering-herd it's meant to cure); loses: one more round trip on the first
  request after purge.
- **Surrogate keys / tag purge**: customers tag responses (`Surrogate-Key: product-42`),
  the purge API takes tags. This is the only purge that scales to real workflows
  ("content changed for this one product" ≠ "purge by URL list, 10^6 at a time"). The
  control-plane cost: tag→key sets must be enumerable somewhere. Trade-off: (a) maintain an
  inverted index of tag→keys globally — massive, but purges are O(index lookup); or (b)
  embed a tag bloom/bitmask as part of the cached object's metadata and purge by *scanning
  caches for the tag* — no index, purge latency ~scan time. Chosen: (a) for hot
  customers/high fan-out tags, with the index built asynchronously from access logs (the
  same log pipeline billing uses: the metadata already flows) — trade-off: a tag purge of a
  recently-created, not-yet-indexed tag can miss keys for seconds; acceptable vs. real-time
  index maintenance on 23M req/s.
- **Propagation time vs correctness, stated**: we promise purges landing fleet-wide in
  seconds (fastly-class public claims ~150 ms global purge, vendor-reported; Cloudflare
  documents purges under ~30 s; our design target is single-digit seconds end-to-end),
  and *correctness-with-grace* is defined as: no request after time T is served from
  pre-purge state, where T = push-fanout bound (~5 s, est.). Requests *in flight* during
  the purge may see old bytes — that is the boundary of the contract, and selling exactly-
  atomic purge would require synchronously draining 2×10^12 req/day of in-flight requests,
  which no operator does.

## 5. The hot-object problem: one cache key at 10M req/s

A launch announcement, patch download, or viral image: one URL receiving 10M+ req/s
(estimating from "publicly reported ~200M req/s" attack-class floods and launch traffic
that is a real fraction of fleet peak). One object is trivially cacheable — the problem
isn't storing it; it's that every layer's *coordination* around it becomes a hotspot:

- **Expiry synchronization is the real failure**: if the hot key expires on all POPs at the
  same instant (same TTL start), every POP misses at once → 300+ edge misses → collapse at
  the shield if it also just expired → all fall to origin together. Fix: **request
  coalescing at every tier** — the cache node owning the key takes a distributed lock (or
  leader-per-key election), fetches once backfilling concurrent waiters ("collapse
  forwarding"), and TTLs are jittered by a random fraction (e.g. +0–10%) so expirations
  across POPs desynchronize. Trade-off: coalescing adds lock coordination on the miss path
  (a contended lock on a 10M req/s key is itself a hotspot — so locks are per-key, held
  for the fetch duration only, and *hits never touch the lock*); jitter trades up-to-10%
  earlier staleness for expirations that never line up. We pay it: the alternative is
  origin death, which is the thing the customer bought the CDN to prevent.
- **The cache node owning the key is hot at every layer**: hash-table buckets, NIC queues,
  TLS CPU on that POP. POP-level: the owning server for the hot key becomes the POP's
  chokepoint. Fix: split the *serving* from the *storage* role for hot keys — once a key
  is flagged hot (adaptive metering of req/s per key), replicas of the object are pushed
  to every cache server in the POP so reads are lock-free and evenly spread, and the
  origin-fetch duty stays with the elected owner. Trade-off: read-replication of hot keys
  multiplies their memory/disk footprint N-ways (acceptable: hot keys are few and small
  *by definition* — a too-big-for-DRAM hot object is a different tier problem, solved by
  the NVMe layer's page cache, not replication).
- **Revalidation storms after purge**: soft purge (deep dive 4) + coalescing + stale-
  while-revalidate compose: only the first post-purge request reaches origin, everything
  else waits at any tier. Without that composition, a purge *is* a self-inflicted
  thundering herd.
- **Logging is a hidden hot key**: 10M req/s on one key produces 10M log lines/s
  (~15 GB/s at ~1.5 KB/line, est. — over 40% of the fleet's ~35 GB/s average log stream)
  referencing one customer + one URL — a single aggregation partition's write amplification (same
  shape as Twitter's celebrity fanout: an outlier consuming a shared per-key resource
  [02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md)). Fix: per-customer log
  partitioning by (customer, time-bucket) not by URL, with the hot key's line count
  downsampled for analytics (billing keeps full fidelity).
- **Failure mode that survives all of it**: origin serving the hot object slower and
  slower; the coalesced fetch stretches, waiters time out. Mitigations: serve stale beyond
  TTL under origin pressure (grace window, customer policy) and bound waiter queues
  (overflow requests are served the stale copy rather than queued) — a stale byte served
  beats a fresh byte not served, for *non-authoritative* content; customers that can't
  tolerate it opt out of grace via config.

## 6. DDoS absorption

Attack volumes publicly reported and vendor-mitigated in the 2023–2025 era: ~200M+ req/s
L7 HTTP floods, tens-of-Tbps volumetric UDP floods (records up to ~70 Tbps), and
DNS-layer floods of the same order (Cloudflare-class disclosures; treat magnitudes as
rough). Design requirement: the same
fleet that serves clean traffic absorbs this, and customer origins stay functional.

- **L3/4 volumetric**: the weapon is anycast itself — attack packets addressed to the
  anycast space are *spread across every POP* by BGP's own routing, dividing the attack by
  POP count before any filtering happens. Per POP: kernel-bypass drop paths (XDP/eBPF
  filter programs pushed via the control plane — a new drop rule is a config push, not a
  kernel change) and hardware ACLs upstream. Trade-off: anycast spreads *clean* traffic
  too, so capacity planning must assume clean+attack coexist in every POP; a POP that
  would be safe under per-region absorption becomes the weak link if a geo-targeted attack
  (spoofed source geography) collapses onto few POPs — mitigated by prefix-withdrawal
  (deep dive 1) and upstream scrubbing contracts for beyond-capacity events.
- **Amplified/reflection UDP (DNS, NTP, memcached)**: absorbed by egress-rate limits and
  source-validation (BCP38-style) at the network edge; the CDN's *ingress* is not the
  scarce resource — its upstream links' ability to receive the flood is, hence drop-as-
  early-as-physically-possible (filtering in NIC hardware before kernel memory exists).
- **L7 floods**: requests at 200M req/s *look like* traffic and must cross TLS to be
  classified. Design: verification is priced *per IP/session, not per request* — a client
  that passes a proof (TLS-fingerprint plausibility, JS/proof-of-work challenge, or managed
  bot score) carries a signed, short-lived clearance cookie/cert held at the edge; L7 rate
  limits apply per (IP → clearance → customer → path) with the check being an O(1) table
  lookup, not an external call. Trade-off: challenges are user-hostile (a challenge page
  is visible to a human), so the ladder is *invisible checks first*: TLS/JA4-style
  fingerprinting, header plausibility, history — only the suspicious residual sees a
  challenge, and only when attack pressure on that (customer, path) exceeds a threshold
  ("under attack mode" is a per-customer switch, not a fleet posture).
- **The CDN as origin shield**: the last-line contract — origins firewall to CDN IP space
  only. Attack traffic that somehow exceeds CDN capacity gets dropped *at the POP*, never
  forwarded; the origin-facing tier additionally rate-limits and caches errors (negative
  caching covers GETs; POST floods are matched against per-path policy — trade-off:
  aggressive per-path POST rate limiting can fail-open checkout endpoints under attack;
  solved by customer-classified path rules, not global defaults).
- **What this costs in the clean path**: every request pays a rules-engine evaluation
  (the <10 ms hit-path budget from 01's NFR table covers it — the WAF rules are compiled to per-customer
  match tables, amortizing to a few hash lookups), and the fleet must keep 2–3× clean-peak
  headroom (est. from 02) reserved for absorption. That idle headroom is the price of the
  DDoS product being the same hardware as the CDN product — there is no separate attack
  fleet.

---

Cross-references in this document: [01.youtube/04-deep-dives.md](../01.youtube/04-deep-dives.md)
(cache hierarchy, tier boundary inversion), [02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md)
(outlier fanout against shared resources, twice).
