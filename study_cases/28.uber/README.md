# Case 28 — Uber Backend: Matching, Pricing, Driver Location

**The system**: the *server* half of a real-time rideshare — the backend that ingests
driver positions at hundreds of thousands of updates per second, answers "which nearby
drivers can take this ride?", runs the matching engine, computes fares and surge
multipliers, and owns the trip state machine from request to completion. It is the
deliberate server twin of [Case 22](../22.frontend-uber/README.md), which scoped this
half out: there, "client renders, server decides" — and this case is the part that
decides.

**Why it's hard**:

- **The core state is a moving set.** A conventional system stores rows; here ~1M drivers
  online each contribute a position that goes stale within seconds. The database is the
  wrong tool — the answer to "nearby drivers" lives in memory, partitioned by geography,
  and every component must reason about how wrong the data is (deep dives 1–2).
- **It is a two-sided market, not a request/response query.** A rider request and a
  driver accept are two independent intentions mediated by the system; the matching
  engine must make an assignment in a second or two, not queue it, and must never assign
  the same driver to two riders or the same rider to two drivers (deep dive 3).
- **Price is a control signal, not a lookup.** Surge pricing is computed from
  supply-and-demand imbalance and immediately changes that demand — the system
  continuously perturbs its own input, so naively designed it oscillates (deep dive 4).
- **Correctness transitions ride on a best-effort world.** Driver went offline mid-trip;
  GPS jumped 2 km; the driver app crashed after accepting; the rider cancelled twice
  at once. The money-relevant state machine must be idempotent and reconcilable even
  though the position stream dropping packets is normal, expected behavior (deep dive 5).

**Reading order**: [01-requirements](01-requirements.md) →
[02-scale-estimates](02-scale-estimates.md) → [03-architecture](03-architecture.md) →
[04-deep-dives](04-deep-dives.md). Read [Case 22](../22.frontend-uber/README.md) first
if you want the client-side picture; its battery-aware throttling and dead-reckoning
decisions are the *other end* of every protocol designed here.

**Patterns taught here**: stale-data-aware write-in-memory read-out-memory stores
(geo-sharded driver availability), adaptive telemetry sampling driven by *usefulness
of the next sample*, geospatial indexing (geohash/H3/quadtree trade-offs), batched
one-to-many assignment optimization under a latency budget, a control-loop pricing
system with hysteresis, and an event-sourced state machine with idempotent transitions —
the same delivery/guarantee shapes as [Case 03](../03.whatsapp/04-deep-dives.md)
(different signals on different channels) and the read-path locality of
[Case 10](../10.distributed-cache/README.md). The routing/ETA engine itself is out of
scope here — it is [Case 26](../26.google-maps/README.md)'s territory; this design treats
ETA as an internal service dependency. The sibling fleet-management case,
[Case 27](../27.ev-charging-network/README.md), manages fleet availability without a
real-time market dimension — a useful contrast for what matching adds.

**Try it first** (before reading 03/04): (1) 1M drivers online, positions every 3–15 s —
can you keep them all in memory, and where do you shard? (2) ~50 rider requests/s arriving from one dense CBD full of free drivers
(global request peak ~460/s): assign greedily one rider at a time, or match
in small batches?
What does the delay buy you? (3) A driver accepts, but their app dies before the
acknowledgment reaches the socket — one driver, two riders: who wins, and how does the
system tell?

**Primary sources**: synthesis case. Uber's engineering blog has publicly described
pieces of its real-time infrastructure (geo-sharding, driver availability, dispatch,
surge mechanics); where a public figure is cited it is marked "publicly reported ~X".
All scale numbers below are rough estimates derived from stated assumptions, not
measurements, and are kept consistent with the twin case
[Case 22](../22.frontend-uber/02-scale-estimates.md), which derives the same traffic
from the client side.
