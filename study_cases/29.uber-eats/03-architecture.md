# 03 — Architecture

## High-level diagram

```
                    │                CLIENTS (thin logic, fat surfaces)                │
                    │   Customer app        Courier app        Restaurant tablet / POS │
                    └──────┬──────────────────┬───────────────────────┬──────────────────┘
                           │ HTTPS            │ HTTPS                │ HTTPS
                           ▼                  ▼                       ▼
                    ┌─────────────────────────── Edge / API gateway ───────────────────────┐
                    │  routing + authn  + idempotency keys + rate limits per client class   │
                    └──────┬──────────────┬────────────────┬───────────────────────┬────────┘
                           │              │                │                       │
              ┌────────────▼───┐  ┌───────▼───────┐  ┌─────▼──────────┐   ┌────────▼─────────┐
              │ CATALOG SVC    │  │ ORDER SVC     │  │ DISPATCH SVC    │   │ TRACKING SVC     │
              │ - menus, mods  │  │ - order state │  │ - supply index  │   │ - ws fanout,     │
              │ - publish ver. │  │   machine     │  │   (geo index)   │   │   latest-wins    │
              │ - stock-outs   │  │   (owner)     │  │ - assignment    │   │ - per-order room │
              └───┬────────┬───┘  └──┬────┬───────┘  │   engine + ETA  │   │   multiplexer    │
                  │        │         │    │          └───┬─────────┬───┘   └───────┬──────────┘
                  │        │         │    │              │         │               │
        ┌─────────▼──┐  ┌──▼─────────▼─┐  │       ┌──────▼─────┐ ┌─▼───────────────▼─┐
        │ CDN        │  │ EVENT BUS    │◄─┼───────┤ ETA SVC    │ │ LOCATION PIPELINE │
        │ (menus,    │  │ ( Kafka )    │  │       │ (prep time │ │ ping ingest →     │
        │  photos)   │  │              │  │       │ + travel)  │ │ downsample →      │
        └────────────┘  │  topics:     │  │       └─────┬──────┘ │ geo-partitioned   │
                        │  orders.*,   │  │             │        │ live supply store │
                        │  dispatch.*, │  │             │        │ (in-mem + Redis)  │
                        │  location.*, │  │             │        └─────────┬─────────┘
                        │  payments.*  │  │             │                  │
                        └──┬────┬────┬──┘  │             │                  │
                           │    │    │     │             │                  │
              ┌────────────▼┐ ┌─▼─────────▼─┐   │      ┌──────▼────────┐ ┌───────▼────────┐
              │ SETTLEMENT  │ │ NOTIFICATION │   │      │ MERCHANT OPS  │ │ SUPPORT/RISK   │
              │ LEDGER      │ │ & PUSH       │   │      │ (analytics,   │ │ (reconciliation│
              │ (immutable) │ │ (state→client│   │      │ prep reports) │ │  alerts, fraud)│
              └─────────────┘ └──────────────┘   │      └───────────────┘ └────────────────┘
                                                 │
                                        ┌────────▼─────────┐
                                        │ BLOB STORE       │
                                        │ (photos, docs)   │
                                        └──────────────────┘
```

(ASCII; arrows show dominant call direction. The event bus is drawn once for clarity —
every service both publishes to and consumes from it.)

## Components

### Edge / API gateway

- One gateway, three client classes (customer, courier, merchant) with different rate
  limits, authn (per-party tokens), and idempotency requirements. Order placement and all
  state transitions require **idempotency keys** — mobile clients retry on flaky networks
  and a doubled "accept" or doubled payment auth must collapse to one.
- Terminates the sockets that become tracking subscriptions (or routes to the tracking
  service's socket tier).

### Catalog service (+ CDN)

- Owns menus, items, modifier groups, prices, hours, per-market fees; publishes versioned
  snapshots; busts CDN keys on publish. Read path is edge-dominated (see 02, ~25K QPS peak,
  ~95% cacheable). Owns nothing about a live order except *at order time* the order service
  takes a **price snapshot** of the cart (see 04.4 for why snapshotting beats live lookup).
- Menu changes from a restaurant are publishes (new version), not in-place edits —
  in-flight orders stay on the version they were priced against.

### Order service (the state machine — the owner of "what is true")

- Single ownership of order state. Accepts transitions from any party (customer cancel,
  restaurant accept, courier actions), validates them against the lifecycle in 04.1,
  writes append-only transition events, and emits them to the bus.
- Exposes reads per party (customer view, merchant view, courier view) as projections of
  the same state — different fields, same truth.
- Order-adjacent writes are sharded per order id; there is no cross-order transaction here.

### Dispatch service

- Two inputs joined in memory: live courier supply (geo-partitioned position store fed by
  the location pipeline) and unassigned/assignable orders (event bus).
- Runs the assignment loop described in 04.2: build candidate set per unassigned order (or
  batch window), score with ETA service, commit an assignment, emit `dispatch.*` events.
- Never writes to order state directly — it emits "course assigned" events that the order
  service consumes and applies as transitions. One owner of truth, many initiators.

### ETA service

- Two predictors behind one interface: **travel time** (courier→restaurant→customer legs;
  feeds from map/road graph + live traffic — the [26.google-maps](../26.google-maps/README.md)
  loop, scoped to city-scale) and **prep time** (kitchen behavior model per merchant,
  fed by merchant ops analytics). Dispatch consumes both; the consumer-facing committed ETA
  is a function of both plus queue position (see 04.5).

### Location pipeline

- Ping ingest at ~hundreds of K/s ([13.kafka](../13.kafka/README.md) for the stream-log
  rationale), immediate downsampling, geo-sharded live supply store (in-memory + Redis
  mirror) for dispatch reads, plus an emit into `location.*` for tracking fanout.
- Consumers: dispatch (supply index), tracking (courier dot), risk (route anomalies),
  analytics (downsampled trails).

### Tracking service

- Per-order "rooms" multiplexed over the three parties' sockets; publishes latest-wins
  state deltas (position, ETA band, lifecycle stage). ~4M concurrent sockets at dinner peak
  (02). Deliberately dumb: it replays order-service state plus position feeds, decides
  nothing.

### Settlement ledger

- Consumes order events; writes immutable ledger rows (charge, merchant cut, courier
  earnings, fees, tips, refunds/promos). No mutable balances in the hot path — balances are
  derived views over the ledger, exactly the pattern in
  [06.apple-app-store](../06.apple-app-store/README.md). Settles asynchronously after
  delivery; customer charge authorization happens synchronously at checkout.

### Merchant ops / support & risk

- Merchant ops consumes `orders.*` to build prep queues, prep-time reports, and funnel
  analytics; support/risk consumes everything for reconciliation alerts (state-machine
  stuck orders — see 04.1's failure modes) and fraud screens.

## Data flows

### Place order (happy path, synchronous part)

1. Customer client builds cart against cached catalog; prices are local until checkout.
2. `POST /orders` (idempotent) → order service validates against price snapshot, authorizes
   payment, writes `CREATED`, emits `orders.created`.
3. Merchant surface receives it (push via bus + notification service); customer gets an
   initial **committed ETA band** (ETA service) — see 04.5.
4. Restaurant accepts → `orders.accepted` → dispatch service picks the order up from the
   bus and starts assignment rounds.

### Dispatch (loop over the bus, not a request)

5. Dispatch every few seconds evaluates the unassigned queue per geo-region, scores
   (courier, order) candidates with ETA + timing alignment (04.2), and offers.
6. Courier accepts → `dispatch.assigned` → order service records the transition; tracking
   rooms update all three surfaces.
7. Courier arrives / picks up / drops off — courier app actions emit transitions the order
   service validates and applies; tracking notifies customer with ETA bands.

### Live tracking (firehose)

8. Location pipeline emits downsampled positions → tracking service pushes to the rooms for
   any order the courier is currently attached to. Lifecycle changes pre-position from the
   order service; positions and ETAs re-derive in the ETA service.

### Settlement (async)

9. `orders.delivered` → settlement ledger writes its rows; payouts settle in T+1-N batches;
   support refunds reference ledger rows, never rewrite them.

## What is cross-region and what isn't

- **Geography partitions everything except the ledger.** Supply, dispatch, ETA, and catalog
  serving are regional and sharded by market geo (same move as
  [26.google-maps](../26.google-maps/03-architecture.md)); the latency of a match matters in
  milliseconds and a match's inputs are all local.
- The **order store and settlement ledger are global-authoritative but market-partitioned**
  for writes; a customer's order is pinned to the market it was placed in — no cross-market
  transactions are needed, which is precisely why the design can afford a strong order
  contract everywhere else.
