# 04 — Deep dives

## 1. Driver location updates at scale — sampling, batching, hysteresis

**Problem**: ~1M drivers online each emitting positions; raw GPS-every-second would be
~1M updates/s for near-zero benefit because most drivers are parked waiting for offers,
and map views only need ~5 s freshness.

**Sampling — adaptive cadence driven by usefulness of the next sample.** The client
changes cadence by state: moving driver every 3 s, waiting driver every 15 s, trip-idle
(arrived, waiting for rider) 10 s, backgrounded paused. Who decides? The *server hints
the cadence down* (per-driver "report interval" field the gateway pushes) but the
client decides up — [Case 22](../22.frontend-uber/04-deep-dives.md) deep dive 2 covers
why battery forces the client-side switch. The backend hysteresis avoids flapping
between cadences: a driver must be confidently moving (speed above threshold for N
consecutive samples) to escalate to fast mode, and confidently stopped for M slow
samples to drop back — otherwise a city bus's stop-and-go thrashes the write rate and
the index.

- **Option A: fixed 4 s cadence for all online drivers.** Wins: dead-simple client,
  uniform load (1M/4 s = 250K/s, no state machine). Loses: ~2× the write volume of
  the adaptive scheme (250K/s vs 140K/s derived in 02), drains driver batteries, and pays full
  fidelity for parked cars nobody queries. **Option B: fully server-decided per-driver
  cadence** wins freshness precision, loses client autonomy and correctness offline
  (a tunnel must have a fallback policy), and adds round-trips to every change.
  **We pick A-with-adaptation — client-executed, server-hinted, hysteresis-flanked** —
  because the dominant variable is the driver's own motion, observable best at the
  source, while the server hint lets us shed load during incidents (e.g. push
  10-s cadence city-wide during a spike: a one-line emergency valve).

**Batching.** The driver app batches consecutive samples when connectivity is poor and
uploads a burst with timestamps. This is a deliberate accuracy trade: during a 60-s
outage the rider sees a dead-reckoned car (client-side, Case 22) and on reconnect the
whole trace arrives at once, so the backend can rebuild the index and, on trip-trace
logging, keep the audit trail complete. Batching by the producer costs extra client
buffer logic; batching at the gateway instead (micro-batching 50–100 ms of samples
into one log append) is the server-side complement — one append transaction for
thousands of updates instead of thousands.

**Server-side handling.** Ingest is *latest-value-wins per driver*: each sample is a
full (id, lat, lng, heading, speed, accuracy, t) tuple, and the geo index overwrites.
No read-modify-write, no cross-driver coordination — the write pattern is
conflict-free by construction, which is what makes 140K/s cheap.

**Failure modes**. (a) GPS jumps 2 km (tunnel handoff): ingest validates against
last position — an impossible-speed jump is *stored* but marked low-confidence, not
silently applied to matching; a jump with high accuracy is trusted. (b) Malicious/
buggy client spoofing: sanity-bound coordinates, rate-limit per driver, cross-check
heading/speed plausibility. (c) Log partition lag: matching sees old positions —
mitigated by age-aware queries (deep dive 2) rather than by making the log unlosy,
which would couple the cheapest and most expensive paths. (d) A stalled driver app
stops sending: offline-detection hysteresis (no updates in 60–90 s ⇒ mark inactive)
rather than instant flagging, to tolerate tunnel silence without churning the
availability counters that feed surge.

## 2. Geospatial indexing for nearby drivers

**Problem**: answer "drivers within R of P" in ~10 ms over 1M moving points, in memory,
sharded by city, updated 140K×/s.

**Index choice**:

- **Option A: flat geohash grid.** Encode (lat,lng) → base32 string; store
  hash→driver-set in memory; a radius query expands from the driver's own cell
  outward (9 nearest cells at precision p). Wins: dead-simple, linear structure,
  per-cell sharding is free (cell string prefixes shard), O(1) updates (set
  mutation). Loses: cells are not size-uniform (equal lat/lng degrees ≠ equal area;
  distorted toward poles); boundaries force neighbor expansion that returns sets
  whose actual distances can be very uneven within one cell; precision must be
  picked per market density.
- **Option B: S2 / H3 hierarchical hex indexing.** H3's hex cells have uniform
  neighbor distance and clean hierarchy (parent/child coarsening); publicly, Uber
  has described using H3 for exactly this class of problem ("publicly reported" —
  their blog on hexagonal hierarchical spatial indexing). Wins: better distance
  uniformity (hex neighbors all equidistant, no corner problem), hierarchical
  resolution for matching's expanding search, natural fit for surge tiles (same
  index serves both pricing and matching — one geography, two features). Loses:
  hex↔latlng conversion complexity, less tooling familiarity, boundary issues
  still exist (just smaller and more uniform).
- **Option C: in-memory quadtree (R-tree family).** Wins: exact bounding and
  arbitrary radius queries without expansion; classic for this problem. Loses:
  trees rebalance under 140K updates/s — continuous node splits/merges are
  expensive and hard to shard by geography (a splitting node moves between
  machines), and a quadtree is per-server in-memory state, so the sharding story
  is painful.

**We pick B (H3-style hierarchical hex) with geohash-flat storage semantics** —
cells stored as `cell_id → set of driver ids` in a per-region in-memory service,
because the index must serve *matching, map views, and surge tiles* off one
geography, and hierarchical cells directly implement matching's expanding-window
policy (start at resolution-8 cell ≈ 0.46 km edge / ~0.74 km² of area, climb to coarser parents if empty).
A pure quadtree loses on sharding; a flat geohash loses on the pricing reuse and on
neighbor-distance uniformity in dense CBDs where that matters most. The residual
cell-boundary falseness (a driver 90 m away in the adjacent cell, a driver 800 m
away in the same cell) is fixed the same way in all systems: index for *candidate
selection*, then score and rank by true distance/ETA — the index is a filter, never
the answer.

**Sharding and mutation mechanics**: assign each cell to a shard by city+cell hash;
a driver's update goes to exactly one shard (their current cell); crossing a cell
boundary is a delete+insert, both in the same atomic operation at the shard. Latest
position per driver additionally lives in a per-shard hash map; the *set* membership
is the index, the map is the payload. Shard count sized by peak update rate, not
memory (200 MB total fits one box — sharding is for write fan-in and blast radius,
per 02).

**Freshness honesty**: queries carry `max_age`; drivers whose last update is older
than ~10 s are filtered from matching results and dimmed on the client. Serving a
stale "nearby driver" that never accepts is worse than serving a shorter list —
it burns a human round trip.

**Failure modes**. (a) Shard hot cell (airport pickup zone): split by cell
subdivision (drop to finer resolution) or by driver-id hash within the cell. (b)
A shard crashes: its cells have no serving view — matching in those cells
soft-fails to expanding search against neighbor shards' cached views rather than
returning nothing; supply counters degrade, surge may over- or under-react until
repopulation (deep dive 4 interplay). (c) Partition lag between shards feeding
aggregators: each shard is the source for its own cells, no cross-shard consistency
needed by design.

## 3. Real-time matching engine — two-sided market

**Problem**: convert rider intentions into driver assignments within seconds, in a
market where both sides can decline, leave, or lie by omission, and where the same
physical driver cannot serve two riders.

**The two-sided shape**: unlike a one-sided read (Case 26's place search, F2 there),
here *both* the object of the query (driver) and the querier's continuation (rider)
can walk away. The matching engine is therefore a small state machine of its own:
request → candidate-set → offer(s) → ack/decline → confirm or re-dispatch, with a
bounding timeout that always ends in either an assignment or an explicit no-match.

- **Option A: greedy, one rider at a time.** Query index, score, offer to best
  driver; if declined, expand to next. Wins: minimal machinery, minimal latency
  (p50 well under 2 s to *first* offer), trivially shardable per city. Loses: up
  to ~5–15 s of sequential human declines in a dense market before a match lands,
  which both degrades rider ETA and wastes driver attention.
- **Option B: continuous matching (offer to many drivers concurrently)**. Wins:
  fastest first-accept. Loses: needs strict reservation discipline to avoid
  double-accept (two drivers both believe they have the ride); burns multiple
  drivers' time per trip; the loser drivers feel "offer spam" (a documented real
  product pathology in rideshare dispatch).
- **Option C: batched assignment — accumulate requests for a short window (~2–5 s),
  solve a bipartite assignment over riders×drivers, minimize aggregate passenger
  ETA.** Wins: globally better assignments (batching reveals pairings greedy picks
  miss; publicly, Uber described exactly this batch-assignment approach in its
  dispatch research "publicly reported"), fewer wasted offers in dense markets,
  and the bipartite framing makes the fairness objective explicit. Loses: adds a
  fixed window to p50 (every request waits ~2 s); in a sparse market there is
  nothing to batch, so it adds latency for zero benefit; requires an assignment
  solver with a hard latency budget.

**We pick C for dense peak traffic with an A fallback** — batching when the local
supply/demand ratio says there is contention (≥ ~K open requests in a cell), greedy
immediate otherwise — because the pathological case for greedy (high contention)
is exactly the case where batching's global optimization pays, and the
pathological case for batching (sparse) is exactly the case greedy handles in
seconds. The batch solver does not need to be optimal: a greedy-by-delta heuristic
(order pairs by ETA improvement, assign greedily down the sorted list) recovers
most of the batching benefit for a fraction of the solve time, and any assignment
is strictly better than the timeout no-match. This mirrors
[Case 26](../26.google-maps/04-deep-dives.md)'s theme: real-time routing systems
prefer a good answer now to a better answer late.

**Matching query semantics**: geo-index query at expanding resolution (cell →
parent cell → parent), candidate filter (online, not on-trip, correct product tier,
fresh position, driver preference: e.g. destination-direction filters), then score
by passenger ETA (routing-call dependency, Case 26), with a candidate cap (~10–50)
so scoring cost is bounded.

**Reservation discipline (the correctness invariant)**: the winner's offer is
guarded by a compare-and-swap on the driver's state: `available → offered-to(trip)` —
a second trip cannot reserve the same driver because the CAS fails. This CAS lives
in the geo index shard that owns the driver's cell, which is also single-writer by
cell partition — no distributed transaction needed. Timeout (`offered-to` →
`available` after countdown expiry or app disconnect) is a timer, not a message,
because the failure mode (silent driver) is the common one, not the exception.

**Failure modes**. (a) Driver double-message (accept sent twice, or network retry):
idempotency key = trip id + driver id; second accept is a no-op returning the same
result. (b) Driver accepts but app dies before receiving confirmation: driver is
timed out on the offer, trip re-enters matching, rider sees "matching…" — the rider
never observes the phantom match unless the driver app surfaces it. (c) Matching
engine crash mid-batch: requests in flight are re-derived from the event log
(requests are events), idempotent re-dispatch. (d) No-supply pathologies: the
engine must answer "no driver" within the bound rather than expanding the search
radius city-wide (which produces matches riders will actually reject — worse than
an honest retry with surge applied). (e) Thundering re-dispatch after a surge step:
re-dispatch storms are rate-limited and jittered.

## 4. Pricing and surge — a control loop, not a lookup

**Problem**: compute a fare from distance/time plus a geographic multiplier; the
multiplier's job is *market clearing* — shifting demand down and supply up until
they balance — while remaining smooth, explainable, and not exploitable by gaming.

**Fare decomposition**: fare = base + per-km × distance + per-min × time + booking
fee, all multiplied by tier and surge multiplier. The distance/time component comes
from the same ETA/routing dependency used for matching (one routing call, shared —
don't compute twice). Surge multiplier is read per pickup tile (the same H3 cell
hierarchy as deep dive 2 — pricing *is* geospatial, which is why one index serves
both features).

**Surge as a control loop**: demand signal = matched-ride-attempts and open requests
per tile per window (~2–5 min); supply signal = available online drivers per tile.
Ratio → target multiplier via step function (`1.0 → 1.2 → … → 2.5` caps product-
limited), then *damping and hysteresis*:

- **Damping**: new multiplier = old + α(target − old) with α ~0.3 — smooths the
  measurement noise (one stadium event's first minutes are spiky) and, more
  importantly, prevents drivers from racing to the tile the second a spike prints.
- **Hysteresis**: the threshold for *raising* to the next step is lower than for
  *lowering* back; without it, a multiplier hovering at a boundary creates
  oscillation cycles — each oscillation is a system perturbing its own input
  (a textbook bad control loop).
- **Fairness/explainability**: multipliers are published per tile, not per rider;
  jurisdiction rules (publicly reported requirements in multiple markets) demand
  the rules be auditable and non-personal. No per-user variables in pricing — this
  is a design constraint from legality and trust, not from engineering preference.
  Every multiplier change is an audited event (retention requirement in 01).

- **Option A: surge as pure read (precomputed tiles)**. Wins: quote path is one
  cache read, ~0 latency, trivially cacheable ([Case 10](../10.distributed-cache/README.md)
  patterns). Loses: stale tiles during fast events; needs a separate update path
  that is the actual hard part.
- **Option B: compute-at-quote (read raw supply/demand, compute multiplier inline)**.
  Wins: always-fresh. Loses: quotes are 4.2K QPS and now each does windowed
  aggregation over the event log; the control-loop math (damping, hysteresis) needs
  *state across quotes*, which inline computation destroys — you need the tile
  history anyway, so you pay for the tile store either way, plus the read cost.

**We pick A** — precomputed tiles in a cache-fronted store, updated by a background
control-loop job off the event log, with a write-invalidate path and short TTL.
Compute-at-quote loses because the control loop is inherently stateful, not because
inline math is expensive. Cache TTL ~30–60 s (estimate) tight enough for the loop's
own cadence.

**Driver-side incentive display**: drivers see the *effective* earnings (multiplied
rate) before accepting, which is the supply-side arm of the control loop — without
it the multiplier shifts demand but not supply, and clearing never happens. The
display is part of the pricing system's correctness, not a UI nicety.

**Failure modes**. (a) Feedback oscillation (surge up → demand flees → supply comes
→ surge crashes → demand returns…): damped via smoothing + hysteresis + minimum
step dwell time. (b) Driver-side gaming (multi-apping to chase surge tiles):
mitigated by product rules (acceptance-rate floors), not by hiding the multiplier. (c)
Stale tile during a mass event (concert exit): the control loop's own latency is the
limiting factor; a faster loop (shorter window) costs noise and compute — the
engineering call is loop period vs. oscillation risk. (d) Pricing service outage:
fail closed to multiplier = 1.0 with trip-time re-pricing *blocked* (never silently
re-fare a rider after the fact; that is the trust-destruction failure mode). (e)
Cross-tile boundary arbitrage: rider walks 200 m to a cheaper tile — accepted;
suppressing it requires fare-smoothing across neighbors, which quietly reintroduces
all the opacity the rules forbid.

## 5. Ride state machine — server-owned truth

**Problem**: one ride spans ~30 minutes end to end (mean driving time 20 min per 02,
plus matching wait and arrival phases — estimate), four participants' worth of flaky clients,
multiple services (matching, pricing, fleet rules, payments-out-of-scope), and
millions of concurrent trips. The state machine must produce a *single correct
story* despite double cancels, crashed driver apps, GPS fiction, and network
partitions between rider and driver.

```
        ┌─────────┐  match   ┌─────────┐ accept ┌──────────┐
        │requested│─────────▶│matching │───────▶│ dispatched│──ack──┐
        └────┬────┘          └────┬────┘        └────┬─────┘       │
             │ cancel             │ no-driver/timeout│ decline/    ▼
             ▼                    ▼                  │timeout ┌─────────┐
        ┌─────────┐          ┌──────────┐◀───────────┘       │arriving │──▶ arrived
        │cancelled│          │requested │◀── re-dispatch ────┘         │    │
        └─────────┘          └──────────┘                              ▼
                                                                     │in-progress
                                                     driver-completes│or timeout
                                                                     ▼
                                                                 completed
```

*(Terminal: completed, cancelled (rider/driver/system/tier-specific), expired.
Every arrow is a guarded transition, not a method call.)*

**Server-owned truth, event-sourced**: the trip row in the trip DB is the only state
that counts; everything else (rider app, driver app, in-memory matching state,
geo-index reservation) is a projection that may be stale and must reconcile. Each
transition is an event `(trip_id, seq, from_state, to_state, actor, ts, idempotency_key)`
persisted synchronously before the side effects (pushes, reservation, pricing) — the
standard "persist intent first, act after" ordering from
[Case 03](../03.whatsapp/04-deep-dives.md)'s delivery work, because the same failure
mode (ack lost after accept) is the same class of bug.

**Idempotency and guards**. Every client retry carries an idempotency key; the state
machine returns the already-recorded result instead of re-applying. Transition
guards are the arbiters of conflict: two cancels — the first wins, second returns
success (already cancelled); "arrived" while state is `matching` — rejected with
explanation, client must resync from a snapshot; completion from `arriving` without
`arrived` — flag for review, not silently accepted. State is *monotone forward*
except `dispatched → arriving` re-dispatch loop, which is the one cycle in the
diagram and is bounded by a global trip-timeout (~30–60 s, estimate) after which
the trip is failed honestly rather than spinning.

**Who owns the trip?** One region-local service instance owns the in-flight trip's
in-memory state during its active life; the trip DB holds checkpoints. Ownership is
assigned by trip-id hash to a service shard; if the owner dies, the next instance
rebuilds from the DB checkpoint + event log tail (events are the truth, the owner
is a cache). This is the same "owner leases state, log supplies recovery" shape as
[Case 11](../11.zookeeper/README.md)-style coordination and
[Case 13](../13.kafka/README.md) partition recovery.

**Failure modes, ranked by damage**:

- *Driver crash mid-trip with no completion* — the worst physical-world case. The
  system notices silence (offline hysteresis, deep dive 1), surfaces a support
  path, and marks trip `interrupted`; never silently auto-completes a fare on
  absent data. Money must never be derived from *absence* of events.
- *Rider cancels after driver arrived* — the fee rule is a policy applied at the
  transition; the state machine just records which transition happened, from which
  actor, at what time. The record, not the rule, is the state machine's job.
- *Out-of-order lifecycle events* (partition/retry): guard by `(seq, from_state)`
  — an event not matching expected `from_state` is dropped-and-reconciled to a
  snapshot read, not applied. Dropped events go to a dead-letter queue with the
  trip id, so no conflict is lost.
- *Stuck trips* (no events for N minutes mid-state): a reaper scans
  `in-progress/arriving` trips older than threshold and drives them to a terminal
  state explicitly — a stuck non-terminal trip is a support incident and a billing
  hazard; the reaper converts uncertainty into a recorded decision.
- *Reconciliation with clients* (Case 22's reopen-snapshot): one snapshot call
  returns state + next expected event seq; the client replays what it missed and
  the loop closes. Snapshot is derived on demand from trip DB + event log — no
  separate snapshot store to drift.

**Why event-sourced instead of just update-in-place**: the audited history is a
legal requirement (pricing decisions reconstructible), the re-dispatch cycle makes
naive row mutation hard to reason about, and the log is already in the architecture
for matching events (deep dive 3) — the state machine is the cheapest consumer of
the cheapest infrastructure in the system.
