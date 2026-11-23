# 01 — Requirements

## Functional requirements

### Guest / discovery

- **F1 — Search**: find listings for a destination (free text or map region), check-in and
  check-out dates, number of guests. Results are ranked, not filtered-only.
- **F2 — Filters**: price range, property type, rooms/beds, amenities (wifi, parking, pool,
  ...), instant-book only, rating minimum, superhost only. Filters refine freely on a
  result set without re-submitting the whole query.
- **F3 — Map browsing**: move/zoom on a map re-runs the search for the new viewport and
  shows a handful of top listings (photos, price) per map tile; tapping a tile opens full
  results for that area.
- **F4 — Listing detail page**: photos, description, amenities, house rules, reviews,
  host profile, a night-by-night availability calendar, and a per-dates price breakdown.
- **F5 — Booking**: pick nights, submit booking. Two modes: **instant book** (host
  pre-approved; confirm immediately after payment authorization) and **request-to-book**
  (host must accept within a response window; auto-decline/expire otherwise).
- **F6 — Payment**: charge the guest at booking (full or split schedule), in the guest's
  currency. Security deposits where hosts request them.
- **F7 — Cancellation & refunds**: per-listing cancellation policy (flexible/moderate/
  strict) determines refund amounts; refunds are issued back to the original payment
  instrument, in the original charge currency.
- **F8 — Messaging**: guest ↔ host thread around a booking and before it (pre-booking
  messages are also where off-platform payment pressure is policed).
- **F9 — Reviews**: after a stay, guest and host may review each other; reviews are
  double-blind (both submitted or window expires, then both revealed). Hosts can respond.
- **F10 — Payouts**: the host receives the booking amount minus platform fee after
  check-in, in the host's payout currency.

### Host

- **F11 — Listing management**: create/edit listing (photos, amenities, description,
  sleeping arrangement), set pricing (base, weekend, cleaning fee, minimum/maximum nights),
  link/merge calendars across platforms.
- **F12 — Calendar**: block/unblock individual dates and ranges beyond bookings; bulk
  blocking.
- **F13 — Instant book control**: opt in/out per listing; automatic booking requirements
  (verified ID, minimum good reviews).

### Platform / trust

- **F14 — Identity verification**: guest/host identity checks (KYC tiers), device and
  account trust signals feeding risk decisions.
- **F15 — Fraud operations**: automated risk holds, review queues, listing takedowns,
  dispute and damage-claim handling, chargeback representation.
- **F16 — Support**: case management for cancellable/refundable events, rebooking help
  after host cancellations.

## Non-functional requirements

- **N1 — Search latency**: interactive search p95 ≤ 300 ms server-side; map-move re-query
  is debounced client-side and p95 ≤ 500 ms.
- **N2 — Availability staleness**: search results may disagree with true availability by
  at most ~minutes; the booking step must consult *current* truth regardless of search
  staleness (search is advisory, holds are authoritative).
- **N3 — Double-booking correctness**: no confirmed booking may overlap another confirmed
  booking or an active hold for the same nights, ever, under any failure. This is absolute —
  no percentage, no degradation mode.
- **N4 — Money correctness**: every charge, refund, payout, and fee moves the ledger exactly
  once in effect; every money event is auditable and reconcilable to PSP statements.
- **N5 — Booking flow latency**: hold + payment authorization p95 ≤ 3 s end to end (the
  guest sees pending → confirmed within seconds for instant book).
- **N6 — Calendar staleness (host view)**: host-side calendar shows holds and bookings
  within ~seconds; a blocking edit takes effect before the next booking can pass against it.
- **N7 — Review reveal fairness**: neither party can read the other's review before
  submitting their own or letting the window expire; the window is fixed policy.
- **N8 — Payout timing**: payouts release on a fixed schedule anchored to check-in; a
  risk flag can delay a specific payout but never accelerates one.
- **N9 — Availability of the write path**: booking and payment accept path ≥ 99.95%;
  search may degrade to cached/ranked-but-not-date-filtered results during partial
  outages, with the calendar on the detail page kept current.

## Scope decisions (explicit cuts)

- **No "Experiences"** (the activities side of the real product): same review machinery,
  different inventory model; cut.
- **No long-term (monthly+) lease logic**: long stays stress the calendar and payment
  schedule differently; we design for stays up to 30 nights and note where it breaks.
- **Payments via PSP integrations only**: we don't acquire; we model the marketplace ledger
  on top of PSP capabilities (authorization, capture, reversals, multi-currency presentment).
- **Dynamic pricing (ML-suggested prices for hosts)** out of scope; the architecture only
  needs pricing to be per-listing data read at search time.
- **Photo derivatives are computed on the fly / precomputed pipeline**, not stored per
  size in the DB — see [04 deep dive 1](04-deep-dives.md) and the CDN reasoning in
  [09.cdn](../09.cdn/04-deep-dives.md).
- **Design scale**: worldwide coverage with hot metro concentrations (London, Paris, NYC,
  Lisbon, etc. dominate); sharding by geography is the main structural consequence, per
  the regional-shard reasoning in [26.google-maps](../26.google-maps/04-deep-dives.md).
