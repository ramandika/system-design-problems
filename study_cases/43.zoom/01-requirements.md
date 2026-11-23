# 01 — Requirements

## Functional

Core v1:

- **Meetings**: create/schedule/join by ID or link; participant list, mute, camera on/off,
  remove-from-meeting, waiting room (admission control), lock-meeting.
- **Media**: one audio stream per participant (always), optional one video stream, one
  *screen-share* stream (a participant shares screen *or* camera-priority applies — both may
  publish but clients prioritize content). Gallery and active-speaker views.
- **Simulcast**: each video sender publishes ~3 encodings (top/middle/low layer) so the
  relay can give every receiver the best layer its bandwidth allows.
- **Screen share / content channel**: full-resolution content stream (5–30 fps), with
  annotations optional.
- **Meeting controls state**: mute/unmute, raise hand, host moderation — synchronized
  meeting state that joins before media and must be re-synced on reconnect.
- **Recording** (opt-in): server-side mixed recording + chat log, stored to object storage.
- **Chat**: in-meeting text (light; see [30.facebook-messenger](../30.facebook-messenger/README.md)
  for messaging done properly).
- **Dial-in / dial-out (PSTN)**: a phone leg joins as a send-audio-only participant.

Out of scope v1: webinar-scale broadcast economics (treated as a degrade path in deep dives
§6), transcription/ML features, virtual backgrounds' client ML, calendar integration
(→ [21.frontend-calendar](../21.frontend-calendar/README.md)).

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Local render latency | camera/mic → own screen < ~50 ms | client-side pipeline, no server hop for self-view |
| Remote media latency | one-way audio < ~100 ms, video < ~300 ms (est. budget) | relay-only media path; no server-side processing in the hot path |
| Jitter tolerance | absorb 30–100 ms network jitter without audio stall | client jitter buffers + adaptive playout (deep dives §3) |
| Loss tolerance | audio intelligible at 5–10% packet loss | Opus FEC/redundancy, audio prioritized over video (deep dives §3) |
| Join latency | p90 < ~2 s from click to media | pre-warmed connections, relay allocation at join |
| Reconnect | media re-established < ~3 s after network return | session persists server-side; deep dives §5 |
| Availability shape | signaling: very high; media: degrade (drop video layers first); recording: eventually | the *media plane degrades in quality, not in participation* |

## Scope decisions that shape everything

1. **The server relays, it does not transcode** (SFU over MCU): every server-side media op
   multiplies fleet cost by concurrent participants; forwarding is O(bytes), transcoding is
   O(decode+encode) per stream. The one exception (composite recordings) is off the
   interactive path (deep dives §1).
2. **Quality adaptation lives on the receiver**, sender-side layers pre-encode the choices:
   the sender cannot know 9 receivers' bandwidths; the receivers each know their one uplink.
3. **Audio is engineered separately from video** throughout the stack (encoding, buffers,
   priority, loss handling) — the two tracks fail at different loss/jitter thresholds.
