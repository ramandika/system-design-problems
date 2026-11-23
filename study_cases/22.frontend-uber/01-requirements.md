# 01 — Requirements

## What we are designing

The rider app and driver app of a ride-hailing service. Their job: show a live map, show
the vehicles that matter, take the user through the ride lifecycle (browse → request →
match → pickup → trip → payment), and do all of it on mobile hardware over cellular
networks. The backend (matching engine, pricing, driver-side telemetry ingestion) is a
given service with a defined contract.

Functional scope concentrates on the *client machinery*: rendering, the position stream,
the ride state machine, and degradation. Out-of-scope subsystems are listed at the bottom.

## Functional requirements (rider app)

- **F1 — Live map**: pan, zoom, rotate-lite map with vehicle markers, pickups, route
  polylines. Base map from a map SDK; vehicles as the app's own animated layer.
- **F2 — Vehicle discovery view**: before requesting, show nearby available vehicles
  (hundreds in a dense city, ~150–400 in the visible viewport — estimate from
  [02](02-scale-estimates.md)), updating every
  few seconds.
- **F3 — Request & match**: rider requests, sees "matching…" (search animation), then the
  matched driver's car, plate, name, ETA, and route to pickup.
- **F4 — Pickup tracking**: driver-car marker moves toward the rider in (near) real time;
  ETA updates continuously; "driver is 2 stops away"-style refinement is nice-to-have.
- **F5 — Trip tracking**: during the ride, the car follows the route; reroutes redraw the
  polyline.
- **F6 — Payment & completion**: fare shown after arrival, confirm/leave, rating. Fare
  authority is the server; client renders.
- **F7 — Reconnect continuity**: app killed mid-ride, reopened → user resumes exactly
  where the ride is (one request reconstructs full state).
- **F8 — Degraded mode**: with no/poor network the app shows last-known-good state,
  clearly labeled, and recovers automatically.

## Functional requirements (driver app)

- **F9 — Driver location**: regular position updates while online (the input the whole
  system runs on). Driver app owns GPS reading, map-snapping to roads, and cadence.
- **F10 — Accept flow**: request offers with a countdown; accept/reject; acceptance races
  between drivers are server-resolved, client just renders win/lose.

## Non-functional requirements

| Requirement | Target | Notes |
|---|---|---|
| Marker animation smoothness | 60 fps on mid-tier phones | the car must not stutter; this is the NFR that dictates the rendering architecture (deep dive 1) |
| Position-freshness (visible cars) | ≤ 5 s stale normally, ≤ 15–30 s degraded (polling; unbounded but labeled when truly offline) | update cadence vs battery/network cost (deep dive 2) |
| Position-update latency | ≤ 3 s p99 to rider during pickup | "publicly reported"-style target; matching/pickup is the freshness-critical phase |
| Rider→driver perceived agreement | marker gap ≤ ~30–60 m | the two clients never agree exactly (deep dive 2) |
| Battery | no continuous GPS/network wake when app backgrounded & idle | throttling policy is battery-aware (deep dive 2) |
| Cold restart of ride UI | ≤ 2 s from reopen | drives the snapshot/state contract (deep dive 3) |
| Crash-freedom of state machine | no unhandled (state, event) pair | an unhandled pair = frozen ride UI (deep dive 3) |
| Offline usable | browsing, cached map, last-known car | deep dive 4 |

## Explicit scope decisions

**In scope**: both apps' client architecture; position stream ingestion/rendering on the
client; ride lifecycle state machine; degraded modes; map SDK integration and tile strategy.

**Explicitly OUT of scope** (owned by other systems; the client renders their output):

- **Matching engine & surge pricing** — backend (Case 03-style server territory). Client
  sees only offers/matches.
- **GPS quality & map-matching accuracy on the server** — the driver client snaps to roads,
  the server refines; we design the client side of that pipeline only, enough to know what
  accuracy claim each message carries.
- **Payments/ledger integrity** — client is a dumb terminal for fare truth; anti-fraud is
  server-side.
- **Safety features (panic button, incident flow)** — requirements-level: they exist as
  state-machine-safe escape hatches (always available from *any* ride state), but the flows
  themselves are out of scope.
- **Driver earnings, queueing, quests** — another app surface, same primitives.
- **City map data pipelines** (which roads exist, ETA graphs) — backend; see the tile/data
  side in [Case 09](../09.cdn/04-deep-dives.md) for the delivery layer.

**Deliberate simplifications** (stated so the design is honest): single-city per region
cluster assumed for stream fanout; no multi-destination/carpool UI states; fare estimate vs
locked quote treated as one "quote" state.
