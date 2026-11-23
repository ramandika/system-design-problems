# 01 — Requirements

## Functional requirements

**Catalog & search**
- F1: Product detail pages (PDP) and category browse pages per SKU, with per-region
  price, availability, and localized content. Media (images/video) served from
  object storage behind CDN edges.
- F2: Faceted search: free-text query + facet filters (category, brand, price band,
  rating, fulfillment option) with per-facet result *counts*, and ranked results
  blending relevance with sales/margin/quality signals.
- F3: A seller/merchandising write path: publish/edit SKUs, price changes, promo
  pricing, availability flips — appearing site-wide within minutes; plus a
  synchronous kill path for takedowns (legal/recall) — within seconds.

**Cart**
- F4: Guest carts keyed by an anonymous session id (cookie or mobile-issued
  pseudo-account), working without login, expiring after weeks.
- F5: Login merges the guest cart into the account cart (union of line items,
  quantity max-ing; conflicts resolved to "keep, quantity adds") without losing
  either side; logout forks back to anonymous sessions.
- F6: Cart is durable across sessions/devices for logged-in users (saved-cart
  semantics), and prices in the cart re-resolve against the current price engine at
  checkout, not at add time.

**Checkout & orders**
- F7: An order is created exactly once per checkout submission under arbitrary
  client retries and network drops; the customer never sees a double charge and
  never loses a paid order to a timeout.
- F8: Checkout validates cart state (price, availability, promo eligibility,
  address) at submission time as a consistent snapshot, then runs payment
  authorization, reservation, and fulfillment hand-off.
- F9: Post-order order-status lifecycle (PLACED → PAID → FULFILLING → SHIPPED →
  (DELIVERED|RETURNED|CANCELED)) with customer-visible tracking and an audit trail.

**Inventory**
- F10: Multi-node availability (region → fulfillment center) with an
  available-to-promise (ATP) number; PDP shows an approximate badge ("In stock",
  "Only 3 left") that may lag reality.
- F11: Reservation semantics: adding to cart may soft-reserve briefly; the *order*
  hard-reserves; unclaimed reservations expire and return stock automatically.

**Promotions**
- F12: Coupon codes, percentage/fixed discounts, threshold ("free shipping over
  $35"), and time-boxed promo pricing, all applied at cart/checkout evaluation, not
  baked into catalog display price.
- F13: Redemption limits (per code, per account, per time window) enforced
  correctly under concurrent redemptions; expired or exhausted codes fail closed
  with a clear error.

**Non-functional requirements**

- N1: Read plane availability ≥ 99.99% (browse and search must survive the
  transactional plane being degraded); order path availability ≥ 99.9% with
  fail-closed behavior — degrade the UX, never the correctness of money.
- N2: Read latencies: PDP server response p99 < 300ms (plus CDN for media);
  search p99 < 500ms; checkout p99 < 1.5s; order API p99 < 3s.
- N3: Scale targets from `02-scale-estimates.md`: ~100K reads/s baseline, ~1M
  reads/s at Black Friday peak; ~50 order/s mean, thousands/s at sale peaks.
- N4: Seasonal headroom: infrastructure must absorb a 10× multi-day surge with 5
  minutes' notice (flash sales are announced, not scheduled) and must shed load
  predictably beyond that.
- N5: Inventory oversell rate target: effectively zero on the order path (system
  invariant); approximate inaccuracy on display stock acceptable up to minutes of
  lag and a few percent of marginal stock.
- N6: Auditability: every order, price change, promo redemption, and inventory
  movement is an append-only, replayable event — the read models are projections.

## Explicit non-goals (out of scope)

- **The money settlement/ledger design**: authorizations, capture, refunds,
  receipts, chargebacks. The purchase ledger, signed receipts, and offline
  entitlement machinery are owned by `../06.apple-app-store/04-deep-dives.md`
  (deep dive 4); this case consumes a payment-authorization call and stops there.
- **Payments fraud/ML risk scoring**: assume a passed-in risk verdict per order.
- **Warehouse/fulfillment robotics**: this case ends at "reserved, handed to a
  fulfillment workflow".
- **Third-party marketplace settlement**: seller accounting is a finance system,
  not a storefront one.
- **Search ad-auction economics**: sponsored slots are a ranking boost input here,
  not a bidding system.

## Scope decisions

- The storefront is *read-plane-first*: we accept display staleness (minutes) on
  everything that isn't money. The one place we do not accept staleness is the
  order path — checkout re-validates against the transactional core, so staleness
  upstream only costs a rerender of the cart, not a wrong charge.
- Inventory is the only cross-domain consistency coupling in the system, and we
  keep it deliberately narrow: checkout is the single writer of hard reservations
  (deep dive 3).
- Flash sales are treated as a distinct traffic shape with their own admission
  machinery (waiting rooms/queues), not as "Black Friday, but bigger" — the
  concurrency math is different (deep dive 6).
