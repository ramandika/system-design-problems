# 04 — Deep dives

## 1. The waiting room: fairness under flash load (THE dive)

At 10:00:00 a pre-announced instant delivers ~2M arrivals (estimate, see
[02-scale-estimates](02-scale-estimates.md)) against an inventory tier that can absorb
~15–20K seat operations/s. The waiting room's job is precisely to make the mismatch
survivable and *perceived* as fair. Three design questions.

### 1a. Who gets in, and in what order

**Option A — first-come-first-served (FCFS) admission.** Arrival timestamp/order in a
Redis sorted set; admit in order.

- wins: matches the naive fan's model of fairness ("I was early, I should be early"),
  simple to explain, and gives a deterministic, auditable order.
- loses: rewards who can *open tabs first* — advantage to automated clients with
  accurate clocks and pre-staged requests; a queue ordered by arrival is a queue ordered
  by bot sophistication; also encourages the 08:55 camp-out stampede that the edge must
  survive anyway.

**Option B — randomized (lottery) admission.** Everyone who shows up in the admission
window enters a pool; winners drawn randomly, admitted in draw order.

- wins: removes the *timing* advantage entirely — bots gain nothing by being first —
  and dampens the pre-10:00 stampede (no benefit to camping). This is essentially what
  "verified fan" registrations + randomized invites attempt.
- loses: fans who queued at 08:55 feel cheated by the 09:59 arrival who got in; it
  *feels* less fair even when it is more adversary-resistant; also requires deciding the
  window close, and everyone's position uncertainty makes refresh traffic worse.

**Option C — hybrid (the shipped shape): FCFS *within* randomized bucketing.** On
entry, assign position as FCFS, but under storm saturation (when true arrival order is
unmeasurable within one second's burst), resolve the burst cohort randomly.

- wins: keeps the honest mental model when load is measurable, and admits that within
  the 10:00:00–10:00:01 wall of traffic, arrival order is meaningless and forgeable —
  so it stops pretending. Bots gain nothing extra, fans in camp keep their advantage at
  08:55 (cohort assignment favors early arrivals there).
- loses: more code, and the fairness guarantee is now probabilistic and needs
  documenting ("positions within the first N arrivals are randomized") — support tickets
  say "but I was FIRST" forever.

**We pick C.** The deeper principle: a fairness mechanism is only as strong as the
bot defense behind it (dive 5), so design for *perceived* fairness at coarse
granularity rather than millisecond-order precision that only machines can exploit.

### 1b. How position is served without melting

2M users polling position at 1/3 s is ~650K req/s (02's arithmetic) — far beyond any
origin tier, so position must be served *without per-user origin state*:

- Position is a **count plus coarse buckets**: "you are between positions
  400,001–410,000; estimated wait ~40 min". A user's exact position is worthless
  anyway — it changes continuously as others abandon (abandonment is high: many in
  queue will leave, so position improves) — and exactness is what scrapers want.
- Served from replicated in-memory counters, bucketed; any replica answers any
  client. Edge-served static page + coarse counter deltas keeps origin at the target
  ≤80K req/s.
- **Server-driven backoff**: each response carries a signed "next-check" time;
  clients honoring it poll at ~1/10 s, not 1/3 s — and the response *is* the
  backoff contract, so honor-and-punish (delay token issuance for violators) is
  cheap extra pressure on bot loops.

Failure modes:

- *Admission rate mismatch*: admit faster than holds convert → inventory churns,
  everyone sees "unavailable", queue rage. Admit too slowly → seats sit unsold while
  users time out; sale under-performs. The admission rate must track the hold-attempt
  budget of dive 6, and be a tunable, monitored knob (admissions/s), not emergent
  behavior.
- *Queue collapse*: waiting-room Redis dies mid-sale → positions lost, users rejoin,
  fairness gone. Mitigate: waiting-room state is small for what it buys (a 2M-entry
  sorted set plus tokens — order of ~100–200 MB, estimate) — replicate with
  automatic failover, and log every position assignment to the event log so a rebuilt
  queue rehydrates. Cross-ref the session/lease failure thinking in
  [11.zookeeper](../11.zookeeper/04-deep-dives.md).
- *The "everyone got in at once" crash*: a promo, a viral moment, admissions jump
  10×. The waiting room must degrade to "position unknown, hold tight" rather than
  5xx — a degraded honest waiting room beats a crashed one for both fairness and brand.

## 2. Seat holds: inventory correctness under contention

The atomic unit of scarcity is a seat. Between "user clicked" and "payment cleared"
there are minutes; during those minutes the seat must be invisible to all others, and
if the user abandons, the seat must come back. This is a *lease*, not a lock — the same
distinction Chubby/ZooKeeper's design warns about: a lock without expiry is a
dead-system-maker (see [11.zookeeper](../11.zookeeper/04-deep-dives.md) on why a
lock alone is never sufficient).

### The hold state machine per seat

```text
AVAILABLE ──hold(hold_id)──▶ HELD(hold_id, expiry) ──commit──▶ SOLD
    ▲                          │     │
    │         TTL elapsed ─────┘     └── release(user cancels)
    └────────── reaper transitions HELD→AVAILABLE on expiry, logged
```

- **Hold TTL: ~7–9 minutes** (estimate, tuned to checkout UX: long enough to enter a
  card and 2FA, short enough that abandons don't starve the sale). Expiry is absolute,
  server-side; the client may request one short extension, granted only if the seat is
  not contested (extension is *not* guaranteed — contested seats must not be extended,
  else bots extend forever).
- **The reaper**: a scanner over HELD seats on TTL expiry, emitting
  `HELD→AVAILABLE` transitions to the event log; inventory rebuild replays them. Failed
  payments, crashed browsers, walked-away users all route here — the reaper is the
  garbage collector of the sale, and its *lag* is directly lost throughput: a 2-minute
  reaping lag on a 7-minute TTL means nearly a third of a short sale's seat supply was
  dead weight. Monitor p99 reap lag as an SLO.

### The contention question: who wins seats 12–14 of row J

**Option A — pessimistic locking.** Acquire a lock per seat/row (external coordinator
or DB row locks) before inspecting and updating.

- wins: familiar; "hold" reads as "lock", long transactions feel natural.
- loses: at ~15K hold attempts/s on one event, lock waits are the bottleneck made
  literal; an external coordinator adds a network RTT to the system's tightest path;
  crashed lock-holders need lease machinery anyway (so you built A plus a TTL).

**Option B — optimistic compare-and-swap (CAS).** Seat state lives in memory owned by
the event's inventory shard; `hold` is a single-threaded atomic transition
`AVAILABLE → HELD(id, expiry)` (compare current state, set new state, all-or-nothing,
including *multi-seat groups atomically*). On conflict — any seat in the requested
group no longer available — the request fails fast with a precise reason.

- wins: no waiting, no deadlocks; conflicts are cheap to detect and cheap to retry;
  the atomic path is microseconds and entirely in-process; and group-atomicity falls
  out naturally (CAS on the *set* of seats, or CAS on the row strip, see dive 3).
- loses: under high contention most attempts *fail* (the honest truth: only ~55
  seats/s can satisfy anyone, while 15K/s try), so the perceived latency of a doomed
  funnel is "click → unavailable" loops; clients need retry hygiene, and the UI must
  express contention gracefully ("this section just sold out — pick another") rather
  than spinning.

**We pick B.** The deciding argument is arithmetic: demand pressure is ~300×
oversubscription (02), so *almost every attempt would fail regardless of locking
strategy* — pessimistic locking converts those failures into lock-queueing (worse
latency for the winners too), while optimistic CAS converts them into fast,
honest, retry-able rejections. When a resource is this scarce, "fail fast and get out
of the way" beats "queue up to lose slowly".

- Cross-check the invariant: even with CAS, a seat is never sold twice, because
  `HELD(hold_a) → commit(a)` and any other `hold`/`commit` on the same seat fails
  against current state; the log replays to replicas keep shards consistent after
  failover.
- **Defense in depth**: purchase caps and hold-count caps enforced again *inside*
  inventory (`max_open_holds_per_account`), because edge/admission controls are
  probabilistic and the invariant is not.

Failure modes:

- *Client-side staleness*: user selects seats that went HELD 400 ms ago. Revalidation
  at the hold step makes this an honest, rare failure; UI refreshes the map from
  cache. Do *not* try to push real-time map deltas to every client — the write fanout
  at 50K map-readers (02) is exactly the "read amplification" trap. Poll-and-revalidate
  instead; freshness of the *read* is a UX nicety, correctness lives in the CAS.
- *Shard failover mid-hold*: HELD state survives via the log replay (see dive 6);
  a hold issued seconds before failover is either in the log or lost with the
  in-flight RPC — the client's retry with the same hold_id is idempotent.

## 3. Contiguous-group allocation: selling neighbors, not singles

"Best available, 4 together" is a different search problem than "is seat (x,y) free".
Naively scanning and selling any N free seats creates single-seat rubble — later
buyers find a 60% empty map with only non-adjacent singles, inventory that cannot
satisfy any group request is effectively unsellable for the majority of demand
(estimates: most on-sale demand is even-sized groups of 2–4).

**Option A — always pick-your-own-seat.** Sell exact seats; no grouping logic.

- wins: trivially correct inventory, perfect seat choice, and the map state is the
  user's own model.
- loses: group buyers can't coordinate during a fast sale (by the time a friend picks
  seat 14, 13 is gone); produces the fragmentation problem by construction; general-
  admission floors dodge this, but arenas have seats.

**Option B — global best-available.** On each request, scan the whole map for the
best contiguous block (fewest rows from stage, then tightest fit). Return THE answer.

- wins: maximizes sale quality (families get seats 12–14 of row J; nobody gets
  stranded singles unless they choose them); minimal client reasoning.
- loses: the scan is global state contention — every request touches the whole map,
  which is the opposite of the hot-key sharding story; and "best available" for
  group N changes the answer for everyone behind you in line (visible rank shuffling).

**Option C — per-strip partitioned allocation.** Divide the venue into allocation
units that are *small and independent*: floor sections, level/section/row strips.
Each strip has its own atomic allocator; a group request for N lands in the strip
best matching the tier, and allocation within a strip is a tiny contiguous-fit search
over that strip's bitmap only.

- wins: contention on strip keys, not one global map state; matches dive 6's
  sharding story (strips are the shard unit); fragments within a strip but
  fragmentation is *localized* — a strip is small (a row or a few rows), so a gap of
  1 seat does not poison the whole venue; and strip-level "remaining count" is the
  coarse availability data the seat map UI shows anyway.
- loses: no true global best-available (strip choice is by tier/section heuristic,
  not distance-from-stage optimum per request); a full-arena "best available"
  requires consulting several strips and picking the best *found*, which admits
  mild suboptimality; also inventory rebalancing (a strip whose group-seats are gone
  but singles remain) needs its own policy.

**We pick C**, with a documented search order (tier → section quality → first strip
fit) so the answer is deterministic per request. The trade-off accepted: we give up
globally-optimal seat placement to keep contention partitioned and allocation O(strip)
instead of O(venue). At 300× oversubscription the *quality* of seats found matters
less than *whether the funnel completes at all* — and the strip model keeps the
atomic fast path narrow.

- **Singles policy**: allow selling singles into a strip's gaps only past a threshold
  (e.g., strip < 30% remaining), converting "unsellable single rubble" into revenue
  rather than blocking buyers. Explicitly a trade-off: wins revenue from otherwise
  dead inventory; loses some ability to satisfy late group requests.
- Map/UI note: with per-strip allocation, availability aggregates per strip are
  already the cache-tier unit — the read path and write path share a shape, which is
  why this design hangs together.

Failure modes:

- *Fragmentation hot spot*: all demand lands in the front 3 tiers (02 estimate:
  ~60% of demand). Front strips fragment first and get hammered; back strips idle at
  100%. Mitigation: admission presents tier choices with live availability; some
  demand shaping via "best available in section X" entry points.
- *Strip allocator hot key*: a single strip's allocator does tens of ops/s — fine;
  but the *most desirable strip* (front-row of the front section) can do thousands.
  It is a bounded key though: once sold out, contention evaporates. Design for
  "contention until sold out, then zero" — don't size for sustained peak on one key.

## 4. Checkout: idempotency and exactly-once purchase

The purchase involves an external payment authorization and multi-second network
round-trips at the worst possible moment: 10:03, user on cellular, refresh clicked
twice. Failure handling is the whole design.

### The mechanism (idempotency keys + order state machine)

- **Idempotency key** minted at hold time (the hold_id doubles as the checkout
  idempotency key). Every downstream call — order creation, payment authorization,
  capture, ticket issuance — carries the key; receiving side deduplicates on it. This
  is exactly the receiver-side dedup pattern in
  [06.apple-app-store](../06.apple-app-store/04-deep-dives.md): producers are
  at-least-once, receivers are idempotent → *effectively exactly-once*.
- **Order state machine** (authoritative, in the order DB):

```text
 PENDING_HOLD → PAYMENT_AUTHORIZING → AUTHORIZED → CAPTURING → PAID → FULFILLED
      │                  │                │               │
      └─ TTL expire      └─ auth fails    └─ capture      └─ capture fails ⇒
         → release seat     → release seat   retry/backoff   reverse auth, release
```

- **The dangerous step is capture-after-authorize.** Authorized money is a promise;
  capture must succeed exactly once. If the capture call times out, retry — *only*
  safe because the idempotency key makes the rails' duplicate-capture a no-op.
  Capture retries happen against a webhook/event-log consumer, not inline in the
  request path (the user wait must end at "authorized", seconds, not at capture).
- **The seat→order atomicity**: commit of a hold to SOLD is a single CAS on the
  inventory shard keyed by hold_id, and the order write + capture proceed only after
  it. If order-DB write fails after commit, the hold_id retry path rebuilds the
  order from the log; the seat never falls between (either HELD or SOLD, and the
  hold_id either resolves to an order or to a reaped release).

**Option A — synchronous capture in the checkout request** (charge then confirm).

- wins: simplest mental model; the user's "Purchase" click settles everything.
- loses: request duration now includes the payment rails (multi-second, sometimes
  retry-worthy), and the retry surface is exactly where a naive implementation
  double-charges; a stalled checkout holds the seat via a request thread, not a
  designed TTL.

**Option B — two-phase (authorize inline, capture async), hold_id as idempotency
key throughout** (described above).

- wins: the user-visible latency path ends at authorization; all the dangerous
  retry/reconciliation lives in an at-least-once pipeline with dedup where retries
  are safe by construction; capture failures are reversible (release the hold,
  reverse the auth) without money vanishing.
- loses: an extra state (AUTHORIZED≠PAID) that support, accounting, and the reaper
  all have to understand; a capture can fail after the user *believes* the purchase
  completed, requiring a "your payment failed, seats released" flow — a real, ugly
  edge case that must be explicit UX, not swallowed.

**We pick B.** The failure modes we are buying protection from (double charge,
seats released on a paid order) are the two most catastrophic in this domain;
both are structural consequences of Option A's single synchronous path.

- **Exactly-once is a lie; effective exactly-once is the deliverable.** Network
  retries, webhook redeliveries, and manual ops replays are all at least once. The
  invariant the system actually maintains: one (hold_id, account, event) → at most
  one PAID order → at most one capture → at most one issued ticket set. Prove it in
  the reconciliation job, N7 in [01-requirements](01-requirements.md).
- **Reconciliation** (async, hourly during spike): money-in vs orders vs seat
  transitions vs holds-timed-out; any inconsistency is a support-grade incident, not
  a silent drift. This is where seat-map correctness and money correctness meet and
  are *checked*, not assumed.

Failure modes:

- *Buyer double-clicks "Buy"* → same hold_id → same order (dedup), no second auth.
  If two different hold attempts exist (user abandoned seat A and grabbed B), the
  cap-of-open-holds logic and, at commit, the "one active order per account per
  event" constraint resolve it — last-committed wins, earlier hold released.
- *Auth OK, capture fails repeatedly* (true flaky rails) → reverse auth, release
  seat after N retries, order state REVERSED, user told plainly. Ticket inventory
  returns via reaper path; nothing leaks.
- *Crash between commit-seat and write-order*: replayable from the event log by
  hold_id; the seat is SOLD and the order re-derived — the log is the source of
  truth, the table is a projection (13.kafka's core pattern).

## 5. Anti-scalping and anti-bot: resisting an adaptive adversary

Publicly reported context: platforms have estimated *bot traffic at a large multiple*
of fan traffic during hot on-sales, and Ticketmaster publicly blamed several-fold bot
surge for the 2022 Eras presale failure. Assume a well-funded adversary: residential
proxies, account farms aged over months, headless browsers that pass naive
fingerprinting. Assume every static defense has been (or will be) reverse-engineered
within weeks. Design for *economics*: make cheating expensive and imperfect, not
impossible — perfect is not on the table against this adversary.

### Layered measures, each with its cost

1. **Edge fingerprinting + challenges** (device/browser entropy, JS proof-of-work,
   TLS/HTTP2 client fingerprints).
   wins: kills the *cheap* bot fleets dead, raises per-account cost for everyone
   past amateur tier; nearly zero effort for legitimate browsers.
   loses: determined adversaries rotate fingerprints; privacy-sensitive browsers and
   proxy users get false-positived; every layer must be re-tuned as adversaries adapt.
   **Keep it:** it is the outer sieve and cheap.

2. **Account-layer restrictions**: verified-fan registration before the sale,
   purchase caps per account *and* per payment identity and per delivery address (a
   bot with 10,000 accounts still needs 10,000 payment methods and 10,000 places to
   receive tickets — these cross-checks are where account farms become expensive).
   wins: caps convert "infinite bots" into "bounded cost per bot"; the resale market
   (where bots monetize) is starved rather than the venue.
   loses: friction on legitimate fans (registration weeks in advance, identity
   friction); false positives lock real fans out with no appeal path under sale-time
   pressure. Must be tuned to *error on the side of letting a scalper through* vs
   locking a fan out — reverse defaults, since a scalper's marginal seat is a
   support-ticket-free event, a wrongly-banned fan is a PR problem every time.

3. **Behavioral scoring in the funnel** (click patterns, navigation entropy, time
   between map load and hold attempt — humans browse; bots go straight to the best
   request sequence).
   wins: cheap, adaptive, exploits the fact that bots optimize exactly what humans
   wander.
   loses: adversarially noisy — the bot authors read the same blog posts; risky for
   power users / accessibility navigation. Use it as *signal*, never as sole
   enforcement.

4. **Non-transferable / delayed delivery tickets** (the strongest anti-resale lever,
   applied per-event by promoter policy): ticket binds to the attendee's identity
   (or is delivered hours before showtime, canceling resale arbitrage).
   wins: attacks the *payoff*, which no in-funnel bot defense can do; if resale pays
   nothing, bot fleets lose their business model regardless of detection.
   loses: legitimate transfers (gifted tickets, groups arriving separately) become
   support heavy; some resale is *wanted* by promoters (fees) and by fans (life
   happens) — a policy lever, not a purely technical one; unpopular (see public
   sentiment around anti-resale laws and platform criticism).

**Trade-off synthesis**: we pick the layered stack above, because no layer is
load-bearing alone — each catches what the previous misses, and total resistance is
the *product* of the layers' capture rates (roughly, even a mediocre 50% catch at four
independent layers leaves ~6% of bots through, vs 50% for any single one). The
adversarial reality we accept: some scalping happens regardless. The design goal is
that a bot fleet's *expected profit* per ticket falls toward the noise floor, not zero
seats sold honestly.

Failure modes:

- *False-positive ban wave during a sold-out sale*: the highest-emotion, worst moment
  to wrongly lock fans; appeals take hours; support melts. Mitigate: bans during
  on-sale are **hold-blocking, not account-terminating**; reversible in minutes.
- *Defense becomes the DoS*: an expensive browser challenge per request at 650K req/s
  is itself the attack surface; challenges run at the edge, precomputed where possible,
  and are skipped for known-good clients (steady-state traffic, logged-in known devices).
- *The queue is the loophole*: bot fleets enqueue thousands of sessions and win by
  volume. Ties to dive 1: admission cohorts randomize within bursts, and per-session
  risk scores gate *admission*, not just purchase — otherwise the queue itself
  distributes seats to bots faster than any hold-time defense.

## 6. Event inventory sharding: the per-event hot shard

Most sharding stories assume keyspace spreads load. Ticketing breaks the assumption:
one event's demand converges on one event's inventory. Shard by user → doesn't help;
shard by geo → doesn't help; shard by event → *every hot event is exactly one shard*.
This is the purest hot-partition problem in the repo (contrast Kafka's partition
keying, which spreads by default, in [13.kafka](../13.kafka/04-deep-dives.md)).

### The problem stated in numbers (02)

- ~15–20K hold attempts/s, all atomic, all against one event's seat state.
- A single process holding the whole event's state can do roughly ~10^5 simple
  in-memory atomic ops/s (estimate for modern hardware) — at first glance it fits.
  But three things bite:
  1. *multi-seat groups* make each atomic op touch several seats → effective
     throughput lower than the naive op rate;
  2. *front-tier concentration* (02: ~60% of demand → 3 tiers) makes keys *within*
     the event uneven too — a few strip keys absorb a majority of ops (dive 3);
  3. *failover* : the shard is a stateful singleton that must survive host death
     mid-sale with committed-holds intact.

### Options

**Option A — one strong node per hot event** (the obvious shape: inventory state
in-memory on one primary, replicas for failover via the event log).

- wins: dead simple reasoning — all atomic ops on one event are strictly ordered by
  one process, group-atomicity trivial, no coordination protocol to get wrong; and
  the arithmetic above says one node's ceiling is (barely) enough.
- loses: headroom is thin (~20K/s against a ~100K/s ceiling feels fine until
  multi-seat groups, GC pauses, and the front-tier skew each take their cut); a
  hot-er-than-assumed event has nowhere to go (vertical scaling only, mid-sale is
  too late); single-node failure during a sale is the worst possible moment for a
  rebuild.

**Option B — shard the event's seats into strips/slices (per-strip allocation from
dive 3), strips spread across several nodes.**

- wins: load actually divides (strip contention is *localized* — a front-row strip
  saturating does not block the upper deck); a seat's atomicity scope is a strip,
  not the venue, matching how allocation actually works; failure of one strip node
  costs one strip, not the sale.
- loses: cross-strip requests exist (a buyer wants "best available anywhere in
  tier" needs to consult multiple strips, then commit against one); the "no double
  sale" invariant now spans a routing/composition layer, more moving parts to
  reason about; global map state (for UI aggregates) becomes an assembled view,
  slightly staler.

**We pick B** — strips are the shard unit (dive 3's allocation model is also the
sharding model, which is why the two dives agree). Strip-sized keys spread across
the inventory fleet, with the event's routing table mapping strip → node; per-tier
"best available" fans out a read to candidate strips and commits to one. It buys
horizontal headroom exactly where the load concentrates (front tiers), and confines
failure blast radius. Accepted cost: cross-strip coordination for tier-wide
availability queries, paid at read time (cached aggregates) rather than at write
time (atomic ops) — the right side to pay on, per 02's read/write ratio.

### Making the stateful shard survivable mid-sale

- **Event log as the source of truth** (Kafka-shaped, [13.kafka](../13.kafka/04-deep-dives.md)):
  every seat transition is an event; a spare node replays the strip's partition from
  its last snapshot + log to become primary, in seconds. Hold semantics make this
  safe: an in-flight hold at failover time either made it to the log (survives) or
  didn't (client's idempotent retry with same hold_id re-executes — dive 2).
- **Snapshot + replay for rebuild** (compact the log periodically per strip) — the
  same machinery as a database, not novel, but often forgotten for "just a Redis"
  seat state. Without it, shard failure during a hot sale = lose the sale.
- **Cross-event isolation**: one on-sale's storm must not touch other events'
  inventory nodes or catalog capacity (per-event routing + cache namespace
  separation, [10.distributed-cache](../10.distributed-cache/04-deep-dives.md) for
  the cache-tier shape). The catalog tier is deliberately *not* co-tenant with
  hot-inventory traffic.

Failure modes:

- *Strip skew*: a viral artist drives 90% of demand to one section → that strip's
  node is the bottleneck again (hot key *within* the hot shard). Answer: bounded
  contention by design (a strip sells out and contention drops to zero — dive 3),
  plus admission shaping away from sold-out tiers. Vertical scaling of that one
  strip's node is a legitimate last resort; pre-stage the bigger node *before*
  on-sale day (N6's freeze means you sized it ahead, not tuned it live).
- *Routing table drift / split brain* after a node swap: the old primary still
  answering from stale memory. Mitigate: strip primaries hold a fencing token (an
  epoch bumped on failover) that the log consumer checks before accepting writes —
  the same fencing-token pattern the lock-service world settled on (see
  [11.zookeeper](../11.zookeeper/04-deep-dives.md), dive 3, exactly the same idea).
- *Reaper lag on the log consumer* (dive 2's SLO) becomes a *shard-level* risk:
  if the consumer group lags, expired holds sit "HELD" on the rebuilt primary too.
  Monitor consumer lag as part of inventory health, not as a queue-metrics footnote.
