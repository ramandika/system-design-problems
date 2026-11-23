# Case 43 — Zoom

**The system**: real-time multiparty video conferencing — join a meeting, send one
audio+video stream, receive everyone else's, screen-share, and keep the call alive when the
network flips. Sounds like "live streaming plus a group chat"; the workload is the inverse of
[Case 01 — YouTube](../01.youtube/README.md): every participant is a *sender*, bandwidth is
symmetric, and the latency budget is ~100s of *milliseconds*, not seconds.

**Why it's hard**:

- Every participant **uploads in real time** — home/mobile upload is the scarcest link, and
  a 10-person mesh needs 9 uploads per person (*see estimate sheet*).
- The server must **relay, not transcode**: an MCU that decodes/re-encodes every stream
  costs server CPU no fleet survives at millions of concurrent participants — the SFU
  (selective forwarding unit) decision is the design's spine (deep dives §1).
- Quality is **receiver-driven**: the sender publishes layered simulcast; each receiver picks
  the layer it can afford, under bandwidth estimation every second (deep dives §2).
- **Audio is 10× smaller than video and yet the make-or-break track** — a meeting that drops
  to audio-only is a success; a meeting that drops to video-only is a failure (deep dives §3).
- Calls must **survive the network**: Wi-Fi→LTE handovers, 30-second tunnels, laptop sleep —
  reconnection is a designed state machine, not a retry (deep dives §5).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: SFU media plane (vs MCU vs mesh; kin of Discord's voice path —
[05.discord](../05.discord/README.md)), sender-side simulcast + receiver-driven layer
selection (the *live* sibling of player-side ABR,
[23.frontend-video-streaming](../23.frontend-video-streaming/README.md)), bandwidth estimation
loops, meeting-session state machines, large-meeting degrade-to-broadcast.

**Try it first**: before reading 03, answer: (1) a 500-person meeting — who receives which
streams, and what does the SFU egress look like? (2) a sender's uplink drops from 4 to 1 Mbps
mid-meeting — what changes, where is it decided? (3) the laptop sleeps 20 s on Wi-Fi, wakes
on LTE — what state must the client rebuild?
