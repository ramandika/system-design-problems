# 04 — Deep dives

## 1. Limit algorithm: fixed window vs sliding window (log & counter) vs token bucket

Four candidates; they differ in burst behavior, state size, and how badly they race.

**Option A — fixed window counter** (INCR per wall-clock bucket, e.g. 1,000/min):
- wins: the cheapest possible state (one integer + TTL); trivially atomic-izable in most
  stores; long cumulative quotas (hourly, monthly) map onto it naturally.
- loses: *window-boundary burst* — 1,000 requests at :59 and 1,000 at :01 pass a
  "1,000/min" rule: an instantaneous rate up to 2× the limit; a bursty client games it
  by aligning to boundaries. Also punishes bursty-but-honest callers (idle customer hits
  the minute fresh with full allowance, which is actually fine — the real problem is the
  2× seam).

**Option B — sliding window log** (keep timestamps of the last N events, evict expired):
- wins: mathematically exact for any window; smooths the boundary seam.
- loses: state is O(events), so a 1,000 req/min key holds ~1,000 entries — memory grows
  with *rate*, not with *time*; on a fat key at 10K req/s that is 10K entries per second
  of window, either in RAM or as list-trims on every check. Exactness is only affordable
  for low-rate rules.

**Option C — sliding window counter** (weighted blend of this bucket and the previous:
`prev_count × (1 − elapsed_fraction) + cur_count`):
- wins: fixed O(1) state (two integers), removes most of the seam, cheap.
- loses: approximation — the weighted estimate can be off by the previous bucket's
  whole remainder in adversarial traffic (the previous bucket's *shape* within itself is
  invisible); under concurrent updates the blend is recomputed from a read-modify-write
  (races, dive 3) unless scripted atomically.

**Option D — token bucket / GCRA** (capacity C, refill R tokens/s, spend on arrival —
or equivalently GCRA: track one value, the "theoretical arrival time"):
- wins: this is the *burst* semantics the product wants — idle capacity is stored up and
  spendable immediately (a customer doing a 2-second burst of their minutes' allowance
  succeeds, which is correct behavior, not a violation); O(1) state (tokens + last-refill
  timestamp, or one TAT value); refill math makes the check a single arithmetic op, and
  GCRA's single-value update is the least race-prone of all of them (dive 3); `Retry-After`
  falls out for free as time-until-next-token.
- loses: steady-state unfairness in one direction — a sustained caller is capped at R
  even in a dead-quiet hour *for the burst above R*, which some customers read as "you
  advertised more"; worst-case instantaneous arrival still bounded by C, so the seam
  excess is C (bounded, controllable via bucket size, versus fixed window's 2× unlimited
  by bucket choice).

**Chosen: D (token bucket, GCRA-style single-value state) for all short/throughput
limits; A (fixed window) for long hard quotas (hourly/monthly) where seam burst is
unmeasurable and the state is one number.** For 1-second/1-minute windows the deciding
factor is that customers' automation syncs against our *advertised burst semantics*, and
token bucket is the only candidate whose semantics ("you may spend your allowance in a
burst up to C") is both what paying customers want and cheap to enforce exactly at one
node. Long windows are inherently cumulative, so A is honest there and cheapest to shard
(dive 4).

- Trade-off made explicit: we accept that short limits are *not* sliding-window exact
  (C's smoothness) and that a 2-second burst can outrun a sustained rate, in exchange for
  O(1) state on every key, an atomic single-value update (fewer races, dive 3), free
  `Retry-After`, and burst semantics we can advertise. Per-IP abuse checks reuse D with
  tiny buckets; nothing in the fleet needs B's exactness.

**Failure modes**: mis-sized C → the platform over- or under-delivers burst; make C and
R config data (dive 6) so they can be tuned without deploys; GCRA across multiple nodes
on one shared TAT value → clock-dependent math — GCRA decisions must be made at one
authority per key (the lease model, dive 2, keeps each node's bucket independent with
its own wall-clock, which sidesteps this); fixed-window seam on monthly quotas →
acceptable (no one bursts a month's quota in seconds, and if they try, the short limits
catch it first).

## 2. Enforcement location: centralized decision vs local leases (quota drift)

The central question. Every check is: *where does the decision happen?*

**Option A — fully centralized** (every check is an atomic script at the counter store):
- wins: exactly correct — one authority per key, no drift, no over-admission ever; one
  mechanism serves all rule types; trivial to reason about.
- loses: per-request network round trip (~0.3–0.8 ms same-DC) on the hottest path in the
  platform; at edge PoPs the RTT is 10–100 ms — fails the ≤1 ms budget in 01 by 10–100×
  (02-scale-estimates); the store inherits the full ~350K checks/s *and* the hot-key
  skew; the limiter's availability is now coupled to the API's (store down = platform
  down, the opposite of the limiter's job).

**Option B — fully local** (each gateway node keeps its own complete bucket per key):
- wins: ~µs checks, zero shared state, availability is per-node.
- loses: N nodes each grant the full allowance → total admission ≈ N × R — a 1,000
  req/s key on 100 nodes is effectively unrated (100,000 req/s passes). Distributed
  limits without shared state are decorative.

**Option C — local enforcement with quota leases (hybrid)**: each node holds a *lease* —
a slice of the key's allowance it may spend without asking — refreshed in the background
(~500 ms) by reporting consumption to the quota authority, which rebalances slices
across nodes (bigger slice to the node carrying more traffic). Checks are local;
coordination is batched and off the critical path.
- wins: no per-request round trip (µs checks); the authority still exists (limits are
  global contracts, not per-node decorations); load on the authority is per-node-per-
  key-per-interval, not per-request — for a key at 10K req/s across 100 nodes, the
  authority sees ~100 nodes × 2 refreshes/s = ~200 ops/s, a ~50× reduction vs 10K
  centralized checks/s; independent availability (authority down → nodes keep current
  leases with decay, dive 5).
- loses: *drift* — between syncs, nodes hand out leases they may not all need; worst-case
  over-admission per interval ≈ N × (R/N) × T_interval ≈ R × T_interval (every node
  saturated at its leased slice for one full interval). For R = 1,000 req/s and
  T = 0.5 s: up to ~+500 requests ≈ +50% of one interval's allowance. Only sane for
  limits where ±10% drift is an accepted contract (01-requirements, scope decision 2) —
  which is why that line exists *before* this design.

**Chosen: C for throughput/burst limits (the ~95% tier), A for the ~5% hard cumulative
quotas and escalated abuse checks.** B is dismissed as semantics-free; A fails the
latency budget and couples availability. The split is by what the limit *means*: tier
throughput is a contract with tolerance (drift is the customer's own traffic spreading
across our fleet — visible only in a worst-saturated instant), while a monthly paid
quota or a brute-force lockout has no tolerance and is low-frequency, so its round trip
fits the budget.

- Trade-off made explicit: we accept bounded, quantified over-admission (≤ ~1 interval's
  allowance; ±10% normally) and the operational cost of a lease-reconciliation protocol,
  in exchange for taking the network off the hottest path and decoupling limiter
  availability from the counter store. Where that price is not payable (quotas, abuse),
  we pay the round trip instead.

**Failure modes**: authority down → leases freeze, decay per dive 5 (degraded but
open); node crash → its lease evaporates, authority re-grants after a heartbeat timeout
(tiny waste, seconds); *lease starvation* — a node with a cold-but-arriving key must
either pay a one-time lease grant round trip (fine: first request on a key is not the
hot path yet) or default to a small trickle lease while the grant is in flight (also
fine); rebalancing churn under swinging traffic → leases scale with measured
consumption, which is naturally stabilizing.

## 3. Read-modify-write races: making check-and-update one atomic step

Every limiter is a read-modify-write under concurrency. Three textbook races, all real:

**(a) The INCR-then-EXPIRE race** (fixed-window tier). Naive: `INCR key` → if result is
1 → `EXPIRE key window`. If the node dies between INCR and EXPIRE, the key persists
with no TTL — a permanent counter: the customer is locked out *forever* (or until
someone notices). This is the most expensive one-line bug in limiter engineering.
Fix: atomic script — increment, when `count == 1` set the TTL, all in one server-side
step; the store's own single-threaded execution makes the pair indivisible (pseudo-
semantics: `INCR k; if count == 1: EXPIRE k, window; return count` — one script, not
three round trips).

**(b) Check-then-act over-admission**. Naive: `GET count` → compare `count < limit` →
`INCR`. Two concurrent requests both read `count = limit − 1`, both pass, both increment:
over-admission exactly under load, the worst time to be wrong; repeated, the effective
limit scales with concurrency, i.e. an attacker multiplies their allowance just by
parallelism. Fix: the comparison and increment must be one atomic step (script reads,
compares, writes, returns admit/reject in one store-side execution).

**(c) Token-bucket refill race**. Naive: read `tokens`, compute elapsed refill
locally, compare, write back `tokens − 1`. Concurrent updaters clobber each other's
refill computations (last-write-wins erases the other's spend — tokens are created from
stale arithmetic). GCRA's single-value trick is the structural fix: one value
(theoretical-arrival-time), the update is `max(now, TAT) + interval` — a single
max-plus that composes under the store's serialization, one value to lose, no separate
token-plus-timestamp pair to clobber. And per dive 2, most token-bucket checks are *per
node's own lease bucket* anyway, where atomicity is just "this thread's mutex" — the
race only bites at the shared authority.

**Chosen: one atomic decision script per rule type, executed by the store's
single-threaded engine (Lua-class), for every centralized check.** CAS/WATCH-retry loops
are the alternative and are dismissed: under a hot key they thrash (N concurrent CAS
retries against one row is self-inflicted lock contention at exactly the load peak),
whereas single-threaded scripted execution is fair FIFO. The cost we accept: scripts
serialize *per shard*, so a script must stay arithmetic-cheap (µs-class) — no I/O, no
rule resolution, no iteration inside the script; all key-shaped pre/post work happens on
the caller side. On the lease tier, the same discipline applies locally (per-bucket
mutex), which is the cheap case.

- Trade-off made explicit: we accept store-side scripting (a shard of our decision logic
  lives in a Redis-class engine with its own toolchain and the discipline of keeping
  scripts tiny) in exchange for making three racy steps one indivisible step, without
  watch/retry loops that degrade exactly under load.

**Failure modes**: hot-key script serialization → one key's scripts queue (a fat client
queues only behind itself, ~µs each — bounded, and dive 4 chunks fat keys anyway);
script bug → one wrong decision record, not a corrupted counter (scripts are the unit of
test); store failover mid-script → the check fails closed/open per class (dive 5), and
the counter's last-write is either fully applied or not at all — atomicity also buys us
crash-consistency.

## 4. Distributed counters at high QPS: hot keys and chunking

Skew is the fact of life here: top ~1% of keys ≈ ~50% of checks (02-scale-estimates).
A centralized check rate for one fat key can approach a shard's entire safe ceiling
(~80K ops/s), and no amount of consistent hashing fixes a single hottest key — it goes
wherever it goes and takes that shard with it (the general theory is dive 5 of
[10.distributed-cache](../10.distributed-cache/04-deep-dives.md); the typeahead
version of the same problem is [45.typeahead](../45.typeahead/04-deep-dives.md) dive 3).
Two mechanisms compose.

**Option A — shard-split (chunked) counters**: split a fat key's allowance into K
chunks, each its own counter entry on a different shard; a check picks a chunk, spends
from it, and admits — done.
- wins: linear write QPS across K shards (10K checks/s on K=8 shards is ~1.2K ops/s per
  shard); no single point of contention; works for fixed-window quotas.
- loses: capacity is only approximately conserved — chunks drain unevenly; a check
  hitting a drained chunk rejects *while other chunks still have tokens* (= false
  rejections for a paying customer, the worst failure in 01); summed over chunks, the
  fleet enforces ≤ the true limit but locally per chunk it under-delivers; dangling
  capacity when a chunk's shard is unreachable.

**Option B — two-choice chunk selection**: pick two random chunks, spend from the fuller
one.
- wins: with the same K chunks, load balances like power-of-two-choices — drained chunks
  are rarely picked once any chunk has tokens, so false rejections drop by orders of
  magnitude over pure-random pick; still linear write QPS; one extra read's worth of
  state, nothing more.
- loses: two store-side reads instead of one (cheap inside one script); residual
  imbalance remains.

**Option C — keep one authority, batch arrivals**: for token-bucket/GCRA keys, don't
check per request at all — each node keeps its lease (dive 2) and syncs *consumption
deltas* at interval; the authority never sees per-request traffic.
- wins: this is what the lease tier already does — 10K req/s key across 100 nodes costs
  the authority ~200 ops/s (~50×↓ vs centralized); no chunk-splitting needed for
  burst-shaped limits.
- loses: it is still per-key single-authority state under the hood, so it inherits a
  (much smaller) hot spot at the authority for very fat keys; drift as in dive 2.

**Chosen: C first (leases + batched delta sync) for anything token-bucket-shaped —
which is most of the load — and A+B (chunked fixed-window counters with two-choice
pick) for the residual class of fat *cumulative-quota* keys, plus a chunk-rebalance
sweeper that drains-and-merges chunks whose sum drifts too far from the limit.** We pick
C-first because it removes 95% of checks from the shared tier outright (02 arithmetic);
B is bolted onto A because a chunked counter's false rejection of a paying customer is
unacceptable at random-pick skew.

- Trade-off made explicit: we accept approximate capacity conservation across chunks
  (sum ≤ limit; brief false rejections possible during extreme chunk skew — bounded by
  two-choice pick and the rebalancer) in exchange for linearizing the only load pattern
  a store sharded by key hash cannot absorb: one key, one shard, one ceiling.

**Failure modes**: one chunk's shard down → that chunk is unreachable; two-choice pick
naturally routes around it if chunk-unreachability is visible in the pick set
(rebalance folds it within seconds); long unreachability → the key's quota
under-delivers by 1/K until rebalance — safe direction (over-delivery is worse than
under for our classes); rebalancer bug → drift between chunk sum and true limit, so a
periodic audit job recomputes sums and alarms on divergence (the counters are the
receiving system of record for billing-adjacent quota, so auditability is not optional).
Chunk count K is config data (dive 6), sized per key from observed check rate.

## 5. Fail-open vs fail-closed when the counter store is down

The failure posture question. When the centralized tier is unreachable, every check in
that tier needs a default. Both defaults are defensible; only per-class is defensible.

**Option A — fail open globally** (counter store down → allow everything):
- wins: the limiter can never be the API's outage amplifier; a paying customer is
  never blocked by *our* infrastructure breakage; simplest possible posture to
  implement and to justify to customers.
- loses: security load-bearing limits (login attempts per IP, brute-force locks,
  abuse thresholds) silently stop enforcing exactly during a disruption — which
  attackers actively watch for; billing-adjacent meters under-count with no evidence
  trail.

**Option B — fail closed globally** (counter store down → 429/503 for the tier):
- wins: no limit is ever un-enforced; security posture is constant.
- loses: the counter store's availability now bounds the API's — a Redis-class cluster
  outage takes down the platform's *whole enforcement tier* even for customers whose
  limits weren't in play; the limiter goes from a protection to a single point of
  failure.

**Option C — per-class posture, with a bounded-degradation lease**:
- *Safeguards* (per-IP login-attempts, abuse, anomaly soft-thresholds): **fail closed** —
  degrade to the node's *local* state (it still remembers recent IPs), and admit unknown
  IPs at a reduced trickle rate. The cost of a false allow here is compromise, not
  inconvenience.
- *Paid throughput/hard quotas*: **fail open** — nodes keep spending their current lease
  with a decay factor (each failed refresh halves the lease, floored at a small
  trickle). Over-admission is then bounded, roughly ΣR×T intervals of decay, instead of
  "everything, forever".
- *Billing-meters*: fail open, but emit explicit "unmetered interval" records so
  reconciliation can charge or forgive from evidence rather than guessing.
- wins: each limit class fails in the direction its risk profile pays for; degradation
  is bounded and *observable* (decision records carry a `degraded` flag); no component's
  outage scales into a platform outage.
- loses: two postures to implement, test, and explain; the bounded-decay logic is
  genuine moving-part surface area (a decay bug is its own class of incident); during a
  real counter-store outage, dashboards built on counters go dark at the same time
  (accept: dashboards are not enforcement).

**Chosen: C.** The choice is forced by requirements, not preference: 01's
accuracy/False-negative table demands near-zero tolerance on abuse limits (→ closed)
and near-zero false-rejection on paid limits in normal operation (→ open within
bounded decay). Global A and global B each violate one of those explicitly.

- Trade-off made explicit: we accept per-class posture complexity and the operational
  risk of decay logic, in exchange for making the counter store optional for throughput
  (leases carry on, degraded but bounded) and load-bearing only where its absence is the
  bigger danger (security limits) — the limiter stays "at least as available as the API
  it protects" as 01 requires.

**Failure modes**: partial failure (some shards/chunks unreachable) → same rules apply
per-key: a key whose chunks are unreachable decays per class posture, not globally;
prolonged outage → decay floor keeps *some* enforcement so recovery does not meet a
fully-unrated fleet; abuse engine's greylists come from config (dive 6), so they are
available even when counters are not — deliberately: static rules survive, approximate
counting may not.

## 6. Limit-config distribution and versioning

Everything above is *data*: rules, bucket sizes, chunk counts, per-key overrides. The
config plane is thus on every decision's correctness path, just not on its latency path.

**Option A — pull-through rule cache** (node asks a store for the rule on cache miss):
- wins: no snapshot infrastructure; only active keys' rules are ever loaded.
- loses: a cold key = a network fetch on its first request; a config change causes a
  read stampede on the popular rules (every node re-fetching simultaneously — the
  thundering-herd family, same fix needed as
  [10.distributed-cache](../10.distributed-cache/04-deep-dives.md) dive 4); rule-fetch
  failure semantics leak into enforcement semantics.

**Option B — full versioned snapshots, pushed** (~200 MB fits every node's memory,
02-scale-estimates): each revision is an immutable numbered snapshot; nodes load a
revision, verify, then *activate* it.
- wins: zero on-path rule fetches (the check is a hash lookup in memory); one artifact
  to version, audit, and roll back; last-known-good is just "the previous revision you
  still hold"; rule resolution is never on the network.
- loses: ~200 MB per node and the distribution system to build/publish it; whole-fleet
  propagation latency is a design parameter (seconds-to-minutes); a bad revision is
  *fleet-wide instantly* unless staged.

**Chosen: B, with two-phase coordinated activation.** A's fetch-on-first-use puts a
network hop inside an enforcement path we just spent dive 2 taking the network off —
not worth it at ~200 MB. Activation runs prepare/activate: nodes load revision N+1 and
hold both revisions, then flip at a coordinated wall-clock deadline broadcast with the
revision. Rationale: rule *precedence* is relative — if half the fleet enforces
revision N and half N+1, a customer whose override tightened gets flapping 429s
depending on which node served them, which is a worse failure than a brief
*coordinated* transition. The deadline makes the skew window seconds, not minutes.

Two rules inside the config plane itself:

- **Precedence is part of the data, not of the reader's discretion.** Most-specific-wins
  (per-key override > tier default; longer window harder than shorter), with ties broken
  by a deterministic order *defined in the revision*. If precedence were emergent across
  nodes, dive-6 skew becomes dive-3-style nondeterminism.

- **Rollback is a revision, not an investigation**: revisions are immutable; recovery
  from a bad push is "activate N−1", done in seconds by the same mechanism, and
  canaried by rolling a fraction of the fleet first (the canary sees the fat clients
  fast because of the traffic skew).

- Trade-off made explicit: we accept ~200 MB in every gateway node and a snapshot
  distribution pipeline, in exchange for never having a rule fetch on the enforcement
  path and making config state (its versions, its rollback, its audit) a first-class,
  testable artifact rather than a cache behavior.

**Failure modes**: revision fetch unavailable → node pins last-known-good — enforcement
continues on stale rules with an alarm (correct default: a stale limit is safer than an
empty one); bad revision → canary catches most, rollback covers the rest; *revision
skew between nodes* → bounded by the coordinated deadline; config store corrupts a
revision → checksums on publish, nodes reject-and-hold (fail toward last-known-good,
never toward no-config); bootstrapping a fresh node → tier defaults are compiled in as
revision 0, so a node is never serving without *some* rules.
