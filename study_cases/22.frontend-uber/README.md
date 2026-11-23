# Case 22 — Frontend System Design: Uber

**The system**: the *client* of a real-time ride app — the rider and driver applications
that render a live map, hundreds of moving vehicle markers, and a ride lifecycle state
machine, all over a cellular network. The backend is a given: matching, pricing, and
trip storage exist (this is the server twin in spirit of [Case 03](../03.whatsapp/README.md)'s
routers-and-mailboxes split). This case designs what runs in the app.

**Scope note**: this is a *frontend* system design, like [Case 19](../19.frontend-chat-app/README.md).
The deliverable is client architecture — rendering, streams, state, storage, degradation —
not a distributed matching system. The server half this case treats as a given is its own
twin case: [Case 28 — Uber backend](../28.uber/README.md) (geo indexing, matching, surge,
server-side ride state).

**Why it's hard** (all client-side problems):

- **The map is fully dynamic.** A maps app mostly renders static tiles and cached places;
  here the entire interesting layer *moves* — every marker has a position that changes
  every few seconds, and 60fps smoothness of hundreds of moving markers is the difference
  between "the car is coming" and "the app is broken."
- **Everything real-time arrives over a broken network.** Positions stream over sockets
  on cellular, in tunnels, on subways, at 4% battery. The client must interpolate the
  world between updates, throttle what it sends to save battery, and keep a usable UI
  when the stream dies.
- **A ride is a long-lived state machine spanning 30+ minutes and many reconnects.**
  Requesting → matched → driver en route → arrived → trip → paid → done. The client
  never holds the truth (the server does); it holds a *projection* that must reconcile
  cleanly after app kills, reconnects, and out-of-order events — the same shape of
  problem as optimistic message state in [Case 19](../19.frontend-chat-app/04-deep-dives.md),
  but with money and physical safety attached.
- **The client is one of two parties that must agree on where the car is.** Rider-side
  and driver-side apps each send/receive positions; GPS error, map snapping, and update
  cadence mean the two rendered cars never exactly agree. The design decides how much
  disagreement is acceptable.

**Reading order**: [01-requirements](01-requirements.md) →
[02-scale-estimates](02-scale-estimates.md) → [03-architecture](03-architecture.md) →
[04-deep-dives](04-deep-dives.md).

**Patterns taught here**: canvas-vs-DOM rendering decisions for moving layers, clockless
stream interpolation (dead reckoning between updates), battery-aware adaptive throttling,
client state machines with server-owned truth and reconciliation, last-known-good degraded
modes, tile prefetch along a predicted route. These are the client-side twins of the
server patterns in [Case 03](../03.whatsapp/04-deep-dives.md) (connection management,
delivery guarantees) and the delivery patterns in [Case 09](../09.cdn/README.md)
(tile bytes come from a CDN, not from your backend).

**Try it first** (before reading 03/04): (1) given a driver position every 3 s, what does
the rider see in between — a frozen car, a dead-reckoned car, or a car animated between
known points, and who computes that? (2) At 2% battery in a tunnel, which signals does
the client stop sending first, and what does the rider still see? (3) The app is killed
mid-trip and reopened 3 minutes later — what single request reconstructs the full UI?

**Primary sources**: synthesis case. Uber's engineering blog has publicly described its
real-time driver-location infrastructure (RTDI) and map-tooling stack; where a public
figure is cited it is marked "publicly reported ~X". All scale numbers below are rough
estimates derived from stated assumptions, not measurements.
