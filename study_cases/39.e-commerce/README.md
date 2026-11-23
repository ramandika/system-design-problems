# 39 — E-commerce storefront

## What the system does

A large multi-category retail storefront (the Amazon/eBay/Walmart class of system) is
five subsystems wearing one shopping cart:

1. **Catalog + search**: hundreds of millions of SKUs of product data, faceted
   search over a heavily denormalized index, and per-region price/availability
   overlays. The dominant workload is *browse* — reading products at enormous
   fan-out.
2. **Cart**: a per-session shopping cart that works for anonymous guests and merges
   into the account cart on login, surviving days of abandonment.
3. **Checkout + orders**: an idempotent order-creation pipeline that turns a cart
   into money, payment authorizations, and a fulfillment workflow.
4. **Inventory**: "can I actually ship you this item" — multi-node availability
   (ATP), reservations against oversell, and eventual consistency between the
   sellable numbers on the PDP and the warehouse.
5. **Promotions**: coupons, discounts, and promo pricing that must survive a
   million people applying the same code at the same second.

## Why it's hard

- **The read/write asymmetry is extreme.** A top retailer serves roughly a hundred
  thousand browses per second and creates on the order of tens of orders per second
  at the daily mean — a read:order ratio around 2,000:1 (02-scale-estimates.md).
  Nearly every
  architectural choice is "serve reads from somewhere cheap and eventually
  consistent; reserve strong consistency for the tiny transactional core."
- **The spike is annual, not random.** Black Friday/Cyber Monday and flash sales
  produce 5–10× multi-day traffic, concentrated on a handful of SKUs. Capacity you
  buy by the day you pay for by the year.
- **Inventory is a consistency trap.** Sellable quantity must be *near* correct on
  every product page (eventual is fine, minutes late) but *exact* at the moment of
  purchase (strictly serialized against oversell). Same datum, two different
  consistency contracts, and the system has to hold both simultaneously.
- **Money with an interactive UX.** Checkout must be idempotent under retried
  requests and flaky mobile networks, exactly like the app-store purchase
  pipeline — but the retail shape adds a cart state machine, multi-line-item
  reservations, and promo redemption, all of which must roll back together.

## Reading order

1. `01-requirements.md` — functional scope, NFRs, and what is explicitly out of
   scope.
2. `02-scale-estimates.md` — traffic asymmetry, peak/spike math, storage. All
   arithmetic shown; numbers are estimates unless labeled otherwise.
3. `03-architecture.md` — the read plane vs. the transactional plane, and the
   event backbone that joins them.
4. `04-deep-dives.md` — catalog denormalization and faceted search, cart merge,
   inventory reservations vs. decrements, idempotent checkout, coupons under
   concurrency, and flash-sale surge shaping.

## Patterns this case drills

Browse-heavy read caching (vs the distributed-cache and CDN patterns), denormalized
search/index read models fed by an event stream (Kafka case), idempotent money
pipelines (the retail generalization of the app-store purchase ledger — this case
does *not* re-derive the receipt/ledger design, see
`../06.apple-app-store/04-deep-dives.md`), reservation-with-expiry vs. synchronous
decrement, and admission control under seasonal surge.
