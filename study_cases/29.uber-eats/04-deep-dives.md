# 04 — Deep dives

## 1. The order lifecycle state machine

The lifecycle is the product's contract between three parties who don't trust each other.
It needs to be: explicit (no implicit transitions from side effects), idempotent (any
party retries on a flaky network without duplicating effects), and recoverable (a silent
party must not wedge the order forever).

### States and legal transitions

```
                       customer                 restaurant
                    places order                accepts/rejects
                         │                            │
                         ▼                            │
  [CREATED] ──────► [PLACED] ──────────► [ACCEPTED] ───┤── ▲
      │                 │                    │        (reject →
  payment auth  restaurant ignores     [REJECTED]     refund,
  fails         (timeout ⇒ auto                ▲      R: cancel)
                cancel or auto-accept)         │
                                               │
  [ACCEPTED] ─► [PREPARING] ─► [READY]         │ (stale-order recovery, 04.1b)
                    │             │            │
                    │             ▼            │
                    │        courier arrives ─►[PICKED_UP]
                    │                          │        │
                    │                          │     [DELIVERED] ─► [SETTLED] ─► [COMPLETED]
                    │                          │                                    (ratings window)
                    │   restaurant never marks │
                    │        ready             │
                    └──────────► [STUCK? no — see recovery below]
                  customer cancels (windows per stage) ──► [CANCELED]
                  courier no-show / failed delivery ──────► [REASSIGNED | FAILED]
```

Deliberate properties:

- **Transitions are named, validated events, not implicit writes.** Any service can propose
  `PREPARING → READY`, but only the order service applies it after checking that the
  proposer is authorized for that transition (courier may not mark ready; restaurant may
  not mark delivered). Option A (any party writes what it observes) wins flexibility and
  loses exactly what we cannot afford here — two sources of truth and irreproducible order
  histories. We pick the validating-owner model because the ledger settlement (see
  [06.apple-app-store](../06.apple-app-store/README.md)) downstream depends on a
  reproducible sequence of transitions, not on whoever's write arrived last.
- **Idempotency at the transition level**: each attempt carries `(order_id, from_state,
  to_state, request_id)`; retry resubmission is a no-op if the order is already beyond the
  target state. This is what makes tablet flakiness survivable.
- **Money-relevant transitions are the only hard gates**: delivery authorization, refund on
  cancel, courier payout on delivery. Everything else is recoverable by forcing state
  forward.

### Failure modes

- *The restaurant's tablet dies mid-service and never marks READY.* **Recovery model**:
  per-merchant watchdog fed by merchant-ops telemetry (historical prep time + current queue
  depth). If an order in `ACCEPTED`/`PREPARING` state exceeds `p95 prep time × 1.5` with no
  transition, the order service escalates: first pushes a reminder, then force-advances to
  READY (or dispatch assumes-ready — see 04.5) and flags the merchant. Option A (a human
  support loop) wins accuracy and loses availability at scale (millions of orders, we can't
  staff the wedge); Option B (auto-force-forward) wins liveness and risks courier/food
  mismatches when the force is wrong. We pick auto-force-forward with a flag and a customer
  apology credit, because a wedged order at dinner peak is strictly worse than one wrong
  force-forward.
- *Courier marked delivered but the food never arrived (fraud or mistake).* The customer
  refund flow reads the ledger + verification inputs (dropoff confirmation code, customer
  complaint, route anomaly from the telemetry trail) and reverses via a compensating ledger
  row — never edits history. Same ledger discipline as
  [06.apple-app-store](../06.apple-app-store/README.md).
- *Customer cancels after pickup.* Policy branch: cancel is legal only before pickup;
  after `PICKED_UP` it becomes a support-initiated return flow (courier may keep the food,
  customer refunded, merchant compensated — an explicit business rule, implemented as its
  own transitions, not as an exception in the state machine).
- *Split-brain between dispatch and order state (assignment lost during dispatch-service
  failover).* The order service is the only writer of courier-assignment transitions, from
  `dispatch.*` events that carry idempotency; a re-run of a dispatch round after failover can
  safely re-offer only orders whose current state permits it (re-assignment is legal for any
  order not yet `PICKED_UP` — the `OFFERED`/`ASSIGNED` states live in dispatch's own machine
  (04.2), not in the order machine above, which records only the resulting
  courier-attached/detached transition — and re-emit collapses to one transition).

## 2. Courier allocation / dispatch

Same matching core as ride-hail ([28.uber](../28.uber/04-deep-dives.md): candidate pruning
→ batch scoring → assignment under latency), with one addition that changes the problem: a
**staged timeline** (courier arrival ≠ pickup, because the food is not ready yet) and a
**second clock** (kitchen prep time, 04.5).

### Candidate generation

Per unassigned order, prune to a candidate set of ~50 couriers (estimate) by geo-proximity
(kd-tree / geohash cells in the supply index) and eligibility (not on another committed
order, or on a batch whose detour is legal — see below). This is the well-understood part;
the interesting part is scoring and commit timing.

### Scoring: more than "nearest courier"

```
score(courier, order) ≈ α·travel_time(courier → restaurant, arriving at ~food_ready_time)
                      + β·|courier_arrival − food_ready|      (alignment penalty, 04.5)
                      + γ·marginal_delay_to_customer          (post-batch detour cost)
                      + δ·courier_supply_erosion              (don't drain a hot zone)
```

- **γ matters because of batching**: assigning a courier who is already 2 deliveries in
  imposes a marginal cost on their committed orders. The scorer prices that detour. Option A
  is nearest-first (wins simplicity and loses exactly the batching case: nearest is often
  already committed); Option B is a full batch-aware assignment. We pick B with a capped
  candidate set because, at dinner peak density, most couriers *are* mid-assignment.
- **δ is the market-level term**: assigning the last courier standing near a dense lunch
  cluster burns future supply. This is the same market-balance idea ride-hail makes with
  pricing ([28.uber](../28.uber/01-requirements.md)) — here we express it in the score, not
  the fare, which is a product decision we note but do not design.

### One-to-one vs batching

**Option A — one-to-one, assign at order acceptance** (courier goes to the restaurant and
waits):
- wins: simple, lowest pickup-latency variance, no batch detours.
- loses: wastes courier supply — the courier occupies a parking spot during prep — and
  starves the market during peaks (supply is idle-but-committed).

**Option B — batch (two orders per courier), assign opportunistically**:
- wins: throughput per courier hour; this is the mechanism that makes dense lunch rushes
  feasible (publicly, delivery platforms describe batching as a core peak-density tool).
- loses: delivery latency for the *second* order (its food waits or the courier detours),
  worse ETA accuracy, degraded courier experience (more stops, more parking), and coupling
  between two customers' orders.

**Chosen: staged hybrid.** Assign a single courier to the order as soon as the cost model
says arriving *after* food-ready is achievable (i.e., "the courier is the *planned* pickup
agent, whose arrival is timed to the kitchen, not to the order"), and offer a batch only in
dense markets where scores say the marginal detour is inside the promised ETA band. Trade-off
stated explicitly: some orders get a slightly later first-assignment in exchange for the
market sustaining peak throughput — and the committed customer ETA has to absorb batch
detours, which is why ETA bands exist (04.5).

### Assignment loop mechanics

- **Round-based, not event-based**: every ~2-5s (estimate) per region, dispatch evaluates
  the unassigned queue and current supply, not per-order event triggers. Option A (assign
  on order placement, immediately) wins perceived responsiveness and loses the chance to
  wait the 30s that produces a much better alignment score; Option B (pure round-based)
  wins planning quality and needs a hard upper bound on time-to-first-offer (we set
  <10s). We pick round-based with a per-order offer deadline.
- **Offer with optionality**: the courier sees fare + pickup/dropoff ETA + a short accept
  window. Decline → re-offer to the next candidate; timeout → re-score. The loop's
  invariant is "no unassigned order outlives its deadline without an explicit reason logged"
  — the dispatch equivalent of the reconciliation watchdog in 04.1.
- **Failure modes**:
  - *Supply collapse in a hot zone* (rain, stadium): score δ mitigates locally, but the real
    remedy is outside this system's write path — courier incentives and fee surging, which
    we deliberately scoped out ([28.uber](../28.uber/01-requirements.md) draws the same line).
  - *Wrong prep estimate ⇒ cascade*: a restaurant that marks food ready at 19:45 when it
    said 19:15 wedges every courier aligned to 19:15 — this is why 04.5's alignment penalty
    must be recomputed online instead of baked into an immutable assignment.
  - *Dispatch service regional failover*: assignments re-scored, not remembered; the
    authoritative state is the order service's.

## 3. Live order tracking

What the customer sees: courier dot on a map, ETA band, lifecycle stage. What makes it hard
isn't the map — it's that (a) the load shape is connection-count-bound (~4M concurrent
sockets at dinner peak, 02) rather than byte-bound; (b) the *update sources* are different
services with different clocks; (c) the courier's privacy needs a mask applied to the
*customer's* view, not the courier's feed.

### Option A — polling by clients

- wins: stateless servers, no socket tier to operate, retry behavior is free.
- loses: at 4M clients polling every 5s we'd generate ~800K QPS of read amplification for
  mostly-unchanged state and give up position freshness (or hammer the same stores).

**Option B — push via per-order rooms over persistent sockets**

- wins: fresh deltas, load proportional to number of *changes*, not number of *clients*;
  per-order room multiplexing means one courier's position push reaches customer, merchant,
  and any support viewer without re-computing.
- loses: a socket tier to operate — connection churn at dinner-start, sticky sessions on
  reconnect, a fanout service with real state.

**Chosen: B** — the load shape (many viewers, few changes per second per order; positions
changing every ~10-30s, lifecycle changes rarely) is exactly the shape a room/fanout design
beats polling on. Same reasoning as the map telemetry case
([26.google-maps](../26.google-maps/04-deep-dives.md), driving-state loop) — but here we can be
simpler: no route re-planning, only position and ETA *display*.

### Tracking updates are recomputed, not recorded

- Courier position → tracking service pushes coordinates at most every ~5-10s per order to
  each room. Positions are latest-wins; nothing in tracking writes them durably beyond the
  downsampled trail.
- ETA updates are *derivation*, carried by the ETA service: every position + stage change
  triggers a re-estimate of remaining legs; the tracking service publishes the band, not a
  single number (see 04.5 for why).
- Lifecycle stage changes come from order-service transitions, not from tracking-side
  inference. Option A (tracking infers "nearby the customer ⇒ deliver arrived") wins
  freshness and loses the one-party-writes principle — it would let a GPS glitch mark an
  order delivered, with a payout attached. We pick stage events only from the courier app's
  explicit actions plus geo-sanity checks that merely *warn*, never write.

### Privacy and correctness of the customer's map

- During the *before-pickup* phase, the customer's view shows the courier en route to the
  restaurant, not the courier's full live location for other jobs; during *on-the-way*,
  the dropoff destination is shown but neighboring orders in a batch are not visible to
  the customer. Option A (full route visibility) wins transparency and loses courier safety
  and batch-order confidentiality; Option B (a masked polyline, recomputed per leg) is what
  we pick — the mask is a projection of the same position feed, not a separate feed to
  maintain.
- Verification at dropoff (order code) exists to stop the "courier says delivered, food
  isn't there" and "wrong building" failure modes; its failure mode is friction (courier
  can't reach the customer's door code) — resolved with support escalation, not by
  weakening the code check.

### Failure modes

- *Socket storm at dinner start*: 4M reconnects. Mitigation: jittered reconnect backoff,
  room restoration via the order service's state (client refetches state after reconnect
  instead of replaying missed deltas), and a global connection-count guard that can degrade
  freshness (10s → 30s push interval) rather than drop sockets.
- *GPS drift indoors / high-rise*: courier "at the building" but geocoded to the wrong side
  of the block. Same map-matching problem as
  [26.google-maps](../26.google-maps/README.md), scoped to last-mile; the UI uses arrival
  states from the courier, not inference from position alone.
- *Stale position* (courier app backgrounded by the OS): the tracking surface degrades to
  "courier nearby" region + ETA-only refresh, and a notification asks the courier to
  reopen the app — a *soft* recovery path, because we cannot force a mobile OS to keep a
  socket or GPS alive.

## 4. Restaurant menu / catalog management

The read:write ratio here is ~50:1 (02), and the catalog is the product surface the
customer spends 80% of their session in. The core question is: what does "menu edit"
mean, and when is it visible?

### Option A — mutable catalog, live lookup at order time

- wins: trivially low storage; menu changes take effect immediately everywhere.
- loses: the order (and its settlement) depends on what the menu says *at checkout* vs *at
  delivery*; a price change mid-flight produces unexplainable totals; and every order-time
  cart validation becomes a live catalog round-trip on the hot write path.

**Option B — versioned publishes + snapshots**

- wins: in-flight orders are priced against the version they were placed against — the
  total the customer saw is the total they're charged; CDN can cache version-keyed content
  aggressively (the immovable-key, bounded-cardinality win the CDN analysis in
  [09.cdn](../09.cdn/04-deep-dives.md) argues for); merchant edits can't corrupt live
  orders.
- loses: staleness (a "sold out today" toggle takes minutes to propagate), two storage
  representations (current + in-flight versions), and a publish pipeline to build.

**Chosen: B**, with a fast-lane for merchant toggles. Stock-out flags and "restaurant
closed today" bypass the versioning lane and go straight to a hot overlay cache (per-market
Redis) checked at cart-add and at order-accept time — because these aren't catalog
*structure*, they're *state*, and their tolerance is seconds-to-minutes, not hours. This is
the same split (structure vs live availability) that makes the price snapshot cheap and the
availability flag responsive.

### What the version actually contains

A menu version is a full JSON snapshot of structure + a set of item ids + prices + modifier
invariants; photos are blob references (immutable, content-addressed, served from CDN).

### Ordering, constraints, and the failure modes worth naming

- **Modifier invariants** ("pick one of: size S/M/L"; "max 3 toppings"; "must choose sauce
  if spice >= medium") are the hardest data problem: they encode restaurant business logic
  that varies per merchant. Option A (encode as a generic DSL the merchant configures) wins
  expressiveness and loses support simplicity (a bad DSL expression can break the checkout
  for one merchant invisibly); Option B (a small fixed grammar of constraint kinds) wins
  predictability and loses rare-but-real cases. We pick B with the escape valve that exotic
  merchants can be served via merchant-ops configuration (a human-supplied exception), not
  via DSL generality — because checkout breaking at dinner peak is worse than a merchant
  needing manual help.
- *Failure: publish storm at menu refresh time (merchant updates all prices at 17:00).* CDN
  purges are keyed by menu version; a publish is idempotent on version id, and a burst of
  publishes is queued by per-merchant sequence so the last publish wins rather than an
  interleaved last-write-per-item.
- *Failure: the order service's snapshot is stale relative to a stock-out toggled 30
  seconds ago.* Consequence: the courier shows up to a restaurant that doesn't have the
  item. Mitigation is operational: order-accept flow rechecks the overlay, restaurant-side
  flows allow item-level substitution/refund at accept time, and settlement handles partial
  fulfillment as a ledger event (refund of a line item), not as an order rewrite.

### Cross-reference

Menu/versioning is a case of the general "immutable snapshots keyed by version, mutable
metadata in a hot overlay" pattern — the same shape as blob-store object immutability
([07.blob-store](../07.blob-store/README.md)) with a hotter availability layer.

## 5. The double-sided timing problem (food ready vs courier arrival)

This is the problem that distinguishes delivery dispatch from ride-hail. Two processes run
concurrently and their outputs have to converge on a single handoff moment:

- **Kitchen process**: prep time depends on the merchant (kitchen load, menu complexity,
  time of day), is *partially observable* (the merchant's queue is visible, the cooking
  rate is not), and is *controllable by a human* under stress — not a pure predictor.
- **Courier process**: travel time depends on roads, weather, parking, and batching
  detours; is *fully observable* once assigned; and is controllable only by changing the
  assignment (courier wait time is the knob).

The naive model `ETA = prep + travel_to_restaurant + travel_to_customer` is wrong term by
term:

- The **prep term** is the least observable and the most variable (estimates vary by
  merchant discipline, not just by menu). It must be a distribution, not a number.
- The **travel-to-restaurant term** is conditioned on *when* the courier leaves and *when*
  the food is ready — the two are joint, not additive.
- The **travel-to-customer term** is a function of the *route taken*, which for batched
  orders has a detour the customer can't see.
- The **handoff term** (parking, finding the right door, verifying the code) is the term
  most often missing entirely from naive models and is often the dominant variance source
  in dense city deliveries (estimate; this is where a "5 min" ETA loses its first 5
  minutes).

### What the system actually does (design, not just prediction)

The timing problem is a **coordination** problem, and the coordination happens by changing
*when* the courier arrives, not only by predicting.

- **Dispatch-time alignment**: the assignment scorer's β term (04.2) explicitly penalizes
  courier-arrival-vs-food-ready mismatch. Option A (dispatch immediately on order
  acceptance, courier waits at the restaurant) wins "food never gets cold" and loses
  courier-hour throughput and supply density — the restaurant queue starts collecting
  couriers. Option B (dispatch only on READY) wins supply and loses 5-15 minutes of
  delivery latency (the whole pickup leg is added after cooking). **Chosen: timed
  dispatch** — send the courier so as to arrive at *predicted* ready time, and recompute
  arrival as the prep estimate updates. This is the middle that both A and B lose
  throughput/latency to avoid.
- **Re-timing on update**: every prep-estimate revision re-scores courier arrival time —
  if prep slips, the courier's ETA band widens (the customer sees an honest band, not a
  stale promise), and if prep is *early*, dispatch may re-route a nearby courier before
  they set out. The system's commitment is to publish the band, not to deny the change.
- **Merchant-side pressure**: for systematically slow merchants, the restaurant tablet
  shows a projected "start cooking now" time rather than just an order queue — prep-aware
  sequencing (an input to merchant ops, not a dispatch decision). Option A (platform
  sequences the merchant's kitchen from the platform) wins throughput and loses
  overreach — kitchens are human systems with their own constraints; we show the timing and
  let the merchant act.

### Trade-offs, stated

- **Commit early and adjust** (vs. compute at pickup): the customer's committed ETA at
  checkout is a joint estimate over prep+both travel legs+handoff, published as a band.
  Recomputing on every state change is the design; the cost is a per-order pipeline load on
  the ETA service (tens of thousands of route queries per second at peak, 02), justified by
  the fact that ETA miss-rate is a top-line product metric.
- **Batch detours priced in the committed ETA**, not hidden (04.2) — Option A (surprise the
  customer post-purchase with a longer ETA) wins short-term conversion and loses trust and
  refund rates; we pick the price-in-the-band model.
- **Courier wait is bounded**: a courier waiting >N min at a restaurant (estimate: 10-15
  min) triggers a compensation policy and an ops flag on the merchant. Option A (always
  compensate) protects couriers and is exploitable (couriers camp); Option B (never
  compensate) sours supply. We pick bounded compensation plus an explicit merchant-facing
  slow-prep metric, which turns the failure into an incentive.

### Failure modes

- *Systematic prep underestimation at one merchant*: couriers queue for a kitchen that
  isn't done. Detected by a widening gap between predicted vs actual pickup timestamps at
  merchant granularity (merchant ops analytics); remediation is a per-merchant prep-time
  multiplier recalibration + courier-comp policy, not a global model change.
- *Prediction overfit to lunch, wrong at dinner*: prep models trained per-daypart because
  the kitchen load distribution differs; monitored per (merchant × daypart) bucket.
- *Batch-order cascade*: courier assigned two orders on opposite sides of the same block;
  the second customer's ETA absorbs the first's parking/verification time. Mitigated in the
  scorer's γ, but if it slips through, support and auto-credit are the recovery — the
  failure mode is *product-visible*, so the recovery has to be too.
- *The courier arrives before food (or food before courier)*: the state machine (04.1) plus
  timed dispatch should make this rare; when it happens, the tracking surface reports
  "waiting at restaurant" honestly and settlement pays the courier for bounded waiting.
  Option A (hide it, show a smooth ETA) wins UI simplicity and loses trust — we show it.

---

**Recap of the five dive positions, in one line each**: the state machine is a validating
owner, not a consensus; dispatch is timed-to-the-kitchen, not nearest-first; tracking is a
recomputed projection, never an authority; the menu is versioned snapshots with a hot
availability overlay; and the double-sided timing problem is solved by *changing* courier
arrival time, not only by predicting it.
