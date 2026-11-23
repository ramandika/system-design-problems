# 04 — Deep dives

## 1. Rendering hundreds of moving markers at 60 fps

**The problem**: ~150–400 vehicles in the visible viewport (dense CBD; range from
02), each with a position that changes every few seconds, plus pan/zoom gestures at
60 fps (16.7 ms frame). The naive app — one native/JS view per marker, updated on each
stream batch — dies here.

**Three options:**

- **Option A — DOM/native view per marker**: wins: trivial event handling, accessibility,
  per-marker styling. Loses: each marker is a full UI object; updating 200×(pos,heading)
  triggers 200 layout/paint paths; at 10–100 µs per marker, 400 markers costs 4–40 ms —
  most of it past the 8 ms render budget from [02](02-scale-estimates.md) — plus
  batching-agnostic invalidation; pan/zoom repositions 200 nodes per frame.
- **Option B — single Canvas 2D layer, redraw-on-demand**: wins: one raster layer,
  drawImage/arc at ~µs scale per marker (~400 markers ≈ ~1 ms of draw calls);
  the *entire* moving layer invalidates as one unit; projection math and easing are yours.
  Loses: no free hit-testing or a11y (must map taps → nearest marker manually, expose data
  to screen readers separately); no sub-pixel text tricks; reticle work (DPR scaling,
  transform management) is on you.
- **Option C — WebGL layer** (or map SDK's built-in GL annotations): wins: thousands of
  markers, 3D tilt, fancy easing. Loses: heavy integration cost on mobile + battery cost
  when the GPU wakes 10×/s for nothing; overkill below ~5K markers.

**Chosen: B (Canvas 2D) for the moving layer, DOM/native for *the* matched car's info
card — A's strengths kept where they matter (one node), B's where they matter (200
nodes).** Trade-off made explicit: we give up free hit-testing and a11y on the moving
layer and accept ~1–2 weeks of extra platform glue (tap→marker hit test, marker
accessibility via a summary layer: "17 cars nearby, nearest 3 min") in exchange for
guaranteed frame budget at the top-left of the scale curve. Upgrade path: the target-list
interface (below) is renderer-agnostic, so swapping in C when cities get thousands of
idle cars changes one component, not the stream architecture.

**Interior details that make B work:**

- **Projection decoupling**: the stream delivers lat/lng; the renderer keeps a
  projection pipeline (lat/lng → world px → screen px) updated only on *camera* changes,
  not on every marker update; markers store world coords, so a pan is a single canvas
  translate, not 200 recomputations.
- **Easing between updates**: positions arrive every ~3–4 s; the renderer eases each
  marker from its last shown position to the newest target over ~1–2 s (ease-out), then
  holds a slow "drift" along heading (dead reckoning, deep dive 2) until the next target.
  Frozen-then-jump markers are what users read as "the app is buggy."
- **Render trigger discipline**: rAF loop runs *only* while any marker is mid-ease or the
  camera is moving; otherwise the layer sleeps. Continuously ticking rAF at 60 fps for a
  mostly-static map is pure battery burn on mobile.
- **Z-order/styling cheaply**: car-type → pre-rendered sprite; "closest car" pulse is a
  radial gradient at one position, not a per-marker style system.

**Failure modes:**

- *Stream burst mid-gesture*: 50 targets arrive at once during a pinch-zoom → drop all
  eases and jump-cut to new targets when a gesture starts (animating during gesture reads
  as jitter); restart easing on gesture end.
- *Battery-saver / low-end GPU*: rAF stalls → ease durations computed from real frame
  times, not fixed clocks, so markers converge even at 30 fps.
- *Canvas memory pressure* (huge DPR phones): clamp the vehicle layer's backing store to
  screen size × DPR cap; never render at 3× DPR on 720p.

## 2. The position stream: batching, interpolation, battery-aware throttle

**The problem**: positions are a firehose of truth that is *always slightly stale*.
Two directions: inbound (your driver's car → rider map) and outbound (driver GPS →
everything). The client must decide cadence, batching, and what the user sees between
fixes — under the constraint that network and GPS radios are the two dominant battery
costs on a phone.

**Inbound (rider: how often to receive car positions)**

- **Option A — server pushes raw updates as fast as the driver emits them (~3 s)**:
  wins: freshest truth, simplest client. Loses: wakeups per rider scale with driver
  cadence × riders-per-car interest; radio never sleeps during pickup; cellular keepalive
  chatter; battery cost paid by the *watchers*, who get no extra value below ~5 s staleness
  when walking to a corner.
- **Option B — client-requested cadence** (subscribe with phase-specific rate): wins: the
  policy engine (03) phases the app: browsing = 4–5 s batches, pickup = 2–3 s, trip =
  3–5 s, background = paused or 30 s; radio sleeps between cycles; server fanout load is
  client-shaped. Loses: a rate knob that must be re-negotiated across phases — protocol
  surface grows (subscribe/re-subscribe on transitions).
- **Option C — server-driven adaptive push** (server varies push rate by predicted use):
  wins: zero client knobs. Loses: server can't see screen state, charging, or battery;
  will over-push to a pocketed phone (worst case: ~280K concurrent in-progress trips ×
  per-second pushes for users who can't look at the screen).

**Chosen: B, with phase transitions driven by the client's state machine** (phases are
already state-machine transitions from deep dive 3; cadence rides the same transition
events re-negotiated with server ack, so the *only* component that changes stream rates
is the state machine — one authority). Trade-off: slightly more protocol surface vs
battery and fanout economics; picking A would cost every active rider continuous
wakeups, picking C pushes blind to device state.

**Interpolation / drift (what the user sees between updates)**. Between two fixes the
car moves ~25–70 m (city speeds 30–60 km/h at 3–4 s cadence: 8–17 m/s × 3–4 s). The client
renders: ease to newest
fix (1–2 s), then dead-reckoned drift along last heading at last speed up to a **max
drift budget (~one typical block, capped ~60 m and ~8 s)** — the 60 m cap is
[01](01-requirements.md)'s rider→driver agreement target, which worst-case rider-side
drift must not itself exceed — clamped to the drawn route
when one exists. Trade-off: the shown car is *sometimes wrong by up to the budget* —
we accept bounded fiction because a frozen-then-teleport car feels worse than a
slightly-wrong moving car. Reconcile rule: when the next fix lands, ease from wherever
the fiction got to — never snap. If real position diverges from dead reckoning beyond
budget (reroute, traffic), cut fiction immediately (jump is *correct* here: it signals
new truth).

**Outbound (driver: how often to send)**. Driving every ~3 s, waiting every ~15 s,
backgrounded = pause (with a 30 s "still alive" heartbeat). Radio batching: read GPS but
queue sends to flush together (per-send radio wakeups, not per-fix). Trade-off:
coalescing adds up to a few seconds of latency at the destination for waiting drivers
(harmless — nothing downstream needs 15 s-stale waiting cars fresher) in exchange for
fewer radio wakes. Uber's engineering blog has publicly described this shape (driver
position pipelined through a real-time ingestion layer; approximate cadences, exact
numbers are not public and treated here as design estimates).

**The accuracy tag**. Every update carries GPS accuracy; the client renders the car at
accuracy ≥ ~50 m with a soft indicator rather than as confident truth. Cheap and honest:
GPS in urban canyons lies, and map-snapping (client- and server-side projection to the
road graph) fixes shape, not position error — see deep dive 5's tile/route dependency.

**Failure modes:**

- *Clock skew between server and client*: all staleness math uses **server timestamps**
  echoed in updates (client renders "X s ago" from its own receive clock — drift-safe
  enough for UI, never used for state, deep dive 3).
- *Out-of-order fixes on lossy transport*: position channel is fresh-wins (no seq
  retransmit) but **still stamps sequence numbers locally** — a late-arriving older fix
  is dropped, not rendered (old > new regressions read as the car going backwards).
- *Battery-saver/Doze modes*: OS suspends sockets/GPS → policy engine explicitly
  degrades (deep dive 4) rather than fighting the OS with wakeups.

## 3. The ride lifecycle client state machine + server reconciliation

**The problem**: one ride spans ~30 min end-to-end — matching wait + pickup + the
~20 min mean trip + payment, per [02](02-scale-estimates.md) — across many network
epochs, backgrounding, app kills. The rider must always see exactly the right screen; the client never owns truth.
This is [Case 19](../19.frontend-chat-app/04-deep-dives.md)'s message state machine
problem wearing a uniform — with higher stakes (money, physical safety).

**The machine** (each ride, client-side):

```
 browsing ──REQUEST(intent)──► requesting ──MATCHED──► matched(pickup)
     ▲                            │          │              │
     │                     NO_MATCH (t)    │ timeout/     │ARRIVED
     │                            ▼        │ CANCEL       ▼
     └──(user closes)──  request_failed   ▼            arrived ──BEGIN──► trip
                                 └──► cancelled ◄──CANCEL(from any)──┤
                                                                      │COMPLETED
                                                                      ▼
                                                                  completed ──receipt──► paid ──(rate/exit)──► done
```

Rules that make it a machine, not spaghetti:

- **Transitions are server-event- OR intent-driven, never timer-alone**: a client timer
  may *arm* a pending state (e.g. "match search > 5 min → show escape hatch, allow
  cancel") but the state change to `request_failed` requires a server event — client-side
  timeouts deciding money/safety states is how clients lie (their clock isn't the
  server's).
  (Exception: *user-visible* timeout UX like spinners is freely client-driven — display
  state is not machine state.)
- **From any state: `CANCEL`, safety escape, and network-loss (degraded overlay, deep
  dive 4) are handled** — the completeness property: every (state, event) pair has a
  defined next state; an unhandled pair is the "frozen ride UI" bug, so the machine is
  exhaustively defined as a table and audited as one.
- **Idempotent user intents**: every intent (REQUEST, CANCEL) carries a client UUID; the
  server dedups. Retry is safe; a retried cancel can't accidentally create two rides.

**Server reconciliation — the actual hard part:**

- Event channel carries monotonic per-ride **seq** (Case 03's wire protocol: at-least-once
  + client dedup + gap detection → on gap, hold UI (with "updating…" subtlety in
  money-adjacent states), request retransmit).
- **Snapshot endpoint** (`GET ride/current`, ~5–20 KB): authoritative state + last seq +
  current quote/route/driver. On app *reopen*, on socket rebuild, on a seq gap that can't
  be fast-filled: **snapshot, then stream, discarding events ≤ snapshot.seq** (idempotent
  fold). The snapshot is what makes cold restart ≤ 2 s from [01](01-requirements.md):
  two round trips, not replay.
- Optimistic UI confined to *display*: `requesting` animation, CANCEL button grays
  instantly. Irreversible-looking transitions (trip begin, payment) render only after
  server events. Trade-off: slightly slower-feeling match/cancel vs never showing a user
  "paid" before the server said so — a lie that's expensive to walk back.

**Trade-off summary (stated explicitly)**: Option A — server-push with no client machine
(server tells UI what to render, client is a canvas): wins simplicity, loses offline UX,
reconnect flicker, testability, and the ability to show *why* (drivers' states aren't
server-streamable every-second). Option B — full client machine + server truth (chosen):
wins deterministic UI across reconnects, offline-displayable states, testable as a pure
function; loses machine-completeness maintenance cost (an auditable table mitigates). We
pick B because client renders during degraded mode (deep dive 4) *require* a client
machine to render against.

**Failure modes:**

- *Out-of-order lifecycle events* (socket + retries): seq discipline + fold rules; no
  event may regress a state machine backwards except defined rollback events (e.g.
  `MATCHED` → later `MATCHED'` for a different driver = defined event, re-render driver
  card, not a crash).
- *App killed post-`BEGIN`, pre-persist*: snapshot on reopen repairs; persist the ride id +
  state at *every* transition (cheap local write, ~KB) so even without network on reopen
  the user sees correct last-known state labeled degraded.
- *Duplicate CANCEL on flaky network*: idempotency UUID server-side — the failure mode
  "car cancels, rider re-taps cancel, second cancel creates a new (wrong) cancel ticket"
  is the class of bug idempotency keys exist to kill (same discipline as Case 03's
  message identity).
- *Server event after terminal state* (`COMPLETED` then straggler `ARRIVED`): terminal
  states absorb late events into a "late event log" for support/reconciliation, not UI.

## 4. Degraded-network mode: polling, last-known-good UI

**The problem**: tunnels, subways, elevators, 4% battery. The client must keep a usable,
honest surface for 1–30 min without the socket, then recover without confusing the user
about the ride's state.

**Principles:**

- **Honesty about staleness**: every degradable surface carries a source-freshness
  indicator — "Car position: last seen 40 s ago", "Map offline: route saved". A stale
  surface *labeled* fresh is the killer bug: a rider staring at a fake car that isn't
  moving, or — worse — moving via dead-reckoning fiction past its budget (deep dive 2:
  during degraded mode, drift budget collapses to zero; render last-known, frozen, labeled).
- **No decisions in degraded mode**: the machine never transitions on absence. Timers
  don't cancel rides; lost network ≠ ride lost. Escape hatches (cancel, call driver)
  attempt and report failure, they don't fake success. Matches Case 19's offline-send
  discipline: queue intents, reveal delivery, never pretend.
- **The *one* thing that may auto-decide: display reversion** — e.g. hide "Driver arrived"
  banner back to en-route *only if* server later contradicts. Never the reverse.

**Mechanics:**

- **Event channel on degradation**: exponential backoff with jitter probing in the
  background (cheap, radio-friendly); the machine stores `last_seq`, so reconnect is
  gap-check → catch-up events, or snapshot if gap too large. (Same reconnect protocol
  shape as [Case 03](../03.whatsapp/04-deep-dives.md) §2 — resume + backoff — plus
  jitter, which matters when a subway platform exits with 200 riders at once: without
  jitter, reconnect storms are self-made DDoS.)
- **Position channel on degradation**: fall to long-poll / periodic fetch at 15–30 s
  (with the freshness label), or pause entirely in true no-network; on reconnect, one
  fresh snapshot burst (current car pos + route) supersedes all fiction.
- **Tile degradation**: map keeps last rendered tiles at higher zoom frozen (dim +
  "map offline"); viewport prefetch (deep dive 5) keeps a *route corridor* tileable
  offline — the corridor is what a rider in a car without signal needs (prefetch-near-need
  is client work — same shape as [Case 19's](../19.frontend-chat-app/04-deep-dives.md) media
  prefetch; the CDN's edge side is [Case 09](../09.cdn/04-deep-dives.md)).
- **Battery interaction**: low-power mode isn't degraded network, but it shares this
  machinery: same policy engine, same freshness labels, stricter budgets.

**Trade-offs (explicit)**: Option A — fully offline mode with client-side ETA prediction
dead-reckoning the trip: wins wow factor; loses lying about ride truth during exactly the
minutes the user most needs it. Option B — last-known-good + honest labels + probe
(chosen): wins trust and simplicity, loses the "map still moves" illusion during tunnels.
We pick B because in this product a wrong moving car is worse than a labeled stopped one.

**Failure modes:**

- *Network returns half-way* (socket up, gateway stale): freshness labels exist
  per-surface, so a half-broken reconnect shows itself; the reconciler's snapshot fold
  (deep dive 3) re-anchors even after long gaps.
- *Degraded → terminal event* (ride actually completed while offline): on reconnect, events
  replay to the terminal state; UI does a single legible transition ("Your ride ended —
  fare $X") rather than resuming into a dead trip screen.
- *Backoff storms after mass reconnects*: jittered backoff + server-side resume tokens
  (socket identity without full rehandshake) keep the storm bounded.
- *Degraded-mode drift into permanent staleness*: probe timer escalates (15 s → 30s poll →
  "connection lost — check network" full banner); degradation levels are themselves a
  small machine, not ad-hoc per-screen timeouts.

## 5. Map SDK choice & tile prefetching

**The problem**: the map engine is the client's heaviest dependency, and the tile layer
is the majority of app bytes. Choose once, live with it; prefetch so the map never
blocks on a fresh tile mid-interaction or — worse — mid-tunnel.

**SDK choice.** Options: (A) platform-native SDK (Google Maps SDK / MapKit), (B) embed
a vector-tile stack (Mapbox GL / MapLibre or self-run vector tiles), (C) raster tiles +
total DIY. Trade-offs:

- **A wins**: guaranteed basemap quality, geocoding/routing parity with the platform,
  near-zero maintenance. **Loses**: styling limits (your brand can't fully own the map's
  look), per-usage license economics at global scale, no control over update cadence,
  and the moving-layer API is designed around annotations (deep dive 1's Option-A trap).
- **B wins**: full style control (dark mode, branded cars as map layers, 3D), vector
  tiles = smaller + styleable + client-side cacheable semantics; GPU-backed markers
  available. **Loses**: significant rendering/UX parity work (labels colliding, gesture
  tuning), you inherit a tile-pipeline dependency.
- **C**: dismissed — you'd rebuild A's gesture/tiling machinery with fewer features.

**Chosen: B-flavored — vector-tile stack with a retained custom canvas/GL vehicle layer
on top — because the map's *look* and the vehicle layer's 60 fps budget are the product
(dark mode + branded + perf) while basemap data can be licensed.** The trade-off is real
support cost vs control; at this product's scale, the license economics and custom
styling win. (Same class of "own the hot path, buy the rest" as rendering in deep dive 1.)

**Tile prefetching** — the client's one lever to make the map feel free:

- **What to prefetch**: (1) concentric rings around user at zoom lattice z-1..z+1 (the
  pattern any maps app uses); (2) **route corridor**: polyline route + ~500 m buffer,
  tiles along it for z13–z17 — computed client-side from the route geometry the trip
  service already sent; (3) destination surroundings once the user opens search/booking.
- **How**: bounded, priority-ordered fetch queue (corridor > nearby > far), LRU eviction
  inside ~200 MB cache, requests through the platform HTTP cache so CDN cache-control
  does the eviction honesty for free ([Case 09](../09.cdn/04-deep-dives.md): you choose
  *what* to need; the CDN's TTL/edge behavior decides *where it comes from* — don't
  reimplement edge logic in the client).
- **Vector tiles specifically pay off here**: style-only changes (dark mode, night
  mode) re-render the same cached geometry with zero network ([Case 09](../09.cdn/README.md)'s
  cacheable-content reasoning, one level lower).
- **Never prefetch** beyond the active corridor/rings at full detail: low-zoom context
  tiles cover "what's over there" cheaply; over-fetching is exactly what kills battery
  and cache hit rates.

**Failure modes:**

- *SDK update churn / deprecation*: pin the vector stack behind the retained-layer
  interface (deep dive 1's target-list); the app's only contract with the SDK is
  camera + projection + tile loading, not its marker APIs.
- *Tile store thrash* (rider moving fast through prefetch queue): deprioritize behind
  current-viewport tiles; prefetching must never compete with what's on screen.
- *Route redraw mid-trip* (reroute event): corridor prefetch is incremental (diff old
  vs new corridor tile set) — re-fetching the whole corridor on every reroute is a
  classic battery bug.
- *Offline corridor gap*: a reroute received during degraded mode can't fetch new tiles →
  corridor-cache keeps the *old* route corridor renderable and labels "map data limited";
  the machine (deep dive 3) continues tracking truth; when network returns, corridor
  prefetch completes and the label drops.
