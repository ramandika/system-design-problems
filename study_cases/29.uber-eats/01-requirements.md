# 01 — Requirements

## Product in one sentence

A customer orders food from a restaurant; the platform routes the order to the restaurant,
dispatches a courier, coordinates both sides' timing, tracks the order live, and settles
payment between the three parties.

## Functional requirements

### Customer side

- R1. Browse/search restaurants by location, cuisine, rating, delivery fee, ETA; filter and
  rank menus.
- R2. View a restaurant menu with item sizes, modifiers (toppings, spice level, notes),
  prices, photos, and stock-outs ("sold out today").
- R3. Build a cart with modifiers, see a complete price (items + fees + tax + tip) *before*
  checkout, and check out with a saved payment method.
- R4. Place an order, see the order accepted by the restaurant, and receive a **committed
  delivery ETA** at checkout that is updated live through the order's life.
- R5. Track the order live: restaurant accepted → cooking → courier assigned → picked up →
  on the way → delivered, with the courier's position on a map (privacy-bounded: see 04.3).
- R6. Contact the courier or support chat during an order; cancel within a policy window.
- R7. Rate and tip after delivery; reorders ("last order, one tap").

### Restaurant (merchant) side

- R8. Publish and edit catalog: menus, sections, items, modifier groups with
  constraints/exclusivity, sizes, prices, hours, item stock-outs, availability windows.
  Edits must be online and take effect fast (see 04.4 for what "fast" costs).
- R9. Receive incoming orders on an in-store surface (tablet app and/or POS integration),
  **accept or reject** within a configurable window (auto-accept exists as a merchant
  setting), see a prep queue with a projected "ready by" time.
- R10. Mark order progress: accepted → preparing → **ready for pickup** → (handoff is the
  courier's action).
- R11. See courier identity and arrival countdown, and mark handoff done if a paper/courier
  flow requires it.
- R12. See earnings statements, refunds, and dispute tools.

### Courier side

- R13. Set availability (online/offline, area preference), receive an offered job with
  **gross fare, pickup ETA and delivery ETA before accepting**, and accept/decline within a
  short window (~15-60s, estimate).
- R14. Navigate to pickup and dropoff, mark arrived, mark picked up, mark delivered; scan
  or confirm order code at dropoff (order-verification pattern).
- R15. Batch mode: accept a job that may later be paired with a second pickup/dropoff
  (two orders, one courier) — see 04.2.
- R16. See earnings per job and daily totals in near-real-time.

### Platform side

- R17. Dispatch: maintain live courier supply, match incoming orders to couriers, and
  re-match when an assignment falls through (courier declines, no-shows, breaks down).
- R18. ETA: predict food-prep time per restaurant and travel time per leg; commit a delivery
  ETA to the customer at checkout and keep it honest through the order.
- R19. Payments: charge the customer, split between merchant, courier, and platform fees,
  settle post-delivery with refunds/promo adjustments — as a ledger of transitions, not
  in-place balance updates (pattern from [06.apple-app-store](../06.apple-app-store/README.md)).
- R20. Trust & safety: order-code verification at dropoff, courier identity checks, fraud
  screens on fake orders/merchants, support tooling for refunds and re-delivery.
- R21. Merchant analytics: prep-time reports, acceptance rates, menu funnel (views → carts
  → orders).

## Non-functional requirements

| Dimension | Target (estimates, justified in [02](02-scale-estimates.md)) |
|---|---|
| Catalog read latency | p99 < 300ms served at the edge, menu page CDN-cached |
| Checkout → order placed | p99 < 1s synchronous write |
| Dispatch decision latency | first offer to a courier < 10s from order acceptance; re-dispatch rounds every ~2-5s while unassigned |
| Tracking update freshness | consumer sees courier position ≤ 5-10s stale |
| ETA accuracy | stated as a target band (e.g. ±5min) rather than a point guarantee; missed-ETA rate is a top-line product metric |
| Availability | browse degrades gracefully (serve stale menus) when the order path is down; order path is the critical tier |
| Consistency | order state is strongly consistent per order; catalog is eventually consistent (minutes); tracking is firehose-latest-wins |
| Durability | every state transition is an immutable event; balances/settlement derived |

## Scope decisions

### In scope

- Three-party order lifecycle + live tracking + dispatch + catalog + settlement, the full
  loop (R1-R21).
- Batching couriers across two concurrent orders (R15) and its consequences.
- Kitchen prep-time prediction as an input to dispatch and ETA (R18).
- Multi-platform clients: consumer apps, courier app, restaurant tablet + POS APIs.

### Out of scope (explicitly)

- **Demand shaping/pricing (surge, fee experiments) as a system**: we treat demand as a
  given input. Pricing interacts with dispatch but is a separate product system; we note
  the interface and stop (cross-ref [28.uber](../28.uber/01-requirements.md), where the
  same line is drawn).
- **Route optimization over a full multi-stop courier plan** (3+ orders, day-route
  planning): we cap at one-to-one and two-order batching; true multi-stop delivery-route
  optimization (the "courier becomes a TSP vehicle") is a different engine.
- **Restaurant inventory management** beyond menu-stock toggles: no per-item ingredient
  counts, no supplier systems.
- **Native search relevance/ML stack**: treated as an input ("a ranked list of
  restaurants"), same boundary as [26.google-maps](../26.google-maps/README.md) treats its
  ETA model internals.
- **Dine-in pickup orders & subscriptions/loyalty features**: they reuse the same order
  state machine minus courier stages; noted once, not designed.
- **Internationalization of catalog data, tax law engines per market**: merchant-side
  concerns with country-specific rules that don't change the architecture.

### The one scope decision that shapes everything else

**The order is the consistency unit; the catalog and the market are not.** A menu can be
stale for minutes; two services can disagree about courier densities for seconds; but there
must never be two live states of one order. Every component below is arranged around that:
the order service owns the state machine, everything else subscribes to it. This is also
what makes the crash-recovery question in the README tractable — there is one authority to
recover from, instead of reconciling three.
