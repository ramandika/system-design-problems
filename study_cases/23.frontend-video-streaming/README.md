# Case 23 — Frontend System Design: Video Streaming Platform

**The system**: the *client* of a YouTube/Netflix-like video player — everything that runs
in the browser tab when you press play. The encoding backend is a given: a ladder of
adaptively-encoded renditions per title (how that ladder is built is
[Case 01](../01.youtube/04-deep-dives.md)). Delivery is a given: a CDN edge that serves the
bytes ([Case 09](../09.cdn/04-deep-dives.md)). This case designs what sits between the two:
the ABR player, its buffer, its state machine, and its fight with the browser's own
schedulers.

**Why it's hard** (all client problems):

- **The client knows the least and owns the outcome.** It has no a-priori knowledge of link
  capacity, must infer it from the same bytes it is about to consume, and every wrong guess
  is a stall or a wasted download — visible to the user within seconds.
- **Throughput estimation is a race with playback.** You download segments on the same
  connection your playback consumes; the estimate is always stale, partly self-induced
  (your own download competes with your last one), and the punishment for underestimating
  is buffering, while the punishment for overestimating is worse quality for minutes.
- **Two regimes, one player.** VOD wants a fat lookahead buffer (this design targets 30s — each extra
  buffered second is nearly free quality, see 02);
  low-latency live wants a thin one (seconds, or the "live" badge lies). Same machine,
  opposite buffer management.
- **The browser is an adversarial co-tenant.** Background tabs get timers throttled,
  `requestAnimationFrame` stops, the network may hand back nothing; the player must keep
  audio going and not stall when the user switches tabs with a video in a corner.
- **Startup is the metric that hides all others.** Users abandon before first frame; every
  design decision (first-segment quality, buffer fill target, parallel manifest+segment
  fetch) trades against it.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: ABR decision logic (throughput estimation vs buffer-based
rules), segment scheduling and preloading, low-latency live client cadence, player state
machines with hysteresis, MSE + worker demux pipeline, playing nice with tab throttling
and PiP/background audio. These are the client-side twins of the encoding-ladder patterns
in [Case 01](../01.youtube/04-deep-dives.md) and the edge-delivery patterns in
[Case 09](../09.cdn/04-deep-dives.md) — the ladder defines *what* qualities exist; this
case defines *which one plays right now*.

**Try it first**: before reading 03/04, answer: (1) how do you estimate link throughput
when your own segment download is the only probe you have, and what does co-competition
do to the estimate? (2) why does low-latency live forbid the 60s VOD buffer, and what
does the client lose in exchange? (3) the user switches tabs for 10 minutes — what stops,
what must not stop, and what do you have to re-establish when they come back?

**Primary sources**: none vendor-specific — this is a synthesis case. Behavior described
(standardized formats: HLS, DASH, CMAF, MSE) follows published specs; scale numbers are
labeled estimates; publicly reported figures are marked as such.
