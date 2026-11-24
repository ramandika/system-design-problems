# Real-time transport across the case catalog

Where WebSocket/WebRTC/SSE/long-polling, HTTP/2·3/QUIC, and segmented media delivery
(HLS/DASH/ABR) appear in the 50 study cases, what role they play, and how deep each goes.
Companion to [CONCEPTS.md](CONCEPTS.md) — its nearest row is "Primary-connection gateway
(WebSocket/long-lived)", Tier 3, 8/43 backend cases; this doc uses a much wider scope
(it also counts media delivery transports and client-side cases), so the counts below are
**not** comparable to CONCEPTS.md's union-concept counts (methodology note at the end).

Headline: **the catalog has two independent real-time shapes, and almost never mixes them.**
One is the *connection plane*: stateful gateway fleets that hold persistent sockets or
SSE streams and push events to clients (chat, ride tracking, charging, contests). The other
is the *delivery plane*: stateless HTTP request/response over a CDN carrying segmented,
adaptive-bitrate media (video, music). Zoom and Discord are the only cases that run both —
signaling and chat on the connection plane, media on raw UDP/SRTP outside HTTP entirely.
Where a case picks the wrong plane for its problem, the trade-off is spelled out and the
wrong choice is rejected (Spotify rejecting HLS for audio, Uber Eats rejecting polling,
LeetCode rejecting both polling and WebSocket).

- Cases mentioning a real-time transport: **22 / 50** (plus 2 boundary adjacencies, 21 and 30, noted below)
- Full case — transport is the product's core problem: **9** (03, 05, 43, 17, 01, 31, 23, 19, 24)
- Designed architectural component: **9** (08, 27, 38, 28, 29, 09, 46, 30, 22)
- Brief mention / historical reference only: **4** (02, 20, 33, 45)
- Explicit transport alternatives weighed and rejected: **4** (32 rejects HLS/DASH; 29 and 38 reject polling; 38 rejects WebSocket; 24 excludes WebRTC from scope)

False-positive filtering (counted nowhere above): case-insensitive `"QUIC"` matched
"quick/quickly" in 40.tinyurl, 41.web-crawler and one line of 28.uber (28 qualifies through
its socket tier regardless); `"TURN"` matched the verb in 26.google-maps (+6 google-maps
hits), 16.dalle ("multi-turn"), 18.sora, 37.ticketmaster, 36.airbnb ("write turn"),
21.frontend-calendar and 48.google-search; `"DASH"` matched "dashboard" in 09.cdn,
31.netflix and ~9 other cases; substring `"ICE"` matched "service/office/notice" in
11.zookeeper, 48.google-search and 26.google-maps. `"SSE"` substring matched little but
needed word-boundary greps to avoid noise.

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [03.whatsapp](03.whatsapp/README.md) | The connection fleet is the case: 100M live sockets, push delivery | Full case |
| [05.discord](05.discord/README.md) | WebSocket gateway + presence + WebRTC SFU voice planes | Full case |
| [43.zoom](43.zoom/README.md) | The case itself: SFU media plane, ICE/DTLS/SRTP, TURN fallback, QUIC signaling | Full case |
| [17.elevenlabs](17.elevenlabs/README.md) | WebSocket audio transport + adaptive jitter buffer as a named deep dive | Full case |
| [01.youtube](01.youtube/README.md) | Encoding-ladder + segment delivery is the data plane | Full case |
| [31.netflix](31.netflix/README.md) | ABR playback + own CDN fleet as the entire data plane | Full case |
| [23.frontend-video-streaming](23.frontend-video-streaming/README.md) | The case itself: HLS/DASH, MSE vs native HLS, LL-live | Full case (client) |
| [19.frontend-chat-app](19.frontend-chat-app/README.md) | WebSocket live channel + HTTPS backfill split | Full case (client) |
| [24.mobile-chat-app](24.mobile-chat-app/README.md) | Foreground socket policy; WebRTC explicitly out of scope | Full case (client) |
| [08.chatgpt](08.chatgpt/README.md) | SSE/WebSocket edge gateway terminating ~100M connections | Core component |
| [27.ev-charging-network](27.ev-charging-network/README.md) | OCPP 2.0.1-over-WebSocket, persistent, protocol-mandated | Core component |
| [38.leetcode](38.leetcode/README.md) | SSE push hubs; poll vs WebSocket vs SSE weighed explicitly | Core component |
| [28.uber](28.uber/README.md) | Edge gateway terminating driver/rider sockets; push offers | Core component |
| [29.uber-eats](29.uber-eats/README.md) | Per-order tracking rooms over ~4M sockets; polling rejected | Core component |
| [09.cdn](09.cdn/README.md) | Edge POP terminates HTTP/2·HTTP/3; QUIC session resumption | Core component |
| [46.google-docs](46.google-docs/README.md) | TTL'd ephemeral presence hub beside the durable op path | Core component |
| [30.facebook-messenger](30.facebook-messenger/README.md) | Presence/typing ephemeral tier; calling deferred to case 05 | Core component |
| [22.frontend-uber](22.frontend-uber/README.md) | Client socket channel with seq'd events; degrade-to-polling | Core component (client) |
| [02.twitter](02.twitter/README.md) | Historical reference: poll-based timeline reads at 300K+ QPS | By-reference |
| [20.frontend-newsfeed](20.frontend-newsfeed/README.md) | Polled/pull-to-refresh cadence, edge-triggered pagination | Brief |
| [33.instagram](33.instagram/README.md) | Adaptive-bitrate ladder for video ingest, one line | Brief |
| [45.typeahead](45.typeahead/README.md) | HTTP/2 multiplexing + in-flight cancellation as latency tool | Brief |
| [21.frontend-calendar](21.frontend-calendar/README.md) | Boundary: REST push/pull sync — no persistent transport (see below) | Boundary adjacency |

## Per-case notes

### Full cases

**[03.whatsapp](03.whatsapp/README.md)** — the catalog's reference connection-plane case.
The system-level hard problem is not messages, it is **live TCP/WebSocket connections per
data center** ([README.md](03.whatsapp/README.md)): a chat-gateway tier terminates
persistent sockets (~1M per server, estimated) and knows "who is connected to which gateway
now" in an in-memory registry; delivery is server push when the receiver is online, park-
and-push on reconnect when not ([03-architecture.md](03.whatsapp/03-architecture.md)). The
transport trade-offs are consolidated in a dedicated presence dive: delivery guarantees are
per-signal — messages are MUST-NOT-LOSE, "presence is MUST-NOT-COST-ANYTHING"; the presence
cluster is ephemeral, best-effort, never persisted, and losing it means a stale badge for
30 s ([04-deep-dives.md](03.whatsapp/04-deep-dives.md)). Its connection-layer mechanics
(gateway-held sockets, client backoff, TLS resumption, resume-from-latest-sequence) are
reused verbatim by 17 and referenced by 38 and 05.

**[05.discord](05.discord/README.md)** — runs both planes on purpose. Chat, presence,
typing and *voice signaling* all ride one multiplexed **WebSocket gateway** plus HTTPS/REST;
media then breaks out to **UDP/SRTP to WebRTC SFUs** ([03-architecture.md]
(05.discord/03-architecture.md)). Deep dive 4 weighs mesh vs SFU
([04-deep-dives.md](05.discord/04-deep-dives.md)): mesh wins nothing after ~5 people and
costs O(participants) uplink; SFU wins O(1) sender uplink and no transcode latency, loses a
new stateful pool whose death is a visible outage. No transcoding on the interactive path;
video gets sender-side simulcast (2–3 spatial layers, SFU picks per receiver) — accepted
cost 2–3× audio / 3–4× video uplink. **Signaling rides the existing gateway socket** (reusing
auth and session context) instead of a second control channel. ICE/SRTP handshake over the
socket, TURN relay as a *capacity line, not an afterthought* (TURN ≈ 2× SFU egress for
relayed participants); beyond ~5% loss, simulcast drops video layers before audio. Presence
is a separate in-memory, never-persisted cluster sharded by user_id, with opt-in bounded
subscribes and megaguild digests.

**[43.zoom](43.zoom/README.md)** — the catalog's purest WebRTC case. Client endpoint does
capture, 3-layer simulcast encode + Opus, bandwidth estimation, jitter buffer; the media
plane is **SRTP/UDP to an SFU fleet**, signaling/meeting state rides **HTTPS (QUIC)**, and a
**TURN fallback** exists for symmetric-NAT/blocked-UDP paths, costed at ~2× relay egress
([03-architecture.md](43.zoom/03-architecture.md)). The join handshake is
ICE/DTLS/SRTP to an allocator-picked nearby SFU. The reconnect design is the standout:
server keeps the participant session for a ~30–60 s grace window while the client re-runs
ICE against the same relay (or reassigns), then requests a keyframe to resume video — so a
 Wi-Fi→LTE flap does not dump the user back through the waiting room ([04-deep-dives.md]
(43.zoom/04-deep-dives.md)). SFU economics are forward-only forwarding with per-subscriber
layer selection, no transcoding (estimated ~40 Tbps peak egress in the case's materials).

**[17.elevenlabs](17.elevenlabs/README.md)** — realtime *streaming inference* over
**WebSocket**: text deltas in, audio chunks out, minutes per session; ~800K sessions/day
≈ 9 connects/s average ([02-scale-estimates.md](17.elevenlabs/02-scale-estimates.md)). The
named deep dive is "WebSocket audio transport and the jitter buffer"
([04-deep-dives.md](17.elevenlabs/04-deep-dives.md)): audio is playback-constrained, not
arrival-constrained, so every chunk carries media time + wall-clock send time. Buffering
weighed: no buffer (uninterruptible audio), fixed 1–2 s (pays its cost on first byte too),
**adaptive client jitter buffer capped ~500 ms** (chosen) because once audio flows faster
than real time, 500 ms absorbs everything and extra buffering hurts turn-taking. Backpressure
outranks buffering: streams whose output buffer exceeds a few seconds of media are errored
loudly, not silently drifted. Connection mechanics are deliberately borrowed from
[03.whatsapp/04-deep-dives.md](../03.whatsapp/04-deep-dives.md) (NAT keep-alives, reconnect
+ resume-from-sequence, per-sequence dedup).

**[01.youtube](01.youtube/README.md)** — the delivery plane: **rendition ladder (1080p@6Mbps
… 240p@0.4Mbps) chopped into 2–10 s segments** with per-rendition playlists; the player picks
the cheapest quality that fits measured bandwidth and switches segment by segment
([04-deep-dives.md](01.youtube/04-deep-dives.md)). The explicit trade-off: segments rather
than byte ranges, because seek must start at clean keyframes and byte ranges of a monolithic
file force front-to-back re-sync — the cost is one round trip per segment, mitigated by CDN
proximity and prefetch. More renditions = better fit = more storage, transcode CPU and CDN
origin diversity; ladders converge to ~5–8 rungs. Delivery rides the
[09.cdn](09.cdn/README.md)-style edge→shield→origin hierarchy because one viral video means
millions of requests for identical segments. Scale note in [02-scale-estimates.md]
(01.youtube/02-scale-estimates.md): total egress is watched-hours × average *adaptive*
bitrate, not peak bitrate.

**[31.netflix](31.netflix/README.md)** — same delivery plane, different constraint set: the
data plane is a **prepositioned private CDN (Open Connect)** serving HTTP chunk/range
content, with **client-side ABR** chosen over server-side because the measurement system
around playback failures outranks any particular ABR heuristic ([04-deep-dives.md]
(31.netflix/04-deep-dives.md)). The transport question becomes "where do the bytes sit and
who decides the bitrate" — answer: ladder at the appliance, ABR at the client, steering into
a prepositioned fleet because a rented CDN cannot guarantee the Friday-9-PM peak at flat
cost. Client machinery itself is deferred to case 23.

**[23.frontend-video-streaming](23.frontend-video-streaming/README.md)** — the client-twin
full case, written against published specs (HLS, DASH, CMAF, MSE — explicitly labeled as
standard, and the case numbers are derived, not invented). VOD uses 4 s segments (DASH-
typical; HLS default 6 s) with a 30 s buffer target ([02-scale-estimates.md]
(23.frontend-video-streaming/02-scale-estimates.md)); low-latency live drops to chunked
CMAF/LL-HLS parts of ~0.5–1 s over held-open HTTP responses with a ~3–8 s buffer, at an
estimated **1M part-requests/s for a 1M-viewer event** — a CDN hot-key problem, with the
client contributing short cacheable part URLs and aborting in-flight fetches on seek
([04-deep-dives.md](23.frontend-video-streaming/04-deep-dives.md)). The pipeline decision:
**MSE + fetch/worker everywhere it exists, native HLS only on iOS Safari** — one policy
layer, two adapters, because a single MSE-only player second-classes too much iOS traffic.
Explicit cross-link to the chat case's network-flap state machine.

**[19.frontend-chat-app](19.frontend-chat-app/README.md)** — the client side of the
connection plane: **WebSocket for live events, HTTPS for bulk backfill and media**
([03-architecture.md](19.frontend-chat-app/03-architecture.md)). The split is deliberate —
the socket carries only low-volume live deltas; anything large or replayable goes over plain
HTTP with cursor/sequence sync (`sync(last_seq per conversation)`), so a reconnect or gap is
a range fetch, not a socket problem. Push notifications are treated as its own wake-up
channel (out of scope for the client's own code).

**[24.mobile-chat-app](24.mobile-chat-app/README.md)** — mobile policy dive: **foreground =
real socket** (live delivery, typing, receipts, presence), background = push notifications
+ offline store-and-forward ([04-deep-dives.md](24.mobile-chat-app/04-deep-dives.md)).
Reconnect suppression is typed: non-message-driven reconnect churn (typing/presence) gets
longer backoff than the message channel. Boundary decision recorded in
[01-requirements.md](24.mobile-chat-app/01-requirements.md): video/audio calling is a
separate connection stack (WebRTC-class) and is explicitly **out of scope**.

### Designed components

**[08.chatgpt](08.chatgpt/README.md)** — the LLM case's gateway tier terminates
**~100M persistent client connections over SSE/WebSocket** ([03-architecture.md]
(08.chatgpt/03-architecture.md)); [02-scale-estimates.md](08.chatgpt/02-scale-estimates.md)
notes this is the same connection bookkeeping physics as Discord's fleet. Notable restraint:
despite streaming tokens, the case keeps delivery boring — auth/TLS/connection bookkeeping
only, with no session state on the gateway (durable resync lives elsewhere).

**[27.ev-charging-network](27.ev-charging-network/README.md)** — the transport is
**protocol-mandated**: OCPP 2.0.1 over WebSocket, with every charger holding one persistent
connection ([01-requirements.md](27.ev-charging-network/01-requirements.md)). Sizing makes
the connection fleet the scale driver: ~95K persistent WebSockets (95% online), and the
reconnect math — 4G jitter on a fleet-wide event → **~400 new WebSocket connections/s
sustained for 2 min**, each charger replaying 10–30 queued events ([02-scale-estimates.md]
(27.ev-charging-network/02-scale-estimates.md)). Keep-alive is layered (WebSocket ping/pong
plus TCP keepalive); the gateway fleet is deliberately **stateless WebSocket acceptors** so
churn costs nothing but auth ([03-architecture.md](27.ev-charging-network/03-architecture.md)).

**[38.leetcode](38.leetcode/README.md)** — the catalog's cleanest transport A/B/C
([04-deep-dives.md](38.leetcode/04-deep-dives.md)): verdict delivery as async event ~10 s
after submit. **Option A polling** wins simplicity, loses a QPS explosion — 800 verdicts/s ×
500 ms polls across ~150K concurrent participants ≈ 300K+ wasted reads/s at peak (arithmetic
in [02-scale-estimates.md](38.leetcode/02-scale-estimates.md)). **Option B WebSocket** wins
one bidirectional socket for verdicts+leaderboard+chat, loses heavier gateway state and
reconnect storms. **Option C SSE + polling fallback** (chosen) wins because the data shape
is one-way; mitigated with HTTP/2 to the gateway, 15 s heartbeats, exponential fallback poll
(500 ms → 2 s → 8 s, cap 15 s), and per-gateway registration caps against reconnect storms.

**[28.uber](28.uber/README.md)** — per-region **edge gateways terminate driver and rider
sockets**; trip offers are pushed over the socket, which makes the offer/ack race a
connection-plane property ([03-architecture.md](28.uber/03-architecture.md)). Nearby-driver
map views: ~50K clients polling-or-pushed every 4 s, with position telemetry deliberately
split onto lossy topics while ride-lifecycle events ride the must-not-lose log.

**[29.uber-eats](29.uber-eats/README.md)** — the poll-vs-push trade-off written out in full
([04-deep-dives.md](29.uber-eats/04-deep-dives.md)): **Option A client polling** wins
statelessness and free retries, loses ~800K QPS of read amplification at 4M clients × 5 s
polls; **Option B per-order rooms over persistent sockets** wins freshness and one-topic-per-
order multiplexing, loses an operable socket tier — **~4M concurrent sockets at dinner peak**
([02-scale-estimates.md](29.uber-eats/02-scale-estimates.md)), with socket-state in
ephemeral Redis (~2 KB/session). B is chosen; failure modes are socket storm at dinner start
(jittered backoff) and fresh-data degradation rather than socket drops under load.

**[09.cdn](09.cdn/README.md)** — the HTTP-side transport component: every POP terminates
HTTP/2 and HTTP/3 with WAF/rate limits ([README.md](09.cdn/README.md),
[03-architecture.md](09.cdn/03-architecture.md)). The QUIC appearance is specific and
quantified ([02-scale-estimates.md](09.cdn/02-scale-estimates.md)): fleet-wide ≈ 8–15M
concurrent keep-alive connections by arrival-rate × lifetime arithmetic; TLS handshakes are
5–10% of requests and cheap per-request, but **session resumption (session tickets, PSK in
QUIC) is what keeps the handshake share that low**.

**[46.google-docs](46.google-docs/README.md)** — the transport angle only (the TTL'd
presence hub's data model is its own topic, cross-linked from [REDIS.md](REDIS.md)): presence
is **ephemeral pub/sub, TTL'd at ~3–10 s liveness, rate-limited to ~1–2 Hz cursor updates
per client**, strictly separated from the durable op stream
([04-deep-dives.md](46.google-docs/04-deep-dives.md)). The traffic insight: a 200-viewer
doc's presence fanout (~200–400 msg/s) exceeds its op stream, which is exactly why a lost
cursor update is "a cosmetic nothing" and presence never enters the op log
([03-architecture.md](46.google-docs/03-architecture.md)).

**[30.facebook-messenger](30.facebook-messenger/README.md)** — presence/typing as an
ephemeral, best-effort tier that is *never durable* (same contract as WhatsApp's)
([03-architecture.md](30.facebook-messenger/03-architecture.md)). Voice/video calling is
explicitly deferred: "WebRTC SFU shape is in Discord's case" ([01-requirements.md]
(30.facebook-messenger/01-requirements.md)) — a boundary adjacency, counted here only for
the presence tier.

**[22.frontend-uber](22.frontend-uber/README.md)** — client-side connection plane: an
**evented socket channel (WS/gRPC real-time) carrying ordered, sequenced events**, sized in
egress terms (map telemetry ≈ 2.6 Gbps peak at the gateway, not client bytes)
([03-architecture.md](22.frontend-uber/03-architecture.md)). Reconnect is `last_seq` resume
→ gap retransmit → store reconciliation; explicit degradation ladder — label staleness
("last updated Xs ago"), back off to polling with backoff, keep working offline
([01-requirements.md](22.frontend-uber/01-requirements.md): ≤5 s stale normal, ≤15–30 s
degraded polling). Battery reality: OS Doze suspends sockets/GPS, so the policy engine treats
socket death as expected ([04-deep-dives.md](22.frontend-uber/04-deep-dives.md)).

### Brief mentions / by-reference

- **[02.twitter](02.twitter/README.md)** — historical reference only: "publicly
  reported ~300K+ QPS of poll-based timeline reads" (2012, QCon), reused as evidence that
  the hybrid fanout split is safe ([02-scale-estimates.md](02.twitter/02-scale-estimates.md),
  [04-deep-dives.md](02.twitter/04-deep-dives.md)). No push transport designed.
- **[20.frontend-newsfeed](20.frontend-newsfeed/README.md)** — the feed is
  deliberately *not* pushed: pull-to-refresh and a top-ranked polled refresh plus
  **edge-triggered pagination, not threshold-polling** ([03-architecture.md]
  (20.frontend-newsfeed/03-architecture.md), [04-deep-dives.md]
  (20.frontend-newsfeed/04-deep-dives.md)). A feed page is the anti-case for a socket.
- **[33.instagram](33.instagram/README.md)** — one line: video ingest produces "an
  adaptive-bitrate ladder of encodings" ([01-requirements.md](33.instagram/01-requirements.md));
  delivery is assumed handled by the CDN/media machinery of the other cases.
- **[45.typeahead](45.typeahead/README.md)** — HTTP/2 as a *latency tool*, not a transport
  choice: in-flight keystroke requests are cancelled via **HTTP/2 multiplexing + client-side
  abort** ([04-deep-dives.md](45.typeahead/04-deep-dives.md)) — a request nobody needs is the
  purest p99 pollution.

### Explicitly rejected (with the trade-off)

**[32.spotify](32.spotify/README.md)** — the sharpest rejection in the catalog. Deep dive 1
([04-deep-dives.md](32.spotify/04-deep-dives.md)) weighs mono progressive download (A),
**HLS/DASH segmented streaming (B)**, and cache-first playback (C). B wins video-style
uniformity and segment-granular CDN caching, loses *one RTT per track start* — which lands
exactly on the time-to-first-sample budget — and mid-stream quality switching "matters ~never
for a 3-minute music track". Chosen: A + C hybrid (range-addressable file streamed into a
player-disk cache). The doc cross-links YouTube's segments-exist-for-seek-and-quality-switch
conclusion and lands on the opposite choice *because the requirement set differs* — the
model decision this repo wants documented.

**[38.leetcode](38.leetcode/README.md), [29.uber-eats](29.uber-eats/README.md)** — polling
rejected as the primary mechanism in both (QPS-explosion arithmetic above), WebSocket
rejected in 38 as heavier than the one-way shape needs. **[24.mobile-chat-app]
(24.mobile-chat-app/README.md)** — WebRTC-class calling rejected from scope, kept as a
recorded boundary. These are option-level rejections; only case 32 rejects the transport
itself at architecture level.

**Boundary adjacency — [21.frontend-calendar](21.frontend-calendar/README.md)** uses a
"REST/push sync API" with a sync-token plus one push-flush of queued mutations, and sizes
device reminder pushes (~333K pushes/s during bursts) ([02-scale-estimates.md]
(21.frontend-calendar/02-scale-estimates.md)). It has **no persistent transport and no media
plane**, so it is counted outside the 22 — but its push/pull cadence arithmetic is the
low-volume cousin of everything above.

## The portable pattern

Two shapes, one decision rule. Cross-linked to [43.zoom](43.zoom/README.md) (connection +
media) and [01.youtube](01.youtube/README.md) (delivery) as the reference ends:

1. **Choose the plane by the question "does the server need to find the client?"** If events
   arrive at the server unannounced and freshness is the product (chat, verdicts, offers,
   tracking, presence), build a connection plane: stateful gateway fleet, per-gateway
   connection registry, server push (03, 08, 27, 38, 28, 29). If the client drives and the
   content is immutable bytes, build a delivery plane: stateless HTTP segments over a CDN
   (01, 31, 23) — and [09.cdn](09.cdn/README.md) is the component both share.
2. **The gateway fleet is stateless-ish by construction**: hold connection state, auth and a
   route hint only; all authority lives behind it (03, 27, 08, 29). Ephemeral signals
   (presence, typing, cursors) never get durable storage — they get TTL'd in-memory clusters
   and a stated willingness to lose them (03, 05, 30, 46).
3. **Guarantees are per-signal, decided up front**: messages/events are must-not-lose with
   sequence numbers; presence/typing are must-not-cost-anything (03, 05, 28, 46). The socket
   is *not* the durability mechanism — resume-by-sequence against a durable store is (03, 17,
   19, 22).
4. **Push must still be cheaper than the polling you replaced it with.** Both cases that did
   this arithmetic (29: 800K QPS of poll reads vs a socket tier; 38: 300K+ wasted reads/s vs
   SSE hubs) found push wins only because the event rate is low relative to the poll rate.
   When the data is one-way and low-rate, SSE beats WebSocket on gateway state; when the
   client also sends on the same channel, WebSocket (38's option C vs B, 05, 08).
5. **Media goes segment-first and client-adaptive**: sender-side ladder (or simulcast), 2–10 s
   segments, per-recipient selection without server transcode — works for video (01, 23, 31,
   05's SFU, 43's SFU) and is explicitly the wrong tool when latency-to-first-byte dominates
   (Spotify's rejection). SFU forwarding with simulcast is the multicast generalization of
   the same ladder idea (05, 43).
6. **Reconnect is a feature, not an error path**: client backoff + jitter, resume-from-latest-
   sequence, session grace windows on the server, and degradation ladders down to labeled
   staleness or polling (03, 17, 22, 29, 38, 43). WebRTC adds keyframe-request resync on top
   — media needs a GOP boundary, chat needs a sequence number (43).

### Where the shape ends

- **No shared plane across products.** Zoom/Discord pay for both planes precisely because
  neither can fake the other: HTTP-segmented delivery cannot meet conversational latency,
  and a socket fleet cannot serve 40-Tbps-class egress as one-connection-per-consumer. When
  a case pretends one plane suffices, it gets the Uber-Eats-style read-amplification bill or
  a second-class experience (23's iOS argument).
- **Carrier and browser reality bounds the socket**: NAT idle timeouts, Doze battery
  suspension, reconnect storms after any gateway deploy (24, 17, 27, 29). The connection
  plane's cost is connection *churn*, and its design point is the storm, not the steady
  state (27: ~400 new WebSocket/s for 2 min is the sizing driver).
- **TURN/relay is where WebRTC economics break**: relayed media costs ~2× the SFU-path
  egress (05, 43), so traversal success rate, not codec quality, sets the media bill.
- **Protocol fiat overrides architecture**: where the transport is mandated (OCPP-on-
  WebSocket in 27), the design questions move to heartbeats, churn, and replay — you cannot
  trade the transport away.
- **Interactive conversational audio is the gap between the shapes**: HTTP delivery is too
  slow for barfing-in, but the catalog still refuses raw UDP for it — ElevenLabs keeps
  WebSocket plus an adaptive jitter buffer capped at 500 ms (17) and accepts that its
  latency floor is TCP's. Below that floor, only 43's SRTP/UDP plane exists, and the catalog
  has no case for consumer-side raw-UDP audio outside conferencing.

---

*Counting methodology*: "cases mentioning real-time transport" here = 22/50, counted per case
(any of the transports named in scope, verified in context) across backend n=43 and client
n=7 combined — not comparable to [CONCEPTS.md](CONCEPTS.md)'s per-counting-basis rows
(backend 8/43 for "primary-connection gateway", client cases excluded from it; CONCEPTS.md's
count covers only persistent-connection gateways, not media delivery or HTTP/2·3 edge
termination). Two adjacencies (21, 30's calling deferral) lie outside the 22 but are
documented in the table.
