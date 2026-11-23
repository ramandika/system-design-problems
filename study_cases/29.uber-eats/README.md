# Case 29 — Uber Eats

**The system**: a three-sided food delivery marketplace connecting ~1M merchants
(restaurants), a flexible courier fleet (~1.2-1.5M couriers online at peak, globally; see
[02](02-scale-estimates.md)),
consumer fleet of roughly 80-90M monthly diners (estimate; see
[02](02-scale-estimates.md)). The surface a user sees is a menu browser and a map
with a courier dot; underneath it are: a catalog system per restaurant, an order state
machine that must reconcile three parties' views of one order, a dispatch engine that
assigns couriers to orders under seconds-long latency, and a live-tracking pipeline
ingesting second-by-second GPS pings.

**Why it's hard**:

- **Three parties, one source of truth, three clocks.** A single order exists as: a
  promise to the customer ("arriving 19:42"), a kitchen queue entry for the restaurant
  ("start now"), and a job for the courier ("pick up in 8 min"). All three see the same
  order through different clients and different networks, and any of them can be offline,
  lying, or late. The order state machine is the contract that keeps them consistent
  without any of them trusting each other.
- **Dispatch is a two-sided matching problem in motion.** Unlike a ride-hail trip, a
  delivery has a *staged* timeline: courier must arrive, food must be ready, handoff must
  happen. Assigning early buys planning; assigning late buys accuracy. The whole dispatch
  chapter is about where to stand on that line (compare
  [28.uber](../28.uber/04-deep-dives.md), where the trip starts the moment the match
  happens).
- **The double-sided timing problem is the product.** Food getting cold on a counter while
  a courier circles for parking is a *system* failure even when nobody did anything wrong.
  Predicting "food ready at t" from kitchen behavior and "courier arrives at t" from road
  conditions — and coordinating both to converge — is a live scheduling problem, not a
  display problem.
- **The traffic shape is spikier than ride-hail.** Demand concentrates in ~2-hour lunch
  and dinner windows, and catalog reads (browsing menus) outnumber orders ~50:1. The read
  path must be nearly free so the write path can afford to be careful.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**:

- **One-to-one matching with time-staged assignment** — the same matching core as
  [28.uber](../28.uber/04-deep-dives.md) with a second, non-vehicle clock added (kitchen
  prep time), which breaks "send the nearest car" and forces batching + ETA-aware
  assignment.
- **State machine as multi-party contract** — an order lifecycle where transitions may be
  initiated by any side, each idempotent and auditable. Payment settlement lives downstream
  and is derived from state transitions (ledger pattern from
  [06.apple-app-store](../06.apple-app-store/README.md)).
- **Massive asymmetric read path** — catalog/menu serving through a CDN with per-merchant
  cache busting; the same "make reads edge-dominated so writes can afford consistency" move
  as [09.cdn](../09.cdn/README.md).
- **Live telemetry with downsampling** — courier GPS ingest at hundreds of thousands of
  writes/s feeding both dispatch and consumer tracking; the ping→road/travel-time loop is
  the one in [26.google-maps](../26.google-maps/README.md), scoped to last-mile delivery
  (see [22.frontend-uber](../22.frontend-uber/README.md) for the client-side split).
- Event backbone decoupling dispatch, tracking, and accounting — the Kafka usage patterns
  from [13.kafka](../13.kafka/README.md).

**Try it first**: before reading 04, answer: (1) if the customer's ETA is just
"courier travel time to restaurant + food prep time + courier travel time to customer",
what's wrong with each of the three terms — and which term is least observable and why?
(2) Should dispatch assign a courier at order placement or when the food is marked ready —
what does each cost? (3) When the restaurant's tablet crashes mid-service and never sends
"order ready", how does the system recover it without a human watching reconciliation
dashes, and who is allowed to force the state forward?

**Primary sources**: scale assumptions are built up from publicly reported Uber figures —
Uber reported roughly $75B annual gross bookings in 2022, rising to ~$140B in 2023, with
the Eats/Delivery segment a large minority share (Delivery segment gross bookings publicly
in the ~$60-70B range in 2023), ~1M+ active merchants, and high single-digit
billions of trips/orders per year across products; specific Eats order counts are our
estimates derived from those ranges and are labeled as such in
[02](02-scale-estimates.md). All numbers in this case are estimates from assumptions, not
leaked figures. Dispatch mechanics referenced as publicly documented in ride-hail
literature and Uber engineering publications; the delivery-specific design is our own
construction for study purposes.
