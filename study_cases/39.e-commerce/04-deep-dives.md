# 04 — Deep dives

## 1. Catalog denormalization & faceted search

The question is where product truth lives and how many copies of it the read path
is allowed to make. 400M SKUs (02 A2), ~104K read QPS hitting them against ~52
order-creates/s — a read:write ratio of ~2,000:1, roughly three orders of
magnitude, so read shape decides storage shape.

**Option A — normalize everything; browse queries join catalog, price, inventory,
promo tables at request time.** Wins: one truth, no staleness class of bugs,
trivial to edit. Loses: every PDP is a 5-table distributed join at ~100K QPS;
faceted search over a join is impossible to serve in <500ms at that rate; the
join fan-out destroys the cache story (fragment caching requires a *composite*
thing to cache).

**Option B — one giant denormalized document store as the source of truth.**
Wins: single read shape. Loses: a seller price edit becomes a rewrite of a
400M-doc corpus; transactional edits against documents are how you get silently
half-updated SKUs; the *order* path still needs relational constraints (cart
lines → SKU ids, exact prices).

**Option C — relational core of record + denormalized read documents as
projections, joined by the event stream.** Wins: writes stay small, correct, and
relational (SKU core, price overlay, ATP, promo rules); reads get exactly the
shape browse needs (one fat "PDP document" per SKU: content + cached price +
availability band + facets) in the KV cache; the search index is just another
projection. Loses: staleness is now guaranteed and must be bounded (minutes,
deep dive 5 of `../06.apple-app-store/04-deep-dives.md` makes the same fork for
the catalog/entitlement split); projection code is real work and rebuild tooling
is mandatory.

**Chosen: C.** The core is relational and small (~5–10TB, 02); every read surface
is a projection over the spine. Denormalization is not a database hack here — it
is the read plane's *definition*: the PDP document exists to make the 99% case a
single cache hit.

**Faceted search on top.** Facets (brand, price band, rating, fulfillment,
category tree) are pre-aggregated at indexing time into per-field posting lists
with result-count metadata, not computed by querying buckets at request time.
Counts can be approximate under filter combinations (the "brand=3,254 results"
number may be a sampled estimate on long-tail queries) — acceptable, because the
number's job is navigation, not bookkeeping. Availability and price are *inside*
the index (per-region bitsets/price bands) so that an out-of-stock result is
never ranked, not merely filtered away post-ranking — post-filtering breaks facet
counts and pagination, the same trap as query-time storefront filtering in the
app-store search dive (06, deep dive 2).

**Failure modes**
- *Projection lag makes price and stock disagree across surfaces* (PDP says one
  price, cart renders another): prices are always re-resolved at checkout from
  the transactional core (F6), so display staleness can only move the cart
  render, never the charge. Alert on `now − latest indexed event` per shard
  (06's freshness SLO shape).
- *Indexer falls behind during a catalog blitz* (holiday merchandising uploads);
  backpressure pushes edit latency to hours rather than dropping events; edits
  are idempotent by (sku_id, edit_seq), so replay is safe.
- *Cache stampede on a viral SKU*: single-flight request coalescing in the cache
  tier + jittered TTLs — the distributed-cache failure discipline
  (`../10.distributed-cache/README.md`).

## 2. Cart: guest carts, login merge, and the session store

The cart is two states wearing one API: an anonymous-session container and an
account-owned container. The hard parts are (a) where it lives, (b) the merge.

**Option A — carts in browser storage / cookies only.** Wins: zero server state,
free scale. Loses: carts silently die with the device (cleared cookies, private
windows, phone upgrades) — unacceptable for logged-in buyers; server-side
checkout validation anyway needs the canonical cart, so "cookie-only" just
relocates the problem to checkout.

**Option B — carts in a relational database, one row per cart.** Wins: durable,
queryable. Loses: cart traffic is ~5K writes/s at the mean, ~15K/s at evening peak, and
~150K/s concentrated at flash bursts (02), mostly on objects that expire worthless — paying relational-writer cost
for abandoned-cart churn; row-count garbage collection becomes its own project.

**Option C — carts in a distributed KV cache store (DynamoDB/Redis-class),
TTL'd by access, with a write-behind event stream for analytics.** Wins:
memory-latency at cart QPS, TTL handles abandonment for free, the store fails
*small* (a lost guest cart costs a re-add; a lost checkout costs money — so
checkout never trusts the cart store alone). Loses: durability semantics are
weaker — acceptable for carts by the same argument that makes the store perfect
for them (compare `../10.distributed-cache/README.md` on ephemeral-tier
placement).

**Chosen: C,** with the account cart additionally mirrored to the order-core
database on login as a durability bump for logged-in shoppers (a lost *logged-in*
cart is a trust failure, not a UX failure). The cart is never on the
transactional plane for reads.

**The guest → account merge.** On login: read the guest cart (by session id) and
the account cart; merge line items by SKU — quantities *add* (two "1 × X"
become "2 × X", capped at per-order limits), differing promo applications
re-evaluate in the account context; unmergeable stale price display is dropped
(price resolves at checkout anyway, F6). Write the merged cart to the account,
tombstone the guest cart, and keep the session-id → account mapping for the
logout fork. Two failure subtleties:

- *Merge must be idempotent*: login retries (double-click, flaky mobile) with the
  same session id must not double quantities. Guard: the merge operation is keyed
  by (session_id, account_id) with a processed-once index — same idempotency
  discipline as checkout (deep dive 4), one layer up the funnel.
- *Concurrent merge vs. active guest writes* (user still has the guest tab open):
  last-write-wins per line item with an update-vector per cart; the lost-write
  window is a few seconds of UX, not money.

**Failure modes**
- *Cart store regional outage*: guests get degraded (in-session cart in memory;
  "your cart was unavailable, try again") — the account cart survives via
  cross-region replica reads; checkout fails closed (N1), because charging on an
  unverifiable cart is worse than blocking.
- *Merge storm from multi-device logins*: cap merge fan-in with per-account
  serialization (a queue keyed by account, executed sequentially) — quantities
  converge instead of racing.
- *Abandoned-cart analytics double counting*: cart events on the spine are
  deduped by (cart_id, item, op, seq).

## 3. Inventory consistency: reservations vs. decrements

One number — sellable quantity — must answer a fuzzy question ("roughly in
stock?") a million times a second and an exact question ("can 1 unit ship to this
order?") thousands of times a second at peak (a few hundred times a second at the
mean — 02's QPS table). The design error to avoid is inventing *two
independent* stock trackers.

**Option A — synchronous decrement at order commit; display stock is the same
row.** Wins: one number, one truth, no reservation machinery. Loses: display
stock now inherits transactional semantics — every PDP badge read is a
transactional read (02's arithmetic kills this instantly); cart abandonment
never returns stock, so "phantom reserved" stock leaks; the hot-row problem is
worst exactly at flash sales (one SKU row, 5K order/s hammering it).

**Option B — decrement at *add to cart*, return on expiry.** Wins: a cart
additions reserve early, checkout is already safe. Loses: reservation TTL churn
is enormous (~5K cart writes/s vs ~250 inventory writes/s — 20× wasted
reservation traffic, 02 QPS table); abandoned carts hold stock that real buyers
want — the oversell-avoidance becomes an undersell generator.

**Option C — reservation-at-checkout with expiry; display stock is a delayed
projection; add-to-cart is unreserveable (or a short, coarse soft flag).**
Wins: reservation traffic tracks *orders* (~250–25K/s), not cart churn; display
stock is served from KV as an eventual projection; the reservation core stays a
tiny hot-shardable in-memory/DB hybrid. Loses: between "customer clicks buy"
and "reservation commits" someone else may win the unit — a race the UX absorbs
by re-checking at submit; and a crashed checkout holds stock until expiry
(seconds-to-minutes), which slightly undersells at peak.

**Chosen: C.** Reservations carry: (sku_id, order_id, qty, ttl ~10 min, state
HELD → COMMITTED | RELEASED). Release happens on checkout failure *or* TTL
sweep, whichever comes first — the sweep is what makes the system crash-safe
without compensating transactions in the happy path. Oversell prevention is a
strict invariant in one place only: the reservation service serializes
`check(qty) ≤ available − held` per SKU, via compare-and-swap / conditional
writes per SKU shard — not via global locks. Hot SKUs shard *logically*: the
inventory pool for a flash SKU can be pre-split into R buckets (e.g. 8 buckets
of N/8) so contending writers hash to buckets and lock convoy collapses to
bucket-level contention.

- **The eventual-consistency trade, made explicit**: display stock may be stale
  by minutes and a few percent wrong on marginal stock (badge said "in stock",
  checkout says "sold out"). We accept that because the cost is a UX apology at
  add-to-cart — the checkout re-validation (F6, snapshot check) is the actual
  gate. What we never accept: oversell. Money-side invariants live on the
  transactional plane, display-side approximations on the read plane; the whole
  architecture (03) exists so these two never mix. Where a *third* class has to
  exist — flash-sale admission (deep dive 6) — it runs *ahead* of both planes so
  neither is stressed by the losing traffic.
- **Where this differs from the money ledger** (06, deep dive 4): purchase
  ledger entries are *irreversibly append-only* and never expire; reservations
  are *reversibly held* and expire by design. Same idempotency discipline, opposite
  durability contract, and confusing the two is the classic bug (a reservation
  that never expires is an undersell bug; a payment that expires is a lawsuit).

**Failure modes**
- *Reservation sweep outage* (expired reservations not released): stock goes
  silently phantom-reserved → undersell during peak. Mitigation: sweep is
  idempotent and sharded with lag-based alerting; a stuck shard is re-scannable
  from the reservation log (events on the spine replay it).
- *Hot-SKU CAS retry storms*: bucketed pools (above) + client-side bounded retry
  with jitter; if buckets are exhausted the SKU flips to "sold out" honestly
  instead of queue-locking the whole inventory shard.
- *Split ATP views between regions* (trans-shipped stock double-broken):
  per-node ATP is authoritative per node; global sellable = Σ nodes as a
  *projection*. Cross-node steals (region A sells region B's last unit) surface
  at shipment time as a substitute/refund path — an accepted, rare,
  operationalized failure rather than a global synchronizer.

## 4. Idempotent checkout & order creation

Checkout is money under a flaky mobile network, with 3 downstream side effects
(reserve, authorize, redeem promo). 1.5× of order volume arrives as retries and
failures (02) — and if the client doesn't send an idempotency key, the gateway
mints one per *request*, not per *intent*, which is exactly wrong (each retry
becomes a new order). We take the app-store pattern wholesale as our template
(`../06.apple-app-store/04-deep-dives.md`, deep dive 4) and generalize it to
multi-line-item commerce.

**Option A — dedup after the fact** (find duplicate orders by fingerprint —
customer, cart, amount, minute-window — and refund). Wins: no client contract.
Loses: detection is probabilistic (same cart, two legit orders, five minutes
apart is routine — subscription + gifts); refunds of accidental duplicates are
customer-hostile; the fingerprint window is a knob nobody can pick safely.

**Option B — client-supplied idempotency key end-to-end, uniqueness at order
creation.** Wins: provable once-only under any retry storm; retries return the
same order id and status instead of a second charge. Loses: the client must
persist the key across app restarts (write-ahead locally before submit — same
mechanic as the app-store receipt-write-ahead); every downstream must propagate
the key (PSP auth keyed by it, reservation keyed by it, redemption keyed by it)
or a retry can still double-spend one of the three.

**Chosen: B.** Mechanics:

- Key = client-generated UUID, minted at *cart submission intent* (not per
  button tap) and written ahead locally. Unique index on the orders table.
- The **order row is written first with state `PENDING`**, before payment and
  before reservation commit — it is the idempotency anchor: every retry sees the
  row and resumes or replays its result. This *de-serializes* failure: a
  timeout mid-checkout leaves a resumable pending order, not a mystery charge.
- **Saga with compensations** (03 flow 4): `reserve → authorize → redeem promo →
  commit → emit`; failure at any step runs release/refund/release in reverse.
  Compensations are keyed by the same idempotency key so the saga itself is
  retry-safe. No cross-service two-phase commit — compensation is cheap here
  precisely because of deep dive 3's expiring reservations.
- **Snapshot validation at submit**: cart prices/promos/availability are
  re-checked from the transactional cores, and the order freezes them (line-item
  price list copied into the order row). Stale cart display prices can lose this
  check ("price changed since you added"), which is a UX negotiation, never a
  silent repricing.
- **Payment authorization (not capture)** at checkout; capture happens on
  fulfillment hand-off. The PSP adapter propagates the same key so PSP-side
  retries dedup there too (06's "the trick applies one layer down").

- **Trade-off made explicit**: a client-side key contract costs app-engineering
  discipline (persist, retry with same key, tolerate replay responses) in
  exchange for an iron once-only guarantee. A server-only scheme (Option A)
  costs ops (duplicate refunds) plus real customer harm. We pay the client
  contract, and we keep the server fingerprint dedup *only* as a monitoring
  control, not a correctness one — it exists to catch contract violations, not
  to enforce correctness.

**Failure modes**
- *Client bug mints new keys on retry*: caught by the monitoring-only fingerprint
  dedup (same cart hash twice within N minutes) → ops signal, not revenue
  damage; the second order is `PENDING`-resumable, not double-charged, because
  authorization and reservation are still keyed.
- *Order row written but saga crashes forever* (poison order): TTL sweep
  releases reservations; payment is only authorized, never captured pre-fulfillment
  (01's N1 "fail closed"), so a stuck saga's worst case is a voided hold — same
  doctrine as 06: "a recorded order without captured money is recoverable;
  captured money without an order is not."
- *Gateway replays a 5xx after the order actually committed*: the retry returns
  the committed result from the idempotency index — the gateway must treat
  idempotent reads as its own replay cache, not as fresh work.

## 5. Promotions & coupons under concurrency

A promo code is a claim on money with an exhaustible budget. Under normal load
that's a lookup; under flash load it's a contention problem with fraud attached.
The failure everyone designs for: 100K people apply `SAVE20` with budget for
1,000 redemptions while the promo service holds the budget in a read-modify-write.

**Option A — check-then-increment on the promo row** (`if redeemed < cap then
redeem++`). Wins: trivial code. Loses: classic TOCTOU — 100 concurrent readers
all see `999/1000` and all increment; the race is invisible at lab load and
exactly reproduces at flash load; oversell of *discount* is real revenue loss
and reconciliation hell.

**Option B — pessimistic lock / serialized queue per promo code** (row lock or
a single-writer actor per code). Wins: trivially correct. Loses: a hot code is a
single lock — 5K redemption attempts/s serialize into a convoy; the failure mode
is the whole checkout path timing out behind one code's lock (a flash sale's
promo *is* the flash sale's hot SKU).

**Option C — atomic conditional decrement** (`UPDATE ... WHERE redeemed < cap`
style single atomic op, CAS/conditional-write per code), with budget *pre-split*
into buckets for doorbuster codes (N buckets × cap/N, same sharding as hot
inventory in deep dive 3). Wins: correctness without holding any lock across a
round trip; buckets absorb the convoy. Loses: bucket re-balancing when one code
outlives its expected distribution (merge or steal from fat buckets is a
background repair, and while repair runs, the code may "sold out" early in thin
buckets); over-provisioning per bucket wastes budget at the tail.

**Chosen: C.** Redemption state per code: cap, redeemed, per-account and
per-window hashes (rate-limit dimensions), TTL. Redemption is one conditional
atomic operation, executed *inside* the checkout saga as a keyed participant
(deep dive 4) so a failed order auto-releases its redemption. Coupon fraud
dimensions (one code, million accounts; one account, thousand codes) are capped
*before* the atomic budget decrement with coarse per-account/per-code-window
counters that may be slightly lossy — lossy rate-limiting is fine (it is a
security heuristic), budget decrement is not (it is money), and the distinction
is the same read-plane/transactional-plane split as everywhere in this design.

- **Trade-off made explicit**: bucketed budgets win a lock-free hot path and
  lose exact global budget conservation during rebalancing — a code may reject a
  good customer at 99.8% of its true budget spent. We take that because the
  alternative (Option B) converts a promo hot spot into a checkout outage,
  which is a worse failure by orders of magnitude; and budget slop is an
  acceptable cost the merchant chooses when they set the cap (a 1% over/under
  margin on a marketing budget, vs. an unbounded convoy).
- **Promo pricing vs catalog price is a layering rule**: promos are an *order-time
  reduction*, never a catalog rewrite (F12) — so a promo change can never race a
  catalog projection into showing the wrong price badge, and promo expiry never
  requires catalog invalidation.

**Failure modes**
- *Redemption succeeded, order failed, refund of redemption lost*: redemptions
  are saga participants keyed by the order's idempotency key; a TTL sweep for
  orphaned redemptions (redeemed, no committed order after X min) closes the
  window — reservation sweep and redemption sweep are the same job, different
  table.
- *Budget bucket skew makes a hot code die early*: bucket merge repair runs
  async; customer-facing symptom is an early-but-honest "code exhausted" and an
  alert, not a phantom "applied then revoked at payment" (the UX crime of
  re-checking promo eligibility at capture).
- *Cascading promo abuse at checkout* (bot farms): rate-limit and risk check
  ahead of the budget decrement; a flagged redemption is async-reviewed after
  order creation while the order path stays fast — blocking money on a fraud
  verdict synchronously couples two systems' failure modes.

## 6. Flash sales & surge shaping: admission before capacity

A flash sale is not "Black Friday per-SKU" — it is *announced in advance*,
concentrated on known SKUs, with a loser:winner ratio near 20:1 (02). Capacity
sized to admit everyone is an annual fleet for one hour; the correct tool is
admission control ordered from cheapest rejection to most expensive.

The shaping ladder, applied top-down:

```
L0  CDN edge: media + page shells for the sale pages are pre-warmed before
    start time; losers cost the edge bytes, not origin QPS (09.cdn pattern).
L1  Static/edge rendering of the sale PDP: stock badge moves to async client
    fetch; the page itself is a cache object with a kill switch.
L2  Waiting room / queue for order admission on flagged SKUs only: clients are
    issued signed queue tokens (deterministically hashed order-of-arrival by
    consistent hash, no centralized counter in the hot path); admission rate =
    inventory drain rate. Everyone else sees honest position + "do not refresh".
L3  Per-SKU token budget at the API gateway: N admission tokens/sec for
    checkout on the flagged SKU = the reservation core's budget; requests
    beyond it get 429+retry-after, which upstream queueing absorbs.
L4  Read-plane degradation ladder (whole site): recommendations off → facet
    counts sampled → search falls back to category browse → read-only
    "browsing is fine, checkout paused" — each rung is pre-enacted config,
    never improvised.
```

**Option A — overprovision the transactional plane for flash peaks.** Wins: no
queue UX, no tokens, every customer races honestly. Loses: 100× multiplier on
order-path capacity buys an experience most customers can't tell from a queue —
they still lose (5K units, ~100K buyers) — while the fleet sits idle 364 days; and
over-provisioning the *order path* doesn't fix the *read* path anyway (PDP views
of the losing crowd still storm the site).

**Option B — queue/waiting room with honest admission control (the ladder).**
Wins: bounded, predictable load on the transactional core; the losing traffic is
served cheaply and *honestly* (a position is a position); flash inventory drains
in an orderly drain rate set by ops, and the queue doubles as the surge absorber
for unflashy spikes (BFCM). Loses: queue latency is visible UX and becomes a
product surface ("queue jumping" bugs, refresh storms, token spoofing are
attack surface); and a queue that is *too honest* can throttle actual conversion
(the trade-off is real revenue shape, not just tech).

**Chosen: B, restricted to flagged SKUs/timeboxes** — the waiting room is an
opt-in mechanism at the gateway, engaged by promo configuration, not a global
design. Key mechanics: signed admission tickets (forging one is the classic
flash-sale exploit; tickets are HMAC'd with a short TTL and server-side
replay-checked by the idempotency key discipline of deep dive 4); queue state is
distributed by consistent-hash sharding so no single queue coordinator exists to
die at peak; admission drain rate tracks the inventory bucket drain rate from
deep dive 3, so the queue can never admit more buyers than the buckets can
serialize.

- **Trade-off made explicit**: the queue trades *perceived latency* (minutes in
  line) for *predictable success* (admitted buyers actually convert — no 5xx at
  pay) and for a fleet that exists year-round at baseline size. We take it
  because overprovisioning buys the same outcome (5K winners) at 100× the cost,
  and because honest visible waiting measurably reduces refresh storms — the
  client behavior that turns a spike into a pile-up.
- **Relation to the app-store phased-rollout trick**: 06 shapes a *delivery*
  burst by admitting 1%→2%→5% of devices into a download; we shape an *order*
  burst by admitting buyers at inventory-drain rate. Same lever — cap the
  concurrency of the expensive path — applied where this domain's expensive path
  is money, not bytes.

**Failure modes**
- *Token forgery / queue bypass*: short-TTL HMAC tickets bound to
  (account/session, sku, window); replay attempts are visible as
  duplicate-ticket rejections and trip rate limits at the gateway — the abuse
  shape is the same as the Twitter abuse dive's "limits before the expensive
  path" (`../02.twitter/04-deep-dives.md`, dive 6; the app-store case applies the
  same discipline, `../06.apple-app-store/04-deep-dives.md`).
- *Queue coordinator falls over at admission time*: no single coordinator
  exists (consistent-hash shard state); a lost shard's queue position resets
  rather than corrupting — admitting a burst of unsorted buyers degrades to
  Option A's race for that shard, caught by per-shard drain-rate alarms.
- *Inventory buckets drain but queue keeps admitting* (mismatched drain rate):
  the sale pages flip to "sold out" honestly via the kill switch, and
  L1's async badge is authoritative for the read plane; the checkout plane is
  never asked about stock it doesn't have.
- *Whole-site BFCM overload without a flash SKU*: the degradation ladder (L4) is
  pre-enacted by config with metric tripwires per rung — degradation is
  *scheduled* like a deploy, rehearsed before the season, so a rung is an
  operational reflex, not a novel incident decision.
