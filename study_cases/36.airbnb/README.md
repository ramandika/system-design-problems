# Case 36 — Airbnb (vacation-rental marketplace)

**The system**: a two-sided marketplace for short-term stays. Hosts list a property (photos,
amenities, house rules, price, a calendar); guests search a city for travel dates, filter
hard (price, type, rooms, instant book), compare on a map, and book; the platform takes the
guest's money at booking time, holds it as escrow, pays the host after check-in, and issues
refunds when stays go wrong. Four surfaces matter here: **search/discovery** (the dominant
read load), **the calendar** (per-listing day inventory, the real inventory model), the
**booking + payment pipeline** (hold → confirm → payout, the money-correctness core), and
**trust** (reviews, host/guest verification, fraud defense).

**Why it's hard**:

- **Inventory is a calendar, not a scalar.** One listing sells ~3.5-night slices of a
  730-day horizon grid. Search must filter on "available for THIS date range" across 8M listings on
  every query, but the search index is always a few minutes behind the booking database —
  so search is advisory by design, and the only correctness boundary is the reservation
  step.
- **Double-booking is a correctness failure, not an inconvenience.** Two guests paying for
  the same nights is a refund, an apology, and often a guest with no place to stay that
  night. The whole booking path is built so that only one writer per listing can decide
  whether specific nights are free.
- **The money moves three times**: guest pays at booking, platform holds as escrow, host
  is paid after check-in — in different currencies, through PSPs, with refunds and
  chargebacks that can arrive months later. The ledger must be exactly-once in effect
  (the discipline of [06.apple-app-store](../06.apple-app-store/README.md)) across the
  three legs.
- **Fraud is adversarial and structural.** The deadliest pattern is one machine: a fake
  listing (scraped photos, no real home) + stolen payment cards + instant payouts = a
  cash-out pump. The defense is not a classifier; it is a *payout architecture* that makes
  the scam unprofitable.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**:

- Spatial search with per-query date-range + filter combination over millions of items —
  the geo-indexing of [26.google-maps](../26.google-maps/README.md) (dive 2 there) bent
  toward marketplace ranking instead of pure POI distance.
- Calendar inventory and search-vs-truth staleness: search filters are advisory, holds are
  authoritative — the same split as occupancy publishing vs reservation in
  [27.ev-charging-network](../27.ev-charging-network/README.md).
- Two-phase reservation (hold → confirm) with payment authorization inside the hold, and
  refunds as compensating ledger entries — reservation-as-timer and credit-not-delete both
  appear in [27.ev-charging-network](../27.ev-charging-network/04-deep-dives.md), exactly-once
  money in [06.apple-app-store](../06.apple-app-store/04-deep-dives.md).
- Escrow + delayed payout as a fraud-control *architecture*, not a policy afterthought.
- UGC photos and map/search results through a CDN with hostile cache keys — the
  cache-key economics of [09.cdn](../09.cdn/README.md), applied to listing imagery.

**Try it first**: before reading 04, answer: (1) a guest is looking at a listing page; the
calendar shows July 10–14 open, but another guest booked July 11 two minutes ago — where in
the pipeline does she get stopped, and how long does the wrong calendar stay wrong?
(2) payment authorization succeeded but the hold commit failed — what are the two possible
worlds, and which reconciliation job detects each? (3) why does the platform hold payouts
until after check-in, and which of the fraud patterns in deep dive 6 does that timing make
structurally unprofitable?

**Primary sources**: Airbnb's own reporting is public on the main numbers: roughly ~8M
active listings (reported in 2023–2024), ~0.5B nights and experiences booked per year
(about 491M in 2024), gross booking value in the high tens of billions of USD per year
(~$81B in 2024, which implies an average of roughly $165 per booked night — derived),
roughly 5M hosts, and 2B cumulative guest arrivals crossed around 2024. Everything below that line
(DAU, QPS, session shapes) is our own estimate — labeled as such in
[02](02-scale-estimates.md). The case is a design study, not a description of Airbnb's
internal architecture.
