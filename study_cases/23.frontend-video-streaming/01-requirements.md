# 01 — Requirements

## What the system is

The in-browser player of a large video streaming platform. It receives from the backend
(given): a manifest describing renditions of one title (HLS/DASH, per
[Case 01](../01.youtube/04-deep-dives.md)), and segment bytes from a CDN edge (given,
[Case 09](../09.cdn/04-deep-dives.md)). It must decide — continuously, on-device —
how much to download, ahead, at which quality, and never stall.

## Functional requirements

**F1. VOD playback.** Play a catalog title from an arbitrary seek position; supports
pause/resume, scrubbing (including scrubbing past buffered data), playback-rate control.

**F2. Live playback.** Play a live stream behind a live edge window; follows the edge
automatically, supports DVR seek within the availability window, and catches up after
stalls instead of drifting.

**F3. Quality adaptation (ABR).** Automatic rendition selection from the encoding ladder
based on measured throughput and buffer state; manual quality override ("144p…4K" picker)
that still keeps the session stable at the pinned quality.

**F4. Track selection.** Audio tracks (languages), subtitles/captions (side-loaded text
tracks, burned-in not required), and audio-video rendition pairs must stay in sync
(video and audio are separate renditions with separate buffers).

**F5. Player plumbing.** PiP (picture-in-picture), fullscreen, keyboard shortcuts, volume,
muted autoplay with unmute affordance, preview thumbnails on the scrub bar (separate low-
resolution sprite track), background/locked-screen audio continuation.

**F6. Telemetry.** Emits per-session playback events (startup time, stalls, quality
switches, errors, buffer levels) for QoE analytics; must be cheap and never block playback.

**F7. Resilience.** Recover from: segment fetch failure (retry, alternate rendition, or
skip), mid-stream manifest update (live), decoder/MSE quota errors, visibility changes,
network flap between Wi-Fi and cellular.

## Non-functional requirements

Numbers are rough design targets, not claimed industry statistics; the low end is what a
major player publicly shows, the high end is where the design should aim.

| Dimension | Target | Notes |
|---|---|---|
| Startup (time to first frame) | p50 < 1s, p95 < 2.5s (design target) | dominates abandonment; trades against first-frame quality |
| Rebuffer ratio | < 0.5% of watch time under normal networks (design target) | stall seconds / watch seconds |
| Quality switch latency | up-switch ≤ 2 segment durations after link improves; down-switch ≤ 1 segment | asymmetric on purpose (see 04 §1) |
| Live edge lag | classic live ≤ 2–3 segment durations; low-latency mode ≤ ~5s (design target) | see 04 §3 |
| Memory (player, per tab) | budget ~50–100 MB (estimate) | buffer + decoded pipeline; see 02 |
| Battery/data | no fetch when buffer target met and tab hidden w/o audio; metered-connection mode lowers buffer targets | explicit data saver |
| Compatibility | Chromium/Firefox via MSE pipeline; Safari/iOS primarily via native HLS support | see 04 §5 |
| Accessibility | caption rendering, focusable controls, prefers-reduced-motion for UI animations | |

## Scope decisions

**In scope**: everything client-side from manifest parse to pixels — ABR logic, buffer
strategy, segment scheduler, live-edge logic, player state machine, MSE/worker pipeline,
telemetry emission, visibility/background handling.

**Explicitly out of scope**:

- **Encoding backend**: rendition ladder construction, per-title encoding, AV1 decisions —
  [Case 01](../01.youtube/04-deep-dives.md) already covers the ladder; this player
  consumes its output.
- **CDN and origin**: edge topology, cache keys, shielding — [Case 09](../09.cdn/04-deep-dives.md);
  here only the client's cache-friendly request behavior matters (byte-range, retry
  friendliness, not busting caches it shares with everyone else).
- **DRM internals**: the CDM/Widevine contract is treated as a black box the player must
  initialize before first append; key-server design is out of scope.
- **Recommendation/feed UI**, comments, upload — other surfaces, not the player.
- **Native mobile SDKs and Smart-TV apps**: we design the web player; the same state
  machine and ABR principles port, but platform plumbing differs.
- **Offline download/expiry**: a separate feature with its own license model; out.
- **Analytics backend** (aggregation, dashboards): we emit events; we don't build the
  collector (cf. [Case 19](../19.frontend-chat-app/04-deep-dives.md) on client telemetry
  batching, if present).
