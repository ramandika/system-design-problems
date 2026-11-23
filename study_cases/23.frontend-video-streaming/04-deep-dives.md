# 04 — Deep dives

## 1. ABR client logic: throughput estimation vs buffer-based switching (THE dive)

The client must pick a rendition from the ladder (given per
[Case 01 §2](../01.youtube/04-deep-dives.md)) with no ground truth about the network.
Two families of rules, best combined:

**Throughput estimation.** The only probe the client has is its own segment download:
`throughput ≈ segment_bytes / fetch_time`. Estimators in increasing care:

- **EWMA** (exponentially weighted moving average, two time constants: fast ~3 samples,
  slow ~8) — reacts to real changes but damps single-segment noise (a segment that
  finished over an idle connection is not the link's speed).
- **Harmonic mean over recent samples** — punishes high variance downward (harmonic mean
  ≈ min-like); safer than arithmetic mean because stall cost is asymmetric.
- **Self-competition correction (critical).** While the player download saturates the
  link, the measured throughput *is* the player's own consumption, not capacity — you
  will measure roughly `bitrate × safety_factor`, and naive estimation then converges to
  believing the link is exactly what you chose to consume. Real systems either probe with
  a deliberately higher rung, use in-browser resource-timing hints for co-tenants, or
  keep a fixed safety factor (~1.5–2×) and never take a rung above
  `estimate / 1.8` (estimate factor; see 02 for the arithmetic tie-in).

**Buffer-based switching (BOLA-style).** Measure seconds of forward buffer instead of
Mbps. If buffer ≥ target (say 18–30s), request the highest rendition the *link sustained
recently*; if buffer < danger threshold, drop a rung or two immediately — buffer is
time-to-stall, the only quantity the user actually experiences.

**The trade-off.** Option A (pure throughput): wins responsiveness to true capacity
changes, loses stability — it oscillates on variable links because each switch changes
the next measurement (self-fulfilling). Option B (pure buffer): wins stability and
stall-avoidance, loses quality ceiling — on a fast link it takes a while for buffer to
fill before it dares to up-switch, and it can't anticipate a cliff that buffer hasn't
seen yet. **Pick hybrid: throughput sets the ceiling, buffer picks within it.** Down-switch
criteria are biased to trigger instantly (one bad buffer reading), up-switch criteria are
biased to lag (need buffer ≥ target AND throughput headroom for 2 consecutive samples) —
the user forgives minutes at lower quality, not two seconds of spinner.

- **QoE trade-off made explicit**: startup favors low first rung (quality after start,
  not at start — abandonment is at the spinner); mid-session favors buffer stability;
  the panic rule (buffer < playhead margin) caps the damage at "down-switch now" so the
  worst case is a lower render, not a stall.
- **Failure modes**: (a) estimate collapse at session start (no samples) → seed with a
  conservative default and the first segment measured at low rung; (b) oscillation on
  cellular → hysteresis band + minimum dwell time between switches; (c) pinned-user
  override → cap by throughput only if stalling, else honor the pin (quality choice is
  the user's, stall prevention is ours).

## 2. Buffer management & segment preloading: when to fetch ahead, when to stop

**The buffer is the player's only shock absorber.** Design:

- **Target forward buffer = 30s (VOD, estimate)**: 2 MB at avg quality (02) — cheap; and
  it's what converts a link dip into a non-event. Below ~10s forward, ABR's down-switch
  criteria arm; the state machine itself goes STALLED only at literal buffer exhaustion —
  playhead hits `buffered` end (04 §4's data-driven rule, never a buffer-level threshold,
  or the STALL↔PLAYING hysteresis collapses to zero).
- **Fetch-ahead rule**: fetch next segment while `forward_buffer + in_flight < target`,
  one segment at a time in playback order (the scheduler's queue is strictly ordered by
  distance from playhead). Never parallel-fetch segments in playback order: out-of-order
  completion wastes bytes (later segment over a fast connection arrives before an early
  one you're re-downloading slower) and corrupts the throughput signal.
- **When to stop**: (a) target reached — hard stop, save the user's data cap and the
  edge's load (aggregated: 20M clients fetching "one extra segment each" is 20M × 2 MB =
  4.0e7 MB ≈ 40 TB per extra segment per pass — 02's segment size and concurrency); (b) live mode: cap is tiny (04 §3);
  (c) **hidden tab without audio**: suspend fetches entirely (04 §6).
- **Backward buffer**: already-played seconds stay in the SourceBuffer until quota
  pressure or a long seek; `remove()` trailing ranges beyond ~30s backward (estimate) —
  instant rewind is user-visible value per byte until the budget is needed.
- **Startup choice**: fetch init + first 2 segments of a *low* rung in parallel with the
  manifest when possible; the first segments are the only bytes on the critical path, so
  they go out immediately, not after ABR has samples. This costs one wasted low-quality
  segment (~0.2 MB — 4 s at the 240p@0.4 Mbps rung, 02's ladder floor) against the
  startup metric — **Option A (start low, switch up
  from samples) wins the p50 first-frame; Option B (wait for ABR) wins total bytes; pick
  A because startup dominates abandonment**.
- **Failure modes**: MSE `QuotaExceededError` (Safari/family-specific limits) → treat as
  "trim backward buffer, pause appends until buffer drains", never fatal; prefetch of
  next-episode or preview sprites must be marked abortable so a seek cancels in-flight
  low-priority requests; a stalled fetch (no bytes, no error yet) needs an explicit
  timeout shorter than `buffer-left × 2`, else the player stalls dead instead of
  down-switching.

## 3. Low-latency live vs VOD: two cadences, one machine

**Classic VOD/segments**: 4–6s segments, buffer target 30s, no manifest refresh during
playback (the segment timeline is static). Simple, edge-friendly, ~2–3 segment lag from
real time is meaningless for VOD.

**Low-latency live (CMAF/LL-DASH/LL-HLS, chunked transfer)**: segments are still
declared, but the encoder pushes *chunks* (parts) of ~0.5–1s as they're produced; the
client fetches not-yet-complete parts over a held-open HTTP response and appends as
parts arrive.

- **What the client must change**: buffer target drops to ~3–8s (else "live" is a lie);
  part-based fetch cadence (`concurrency ÷ 1s ≈ 1M req/s for a 1M-viewer event`, 02) —
  a hot-key problem whose fix lives on the edge ([Case 09 §5](../09.cdn/04-deep-dives.md)),
  but the client's contribution is short cacheable part URLs and abandoning long-poll
  requests on seek; **catch-up playback rate** (play at 1.05–1.1×) for the seconds after
  a stall instead of jumping to edge (jump = visible glitch, catch-up = invisible);
  live-edge tracking from the manifest's program-date-time, not local clock skew.
- **The trade-off.** Option A (fat buffer everywhere): wins robustness — a 30s buffer
  survives almost any part delay — loses the product (a 30s-behind "live" stream is
  unusable for sports/chat-along content, and it doubles the encoding→viewing latency
  the product is explicitly selling). Option B (thin buffer): wins latency, loses margin —
  any part-delay > buffer becomes a visible stall on exactly the highest-concurrency
  streams. **Pick thin buffer with catch-up rate + panic down-switch (04 §1's rule
  becomes "drop to lowest live rung") because latency is the product here; accept higher
  stall risk and compensate with the aggressive side of the asymmetric switch bias.**
- **Failure modes**: part fetch hanging (server holds but produces nothing) → client
  timeout ≈ 1 part duration and move to next part; manifest updates lagging the encoder
  → availability window math must never "rewind" the playhead on a stale manifest;
  DVR seek near the window start → clamp against the server's declared window, re-derive
  edge position on return-to-live.

## 4. The player state machine with network flap

```
EMPTY ──manifest ok──► READY ──buffer≥2 seg──► CAN_PLAY ──play()──► PLAYING
  │                      │                        ▲ pause            │
  │ manifest error       │ seek (unbuffered)      │                  │ playhead hits
  ▼                      ▼                        │                  ▼ buffered end
ERROR              SEEKING ──appended──► READY ◄──┘              STALLED
                          │                                       │  buffer ≥ resume margin
                          └───────────────────────────────────────┘  (auto-resume)
```

Rules that make it survive real networks:

- **One transition owner** (the player core, 03): UI, visibility events and controller
  callbacks *request*; the machine decides. The stall path is entirely data-driven —
  playhead vs `video.buffered` end — never a timer, because timers are exactly what the
  background tab throttles (and a throttle-delayed timer would mis-declare STALLED).
- **Hysteresis everywhere.** STALL→PLAYING requires buffer ≥ ~2s (margin, estimate);
  PLAYING→STALLED only at literal buffer exhaustion. Without the gap, a marginal link
  flips the UI between spinner and picture every 500ms — the same thrash bug as an ABR
  without dwell time (04 §1), at state level instead of quality level.
- **Network flap (Wi-Fi↔cellular) sequence**: connection type changes → in-flight segment
  likely fails or times out → scheduler retries *on the new connection* with a fresh
  throughput slate (flush EWMA: samples from the old link are lies) → ABR re-seeds
  conservative. Do not "fail" the session on connection-change: the old samples' evidence
  expired, not the network.
- **Fatal vs recoverable errors**: manifest 404 / empty ladder / MSE init failure =
  ERROR (terminal, surface an upgrade-able message); single segment failure = retry ×3
  with backoff, then try the same segment at a lower rung (renditions are alternative
  encodings of the same timeline — Case 01 §2), then skip-one-segment (visible as a
  ≤4s content skip; strictly better than a stall). **Trade-off**: skip corrupts content
  vs stall interrupts it — pick skip after retries because a 4s gap at rung-low beats an
  unbounded spinner in every public QoE ranking we know of (qualitative).
- **Cross-link**: this is the client-twin of the message state machine and its
  network-flap discipline in [Case 19 §1–§4](../19.frontend-chat-app/04-deep-dives.md) —
  same pattern (in-flight states owned by a store, optimistic UI reconciled by data),
  different currency (seconds of buffer instead of delivery receipts).

## 5. MSE + fetch/worker pipeline vs native HLS

**Option A — native HLS** (`<video src=playlist.m3u8>`): the browser (Safari, iOS;
increasingly others) owns fetch, demux, ABR, and buffer. Wins: zero pipeline code,
system decoder and battery behavior, the only practical path on iOS web where MSE
appending of raw media is restricted/absent in practice. Loses: no custom ABR, no
byte-level scheduler, no visibility into stalls beyond `waiting` events — the case's core
problems (04 §1–§3) are unsolvable inside it.

**Option B — MSE + fetch/worker**: the player owns everything upstream of SourceBuffers.
Wins: full ABR/scheduler control (= every dive in this file), cross-browser uniformity,
testable-bytes pipeline. Loses: you now own demux, retiming, quality-switch continuity,
quota errors, and battery behavior you cannot fully control.

**Pick: MSE pipeline everywhere it exists, native HLS exactly where it doesn't (iOS
Safari)**, with the *same* state machine, ABR policy and telemetry in both (the native
path forwards browser events into the same controller; its ABR is the browser's, our
telemetry still measures the outcome). **Trade-off accepted**: two pipelines → one policy
layer split by an adapter; the alternative — a single MSE-only player — yields a
second-class iOS experience, and iOS web traffic is too large to second-class.

**Worker demux in the pipeline**: parse container boxes in a Web Worker, return only MSE-
ready fragments via transferable buffers. Trade-off: one more thread + structured-clone
boundary (cheap for a handful of MB/s segments) vs main-thread parse jank on controls
and scrubbing — pick worker; parse cost scales with bitrate while UI responsiveness is a
fixed p99 requirement.

- **Failure modes**: codec-support drift (new codecs on some browsers) → the manifest's
  codec strings gate appends; fail at READY with a clear error, not mid-playback.
  Quota errors (04 §2). Worker crash → fallback parse on main thread for short segments
  is the emergency hatch, logged loudly.

## 6. Background playback & tab throttling

The browser is a hostile co-tenant exactly when the user multitasks, which is when video
is playing:

- **What the browser throttles in a hidden tab**: `setTimeout`/`setInterval` clamped to
  ~1/s (heavily), `requestAnimationFrame` stopped, timers inside service workers
  different. What keeps running: media playback itself, `fetch`/network, MediaSource
  appends, WebAudio, `video.timeupdate`.
- **Design rules that follow**:
  1. **Drive nothing critical off timers.** Stall detection = playhead vs buffered (04 §4)
     — event-driven, throttling-immune. Fetch scheduling hangs off `timeupdate`/append
     callbacks, not `setInterval`.
  2. **Audio-first background continuity**: audio track keeps appending at the *audio*
     cadence (smaller appends, cheaper); with screen off/hidden, keep the ABR decision
     on audio bitrate only — downloading video bytes for a screen nobody sees fails
     both the data-cap and battery requirements, and the scheduler marks video appends
     suspended on `visibilitychange` when only audio is rendered.
  3. **Visible-again recovery**: on `visibilitychange → visible`, resync: read real
     playhead/buffer state (which kept moving), flush EWMA (network conditions may have
     changed while nobody was measuring), resume video appends, recompute ABR. Expect a
     first-segment latency burst — budget for it, cap it, don't stall.
  4. **Picture-in-picture / locked-screen** are product surfaces, not edge cases: PiP
     keeps video rendering outside the tab (throttling doesn't apply to the PiP window);
     lock-screen control = the platform media session (metadata + play/pause handlers),
     which must be wired to the state machine, not to the UI layer — otherwise the OS
     button and the player drift apart.
- **Trade-off**: Option A (keep fetching video in hidden tabs at full cadence) wins
  instant visual return when the user comes back — nothing to reload; loses battery and
  data (the two complaints app-store reviews are made of) and, in aggregate, wastes 02's
  edge arithmetic on unwatched pixels. Option B (suspend video appends when hidden) wins
  the resource budget, costs a visible re-buffer burst on return. **Pick B for hidden
  tab, full pipeline for PiP/audible+visible surfaces, because hidden-means-not-watched
  is right more often than not, and the return-burst is amortizable (fetch early, buffer
  low)** — the escape hatch is audio: if audio is playing, the session isn't idle, and
  the buffer floor is the audio cadence, not zero.
- **Failure modes**: media session double-control (OS pause + in-page pause fighting →
  route both through the state machine, 03); WebAudio context suspended by autoplay
  policy on some browsers in background → degrade to `video.muted` + element playback
  path; throttled `timeupdate` making the scrub bar look frozen → UI-only concern, never
  let UI polling drive the core (it can be wrong; the machine can't).
