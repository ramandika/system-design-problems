# 01 — Requirements

## The system in one paragraph

A primary ticketing platform for live events (arenas, stadiums, theaters, clubs).
Steady state: a searchable catalog of tens of thousands of live events (~50K, estimate), venue seat maps,
purchase flows for low-contention shows. The interesting state: a promoted on-sale for a
hot show, where a pre-announced instant turns a catalog into a flash-sale with
structural over-subscription — millions of demand for thousands of seats — plus an
automated adversary trying to beat the fans to the inventory.

## Functional requirements

**Steady state (catalog and purchase):**

- F1. Event catalog: search, filter, venue info, seat maps and availability by price
  tier and section.
- F2. Account registration and login; purchase history; stored payment methods.
- F3. Purchase flow: pick seats (or "best available"), hold them, pay, receive a
  deliveryable ticket (mobile barcode / NFC).
- F4. Ticket management: transfer to another person, list for resale per
  artist/promoter rules, entry validation at the gate.
- F5. Pre-sales and restricted windows: verified-fan cohorts, cardholder presales,
  promotional codes, per-cohort inventory allocation.

**On-sale state (the hard requirements):**

- F6. Queued admission: when load exceeds capacity, arrivals enter a waiting room and
  are admitted in a controlled, perceived-fair order, with a live position indicator.
- F7. Seat holds: an admitted user can hold a specific seat or best-available group for
  a bounded window (minutes) while checking out; the hold makes the seat unselectable
  by others and expires automatically.
- F8. Group integrity: a request for N seats must return N contiguous seats in one row
  (or declare failure) — never N scattered singles.
- F9. Idempotent checkout: retries, double-clicks, and network stalls must never
  produce two orders or two charges for one intent.
- F10. Per-account purchase caps: e.g., maximum 4–8 tickets per customer per event,
  enforced across presale + public sale and across delivery identities, not just per
  order.
- F11. Bot resistance: admission queue position and purchase eligibility must degrade
  for automated traffic patterns (device/behavior signals), without depending on any
  single signal.

## Non-functional requirements

- **N1. Spike scale (the target).** One promoted on-sale: an estimated tens of
  millions of interested fans for a super-hot show, millions of actual visitors in the
  first hour, against ~20,000 seats. Roughly 3.5 billion system requests were publicly
  reported for one day during the Taylor Swift Eras presale — that is the anchor for
  "hot on-sale". Design target: survive ~10× the biggest observed on-sale without
  selling a seat twice.
- **N2. Concentration, not volume.** Load is measured in minutes, not days (see
  [02-scale-estimates](02-scale-estimates.md)). The first ~5 minutes carry most of the
  traffic; the system must be at full capacity at second zero, because the ramp *is*
  the event.
- **N3. Inventory correctness — the invariant.** No seat may be purchasable by two
  buyers; no seat may be purchasable at all after being sold; holds must never outlive
  their TTL in a way that blocks sale. This trumps availability: it is acceptable to
  pause the sale to preserve it, not the reverse.
- **N4. Fairness as a product feature.** Among fans who follow the rules, allocation
  must be *perceived* as fair (queue order or documented lottery), and queue position
  must not be forgeable more cheaply than the defense makes it. Perfect fairness is not
  achievable against an adaptive adversary; the requirement is that cheating costs
  real money and effort.
- **N5. Latency profile.** Steady state: catalog p99 < 300 ms (normal e-commerce).
  On-sale: seat-map and hold operations p99 < 1 s from an admitted user; everyone else
  gets a fast waiting-room page (sub-second) with position updates. *Not* serving
  non-admitted users the seat map is a feature — it protects inventory from scraping.
- **N6. Availability semantics.** The catalog tier must stay up for *all* events
  (other shows' traffic doesn't stop because one show is on sale). The on-sale path
  may shed load (queueing) but must not crash or leak inventory. No maintenance window
  may intersect a scheduled on-sale; freeze deploys for the on-sale path ~1 hour
  before until inventory zeroes out.
- **N7. Money correctness.** Authorization must be captured exactly once per order;
  partial captures (buyer held 4, paid for 2) must be impossible; refunds/reversals of
  failed holds are idempotent. Reconciliation must be able to prove, after the dust
  settles, that money ↔ seats ↔ orders is consistent (cross-ref the money pipeline in
  [06.apple-app-store](../06.apple-app-store/04-deep-dives.md)).
- **N8. Auditability.** Every seat-state transition (available → held → sold →
  returned) is an immutable event with actor, reason, and timestamp; disputes and
  anti-fraud investigations replay it.

## Scope decisions

- **In scope**: primary on-sale, holds, fairness queue, purchase integrity, bot
  resistance, per-event hot-shard handling.
- **Out of scope**: secondary-market pricing engines, dynamic/"platinum" pricing
  strategy, box-office/hardware entry gates (except the ticket-delivery interface),
  and the promoter's marketing tooling.
- **The single most important decision**: most visitors are *not* shown inventory.
  Capacity for seat-map and hold traffic is reserved for the admitted population; the
  waiting room is the load-shedding interface. This inverts the usual web instinct
  ("keep everyone in the funnel") and everything downstream depends on it.
