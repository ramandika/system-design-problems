# 01 — Requirements and scope

## Functional requirements

**Rider side**

1. **Ride request**: rider specifies pickup (usually current location from the client)
   and destination; the backend returns a fare estimate and ETA, then creates a ride
   request and begins matching.
2. **Matching**: the backend assigns a concrete driver, communicates the offer, handles
   driver accept/reject, and can re-dispatch if rejected or timed out.
3. **Trip lifecycle**: track a ride through requested → matched → arriving → arrived →
   in-progress → completed, exposing state changes to both apps in near-real-time.
4. **Live vehicle position**: during the arriving phase, stream the assigned driver's
   position to the rider (the client-side rendering of this is Case 22).
5. **Pricing**: compute a fare from base + distance + time + booking fee, plus a
   geographic multiplier (surge) when supply is short; expose the multiplier to riders
   and the *incentive* version to drivers before they see requests.
6. **Cancellation**: rider or driver can cancel; policy (who pays a fee) is a product
   rule on top of the state machine, not the state machine itself.
7. **History and receipts**: completed trips are queryable; the fare breakdown is
   reconstructible per trip.

**Driver side**

8. **Availability**: drivers go online/offline; only online drivers are matchable.
9. **Location reporting**: every online driver's app streams position updates; the
   backend ingests, keeps the latest per driver, and maintains the geospatial index.
10. **Offers**: the backend pushes trip offers to selected drivers with a countdown;
    the driver accepts or declines; acceptance drives the state machine.

**Platform**

11. **Geospatial query API**: "drivers near point P within radius R" used by matching
    and by the rider's map view (viewport vehicles).
12. **Event pipeline**: all state transitions and position samples flow through an
    ordered event log feeding analytics, ML feature pipelines, and replay.

## Non-functional requirements

- **Match latency**: rider-perceived time from request to driver-assigned: p50 ≤ ~3 s,
  p99 ≤ ~10 s (target; product-visible). A "no driver found" answer within ~60 s is
  better than a silent 3-minute wait.
- **Position ingest**: ~140K updates/s average, ~280K/s at peak (derived in 02);
  individual updates may be dropped — the system is *lossy by design at the sample
  level*, never at the state-transition level.
- **Freshness**: nearby-driver views and matching operate on data ≤ ~5–10 s old
  (estimate target); the state machine operates on data that is never silently lost.
- **Availability**: request/quote paths available through regional failures; matching
  degrades to "wider search radius / no-match" rather than hanging.
- **Consistency (narrow and critical)**: a driver is assigned to at most one active
  trip, ever. Double-assignment is a correctness failure, not a UX bug — this is the
  one strongly-consistent invariant in an otherwise eventually-consistent system.
- **Fairness/legality**: pricing must be explainable (no discriminatory variables),
  and surge must be auditable after the fact; several jurisdictions require publishing
  the rules.
- **Scale**: hundreds of cities, multi-region deployment, per-city traffic isolation
  so a mega-city's load cannot starve a small market.
- **Safety-critical data**: trip state is financial and physical-safety record; treat
  its durability like an OLTP system, unlike position telemetry.

## Explicit scope decisions

**In scope**: driver location pipeline (ingest, storage, indexing), nearby-driver
search, dispatch/matching engine, fare + surge computation, trip state machine,
offer/accept flow, the event stream that carries all of the above.

**Out of scope, with the owning case**:

- *Client rendering, map tiles, dead-reckoning UX* → [Case 22](../22.frontend-uber/README.md).
  This case defines the wire cadence those clients depend on, nothing more.
- *Routing, ETA computation, map-matching to roads, tile serving* →
  [Case 26](../26.google-maps/04-deep-dives.md). Uber-style matching calls an internal
  ETA service as a dependency; we specify the call, not the algorithm.
- *Payments accounting, fraud, driver earnings ledger* → out of scope entirely; the
  state machine emits the events those systems consume.
- *CDN/static delivery* → [Case 09](../09.cdn/README.md).
- *ML feature stores and training pipelines that consume position/trip events* → out
  of scope; the event log in 03 is their upstream.
- *EV charging/fleet availability domain logic* → [Case 27](../27.ev-charging-network/README.md)
  covers availability without an open market.

**Deliberate ambiguity resolved here**: the "nearby drivers" list on the map uses the
same in-memory index as the matching engine, but with looser freshness and no locking;
matching is the only consumer allowed to *reserve* a driver.
