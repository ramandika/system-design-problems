# 04 — Deep dives

## 1. Spatial search over 8M listings: geo indexing + filters + ranking, on one query

The query that defines this system: *a map viewport (often after panning), a date range,
guest count, and a stack of filters — return ~200 ranked listings, in 300 ms.* The corpus
is ~80GB and RAM-resident ([02](02-scale-estimates.md)), so this is not a distributed-
storage problem; it is a query-shape problem. The geo machinery is the same family as
POI search in [26.google-maps](../26.google-maps/04-deep-dives.md) (dive 2), bent to a
different shape: Maps ranks by distance and detail quality with no per-item *ranges*;
we must intersect **which cells** with **date-range availability** with **price/amenity
postings** and then **rank**.

Design decisions inside this:

- **Geohash/H3-cell posting lists inside the search engine, not a separate geo database**:
  each listing gets posted to the identifiers of the cells that cover its point at several
  precisions. A viewport query expands to a *cell set* (children of coarse cells covering
  dense regions, parents over sparse ones) and takes the union of postings, then
  point-in-polygon verifies results against the true viewport shape. The trade-off —
  cell granularity vs posting-list length vs how much gets post-filtered — is exactly the
  cell-match/false-positive dial of geospatial engines: precise cells make short posting
  lists but query-time expansion explodes near boundaries; coarse cells make one posting
  list serve everything but return junk to verify. We pick mid-precision cells (H3-like,
  few-hundred-meter cells) with the standard *parent-inclusion trick* — include the parent
  cells only around viewport edges — because post-verification of a few false candidates
  is far cheaper than expanding 50 children for sparse areas.
- **Date-range availability as per-listing bitmaps evaluated in the scorer, not as
  postings**: a "day D available" posting list intersected across a 7-night stay means
  intersecting 7 giant lists per query — a stay over a popular holiday month drags nearly
  every listing through 7 postings. Instead, each indexed listing carries a per-month
  availability bitmap (~31 bits) plus a *blocked-days-in-range* count column, so the
  filter is "count(booked∩range) == 0" evaluated per candidate — O(nights) bit ops on an
  in-memory doc value. Candidates arrive from the geo postings already (~thousands), so
  the filter is cheap; the details and the staleness problem are dive 2.
- **Two-stage ranking**: cheap candidate retrieval (geo + date + hard filters) leaves ~1–10K
  docs; a first pass by a precomputed quality score (review aggregate, response rate,
  recency, price competitiveness relative to the local market) cuts to ~500; a
  learning-to-rank model scores those, blending expected-bookability. This is the standard
  retrieval→LTR funnel of [15.ml-ranking-system](../15.ml-ranking-system/README.md). The
  marketplace twist ranking must solve: **relevance is dates-dependent and
  conversion-dependent** — the "best" listing is one that gets booked *this guest, these
  dates*, and over-ranking an unavailable-but-gorgeous listing wastes the other 199 slots.
  Hence availability is not just a filter tail: near-miss listings (available one night
  earlier) can be surfaced deliberately, which moves the availability filter from
  hard-kill to a score feature for a *secondary* "matches nearby dates" section.
- **Cache keys for map-move queries**: the panning pattern means near-identical queries at
  jittered viewports — the [09.cdn](../09.cdn/04-deep-dives.md) cache-key economics apply
  inverted (we are the origin). Viewport is normalized to a cell-tree closure before key
  computation, and tile-bucketed top-K results are cached per (cell set, date range,
  filter hash) with short TTL — price calendars move slowly, so the cache is safe at
  minutes. The trade: normalized keys raise hit rate but return slightly wrong tiles
  for panning users (a tile edge shows a listing that isn't quite in view) — visible and
  bounded, and cheaper than answering ~2K distinct panning shapes per second.

**Failure modes**: a listing's calendar snapshot in the index silently stops updating
(fix: index updater tracks per-listing last-published epoch and an auditor re-drives any
listing whose snapshot age exceeds an SLA — the same "staleness must be observable" rule
as the partner fanout in [27.ev-charging-network](../27.ev-charging-network/04-deep-dives.md));
hot metro cells (Lisbon in July) make one posting list the hottest object — same
split/jitter playbook as the hot-object dive in [09.cdn](../09.cdn/04-deep-dives.md),
applied at the engine level (replica-fan by shard-per-city, since shards fit in RAM).

## 2. Availability calendar integrity: a grid, a snapshot, and an honest staleness split

The inventory model is not "stock count" — it's a **per-listing grid of day states**:
open, blocked (host), on-hold, booked. Derived facts hang off it (price per night,
minimum stay rules). Three questions matter: how is truth stored, how does search see it
without waiting, and what happens when they disagree.

Design decisions inside this:

- **Truth lives in one place per listing**: the calendar service owns the day grid, and
  every mutation (hold, booking, unblock) commits there inside the same ordering authority
  as the booking service (dive 3 — they share the per-listing writer). Options for the
  write model: Option A, a generic relational `availability` table with row-per-day and
  conditional writes; Option B, an append event log per listing with the grid as a
  materialized projection. A wins a decade of tooling (conditional `UPDATE ... WHERE
  state='open'`, foreign keys to bookings, no projection machinery) and loses only the
  audit narrative — which we recover because bookings/holds themselves are append-only
  ledger rows and the grid is reconstructible from them. We pick A: the audit property
  matters, but it is carried by the booking table, not needed inside the grid.
- **Search sees snapshots, never truth**: a change event (booking confirmed, hold
  committed, host unblocked dates) flows onto the event bus; the search index updater
  applies *delta* updates to the affected day bitmaps and to the listing's snapshot epoch.
  Full listing re-indices happen on metadata edits. The design question is push-delta vs
  periodic rebuild: Option A, delta push on event — wins freshness (seconds) and low
  steady-state cost, loses the guarantee shape (a lost/duplicated event silently corrupts
  a bitmap, and bitmaps are self-healing only via full rebuild); Option B, periodic full
  snapshot rebuild — wins eventual self-correction, loses minutes of staleness per change
  and burns a rebuild for ~8M listings to fix one booking. We pick A with B as a
  reconciliation sweep: deltas for freshness, a nightly (and on-demand per-listing)
  rebuild-as-audit that recomputes bitmaps from the booking database and diffs — drift
  found is an incident, not a repair mechanism (N2's staleness bound is enforced by the
  sweeper's age alarms, mirroring the reconciliation discipline in
  [06.apple-app-store](../06.apple-app-store/04-deep-dives.md)).
- **Holds show in the grid immediately**: a guest's 10-minute hold turns nights to
  `on-hold`, and the snapshot carries it — search drops the listing for those dates within
  seconds, because a held night that still searches as open is just a double-booking we
  discover later (the same "a published available that is actually a paid hold is a
  double-booking with a refund" observation as
  [27.ev-charging-network](../27.ev-charging-network/04-deep-dives.md), there at
  connector level, here at date level).
- **Min/max-nights and gap rules are evaluated at booking time, search only approximates**:
  rules like "3-night minimum over weekends" or a 1-night gap after checkout cannot be
  cheaply pre-filtered for arbitrary requested ranges; search applies a simplified
  approximation, booking applies exact rules. Fail-closed direction: search may *fail to
  show* a bookable listing (approximation rejects a real match), never *show* one the
  rules reject — a false negative wastes a search impression, a false positive wastes a
  whole booking flow.

**Failure modes**: event-bus backlog after an incident → bitmaps go stale at scale, search
shows half a city's booked-out dates as open — fix: updater publishes *lag* as a first-class
metric and search degrades to "don't trust date filters beyond X seconds of lag" behavior
(cached results with the caveat UI) rather than confidently-wrong results; host bulk-blocks
3 months and the delta updates collapse into a per-listing storm — fix: batch host edits
into one event per listing per second; grid reconstruct vs live write divergence during
reconciliation rebuild (fix: rebuild computes on the side and the per-listing writer swaps
it in atomically under its write turn — a rebuild never mutates the live grid mid-flight).

## 3. Booking reservation: hold → confirm, and the one-writer-per-listing rule

The absolute requirement (N3) is that no two confirmations or holds may overlap on the
same nights. The shape is a two-phase transaction with an external system (the PSP) on
the critical path, which is the hard part.

Design decisions inside this:

- **One ordering authority per listing**: booking requests for a listing partition to a
  single writer (by listing_id hash), and the hold critical section is:
  (1) re-check the grid *inside* the same writer that will commit; (2) mark nights
  `on-hold` with a hold record carrying TTL; (3) PSP authorize; (4) commit (instant book
  → confirmed; request-to-book → pending-host, payments authorized but not captured).
  Options for cross-instance correctness: Option A, hash-partition with one writer
  process per listing — wins trivially serialized nights, loses that a partition instance
  is a failover concern; Option B, database conditional transactions (optimistic row
  checks) from any instance — wins stateless services and easy failover, loses holding a
  PSP call across a row lock (lock hold time balloons to the PSP round trip, or you
  re-check after the PSP and open a race). We pick A for night-granting, with B as the
  inner mechanism (the conditional `state='open'` update is still the last check inside
  the writer) — belt and braces, because the failure mode of getting this wrong is paying
  real money twice.
- **The hold buys time for the money, not convenience**: TTL ~10–15 minutes for
  request-to-book authorization-holds, seconds-level for instant book (essentially one
  transaction). An expired hold releases nights via a TTL sweeper and reverses the
  authorization — the reservation is a timer-bearing state, exactly the pattern from
  [27.ev-charging-network](../27.ev-charging-network/04-deep-dives.md) (charging
  reservations there), the only difference being that our money is already authorized
  during the hold rather than at start.
- **The saga for the two failure worlds** (N4): payment captured and confirm failed
  (calendar committed, ledger intent never finalized) → compensating full refund via
  the original charge id; payment authorized and hold commit failed (nights released,
  authorization orphan) → explicit void of the authorization; and the worst world —
  PSP timeout with unknown outcome — treated as *in-flight*, resolved by reconciliation
  of PSP statements against ledger intents (idempotency keys per intent; intents are the
  truth, PSP responses are facts). The discipline is the exactly-once-money machinery of
  [06.apple-app-store](../06.apple-app-store/04-deep-dives.md); what's marketplace-specific
  is that a "stuck" intent means a guest thinks they have a room tonight, so stuck-intent
  alarms page a human rather than wait for the nightly job.
- **Instant book vs request-to-book is a correctness-equal, latency-unaffecting mode
  flag**: both run the identical hold critical section; request-to-book differs only in
  that commitment waits for a host event with a timeout (auto-decline), during which the
  nights are `on-hold` — so a host cannot decline because the nights got booked from
  under her, and a guest cannot hold forever. Trade-off: holding nights for up to 24h on
  an uncaptured authorization (PSPs expire auths in days, so the window is fine) burns
  availability during the request window — accepted, because the alternative (release
  nights, re-grant on accept) reintroduces the double-book race the hold exists to kill.
- **Guest cancellation and host cancellation are both ledger-first**: cancel walks the
  reverse path (release grid nights, policy-computed refund entries, escrow adjustments)
  — never "delete the booking row".

**Failure modes**: the writer's failover during a hold (fix: hold records are durable
before any PSP call, so recovery scans in-flight holds and either finishes or reverses
each — a hold is never lost, only ever resolved); double-authorization on a retried
"authorize" (fix: same idempotency key per booking intent — a retried call returns the
first authorization); the reconciliation job finds a PSP-funded authorization whose
intent is missing (money exists, no row) — fix: auto-create intent from PSP data, refund
by default, because orphaned guest money is a support ticket while orphaned nights are
an incident.

## 4. Payments: escrow, deposits, refunds, and the three currencies

Guest pays at booking in *guest currency*; platform holds it as escrow; host is paid
after check-in in *host currency*; the listing is priced in *listing currency*. Three
currencies and a hold period in the middle of every booking — this is the marketplace
money machine.

Design decisions inside this:

- **Ledger-first with escrow states**: every booking spawns ledger entries: charge
  (guest → platform escrow), fee accrual (platform revenue), payout (escrow → host),
  refunds as new debit/credit entries **offsetting**, never mutating, the originals —
  the same credit-not-delete doctrine as Credit CDRs in
  [27.ev-charging-network](../27.ev-charging-network/04-deep-dives.md), and the
  ledger-is-truth-over-PSP stance from [06.apple-app-store](../06.apple-app-store/README.md).
  Escrow as a *liability account* means the money state machine (funds held → released →
  partially refunded) is queryable per booking without asking a PSP. Option A, PSP as the
  record; Option B, ledger as the record — B wins auditability, refunds-on-anything
  (chargebacks, damage claims against already-payout funds), and multi-PSP portability;
  it loses the double-entry plumbing you must build yourself. Not a close call at $81B/yr
  of booked value sitting in a hold state ([02](02-scale-estimates.md)).
- **Payout timing is the fraud control and the reconciliation anchor**: escrow releases
  ~24h after check-in (policy-configurable), batched to hosts on a weekly cadence by
  default. Why the delay: it makes every fraud scheme in dive 6 structurally unprofitable
  because the platform still holds the money when the scheme is detected; it also makes
  chargeback handling possible *before* host funds left. Trade-off: it holds back hosts'
  cash flow (the real product pressure to release earlier); the counter is that early
  payout converts fraud defense from architecture into collections.
- **Currency handling — presentment vs settlement**: guests see and are charged in guest
  currency with the FX rate *quoted and locked at booking time*; the charge executes at
  that rate (or the PSP executes and we reconcile to the quoted rate, absorbing small
  drift as revenue/loss entries rather than repricing the guest). Hosts are paid in payout
  currency at payout-time FX. The precision rules: all ledger amounts in **minor units**
  (cents) of the *original* presentment currency; refunds reference the original charge
  and refund in that currency even if the payout already converted — an FX gain/loss
  entry absorbs the difference. Trade-off: Option A, refund by reversing the original
  transaction (PSP reversal) — wins "guest never sees two charges", loses when a payout
  happened after the original charge (reversal of a captured-and-paid-out charge is
  impossible); Option B, refund as a new transaction against the original — works after
  payouts but looks like a second charge. We pick A up to payout, B after — the state
  machine switches at the escrow release boundary, and *that* boundary is why payout
  timing belongs in the payments design, not just operations.
- **Security deposits and damage claims** are separate ledger accounts with their own
  state machines (authorized-not-captured at booking, released days after checkout,
  claimed with evidence) — not amounts inside the main escrow, because their refund
  rules, timers, and dispute paths differ entirely.
- **Partial captures / split schedules** (pay-now vs pay-over-time, long stays): the
  charge schedule is a first-class ledger plan (dated intents), not ad-hoc calls, so
  "capture second installment" is a scheduled intent executed with idempotency keys —
  a missed capture is a reconciliation finding, not lost revenue.

**Failure modes**: FX drift between quote and PSP execution on high-volume days (fix:
rate locks at quote + drift-absorbing rounding entries, and alerts when drift exceeds a
threshold — drift is a *signal* of PSP misconfiguration, not just cost); multi-PSP
regional failover mid-booking (fix: PSP choice per intent, recorded, so all lifecycle
operations target the PSP that holds the funds — you cannot refund money at a PSP that
never had it); chargebacks arriving after payout (fix: the escrow delay above, plus a
host-deductible balance, since some liabilities will still land late); double-release of
escrow on a retried payout (fix: payout intents keyed idempotently against the ledger,
PSP payout calls carry that key).

## 5. Reviews and trust signals: double-blind by design, integrity by graph

Reviews are simultaneously a *product feature* (guests read them, hosts answer them) and
the platform's highest-signal ranking/trust input (dive 1 consumes them). The design
problem is that the two sides want opposite shapes: guests want *current, verified*
reviews; ranking wants *stable aggregates*; integrity wants *detectable manipulation*.

Design decisions inside this:

- **Double-blind with a fixed reveal window, both-or-expire**: after checkout, both
  parties get N days to submit; neither sees the other's review until both submitted or
  the window closed. The reason is the reciprocity problem: hosts control guests'
  bookings; visible reviews make guests rate to avoid retaliation. Option A, immediate
  publish — wins freshness, loses honesty (publicly reported pattern in every two-sided
  marketplace: reviews skew positive and retaliatory); Option B, forced double-blind with
  auto-publish on window expiry — wins the honesty property that makes the aggregate
  trustworthy enough to rank on, loses the "review while it's fresh" psychological effect
  (submissions happen later, and late reviews are worse reviews). We pick B: the
  aggregate is load-bearing downstream, so input quality is worth the freshness cost, and
  expiring-without-submit is itself a signal (a host who never reviews is a data point).
- **Trust is a tiered, multi-signal model, not a star rating**: superhost is computed
  from a *vector* (rating, response rate, cancellation rate, compliance) — because a
  single mean is both gameable and lossy. Identity/KYC tiers gate *what actions* an
  account can take (book without verification, instant book as host, high-AOV booking),
  which is where trust connects to dive 6's fraud economics.
- **Aggregates are streams, not queries**: review events are append-only; aggregates
  (count, average, time-decayed average, per-category scores) are maintained by the event
  stream and pushed into the search index like any other listing field. Recomputation
  from source must always be possible and is reconciled periodically — the same
  materialized-view-with-audit shape as the calendar in dive 2. A review removal (fraud
  takedown) is a decrementing event, not an aggregate edit.
- **Integrity detection is graph-shaped, not review-shaped**: fake review rings (hosts
  leaving each other 5-stars through fake stays, or paid guests) look individually normal
  and collectively dense — detection needs the review graph (who reviews whom, account
  age, device overlap, booking AOV vs review patterns), not per-review classifiers.
  Review gating on *verified stays only* is the structural precondition for all of this:
  a review must reference a completed booked stay, so the reviewer set is bounded by
  people the platform already has money signals on.
- **Moderation stance**: post-moderation (publish, then remove on detection) for the
  bulk of content; pre-moderation only for accounts already flagged. Option A,
  pre-mod everything — wins zero-visible-fraud, loses review velocity (the whole
  marketplace rhythm) and burns human moderation on a 90M-review/year volume
  ([02](02-scale-estimates.md)); we pick post-mod with verified-stay gating because the
  damage window of a fake review is small (its aggregate weight is one out of hundreds)
  while the cost of pre-mod is permanent friction.

**Failure modes**: the trust vector's feedback loop — ranking favors superhost traits,
hosts optimize the traits not the hospitality, the signal decays (the same model
feedback-loop problem as [14.ml-production-system](../14.ml-production-system/README.md),
fix: keep the objective = bookings and satisfaction, re-evaluate feature drift);
double-blind window expiry auto-publishing when one side genuinely never saw the trip
(no-shows) — fix: publish the traveler's side even if host never does, and treat missing
host review as neutral, never negative-slash-vengeful defaults; review-bombing after
policy incidents (fix: time-bounded rate limits on review submission per listing and
sudden-shift detection on aggregates).

## 6. Marketplace fraud: fake listings, booking scams, and the payout architecture

The fraud threat model, concretely: (a) **fake listings** — scraped photos and text, no
real home, priced attractively; (b) **booking scams** — guest pays on-platform then
pressured off-platform, or host lures a check-in that doesn't exist; (c) **payment
fraud** — stolen cards booking *real* (or fake) listings; (d) the combination (c)+(a),
the most dangerous one: **stolen cards + fake listings + payout = a cash-out machine**
that launders card fraud into host payouts. The design answer is one architectural
stance and a set of detectors around it.

Design decisions inside this:

- **The escrow release delay is the load-bearing defense** (dive 4): because payouts
  release ~24h after check-in and money is still ours when almost every signal has
  arrived (chargeback windows, bank disputes, support reports, check-in reality —
  a guest physically failed to enter a nonexistent home), the fraud economics invert:
  the scammer books on a stolen card, but never touches funds before the platform can
  intercept the payout. Option A, instant payouts (a real product feature demand);
  Option B, delayed escrow release with risk gates — A wins host satisfaction, loses the
  ability to stop any of (a)–(d) after the fact; we pick B, because an instant-payout
  marketplace is a fraud collection problem, and collections are where you pay anyway.
- **Fake listings are caught by image and text identity, not just review**: photos are
  content-addressed in blob storage and served by CDN (dive 1), so a perceptual-hash /
  reverse-image index over all listing photos makes *photo reuse across listings*
  computable at listing creation — the single highest-precision fake-listing signal
  (scraped properties reuse real photos, almost always). Around it: new-listing velocity
  rules per host and per device, price-vs-market anomalies ("$40/night downtown" that
  the pricing model itself flags as a 2-sigma outlier), payment-instrument mismatch
  (payout account country/name vs listing location), and a no-booking-history cold-start
  review period for new listings. Trade-off: Option A, block suspicious listings until
  human review — wins fewer guests burned, loses host onboarding speed and risks
  suppression of legitimate supply in supply-hungry markets; Option B, publish-then-
  takedown — wins supply growth, loses guests who book fakes in the window. We pick a
  hybrid, deliberately asymmetric: new listings with *any* high-risk signal are held;
  clean new listings publish immediately; takedown is also possible post-publish since
  escrow delay caps the guest loss (they paid but check-in is where the scheme fails,
  and money hasn't left).
- **Booking-scam detection rides the messaging service** (F8): the classic scripts (pay
  me directly, my card doesn't work, wire transfer, external payment link, "send your
  ID details here") are pattern-matched on message content and metadata, with outcomes
  from warning banners (both sides), to blocking the message, to account
  suspension. This is also where *host-side* scams surface (host cancels after booking
  and re-offers off-platform) — so the signal consumer is the same for both market sides.
- **Payment fraud detection is velocity+network, run *before* capture**: device trust,
  payment instrument history, booking-pattern anomalies (last-minute bookings of the
  most expensive unit in fake-prone markets is the classic stolen-card signature) —
  decisions land as *risk scores on the booking intent* (dive 3), and gates on the
  *payout* leg (N8: risk can delay a payout but never accelerates one). Note the division
  of labor with ML infra: models and feature pipelines per
  [14.ml-production-system](../14.ml-production-system/README.md), outcomes and accountability owned by trust & risk, and the money architecture (this dive) is what makes
  model errors survivable.
- **KYC placement by economic exposure, not by ideology**: guests can book low-AOV
  instant-book listings with minimal verification; high-AOV bookings, request-to-book
  host payouts, and payout-account changes trigger stepped verification. Payout-account
  changes deserve special attention — the account-change-then-payout is the standard
  account-takeover cash-out — fix: change-then-payout gets a hold window of its own.

**Failure modes**: false-positive host suppression during a supply crunch (fix: per-market
threshold tuning and an appeals path that is *actually staffed* — a wrongly-held payout
or blocked listing is a churn event in a marketplace where both sides are optional);
fraud rings adapting to image hashing (fix: multi-signal detection — photos alone never
decide, because sophisticated scams use freshly-shot photos of a real but non-rented
property); risk models gone stale as attack patterns shift (fix: the payout-delay
architecture is the model-independent backstop — when the model misses, escrow is still
ours); support/ops becoming the real system at scale (fix: case-management and evidence
flows built into dispute handling from day one, because at ~140M stays/year the
manual-review population is a design input, not an afterthought).
