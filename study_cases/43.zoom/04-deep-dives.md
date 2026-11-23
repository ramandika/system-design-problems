# 04 — Deep dives

## 1. Mesh vs MCU vs SFU (THE dive)

Three ways to connect N participants:

**Option A — mesh (P2P: every participant sends directly to every other)**:
- wins: no media servers at all; lowest latency; zero platform egress cost.
- loses: every participant needs (N−1) upstream streams — at N=10 (our mean meeting) that is
  ~9 × 3 Mbps ≈ **27 Mbps of upload per video sender**, and consumer upload lines are the
  scarcest resource in the whole system; encoding 9 separate streams on the laptop CPU; no
  central point for recording or admission.

**Option B — MCU (multi-point conference unit: server decodes all N streams, composites, and
sends each participant one re-encoded stream)**:
- wins: single downstream per participant (dial-up-era friendly), uniform output quality, a
  natural place to mix audio and record.
- loses: server does decode+compose+encode per meeting — at our peak (~420K live meetings,
  est.) the transcode fleet is astronomical; encoder latency (~100s of ms) added to
  everyone's glass; every participant's stream quality is capped by the composite.

**Option C — SFU (selective forwarding unit: server routes packets, never decodes)**:
- wins: server work is O(packet forwarding) — the fleet cost is line-rate egress
  (~40 Tbps peak, ~3K nodes at ~15 Gbps, est. sheet) instead of transcoding per stream; no
  added encoder latency; each receiver picks its own quality from the sender's published
  layers (deep dive 2); the sender encodes a *fixed* small number of layers (3), not N.
- loses: clients must publish multiple simulcast layers (upload cost ~3 Mbps vs 1.5) and run
  per-receiver adaptation logic; weak clients that can't decode more than one stream get no
  server-side help; recording needs a separate composite pipeline anyway.

**Chosen: Option C**, with A acceptable for ≤ ~4-person calls as an optimization and B
surviving only as the offline recording path. The decisive arithmetic: server cost per
meeting under MCU scales with N × transcode, under SFU with N × forward; at our
15M meetings/day with mean size 10, only forwarding fits a fleet. Kin:
[05.discord](../05.discord/README.md) made the same choice for voice; Zoom-class video
products converged on SFU for the same reason.

**Failure mode**: SFU node failure drops every meeting it hosts. Mitigate: allocator caps
sessions per node, regional overprovisioning, and the client's reconnect state machine
(deep dive 5) makes node loss an 8-second blur, not a dropped call.

## 2. Bandwidth estimation + simulcast layer selection

The sender cannot know its 9+ receivers' bandwidths; each receiver knows exactly one uplink
(itself). Two mechanisms cooperate:

- **Sender-side simulcast**: publish 3 fixed layers — ~150 kbps (thumbnail), ~600 kbps
  (half-res), ~2.5 Mbps (full 720p-ish; est. sheet). Cost: fixed ~3 Mbps upstream whenever
  video is on, regardless of who watches.
- **Receiver-driven selection**: each receiver runs a bandwidth estimator (loss + delay
  gradient, GCC-style) and asks the SFU for the highest layer estimated to fit, re-visiting
  the choice ~every few seconds.

**Option A — single stream per sender, server transcodes per receiver (MCU-style
personalization)**: perfect per-viewer quality; per-viewer server transcode cost, and this is
the fleet-killer of dive 1. **Option B — receiver-driven from simulcast (chosen)**: near-zero
server media cost, adaptation is client-correctness work; the trade is that the *sender's*
published layer set bounds quality, and a receiver below the lowest tier gets nothing unless
the product defines a floor (we keep an audio-only floor).

**Failure modes**: estimator oscillation → every receiver flap-switching layers creates
keyframe request storms at the SFU → add hysteresis (hold a layer change for N seconds; bias
downward faster than upward). All-receivers-in-a-meeting estimate upward simultaneously after
a network recovery → SFU egress overshoot → relay answers layer-up requests progressively
(top layer is best-effort). Estimator starved for data during a mute-heavy period → keep
probing on audio only.

## 3. Audio: small stream, big discipline

Audio is ~40 kbps mean vs ~3 Mbps for video — ~1.3% of the bytes — but it is the track users
cannot tolerate losing (a meeting that degrades to audio-only is a success; video-only, a
failure). Consequences, per layer of the stack:

- **Encoding**: Opus with silence/DTX (mute + pauses cut the already-tiny stream ~10× more),
  and FEC/redundancy so 5–10% packet loss stays intelligible (est. requirement in 01).
- **Playout**: adaptive jitter buffer; the buffer absorbs reordering/bursts at the cost of
  ~tens of ms of latency; it shrinks when the network is calm.
- **Priority**: at any congestion point, audio is marked to survive first — an SFU under
  pressure drops video layers before audio, and any receiver whose bandwidth estimate
  collapses keeps audio (deep dive 2's floor).
- **Mute as dominant state**: in a 10-person meeting, typically 1–2 people speak at once;
  audio duty cycle on the fleet is a fraction of video's even though "audio participants" is
  100%. This is why the est. sheet's audio ingress (~0.24 Tbps peak) is noise next to video
  (~18 Tbps) despite equal participant counts.

**Trade-off stated**: Option A — one unified transport with per-stream QoS tagging, simpler
client. Option B — audio engineered as its own track (chosen): two codepaths to maintain, in
exchange for the failure asymmetry users actually experience.

## 4. Screen share: the content channel

A screen share is not "video of a screen": typing text needs resolution (a 5-fps stream at
full res is readable; scrolling/animation needs ~30 fps; lossy compression on text produces
smearing). So the content stream is encoded with different parameters than the camera
(hi-res, adjustable fps, lossless-preferred on static regions) and published as a separate
track alongside camera+audio.

- **Option A — treat screen share as just another video stream**: one pipeline; loses
  readability (text needs ~4× the pixels of a face for the same perceived quality) and the
  fps policy differs by content type.
- **Option B — dedicated content track (chosen)**: a second publishing path and a second
  subscription decision; the payoff is the meeting's media shape changes — everyone subscribes
  to *content* at higher priority than gallery tiles (content is the focal stream; est.
  bandwidth: full-res content ~1–3 Mbps replaces ~3.4 Mbps of a receiver's downstream budget).

**Failure modes**: share fails when a window is minimized (capture API) — fall back to "share
stopped" UX, never a black rectangle; a 4K monitor shared at full res = 4× the sender's
layers — cap content top layer by sender uplink (estimator-gated, same machinery as dive 2);
animated content at text-tuned fps → client-side content classifier switches fps (smearing
worst case bounded to the low-fps tier).

## 5. Meeting session state + reconnection

State machine (per participant): `INIT → AUTH → ADMITTED (waiting-room gate) → IN_MEETING →
RECONNECTING → IN_MEETING | EXITED`. Signaling state lives server-side and survives client
network flaps; media state must be rebuilt quickly.

The reconnect timeline (the case that happens daily): Wi-Fi→LTE handover, tunnel blink, laptop
sleep 20 s. Design:

- **Server keeps the participant's session** (roster slot, subscriptions) for a grace window
  (~30–60 s): the other participants see "reconnecting", not "left".
- **Client re-establishes transport**: new ICE against the same relay; relay reassignment if
  the region changed (network switch can move you); if UDP is blocked on the new network,
  TURN fallback (est. sheet: 2× egress cost on that path vs a disconnected user).
- **Media resync**: keyframe request on the video layers (the encoder only forwards
  decodable GOP boundaries); audio self-synchronizes (short frames).
- **State resync**: roster, mute states, controls, chat catch-up from the meeting service —
  one idempotent "get full state" call keyed by session token; duplicate JOIN messages after
  a flap are deduped server-side.

**Option A — client re-joins from scratch after any flap** (simpler, "just refresh"):
loses: a 10-second sleep causes full re-admission (waiting room again!), lost chat, roster
flicker for 9 other people. **Option B — persistent session + targeted resync (chosen)**:
harder state machine, and the state store must tolerate client-visible staleness during the
grace window. Chosen B; a call that survives a subway ride is the product.

## 6. Large meetings and webinars: degrade toward broadcast

At mean meeting size 10 the SFU story is easy. At 500–1,000+ participants, two things scale
badly if unattended: control-plane chatter (everyone sees everyone) and gallery subscriptions.

**Trade-off stated**: Option A — stay fully interactive at any N (everyone publishes, full
roster, full subscription graph) wins equality, but the state machine and moderation become
the bottleneck before bandwidth does. Option B — panel-publish / viewer-receive (chosen):
bounded per-viewer downstream regardless of N, but the silent majority loses its video path
(unmute/raise-hand or panel status is the price).

- **Viewer mode**: participants beyond a small "panel" stop publishing audio/video by default
  (host control) — the meeting becomes send-few / receive-many, i.e. structurally a live
  stream with interactive edges. Receiver subscription is capped (content stream + active
  speakers), so per-viewer downstream stays ~3–4 Mbps regardless of N (est. sheet's
  per-receiver math holds because *subscription count*, not meeting size, is the variable).
- **SFU load**: egress for a 1,000-viewer webinar ≈ 1,000 × ~3 Mbps ≈ **3 Gbps per meeting**
  — one strong node, or a small multi-node relay tree for the meeting; contrast the *control*
  fanout if all 1,000 were full participants (roster+raise-hand+moderation state becomes the
  bottleneck before bandwidth does).
- **The hard part at scale is moderation, not media**: admission control (waiting rooms,
  screen-share permissions, removing participants) is a write-through state machine at N=
  1,000 with hostile users — rate-limit control events per participant, gate stream
  publication behind host permission.

Beyond ~5–10K viewers the product is broadcasting; past that point stop pretending it's a
meeting — prepositioned CDN delivery does the rest
([01.youtube](../01.youtube/README.md), [31.netflix](../31.netflix/README.md)); the SFU path
remains only for the interactive panel.
