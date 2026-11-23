# 10.distributed-cache — 04 Deep dives

## 1. Sharding: consistent hashing with virtual nodes vs hash slots

The tier is partitioned so every key has exactly one owner. Two mainstream mechanisms:

**Option A — consistent hashing ring with virtual nodes.** Each node owns many points
(vnodes) on a hash ring; a key belongs to the next vnode clockwise. Migration moves only
keys between the leaving node and ring neighbors.

- wins: any node can join/leave; rebalancing is automatic; vnode count per node is a tuning
  knob that smooths out capacity differences.
- loses: key→owner is *computed* against a ring membership the client must know and keep
  fresh; heterogeneous additions shift load onto neighbors unpredictably; the migration set
  is "everything between two ring points," which is harder to bound than it sounds.

**Option B — fixed hash-slot table (Redis Cluster's model).** 16,384 slots; each shard owns
a contiguous slot range. Key → `hash(key) mod 16384` → slot → shard. Migration is per-slot:
source streams the slot's keys, the map flips, clients re-route.

- wins: slot→shard is a *table* — cheap to publish, cheap to reason about, and migration
  moves a chosen slot, not a ring arc; capacity rebalance is an explicit slot reassignment.
- loses: a fixed 16,384-way split is coarse (a shard can't be finer than one slot) and the
  table still must live somewhere authoritative (failover quorum), which is a control-plane
  dependency.

**Chosen: B (hash slots).** We pick slots because the operational win is the one that bites
in practice: resharding becomes an explicit, schedulable, *per-slot* operation with a bounded
migration set, instead of an emergent property of ring membership. Consistent hashing wins
when nodes churn constantly, which a cache tier does not — the topology changes weekly, not
per request.

**Migration mechanics and the hit-rate cost.** Keys do not move to be useful — they move
because the ring shrank or grew. Every moved key that isn't copied *with its TTL and access
history* becomes a guaranteed miss at its new home. So migration is: for each slot, bulk
copy key+TTL from source, mark slot as migrated, flip the map. Moving 10% of slots at once
costs roughly 10% hit rate for one reheat window (~minutes to tens of minutes for hot keys)
— visible in the DB load chart, so we drip slots: at most one or two slots move
concurrently, never during peak. Trade-off accepted: reshard takes hours instead of
minutes, in exchange for the DB never seeing a migration-induced spike.

**Failure modes**: (a) split-map — client's slot map briefly disagrees with the shard's
owner; mitigate with MOVED-style redirects instead of errors during flips; (b) half-migrated
keys — bulk copy then flip means a GET in the window sees the old owner; make migration
read-through safe: after the flip, old owner redirects rather than 404s; (c) hash-tag
abuse — app teams co-locating everything under one tag to get multi-key atomicity
re-creates a single hot shard; enforce tag cardinality in review, not at runtime.

## 2. Replication and failover — and what consistency reads can actually expect

Per shard: 1 primary + 2 async replicas in different failure zones. Writes ack at the
primary; replication is asynchronous. Failover: a controller quorum promotes a replica when
the primary's heartbeat lapses (target ≤ 15 s, N2).

**Option A — async replication, promote-latest.** Ack = primary accepted.

- wins: write latency is one local hop; the tier stays fast, which is its entire job.
- loses: a failover can lose writes acked in the last replication interval, and replicas
  can serve reads that are older than the primary's acked state.

**Option B — quorum writes (ack only after a replica confirms).** E.g., WAIT on 1 replica.

- wins: acked writes survive any single-node failover; reads against an up-to-date replica
  see every acked write.
- loses: adds a cross-zone round trip to every write (a few ms typical, several times the
  1.5 ms N1 op budget); a zone outage degrades write availability unless you fall back to
  async anyway.

**Chosen: A for the object class, B-style (sync to one replica, quorum failover) for the
session class.** Justification is the class split from 01: objects are reconstructible —
a lost cache write costs at most a miss and a reheat; sessions are not reconstructible in
the hot path, so they pay the extra hop, and they are only ~10% of write volume (see 02),
so the latency bill is small and paid where it matters.

**What a read can expect, honestly stated.**

- Object-class reads from replicas: **eventual consistency** — a read right after an
  invalidation can return the pre-invalidation value; bounded by replication lag (usually
  single-digit ms, worst-case during failover).
- Session-class reads: **read-your-writes within a session** as long as the client pins to
  the primary (or the map pins the session's owning shard); reads from stale replicas can
  resurrect a just-updated session field. The guarantee is per-key (per-session), not
  causal across keys.
- After failover: the promoted replica's state is the newest surviving state; anything not
  replicated is gone *for objects* (acceptable: origin rebuilds) and *must not be gone for
  acked sessions* — hence sync replication for that class. This is the same
  "which signals are MUST-NOT-LOSE vs MUST-NOT-COST-ANYTHING" call WhatsApp's presence
  design makes — see [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md), dive 6.

**Failure modes**: (a) failover storm — many primaries fail at once (zone outage,
rolling deploy gone wrong); staggered promotion and client retry-with-jitter keep the
origin from seeing 1.5M QPS of simultaneous pass-through; (b) split brain — an isolated old
primary still accepting writes; mitigate with controller-quorum fencing (old primary stops
serving when it can't renew its lease); an unfenced primary serving a partitioned zone's
reads is survivable for objects, not for sessions; (c) promotion of a *lagging* replica —
controllers prefer recency of replication stream over raw liveness when choosing.

## 3. Invalidation: cache-aside vs write-through and their races

**Option A — cache-aside (write-around).** App writes DB, then DELs cache; reads miss →
fetch DB → SET.

- wins: cache holds only keys actually read; write path doesn't spend cache cycles on
  cold or unread data; a cache outage doesn't block writes.
- loses: the first read after a write always misses (one-strike staleness), and the write
  path has a race (below).

**Option B — write-through.** App writes DB and cache in one logical step (or the cache
write happens through the same writer).

- wins: cache is warm immediately after every write; no read-path race between fetch and
  write.
- loses: write amplification and cost (writing data nobody reads); if the writer dies
  between DB commit and cache write, the cache is stale until TTL with *no* invalidation
  signal; a cache outage is in the write path.

**Chosen: cache-aside** for the object class — its read-set skew (hot keys are a small
fraction) means write-through pays to refresh data nobody will read. We pick it despite the
race because the race is *bounded* (see below) while write-through's cost scales with all
writes.

**The cache-aside race, precisely.** Two interleavings, both real:

1. Stale-set pin: reader R misses; R reads DB row (old value V1); writer W commits V2 and
   DELs the key — DEL fires *before* R's SET; R then SETs (V1). Cache now holds V1 under an
   otherwise-fresh key until TTL expires. TTL 5 min ⇒ up to 5 minutes of staleness, per
   occurrence. Interleaving needed: `read-DB(V1) → commit(V2) → DEL → SET(V1)` — narrow but
   common under load.
2. Delete-then-miss: W commits V2, DELs; R misses (because of the DEL), reads DB, gets V2,
   SET V2 — benign. The race only bites when the read *fetched before* the DEL.

Standard mitigations, in increasing order of cost:
- **Order discipline**: always DEL *after* DB commit (write-through-of-delete), never
  before; prevents the worst variant where the DEL precedes the commit and is then undone
  by a concurrent reader's pre-commit fetch.
- **Versioned keys / write-version bump**: each DB row carries a version; keys are
  `key:v{version}` — a stale SET(V1) writes to `key:v1` which no one reads. Kills the
  stale-set race at the cost of unbounded key churn and needing a version source.
- **Delayed double-delete**: DEL, tiny delay (~read-latency window), DEL again, catching
  the straggler SET from interleaving 1. Ugly (a timed delete) but cheap; used as a
  targeted patch for known-bad rows, not fleet-wide.
- **Short TTL as the backstop**: the race is only unbounded if TTL is. With 5-minute TTLs
  the worst stale window is 5 minutes — the real design answer is that staleness *deadline*
  is a product requirement (N4 doesn't promise it, the class policy does).

**Consistency expectation**: cache-aside gives **read-after-TTL staleness with a small
unbounded-under-race tail** — not linearizability, not even per-key monotonicity for object
reads. Sessions bypass this: they are write-behind to a key nobody else invalidates
concurrently.

**Failure modes**: (a) invalidation floods — a bulk backfill or moderation sweep DELs
millions of keys at once ⇒ instant mass-miss thundering herd at the DB; sweep invalidations
must be rate-limited per origin budget (same physics as Twitter's fanout-of-delete problem —
see [Twitter deep dives](../02.twitter/04-deep-dives.md), dive 4); (b) missing
invalidation on writes through a different path (a batch job that writes the DB directly) —
stale until TTL, usually invisible, occasionally a production incident; the fix is
organizational (all writers invalidates-or-declares), not technical.

## 4. Thundering herd and request coalescing

A popular key expires (or the tier restarts); 50K concurrent readers miss and all 50K go to
the origin DB for the same row. That is a thundering herd, and it is the classic way "we
have a cache" becomes "the DB is down."

**Option A — in-process coalescing (single-flight) per client instance.** Concurrent GETs
for the same key inside one process → one origin fetch, others wait.

- wins: no new infrastructure; removes the 50×–500× multiplier *within* each instance
  (a service with 500 instances still fans 500 fetches — needed but insufficient).
- lives in the client library (03's component 1) for this reason.

**Option B — proxy/tier-level coalescing.** The shared tier or proxy detects an in-flight
key and shares the origin fetch.

- wins: collapses across all instances (the 500 becomes 1).
- loses: puts application awareness (which origin? which fetch?) into the proxy — the
  proxy now must speak "fetch origin," which couples the cache layer to every backend; or
  clients must register waiters on a lease, which is complex and a new failure mode.

**Option C — probabilistic early refresh.** A small percent of readers refresh the key
*k before* expiry (rate proportional to time remaining-to-expiry), so hot keys rarely hit a
hard expiry boundary.

- wins: eliminates the expiry moment rather than reacting to it; simple, no coordination.
- loses: does nothing for cold-start (whole tier empty); requires the read path to be able
  to refresh (acceptable for object class).

**Chosen: A + C**, B only where it already exists (a proxy-based deployment). A kills the
per-instance multiplier, C kills the expiry spike; between them, the DB sees ~one fetch per
key per expiry, not thousands. B's cross-service variant is the thing to reach for when
*instances per service* themselves number in the thousands — the decision point is fan-in
size, not preference. The same coalescing physics shows up one layer down as Discord's
data-services shim coalescing concurrent same-row readers ([Discord deep dives](../05.discord/04-deep-dives.md),
dive 5) and in YouTube's CDN cache hierarchy, which exists to collapse the thundering herd on
the same bytes ([YouTube deep dives](../01.youtube/04-deep-dives.md), dive 3).

Complementary defenses: (i) **soft TTL** — serve a slightly stale value while a refresh is
in flight, so readers never block on expiry (trade-off: staleness during refresh vs a wait;
pick serve-stale for object class); (ii) **staged warmup after restart** — a restarted
service throttles its own pass-through rate (e.g., token bucket from 1% to 100% over
minutes) so a deploy doesn't cold-crash the DB; (iii) **negative caching** — cache the
absence of a value for a short TTL so a stream of nonexistent-key lookups (attacks, broken
clients) doesn't become DB QPS.

**Failure modes**: (a) coalescer leak — a waiter registered before a crash never completes;
put timeouts on waiters, not just on fetches; (b) lock key is *itself* a hot key — the
distributed lock used to elect the single refresher becomes its own thundering-herd target;
prefer per-process coalescing plus short soft-TTL and skip distributed locks; (c) cold tier
+ no staged warmup = the 5×-over-provisioned origin cliff from 02 — the most common way
cache outages outrun their blast radius.

## 5. Hot keys and oversubscription

Hash partitioning distributes *keys*, not *requests*. One key — a celebrity's hydration
blob (100M-follower accounts post per-second traffic; see
[Twitter deep dives](../02.twitter/04-deep-dives.md), dive 1, for that skew) or a flash-sale
SKU — lands on one shard: 100K QPS against a node that comfortably serves 50K. The shard
melts; the other 29 shards are idle. This is oversubscription: capacity planned for average
key-ownership fails under request skew.

Options, roughly in order of violence:

- **Replicate the hot key** `key:replica{n}` and have clients pick a replica by hash of
  (request id) — read traffic spreads over r shards. Wins: linear relief, no new infra.
  Loses: r× invalidations and r replicas to keep coherent (acceptable: staleness ≤ r× lag);
  needs to know *which* keys are hot.
- **Local (L1) caching of the hot key**: the hottest keys are exactly the ones an
  in-process cache serves best (hit rate → ~100% within each instance). Wins: removes the
  load from the network tier entirely. Loses: L1 invalidation is per-instance best-effort —
  hot key staleness grows by up to L1's TTL.
- **Split reads from the hot key's data shape**: if the hot value is 10 KB of a 1 KB-used
  projection, serve the projection (smaller key ⇒ less bandwidth ⇒ shard survives longer).
- **Do nothing, rate-limit**: drop-oldest or serve-stale on the melting shard, protecting
  the shard's *other* keys — the blast radius stays one shard instead of the fleet.

**Chosen: detect + L1-first, replicate as the second line.** Detection is the actual
problem: per-key op counts sampled on the shard (not client-side) with a threshold ~2×
mean ops/s; auto-promotion into a small "hot set" replicated r=3–8 with client-side
round-robin. Trade-off accepted: hot keys may be up to (L1 TTL + r× replication lag)
stale; we buy shard survival with a bounded extra staleness on exactly the keys whose
staleness shows. Counter-class keys (rate limits) instead shard by *bucket* and accept
approximate INCR on replicas — an atomic counter's consistency is negotiable where it's a
limit check, not money.

**Failure modes**: (a) hot key *during* migration — a hot key's slot moves while it's hot;
migration bandwidth and traffic skew multiply; pause hot-key migrations; (b) detection
lag — skewed traffic arrives faster than sampling windows close; a shard can CPU-saturate
before it's flagged; mitigate with pre-emptive L1 for the top-K keys *by request count
seen anywhere*, which is cheap to compute client-side; (c) many lukewarm keys — 500 keys
each at 1K QPS on one shard don't trip a 100K threshold but sum to 500K; the mitigation is
capacity checks per slot-range, not per-key alarms only.

## 6. Eviction (LRU vs LFU) and memory sizing math

Capacity is finite (02): ~1.4 TB primary footprint for ~1 TB of hot values. When memory is
full, the tier must evict. The policy decides *which* keys die — and, implicitly, the hit
rate.

**Option A — LRU (approximate; the usual segment/window approximations).** Evict the
least-recently-used.

- wins: simple, well-understood; one-time scans (a nightly job reading every user once)
  don't pollute it forever.
- loses: **count-based pollution**: a key accessed 1M times once an hour and a key accessed
  once both count as "used" — LRU keeps the once-touched tail as long as it's recent. With
  request skew, recency ≈ frequency for top keys, so the loss is small — until a scan or a
  flash trend (one-time viral lookups) flushes the real working set.

**Option B — LFU (approximate; e.g., frequency-sketch based, as in Redis 4+ LFU or
W-TinyLFU's count-min sketch + doorkeeper).** Evict the least-*frequently*-used.

- wins: keeps genuinely hot keys under pollution from one-time access storms; matches the
  Zipf-ish access skew caches actually see.
- loses: needs per-key counts (or a sketch) — more memory per entry and a *new-entry
  problem* (a fresh key's count is 0 and gets evicted before it has a chance; mitigations:
  initial count + decay).

**Chosen: LFU for the object class (access-skew driven), plain LRU (or nothing) for
sessions (session expiry is by TTL, not by policy)** — object reads are heavily skewed; LFU
exploits that; sessions don't compete in eviction at all (they expire). Trade-off accepted:
slightly more bookkeeping memory (a sketch, doorkeeper) in exchange for higher hit rate on
the skew.

**Memory sizing math (the part that's load-bearing).** Hit rate vs working-set fraction:
with Zipf-like access (skew α ≈ 0.9–1.0 is the commonly observed range for item popularity —
a rough characterization, not a measured constant), the hot fraction of keys covers most
requests. If the hot 20% of ~500M keys = 100M keys × 2 KB = 200 GB covers ~80% of reads,
then a tier holding 1 TB (with the 5-minute TTL of object class naturally keeping the tier
close to the working set) delivers ~90%+ hits; **starving the tier to 0.3 TB drops the hit
rate toward ~60–70% (estimate) — and a 1.5M-QPS tier at 60% hit sends 600K misses/s at the
origin, ~4× its steady-state residual.** The sizing rule that follows: provision memory for
the working set at target hit rate, *then* leave 30% headroom (eviction efficiency degrades
non-linearly as utilization → 100%, because the approximate LRU/LFU samples rather than
scans, and hash-table rehash + fragmentation need slack).

Decay and window: LFU counts decay (divide by 2 every N minutes) so a formerly-hot key
releases its grip; without decay, LFU pins yesterday's hot keys over today's — the
cache-tier analog of a stale index.

**Failure modes**: (a) eviction avalanche — memory pressure evicts the working set; miss
rate rises; misses add *writes* (SET after origin fetch), raising memory pressure further —
a positive feedback loop that ends at 100% miss; the circuit breakers are: admit-window on
new SETs under pressure (don't admit what you can't keep), and origin-side rate limiting as
the final wall; (b) TTL all-expire-at-once — a bulk load that sets uniform TTLs produces a
synchronized expiry cliff hours later; jitter all TTLs ±20% as policy; (c) sketch/counter
memory forgotten in sizing — LFU's bookkeeping is ~tens of bytes per entry; at 500M keys
it's tens of GB, which is free only if you counted it.
