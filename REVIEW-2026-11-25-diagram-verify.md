# Diagram spec verification 2026-11-25

**Status: fixes applied 2026-11-25 — 108/111 findings applied (43 fixer agents, run wf_339354a1-f10); 3 skipped as already-fixed no-ops (31.netflix ×2, 32.spotify ×1), 1 skipped as a fallback option not taken (06.apple-app-store, primary fix applied instead). All 56 repo mermaid blocks re-render PASS (independent sweep).** This record's findings were produced by verification run wf_69239ab4-75d (50 case validators, one per case).
Every diagram checked against `study_cases/DIAGRAMS.md` (§1–§8) + mermaid syntax + the case's own written specs (01–04).

## Tally

- All **50** cases validated, **0** agent failures; **7 clean**, 43 with findings.
- **111 findings: 6 high, 68 medium, 37 low.** Medium-weight clusters: §5 verb-phrase edge labels, §6 missing transport/format half on cross-boundary edges, store edges not in OP-arg form, minor prose-flow omissions.

Clean: 01.youtube, 02.twitter, 05.discord, 10.distributed-cache, 12.bigtable, 15.ml-ranking-system, 40.tinyurl.

## High findings

### study_cases/04.deployment-system/ — §3 labels
- **Where**: 04.deployment-system/03-architecture.md, line 25
- **Finding**: `fro -->|"propose: platform rollout"| ctl[deployment controller<br/>rollout state machine · canary]` — node ID `fro` is used but never declared, so Mermaid renders a node literally labeled "fro" (a reader cannot decode it; §3: "IDs are scaffolding; the label is the truth"). The prose (03 components §12, dive 5) names it "fleet rollout coordinator".
- **Fix**: `fro[fleet rollout coordinator<br/>platform version -> waves · global pause] -->|"propose: platform rollout"| ctl[deployment controller<br/>rollout state machine · canary]`

### study_cases/14.ml-production-system/ — prose-consistency / §4 arrows / §5 transfer-vs-transformation / §6 edges / §1 shapes
- **Where**: study_cases/14.ml-production-system/03-architecture.md, mermaid block lines 9–15 (nodes fs, ms) and line 22 (`fs -->|"3· Score()"| ms`)
- **Finding**: The serving path draws a synchronous deploy-boundary call: `fs -->|"3· Score()"| ms` from `fs[feature service ...]` to `ms[[model server<br/>N versions in-process<br/>champion · challenger · shadow]]`. A reader decodes a separate synchronous inference-service hop over the wire — exactly the design the prose rejected. 03 key data flows: "the only network hops are the feature fetches ... the model itself runs in-process, no separate 'inference service' hop (trade-off in deep dive 4)"; deep dive 4 Chosen = Option B "one co-located serving binary, many in-process models" ("no hop; feature fetch and scoring are in one memory space; one deployment"); 01 NFR: "model code in-process or co-located, no extra network hop". The sync `-->` Score() edge contradicts this (and per §6 a function call inside one deploy unit is not diagrammed at all; per §5 the resolve->fetch->score transformation belongs as a -> chain in the owning node's label). The `ms[[...]]` subroutine shape is also wrong either way: if the model server is a separate deployed unit, §1 says deploy rectangles, and if it is in-process, neither the sync edge nor the call is drawable.
- **Fix**: `Delete `fs[feature service<br/>resolve feature list<br/>per-request cache]` and `ms[[model server<br/>N versions in-process<br/>champion · challenger · shadow]]`, and replace the four serving-path lines with:
    s[rank serving binary<br/>resolve feature list -> MGET -> Score<br/>N versions in-process · champion · challenger · shadow]
    u ==>|"1· POST /rank<br/>HTTPS · JSON"| s
    s -->|"2· MGET feature keys"| o
    s -.->|"produce: prediction<br/>scores · inputs · model version"| plog
    s -->|"SQL production pointer"| reg
(i.e. `fs ==>` becomes `s ==>`, the three `fs -->|"2·..."| o` / `fs -->|"3· Score()"| ms` edges and the `ms -.->|"produce: prediction..."| plog` edge are replaced by the `s -->` / `s -.->` lines above; the `ms -->|"SQL production pointer"| reg` line becomes `s -->|"..."| reg`.)`

### study_cases/18.sora — §6 API edges / §5 transfer-vs-transformation / prose-consistency
- **Where**: study_cases/18.sora/03-architecture.md, mermaid block, line 14
- **Finding**: Edge `jo -.->"produce: job.chunks<br/>gRPC LeaseChunkJob() · GPUs pull"| mf` internally contradicts itself and the prose: it is labeled 'GPUs pull' yet drawn as a jo->mf push; it mixes an event contract (produce: + event name) with an RPC method (LeaseChunkJob()) on one edge; and it is dotted/asynchronous while the prose describes a synchronous gRPC lease pull (04-deep-dives dive 2: 'GPUs are stateless executors that pull work'; dive 3 failure mode (b): 'lease-based job claiming and TTL'd leases'). There is also no event bus/topic node anywhere in the diagram or prose, so produce: names an event system that does not exist.
- **Fix**: `mf -->|"gRPC LeaseChunkJob()<br/>pull (job_id, chunk_idx)"| jo`

### 28.uber — prose-consistency
- **Where**: study_cases/28.uber/03-architecture.md, mermaid block lines 12-13 and 29 (q, pr, st), vs 'Pricing service' component paragraph and 'Quote flow'
- **Finding**: The diagram routes the surge read + fare computation through the quoting service, contradicting the prose. Diagram: `q -->|"⚡ GET surge:{tile}"| st[("surge tile store<br/>cache + log")]` with q labeled `quoting<br/>fare + ETA fan-out`, while the pricing service `pr["pricing svc<br/>fare calc · damping + hysteresis"]` has only a SET edge to the tile store and is never called by anyone for fares. The 03-architecture Components prose is explicit: "Pricing service ... Stateless fare computation (base + distance + time + tier) reading *surge multiplier* from the tile store" — i.e. the pricing service reads surge from the tile store and computes the fare. As drawn, a careful reader concludes quoting computes fares directly against the tile store and the pricing service only writes tiles — the wrong owner for the exact thing the Q: caption asks about ("matched to a driver with the right surge price").
- **Fix**: `Replace the q->st line `    q -->|"⚡ GET surge:{tile}"| st[("surge tile store<br/>cache + log")]` with `    q -->|"PriceQuote(dist, time, tier)<br/>gRPC · proto"| pr["pricing svc<br/>fare calc · damping + hysteresis"]`; change q's definition to `    q["quoting<br/>ETA fan-out · fare assembly"]`; and change line 29's `    pr -->|"SET surge:{tile}"| st` to `    pr -->|"GET+SET surge:{tile}"| st[("surge tile store<br/>cache + TTL")]` (read+write is one store edge per §4).`

### study_cases/36.airbnb — §1 shapes / §4 async edges / prose-consistency
- **Where**: 03-architecture.md, mermaid block line 39, edge eb -.->"consume: booking.confirmed" -> cal
- **Finding**: Edge `eb -.->|"consume: booking.confirmed"| cal` makes the calendar DB (the authoritative store) the consumer of the booking event. The 03-architecture prose says the event bus "Feeds the search index updater and the calendar-vs-bookings reconciliation job" — the reconciliation job is the consumer, not the calendar DB. As drawn, a reader concludes the availability truth is fed asynchronously from the bus, contradicting both the case's central stance (calendar day grid = authoritative synchronous write, events are advisory/derived, N2/N3) and the diagram's own Q: caption. Also violates §1 semantics: a cylinder is written with OP-arg `-->` edges, it does not `consume:`.
- **Fix**: `Replace the line `    eb -.->|"consume: booking.confirmed"| cal` with:
`    rj[reconciliation job<br/>calendar vs bookings · drift alarm]` and `    eb -.->|"consume: booking.confirmed"| rj` and `    rj -->|"read+write · day grid audit"| cal` (brings the diagram to 15 nodes, still within the ~15-node limit)`

### study_cases/37.ticketmaster — §6 API edges / §5 transfer-vs-transformation / prose-consistency
- **Where**: study_cases/37.ticketmaster/03-architecture.md, ord→psp edge (line 18)
- **Finding**: `ord -->|"authorize · capture<br/>HTTPS"| psp[/payment PSP/]` — the sync edge to the PSP lists both steps, but 04-deep-dives.md dive 4 explicitly chose Option B (authorize inline, capture async against a webhook/event-log consumer, "not inline in the request path"); the diagram and prose genuinely disagree on where capture runs, the label has no METHOD /path (§6), and it is a verb phrase rather than the payload (§5).
- **Fix**: `ord -->|"POST /payments/authorize<br/>HTTPS · JSON"| psp[/payment PSP/]`

## Findings by case (medium + low)

### 09.cdn
- **[low] §6 edge contracts** — study_cases/09.cdn/03-architecture.md, mermaid block, edges into lp[Log pipeline]
  - The async log-shipping edges `ep -.->|"log records · compressed"|` and `sh -.->|"log records · compressed"|` state only a raw payload; §6 requires async event edges to carry `produce:` + event name (the event name is the API), and the control-plane edges two lines above already set that form ("produce: config.update …"). The API/contract of the log feed is not decodable from the label.
  - Fix: `ep -.->|"produce: pop.logs<br/>compressed"| lp[Log pipeline<br/>analytics + billing]`

### 20.frontend-newsfeed
- **[medium] prose-consistency** — study_cases/20.frontend-newsfeed/03-architecture.md, block at line 13 — missing ssr --> tla edge on the paint (primary) path
  - The primary `1· GET /feed` path shows ssr rendering the page-1 items (label "streams above-fold HTML / 1–2 items · inline JSON page 1") but the diagram has no edge from ssr to timeline-api, even though timeline-api is the only source of the pre-ordered entries the SSR tier renders. Every later page fetch (pf, isl, ins) goes through tla, so the diagram implies page 1 arrives without fetching it. Prose requires the fetch: "a *thin* SSR tier that streams the first viewport: one or two rendered feed items plus an inline JSON payload (page 1 + cursor token...)" and the Case 02 contract ("the timeline is a pre-ordered list of entries — the client never ranks, only renders and paginates") applies to the SSR tier as the client. Missing flow on the stated request path (§8 one-question coverage: the question is "How does one feed session paint...").
  - Fix: `Add this edge after the ssr --> ab line: `    ssr -->|"GET /v1/feed<br/>page 1 · 20 entries · HTTPS · JSON"| tla["timeline-api<br/>Case 02: pre-ordered entries<br/>~15 items / page"]``
- **[medium] §6 API edges (also §4 arrows)** — study_cases/20.frontend-newsfeed/03-architecture.md, line 17: `ssr -.->|"inline payload<br/>page-1 entries · cursor token"| fs`
  - `ssr -.->|"inline payload<br/>page-1 entries · cursor token"| fs` — a cross-deploy-boundary edge (server SSR tier -> client store) with no contract in the §6 form (no METHOD/path + transport, no RPC method, no produce:/consume: + event name), on a `-.->` edge that per §4/§6 signals an event handoff but here carries an in-band payload. The payload is already accounted for twice elsewhere: the primary edge label u ==>\|"1· GET /feed<br/>HTTPS · HTML + inline page-1 JSON"\| and fs's own label "page 1 inline-seeded". A reader could decode a separate async push of data from SSR to the store.
  - Fix: `Delete the whole line `    ssr -.->|"inline payload<br/>page-1 entries · cursor token"| fs` — the inline seed stays documented in fs's label ("page 1 inline-seeded") and the primary edge label ("HTML + inline page-1 JSON").`

### 21.frontend-calendar
- **[medium] §4 requests direction / §6 store edges** — study_cases/21.frontend-calendar/03-architecture.md, edge `ls --> exp`
  - The read is drawn backwards and unlabeled: `ls --> exp` says the Local store calls/feed the Expansion cache, but the cache is the reader (prose: expansion "derives occurrences from masters for a requested window"), and per prose its invalidation comes from the change hub ("Invalidate exactly (master_id) on master edit — the change hub's keying"), not from the store. Everywhere else in this block edges into the store carry an OP-arg label (ui->ls, sync->ls), so this unlabeled edge out of the store has no decodable contract and reads as either a response-direction edge or as hub bypasses by invalidation, both wrong.
  - Fix: `Replace `  ls --> exp` with:
  exp -->|"SQL masters in window"| ls
  hub -->|"master_id keys"| exp`
- **[medium] §6 event contract / §5 payload naming** — study_cases/21.frontend-calendar/03-architecture.md, edges `pch -.->|"invalidate hint<br/>push · no payload"| sync` and `pch -.->|"reminder.push<br/>push · JSON"| bell`
  - The two push-channel deliveries don't follow the §6 event contract. `pch -.->|"invalidate hint<br/>push · no payload"| sync` names no event and uses no produce:/consume: prefix — the event name is the API and there is none to decode — and "invalidate" is outcome phrasing rather than a payload name (prose calls it an "invalidate, pull now" hint). The sibling edge `pch -.->|"reminder.push<br/>push · JSON"| bell` names an event but also omits consume:. Together they read as two different labeling conventions on the same channel.
  - Fix: `  pch -.->|"consume: sync.hint<br/>push · no payload"| sync
  pch -.->|"consume: reminder.push<br/>push · JSON"| bell`
- **[low] §5 edge labels** — study_cases/21.frontend-calendar/03-architecture.md, edge `sel -->|"expand window a–b"| exp`
  - Edge label is a verb phrase: `"expand window a–b"` states a command/outcome, not what crosses. The thing crossing is the query for window a–b.
  - Fix: `  sel -->|"query: window a–b"| exp`
- **[low] §6 API edge contract** — study_cases/21.frontend-calendar/03-architecture.md, edge `np -->|"pull event data<br/>HTTPS · JSON"| api`
  - `np -->|"pull event data<br/>HTTPS · JSON"| api` is drawn as a cross-boundary HTTP edge (it carries a transport line) but has no METHOD /path — §6 requires method + path + transport · format — and "pull event data" is a verb phrase, not the query that crosses.
  - Fix: `  np -->|"GET /events<br/>HTTPS · JSON"| api`
- **[low] §2 emoji table** — study_cases/21.frontend-calendar/03-architecture.md, node `pch["Push channel<br/>FCM/APNs/WS · sync hints · reminder payloads"]`
  - The emoji table maps "notifications / push" to 🔔, and the sibling mobile case (24.mobile-chat-app) gives its push service 🔔, but the Push channel node here is bare while the same-block Notification presenter carries 🔔 — the two notification components disagree on the table's emoji.
  - Fix: `  pch["🔔 Push channel<br/>FCM/APNs/WS · sync hints · reminder payloads"]`

### 28.uber
- **[medium] prose-consistency** — study_cases/28.uber/03-architecture.md, mermaid block line 18 (me→d edge), vs 'Edge gateway (per region)' and 'Matching engine' component paragraphs
  - `    me -->|"6· offer push<br/>WS · JSON"| d` draws the matching engine holding a WebSocket to the driver app, bypassing the gateway. Prose says the edge gateway "Terminates driver and rider sockets, does auth and connection lifecycle" and the matching engine "sends offers over the socket" — the socket is between driver app and gateway only. The edge carries `WS · JSON`, so a reader decodes a direct me↔d WebSocket that does not exist in the design.
  - Fix: `Replace `    me -->|"6· offer push<br/>WS · JSON"| d` with `    me -->|"6· offer push<br/>gRPC · proto"| gw` (offer rides the gateway's driver socket). Optionally, to make the socket direction explicit per §4, additionally replace line 9's `    d -->|"1· pos batch every 3–15 s<br/>WS · JSON"| gw` with `    d <-->|"WS session<br/>pos batch · accept up · offer push down"| gw` and drop the separate line-19 accept edge.`
- **[low] prose-consistency** — study_cases/28.uber/03-architecture.md, mermaid block, matching engine `me["matching engine<br/>query → score → offer"]` has no ETA edge; vs 'Data flows — Ride flow' and 04-deep-dives §3
  - The ride/matching flow the Q: caption covers includes the ETA dependency — prose ride flow: "candidate scoring (calls ETA service)"; deep dive 3: "score by passenger ETA (routing-call dependency, Case 26)"; 01-requirements: "Uber-style matching calls an internal ETA service as a dependency; we specify the call". The diagram shows me's scoring (`matching engine<br/>query -> score -> offer`) but scores with no drawn dependency — every other cross-service dependency in the question (geo index query, trip state machine, tile store) is an edge, so ETA is silently absent.
  - Fix: `Add after line 17: `    me -->|"GetETA(P, D)<br/>gRPC · proto"| eta["ETA svc<br/>routing + drive-time<br/>vendor: Case 26"]` (rectangle — internal service, no emoji; brings the diagram to 14 nodes, still ≤ ~15).`

### 43.zoom
- **[medium] §6 API edges** — study_cases/43.zoom/03-architecture.md, flowchart block line 9, edge u==>sg
  - Primary edge `u((🧑 participants<br/>3 video layers · Opus<br/>bandwidth estimator)) ==>|"1· HTTPS signaling<br/>join · mute · roster"| sg[...]` carries no METHOD /path and no body format. §6 requires every cross-deploy-boundary edge to carry `METHOD /path<br/>transport · format`; the reference example is `==>"1· GET /v1/feed<br/>HTTPS · JSON"`. Here 'HTTPS signaling' is not a METHOD /path and 'join · mute · roster' is a payload list where the format line should be.
  - Fix: `    u((🧑 participants<br/>3 video layers · Opus<br/>bandwidth estimator)) ==>|"1· POST /join<br/>HTTPS · JSON · join/mute/roster"| sg[signaling · meeting svc<br/>meeting state machine]`
- **[medium] prose-consistency** — study_cases/43.zoom/03-architecture.md, flowchart block line 13, node tn
  - The TURN fallback node is a dead end: `u <-->|"SRTP · symmetric NAT"| tn[TURN fallback<br/>relay when UDP blocked]` but no edge leads from tn onward. The Q: caption explicitly asks about TURN fallback, and the prose (media session to 'that SFU' with TURN fallback; 04 §5 'TURN fallback (est. sheet: 2× egress cost on that path)') only makes sense if TURN-relayed media still terminates at the SFU. As drawn, a careful reader decodes a media path that goes nowhere.
  - Fix: `    tn <-->|"SRTP media"| sfu`
- **[low] §4 arrows** — study_cases/43.zoom/03-architecture.md, flowchart block line 11, edge ra-->sfu
  - Edge `ra -->|"3· gRPC AssignRelay()<br/>SFU address · token"| sfu[...]` puts the response contents ('SFU address · token') on a request-direction edge; §4 says draw the request direction only, and a reader can misread the address/token as request arguments to AssignRelay() rather than what the allocator hands back for the client to connect with.
  - Fix: `    ra -->|"3· gRPC AssignRelay()"| sfu[SFU fleet at edge POPs<br/>primary media path<br/>forward-only · per-sub layer pick]`

### study_cases/03.whatsapp/
- **[medium] §4 arrows / prose-consistency** — 03-architecture.md mermaid block, edge `dl -.->|"push envelope"| cg`
  - `dl -.->"push envelope"| cg` draws the online push as an async fire-and-forget edge, but the prose makes it a synchronous hop on the sub-second hot path: "delivery service pushes to the receiver's gateway", "Delivery latency: dominated by 2 hops (sender gateway -> delivery service -> receiver gateway)", and 01-requirements targets "< 1 s when both online". §4 reserves `-.->` for edges where "the caller does not wait" (events, queue consumption) — this push is neither; it is the second sync RPC of the delivery chain. The label "push envelope" is also a verb phrase, §5 style, not the payload alone.
  - Fix: `dl -->|"3· gRPC push(envelope)<br/>to receiver's gateway"| cg`
- **[low] §6 edge contracts** — 03-architecture.md mermaid block, edge `s <-->|"E2E envelope<br/>persistent TCP"| cg` (same issue on the `r` edge)
  - Both client session edges `<-->|"E2E envelope<br/>persistent TCP"|` name the payload and the transport but omit the format half of the §6 contract (`transport · format`), unlike the §7 reference form `WS feed.delta<br/>wss · JSON`. The edge crosses the client↔server deploy boundary, so it must decode alone.
  - Fix: `s((🧑 sender)) <-->|"E2E envelope<br/>persistent TCP · binary"| cg[chat gateway<br/>sockets · framing · session registry]`
- **[low] prose-consistency** — 03-architecture.md mermaid block, missing edge after `cg -.->|"client ack<br/>message id"| dl`
  - The prose online flow ends "receiver ACKs -> 'delivered' state propagated to sender (and read receipts when receiver opens)". The diagram draws the inbound ack (`cg -.->|"client ack<br/>message id"| dl`) but not the delivered-state propagation back to the sender's gateway, which belongs to the caption's question ("one 1:1 message ... online"). A reader of the diagram alone can't tell the sender ever learns the message was delivered.
  - Fix: `add below the ack edge: `dl -.->|"delivered state<br/>message id"| cg``

### study_cases/04.deployment-system/
- **[medium] §5 transfer-vs-transformation / §6 edge contracts** — 04.deployment-system/03-architecture.md, line 25, edge fro --> ctl
  - `-->|"propose: platform rollout"|` — a verb phrase, not data crossing the boundary (§5: an outcome/trigger like "push config" gets rewritten as what crosses), and a cross-deploy-boundary edge with no transport contract (§6 requires METHOD/path + transport·format or RPC method + transport).
  - Fix: `-->|"gRPC ProposeRollout()<br/>gRPC · proto"|`
- **[medium] prose-consistency** — 04.deployment-system/03-architecture.md, diagram edges around run/art (lines 19–21); prose 03 "Data flows" step 4
  - `run -->|"CAS put artifact:{sha}"| art` is the only runner–artifact edge; nothing produces the `artifact.published` event that line 21 then consumes (`k -.->|"consume: artifact.published"| ctl`). Prose flow 4: "On artifact.published, downstream jobs and the deployment controller are notified via the bus" — the diagram's own Q: ("what does the event bus fan out?") covers this, and the dag executor that opens downstream jobs never consumes it.
  - Fix: `run -.->|"produce: artifact.published"| k    and    k -.->|"consume: artifact.published"| dag`
- **[medium] prose-consistency** — 04.deployment-system/03-architecture.md, run[runner fleet] edges (lines 17–20); prose 03 "Data flows" step 3 + components §5
  - No edge carries the runner's completion report: prose flow 3 says the runner "reports success/failure with the fence token (stale tokens rejected)" and the dag executor "Persists each transition", but the diagram only shows claim, creds, artifact put and log.chunk. Without it, "a commit becomes a tested artifact" — the diagram's stated Q: — has no closed loop. Prose does not pin the transport; the bus-produce form below matches the case's "DB-rows-are-authority, bus-is-fanout" decision and dive 1's fencing-at-report mechanism.
  - Fix: `run -.->|"produce: job.status"| k    and    k -.->|"consume: job.status"| dag`
- **[low] §6 edge contracts** — 04.deployment-system/03-architecture.md, line 13
  - `db -.->|"outbox rows"| rl[outbox relay<br/>outbox rows -> bus events]` — a store edge whose label states only the payload; §6 says store edges use OP-arg form (`SQL <table/purpose>`), so the edge does not decode its own operation.
  - Fix: `db -.->|"SQL poll outbox"| rl[outbox relay<br/>outbox rows -> bus events]`

### study_cases/06.apple-app-store/
- **[medium] prose-consistency** — study_cases/06.apple-app-store/03-architecture.md, edge sf --> ct (line 17)
  - Edge `sf -->|"SQL released versions<br/>per storefront"| ct` draws the storefront/check-in tier sync-reading the single authoritative catalog metadata DB. The prose contradicts this: 03 component 6 says "reads eventually consistent across regions (deep dive 5)" and deep dive 5's chosen design is "every region's read model (storefront caches, search index delta layer, manifests, CDN pre-warm) subscribes to the same catalog-change event stream" — storefront reads come from event-fed per-region read models, never directly from the authoritative store (ct's label even says "writes only via review gate", implying it is the authoritative node). A reader would decode the check-in path as strongly consistent against the authoritative catalog.
  - Fix: `Replace `sf -->|"SQL released versions<br/>per storefront"| ct` with `eb -.->|"consume: catalog.changed"| sf` (sf's own label keeps `release rows · storefront cache` — its reads serve the event-fed regional read model, not the authoritative DB).`
- **[low] prose-consistency** — study_cases/06.apple-app-store/03-architecture.md, edges ct -.-> eb -.-> sc (lines 14-15)
  - The diagram shows only `search cluster` consuming `catalog.changed`, but the 03 prose data flows say "catalog change event -> search indexer, CDN pre-warm, storefront caches", and deep dive 1 states "the release-gate event triggers CDN pre-warm of the new variant's objects". CDN pre-warm is part of release fanout and squarely inside the diagram's own Q: ("how does a developer upload become a released app"); it is absent from the diagram.
  - Fix: `Add after the eb line: `eb -.->|"consume: catalog.changed<br/>CDN pre-warm"| dist` (or, since dist is already on the download path, fold it into the dist label: `dist[distribution-service<br/>signed URLs · throttling<br/>pre-warm on catalog.changed]`).`
- **[low] §8 one question / ~15 nodes** — study_cases/06.apple-app-store/03-architecture.md, whole mermaid block (lines 7-26)
  - The block has 17 nodes (dev, gw, pipe, bl, rv, ct, eb, sc, u, sf, cdn, dist, iap, psp, lg, k, agg), over the ~15-node budget of §8. It grew past the budget with bonus coverage: the search cluster (sc) belongs to deep dive 2's question, not "upload -> release, check-in, download, pay".
  - Fix: `Delete line 15 (`eb -.->|"consume: catalog.changed"| sc[...]`) — drop sc to 16, or drop both sc and rv's search-adjacent detail after fixing the read-model edges; if keeping sc intentionally, state in prose that search is shown for the release-fanout context and accept the ~ in ~15. Cleanest trim: remove sc and its consume edge (bringing the count to 16) and let the finding-2 fix's pre-warm edge replace it as the second consumer.`

### study_cases/07.blob-store/
- **[medium] §6 API edges + prose-consistency** — /home/pranamulia/system-design/study_cases/07.blob-store/03-architecture.md, mermaid block line 11: fe -->|"GET / 2· write index record:bucket-key"| mt
  - The edge label "GET / 2· write index record:bucket-key" is garbled: the leading "GET /" makes a solid sync edge into a store decode as an HTTP GET against the metadata index (violating the §6 store-edge OP-arg form — "a --> edge into a store is the operation named in its label"), and it also contradicts itself since the prose says a PUT commits the index record. Additionally, the GET path the Q: caption explicitly covers ("authenticate -> loads index record -> parallel read of ≥k shards") has no index-read edge anywhere in the diagram, so GET decodes as fe reading shards with no mapping source.
  - Fix: `    fe -->|"GET index record:bucket-key"| mt
    fe -->|"2· write index record:bucket-key"| mt`

### study_cases/08.chatgpt/
- **[medium] prose-consistency** — study_cases/08.chatgpt/03-architecture.md, mermaid block line 72: `SCH -->|"affinity tbl: prefix id → worker set"| PF`
  - Edge "SCH -->|"affinity tbl: prefix id -> worker set"| PF" draws the scheduler pushing the affinity table into the prefill pool, but the prose (04-deep-dives.md DD3: "The scheduler keeps an approximate affinity table") makes the table scheduler-internal state — no such flow exists. It is also a second parallel SCH->PF edge duplicating the "affinity routing" already in the SCH node label, and it appears to be a mangled stand-in for the Prompt/prefix cache component (§1 "Never group/substitute" spirit), which appears nowhere as a node in this diagram. A careful reader could decode it as a real table-distribution flow into prefill workers.
  - Fix: `Delete the line `  SCH -->|"affinity tbl: prefix id -> worker set"| PF` entirely. The prefix id -> worker-set routing is already carried by the node label `SCH[Inference scheduler<br/>affinity routing · batching · preemption]` and by the `SCH -->|"gRPC Prefill()<br/>prompt · prefix id"| PF` edge; the KV prefix cache is covered in prose (03 Components) and in the 04 sequence diagram's IP participant label.`
- **[medium] §6 API edges** — study_cases/08.chatgpt/04-deep-dives.md, sequence block message: `AG->>IP: 5· dispatch prompt · prefix id · pinned v`
  - Cross-deploy-boundary edge labelled "5· dispatch prompt · prefix id · pinned v" carries no API contract — §6 requires METHOD+transport or RPC method per boundary edge, and §5 forbids verb phrases ('dispatch') in edge labels; labels state what crosses. The 03 flowchart names this exact hop `gRPC EnqueuePrompt()`, so the two diagrams are also inconsistent at this hop.
  - Fix: ``        AG->>IP: 5· gRPC EnqueuePrompt(prompt · prefix id · pinned v)``
- **[low] §3 labels** — study_cases/08.chatgpt/03-architecture.md, mermaid block all node declarations (lines 62–76), e.g. `GW[Edge gateway tier<br/>auth · TLS · conn bookkeeping]`
  - Node IDs are uppercase (C, GW, CS, DS, RL, MOD, SCH, REG, PF, DC, SA, AM); §3 requires "Node IDs: 1–4 lowercase letters". Mermaid still parses it, but the case drifts from the spec and from all compliant sibling-case diagrams, which use lowercase IDs.
  - Fix: `Lowercase every ID in the 16 statement lines: c, gw, cs, ds, rl, mod, sch, reg, pf, dc, sa, am (e.g. `  c((🧑 clients<br/>web · app)) <-->|"1· user turn · SSE token stream<br/>SSE/WS · JSON"| gw[Edge gateway tier<br/>auth · TLS · conn bookkeeping]` and likewise on each edge line).`

### study_cases/11.zookeeper
- **[medium] prose-consistency** — study_cases/11.zookeeper/03-architecture.md, write-path edges between step 4 and step 5
  - The diagram's own question is "How does a write reach a ZAB quorum", and the prose says "followers fsync to the txn log and ACK. On a quorum of ACKs the leader sends COMMIT" — but the ACK edge from follower to leader is not drawn. The write path jumps from `f -->|"4· fsync proposal"| tl` to `l -->|"5· COMMIT zxid"| f` with no quorum-forming step visible, so a reader cannot see how the quorum is reached (the 'ZAB in one picture' ASCII in the same file does show `fsync txn log -> ACK` and `quorum ACKs -> COMMIT`).
  - Fix: `Insert one line after `f -->|"4· fsync proposal"| tl`: `f -->|"ACK zxid"| l``
- **[low] §6 API edges** — study_cases/11.zookeeper/03-architecture.md, edge c -->|"read /path"| o
  - Cross-deploy-boundary edge `c -->|"read /path"| o` carries no transport/format line, while the write edge `c -->|"1· write /path<br/>TCP · session"|` on the same pair does — §6 requires every cross-boundary edge to carry the contract so it decodes alone.
  - Fix: `c -->|"read /path<br/>TCP · session"| o`
- **[low] §6 API edges** — study_cases/11.zookeeper/03-architecture.md, edge c -->|"read /path"| f
  - Same §6 contract omission on the follower read edge `c -->|"read /path"| f` (transport line missing).
  - Fix: `c -->|"read /path<br/>TCP · session"| f`

### study_cases/13.kafka
- **[low] §5 transfer-vs-transformation** — study_cases/13.kafka/03-architecture.md, edge t -.-> og
  - Edge label second line "independent cursor" is not a contract line — §5 requires edge labels to state only what crosses (payload/event/query, plus transport·format); "independent cursor" is a semantic property of the consumer, so it belongs in the node label. Quote: t -.->|"consume: orders · clickstream<br/>independent cursor"| og
  - Fix: `    t -.->|"consume: orders · clickstream"| og

and move the property into the target node's label: change
    og((🧑 other groups<br/>warehouse · streaming services))
to
    og((🧑 other groups<br/>warehouse · streaming<br/>independent cursors))`

### study_cases/16.dalle/
- **[medium] §4 arrows / §5 transfer-vs-transformation** — 03-architecture.md, mermaid block line 17: js -.->|"poll status · POST /client webhook<br/>image.done · at-least-once · client dedup"| u
  - One edge merges two different flows and encodes the poll backwards: polling is a client request toward the job-service, but as drawn (js --> u) it reads as job-service pulling from the client, violating the request-direction-only rule. The label also carries behavior, not payload: "at-least-once" and "client dedup" are delivery semantics/outcomes, not what crosses the wire (§5). The webhook push direction itself (js -.-> u) is correct.
  - Fix: `Replace line 17 with two edges:
    u -->|"GET job status<br/>HTTPS · JSON"| js
    js -.->|"produce: image.done<br/>webhook · at-least-once"| u
(client-side dedup stays in the prose, where 03 "Deliver" already covers it)`
- **[low] §5 transfer-vs-transformation** — 03-architecture.md, mermaid block line 20: rb -.->|"warm: derived sizes"| cdn[/🌍 CDN/]
  - Edge label carries a verb phrase prefix ("warm:") instead of stating only what crosses; per §5 an outcome/trigger label like "push config" is rewritten as the data that crosses. The proactive-push-at-completion detail belongs in prose (04 deep dive 5 already states it).
  - Fix: `rb -.->|"derived sizes"| cdn[/🌍 CDN/]`

### study_cases/17.elevenlabs
- **[medium] prose-consistency** — study_cases/17.elevenlabs/03-architecture.md, edge "bjs -->|\"PUT WAV / MP3 objects\"| ob" (block line 19), vs. key data flow "Batch job" and 04-deep-dives §4
  - The batch pool node "bp[🧠 batch-pool<br/>preemptible at segment boundaries]" is a dead end: prose says "segment jobs are enqueued to the batch pool only -> outputs written as WAV objects, manifest updated", i.e. the batch pool renders the audio and its outputs are what get stored, yet bp's only edges are the two input/lookup edges to va and vr. The object write is instead drawn from batch-job-service, which the Components table describes as queue/segmentation/retries and never synthesizes audio; as drawn, a reader decodes that bjs produces the WAV bytes while bp's synthesized audio goes nowhere. Segment outputs must also be durable for the prose's resume-from-manifest guarantee.
  - Fix: `Replace the single line
    bjs -->|"PUT WAV / MP3 objects"| ob
with two lines:
    bp -->|"PUT segment outputs"| ob
    bjs -->|"PUT finalized WAV / MP3 objects"| ob`
- **[low] prose-consistency** — study_cases/17.elevenlabs/03-architecture.md, clone flow edges "ca -->|\"append attestation · instant-clone embedding\"| va" / "ca -->|\"UPSERT voice:{voice_id}\"| vr" (block lines 22–23), vs. "Voice registration (cloning)" flow and 01-requirements F4
  - The drawn /clone flow ends at the instant-clone leg and omits the professional-clone tier entirely — prose: "professional clone: training job queued on the batch pool, checkpoint written to asset store, registry entry flips to ready when landed", and F4 defines it as one of the two clone tiers. A reader of this diagram alone decodes cloning as embeddngs-only; the pro-clone fine-tuning load (one of the batch pool's jobs, which the diagram's Q about the shared fleet covers) never appears as an edge into the queue q.
  - Fix: `Add after the existing ca edges, and extend the queue label:
    q{{📨 job queue<br/>durable · synth · clone jobs}}
    ca -->|"produce: clone.train"| q`

### study_cases/18.sora
- **[medium] §6 API edges / §5 transfer-vs-transformation** — study_cases/18.sora/03-architecture.md, mermaid block, line 13
  - Edge `jo -.->"produce: draft.pass<br/>~1/10 of a job"| dp` names an event (produce: draft.pass) with no topic/bus in the design — the draft pool pulls work the same way the main pool does — and '~1/10 of a job' is a cost annotation, not the payload that crosses (§5: an edge label states only payload/event/query; the number belongs in the node label or prose, where 03/04 already state it).
  - Fix: `dp -->|"gRPC LeaseDraftJob()<br/>pull draft job"| jo`
- **[medium] §5 transfer-vs-transformation / §6 API edges** — study_cases/18.sora/03-architecture.md, mermaid block, line 17
  - Edge `jo -->|"assemble job"| pp` carries a verb phrase describing what pp does, not what crosses (§5), and carries no contract at all (no method/path, no transport, no event name) on an edge crossing two named deploy boundaries (§6: every cross-boundary edge carries the contract). Per deep dive 2 the assemble/upscale/encode stage is part of the same pull-a-job DAG, so the request direction is also a pull.
  - Fix: `pp -->|"gRPC LeaseAssembleJob()<br/>pull assemble job"| jo`
- **[low] prose-consistency** — study_cases/18.sora/03-architecture.md, mermaid block, node pin (line 10)
  - Node `pin[prompt-intake<br/>text safety classifiers<br/>prompt-hash dedupe]` claims prompt-hash dedupe but has no store edge, while every other component's store read/write is drawn (jo->mdb, pp->mdb, srv->mdb). Prose 03 Components 2: 'a (prompt-hash, params) key hitting a screened artifact returns it instead of scheduling a new job' — that key/artifact lookup lives in the metadata DB, which stores 'verdicts'. A reader of the diagram alone cannot decode where the dedupe key comes from.
  - Fix: `pin -->|"GET (prompt-hash, params)"| mdb`

### study_cases/19.frontend-chat-app/
- **[medium] §4 arrows** — study_cases/19.frontend-chat-app/03-architecture.md mermaid block, edge `v ==>|"intent: send · read"| se[sync-engine · the only writer ...]` (second `==>` in the block; the first is `u ==>|"tap send(text)"| v`)
  - Two `==>` primary-path edges in one diagram. Spec reserves `==>` for "the one data path the case is about" / exactly one primary sync path per diagram (the §7 reference example has exactly one). Here both the user's tap into the view layer and the view-to-engine intent hop carry the thick arrow.
  - Fix: `v -->|"intent: send · read"| se[sync-engine · the only writer<br/>outbox drain · seq-ordered intake<br/>reconnect backfill]`

### study_cases/22.frontend-uber/
- **[medium] §6 API edges (prose-consistency with 02/deep-dive-2 cadence policy)** — study_cases/22.frontend-uber/03-architecture.md — edge `d((🧑 drivers)) -->|"GPS ping every 3 s<br/>gRPC · proto"| gw`
  - The driver->gateway edge crosses a deploy boundary but its label `"GPS ping every 3 s<br/>gRPC · proto"` carries transport only — no RPC method name as §6 requires for RPC edges — and "every 3 s" states the moving-driver cadence as constant, while 02-scale-estimates.md and deep dive 2 define 3 s moving / 15 s waiting / paused backgrounded. A careful reader decodes a fixed 3 s protocol; the label also says what the schedule is, not what crosses the wire.
  - Fix: `d((🧑 drivers)) -->|"gRPC StreamPosition() · proto<br/>pos updates · 3 s moving / 15 s waiting"| gw`
- **[low] prose-consistency (§8 sparse-diagram intent)** — study_cases/22.frontend-uber/03-architecture.md — subgraph backend, node declarations `ts[trip service<br/>source of truth]` and `ms[matching service<br/>offers · matches]`
  - Two backend nodes are declared but have zero edges: `ts[trip service<br/>source of truth]` and `ms[matching service<br/>offers · matches]`. They render as disconnected boxes and contradict the file's own stance ("No server diagram beyond the contract: … this doc defines only their client-facing contracts"); a reader cannot decode how offers/matches reach the ride store from them. Either remove them or wire the event they emit to the gateway.
  - Fix: `Delete the two lines `        ts[trip service<br/>source of truth]` and `        ms[matching service<br/>offers · matches]` (alternative: `ms -.->|"produce: ride.events"| gw`), keeping only gw and ss in the backend subgraph.`

### study_cases/23.frontend-video-streaming
- **[medium] prose-consistency** — 03-architecture.md mermaid block, ABR controller inputs (around line 28)
  - The diagram's Q is "how does ABR steer quality", but ABR's only drawn input is `bm -->|"buffer level secs"| abr` — the throughput samples from segment fetches are missing, so a reader decodes ABR as buffer-only BOLA. Prose disagrees with that reading: 03 Components says "Consumes throughput samples (per segment fetch) and buffer level", F3 says "based on measured throughput and buffer state", and 04 §1's decision is "hybrid: throughput sets the ceiling, buffer picks within it".
  - Fix: `Insert after the `bm -->|"buffer level secs"| abr` line the edge: `    sched -->|"throughput samples"| abr``
- **[low] §5 transfer-vs-transformation** — 03-architecture.md line 20: `u((🧑 viewer)) -->|"press play · seek"| sc`
  - Edge label `-->|"press play · seek"|` is a verb phrase. §5: an edge label states only what crosses (payload/event/query); "press" is a trigger verb, and the spec explicitly says triggers get rewritten as the data that crosses.
  - Fix: ``u((🧑 viewer)) -->|"play · seek commands"| sc``
- **[low] §3 labels** — 03-architecture.md line 13: `sched[segment scheduler<br/>only component on network]`
  - Node ID `sched` is 5 lowercase letters; §3 caps node IDs at "1–4 lowercase letters". The other compliant diagrams in the corpus (09.cdn, 13.kafka, etc.) all stay ≤4, so this breaks the convention.
  - Fix: `Rename the ID `sched` to `sch` in the node def line `sched[segment scheduler<br/>only component on network]` and in its 7 edge references (the ==> cdn edge, --> wrk, the two bm/abr targets, "target quality", "commands" edges), e.g. `    sch[segment scheduler<br/>only component on network]` and `    sch ==>|"GET /videos/{id}/manifest.m3u8 · /segments/*.m4s<br/>HTTPS · fMP4"| cdn[/🌍 CDN<br/>edge vendor/]``

### study_cases/24.mobile-chat-app
- **[medium] prose-consistency** — 24.mobile-chat-app/03-architecture.md, mermaid block line: u((🧑 user)) ==>|"composed msg"| ctl
  - The user's composed message enters the controller directly: "u((🧑 user)) ==>|\"composed msg\"| ctl", leaving the drawn ui node output-only (its only in-edge is "st -.-> ui"). Prose F1 states the send path starts "UI -> controller", so the diagram contradicts the prose on the primary path of its own question.
  - Fix: `    u((🧑 user)) ==>|"composed msg"| ui
    ui -->|"composed msg"| ctl`
- **[medium] prose-consistency** — 24.mobile-chat-app/03-architecture.md, mermaid block edge: push -.->|"notification<br/>seq range N"| conn
  - The push-delivery edge "push -.->|\"notification<br/>seq range N\"| conn" asserts the push notification carries a sequence range. 04-deep-dives §1 (chosen design) says "Push carries only conversation-ID/sender metadata (no plaintext payload)", and F2 says the client wakes, reconnects, and the sync engine compares cursors — the push carries no sequence data, so the label is an invented payload that could be read as push-driven sync.
  - Fix: `    push -.->|"notification<br/>conversation-id · sender metadata"| conn`

### study_cases/25.mobile-newsfeed/
- **[medium] syntax** — study_cases/25.mobile-newsfeed/03-architecture.md, edge fc --> api (~line 29)
  - Edge label "GET /feed?cursor=<opaque><br/>HTTPS · JSON" — the raw HTML-like fragment `<opaque>` is parsed as an unknown tag by Mermaid's htmlLabels renderer and silently stripped even though the label is quoted. Verified by rendering with mermaid-cli: the output SVG contains `GET /feed?cursor=` and zero occurrences of the text `opaque`. A reader of the rendered diagram cannot see the load-bearing fact that the cursor is opaque (prose: "paged by opaque cursor"), and the truncated label looks like a mistake.
  - Fix: `fc -->|"GET /feed?cursor={opaque}<br/>HTTPS · JSON"| api`
- **[medium] prose-consistency** — study_cases/25.mobile-newsfeed/03-architecture.md, edge ip --> cdn (line 33)
  - Edge label "GET /img/{variant_url}<br/>HTTPS · JPEG" states the format as JPEG, but 02-scale-estimates.md says the optimized ~60 KB variants at ~1080 px are "WebP/HEVC-class compression". The diagram contradicts the written estimate on the media format a reader decodes from the edge.
  - Fix: `ip -->|"GET /img/{variant_url}<br/>HTTPS · WebP/HEVC"| cdn`

### study_cases/26.google-maps/
- **[medium] §6 edge contracts** — study_cases/26.google-maps/03-architecture.md, flowchart block, edge `mm -.->|"matched pings"| agg` (line 23)
  - The service-to-service async handshake between two separately drawn deployed units (map matcher -> segment aggregator) carries neither produce:/consume: + event name, nor an RPC/HTTP contract — just a bare payload. Every other async edge in this block follows the §6 event contract (u -.->|"produce: gps.ping"|, bus -.->|"consume: gps.ping"|), so this one reads as an undocumented event transfer into a rectangle.
  - Fix: `    mm -.->|"produce: segment.matched"| tk{{📨 matched pings<br/>topic · by region}}
    tk -.->|"consume: segment.matched"| agg`
- **[low] §6 edge contracts** — study_cases/26.google-maps/03-architecture.md, flowchart block, edge `pipe[...] -->|"rendered tiles · epoch id<br/>HTTPS · protobuf"| tile` (line 25)
  - Cross-deploy-boundary edge carries a payload ("rendered tiles · epoch id") and transport · format but no METHOD /path, which §6 requires for HTTP edges. A reader decodes the payload but not the contract.
  - Fix: `pipe[map-data pipeline<br/>imagery -> graph -> tile render<br/>region-sharded · re-customizes rout] -->|"PUT /tiles/{epoch}<br/>HTTPS · protobuf"| tile`
- **[low] §1 shapes / §6 events** — study_cases/26.google-maps/03-architecture.md, flowchart block, edge `pipe -.->|"produce: map-edit"| rout` (line 26)
  - An event edge (produce: map-edit) terminates on a cylinder store. Per §1/§7 the produce:/consume: event API targets a hexagon topic; an event flowing into a keyed store is the only shape/edge mismatch in the block and could be misread as the routing graph being a log. The pipe label already carries "re-customizes rout", so the store-write reading is the closest decode.
  - Fix: `pipe -.->|"write region graph edits"| rout`

### study_cases/27.ev-charging-network/
- **[medium] prose-consistency** — 03-architecture.md diagram, edges into `av["availability svc..."]` and `dv["device mgmt..."]` from `bus`
  - Two consume edges have no producer anywhere in the diagram: `bus -.->|"consume: status"| av` and `bus -.->|"consume: commands-ack"| dv`. The gw produce edge `gw -.->|"2· produce: tx.events · meter-values"| bus` only names tx.events/meter-values, but per the prose (Components #2 partitioning lists 'status by charger'; Components #8 'per-device ACK tracking'; deep dive 5 'gateway connection-loss events feed the same normalizer') the gateway relays charger status and command ACKs onto the bus. Since the Q covers occupancy over OCPI, a reader can decode 'status' as appearing from nowhere.
  - Fix: `Add one line after the edge-2 line: `gw -.->|"produce: status · commands-ack"| bus``
- **[medium] §5 transfer-vs-transformation** — 03-architecture.md diagram, edge `oca -.->|"token list → cache fill"| tok`
  - Edge label `|"token list -> cache fill"|` mixes a transformation chain and an outcome ('cache fill') into an edge label. §5: an edge only moves data; the transformation belongs in a node label.
  - Fix: `oca -.->|"token list"| tok`
- **[medium] §4 arrows** — 03-architecture.md diagram, edge `em -.->|"PUT /oicg/tokens<br/>OCPI 2.2.1 · JSON"| oca`
  - The partner push `em -.->|"PUT /oicg/tokens<br/>OCPI 2.2.1 · JSON"| oca` is drawn dotted (async, caller does not wait) while its sibling inbound push `em -->|"POST /ocpi/cdrs<br/>..."| oca` is solid (sync). Both are external HTTP POST/PUT requests the partner waits on (deep dive 5: OCPI pushes are individual requests with a per-CDR ACK); the mixed semantics decode as tokens arriving queued but CDRs synchronous, which the prose does not claim.
  - Fix: `em -->|"PUT /oicg/tokens<br/>OCPI 2.2.1 · JSON"| oca`
- **[medium] prose-consistency** — 03-architecture.md diagram, node `em -->|"POST /ocpi/cdrs<br/>ocpi 2.2.1 · JSON"| oca` chain — no edge from `oca` to `led`
  - The inbound-roam flow dies at the adapter. Prose (Key data flows, 'Inbound roam'): inbound CDRs '-> validation -> priced against our tariff -> ledger entry in the "we owe partner X" direction', but the diagram has no edge from `oca` to `led["billing ledger"]`; the Q covers 'partners exchange ... CDRs', so the missing leg hides where the money lands.
  - Fix: `Add: `oca -.->|"append inbound CDR entry"| led``
- **[low] §5 transfer-vs-transformation** — 03-architecture.md diagram, edge `tok -->|"POST /oicg/authorize<br/>OCPI 2.2.1 · JSON (cache miss only)"| em`
  - Edge label `|"POST /oicg/authorize<br/>OCPI 2.2.1 · JSON (cache miss only)"|` carries the outcome qualifier 'cache miss only' on the edge; §5 says an outcome is not payload — the prose (Components #3) already owns the cache-miss behavior.
  - Fix: `tok -->|"POST /oicg/authorize<br/>OCPI 2.2.1 · JSON"| em`
- **[low] prose-consistency** — 03-architecture.md diagram, edge `em -->|"POST /ocpi/cdrs<br/>ocpi 2.2.1 · JSON"| oca`
  - Label text 'ocpi 2.2.1' is lowercase, inconsistent with every other OCPI version label in the same diagram ('OCPI 2.2.1' on the gw/tok/em/oca edges).
  - Fix: `em -->|"POST /ocpi/cdrs<br/>OCPI 2.2.1 · JSON"| oca`

### study_cases/29.uber-eats/
- **[medium] §5 transfer-vs-transformation** — 03-architecture.md, block 1, edge cdn -> cat
  - Edge label `-->|"miss: GET /menus/{id}"| catalog svc` describes a cache outcome, not what crosses — DIAGRAMS.md §5 explicitly names `miss` as a banned label ("An outcome or trigger is not a payload: `miss`, `hit 95%`, `push config` get rewritten as the data that crosses").
  - Fix: `cdn -->|"GET /menus/{id}<br/>origin fetch"| cat["catalog svc<br/>versioned menu snapshots"]`
- **[medium] §6 API edges** — 03-architecture.md, block 1, edge os -> psp (payment PSP)
  - Edge label `-->|"payment auth<br/>HTTPS · JSON"|` is a verb phrase with no API contract — §6 requires METHOD /path + transport · format (or an RPC method name) on every cross-deploy-boundary edge; 'payment auth' names the transaction, not the contract.
  - Fix: `os -->|"POST /payments/authorize<br/>HTTPS · JSON"| psp[/"payment PSP"/]`
- **[medium] prose-consistency** — 03-architecture.md, block 1, edges into/from tr (tracking svc) referencing WS pos + ETA bands
  - Node label `tr["tracking svc<br/>per-order rooms · latest-wins"]` with edge `tr <-->|"WS pos + ETA bands<br/>wss · JSON"| cu` promises ETA bands, but no edge in the diagram ever delivers ETA data to tracking — ETA svc's only edge is the dispatch-side `ds -->|"gRPC GetEta()"| eta`, so the diagram implies tracking pushes ETAs it never receives (prose data-flow 8: 'positions and ETAs re-derive in the ETA service', and flow 3: customer 'gets an initial committed ETA band (ETA service)'). A reader could conclude tracking derives ETAs itself, contradicting the ETA service's ownership.
  - Fix: `eta -.->|"produce: eta.updated"| ob{{"📨 event bus<br/>orders.* · dispatch.*<br/>location.* · payments.*"}}`

### study_cases/30.facebook-messenger
- **[medium] §1 node shapes** — study_cases/30.facebook-messenger/03-architecture.md, mermaid block lines 16-17 (diagram starts line 8)
  - Node `mb[("per-user mailbox store<br/>rows (user, thread, seq)<br/>unread counters")]` is drawn as a keyed-store cylinder, but two edges source it as an event stream: `mb -.->|"consume: mailbox.append"| ib["inbox svc<br/>pagination · snippets"]` and `mb -.->|"consume: mailbox.append"| si[("🔍 search index<br/>derived from mailbox")]`. DIAGRAMS.md §1 says "cylinder only if every access is keyed read/put; if consumers read it as a log (consume:), draw hexagon" — and every other case in the repo sources `consume:` edges from a hexagon event log/bus (28.uber `el{{...}}`, 27.ev-charging `bus{{...}}`, 05.discord `k{{...}}`), never from a cylinder. A reader decodes the mailbox store itself as publishing a CDC event stream, which the case's prose never states anywhere (03 prose: "inbox ... updated async from the mailbox rows"; dive 1 anchor: "per-user mailboxes fed by a unidirectional message log, with the inbox built as mail-service views").
  - Fix: `Move the event to the actual writer (fanout) and give it a proper topic node; replace the two lines:
    mb -.->|"consume: mailbox.append"| ib["inbox svc<br/>pagination · snippets"]
    mb -.->|"consume: mailbox.append"| si[("🔍 search index<br/>derived from mailbox")]
with:
    fs -.->|"produce: mailbox.append"| ev{{"📨 mailbox log"}}
    ev -.->|"consume: mailbox.append"| ib["inbox svc<br/>pagination · snippets"]
    ev -.->|"consume: mailbox.append"| si[("🔍 search index<br/>derived from mailbox")]
(3 lines replace 2, adds 1 node for 12 total, still under the ~15 cap; renders clean under mermaid-cli.)`

### study_cases/31.netflix
- **[medium] §4 arrows** — 31.netflix/03-architecture.md, line 20 (edge 1, member → control-plane)
  - The thick primary arrow is on edge 1: `m ==>|"1· GET /ui/rows<br/>HTTPS · JSON"| cp[...]` — but the one data path the case is about is byte delivery via Open Connect (§4: `==>` is reserved for the primary sync path, "the one data path the case is about"; the case thesis is that delivery is the product and "the control plane is not touched again"). A reader taking the thick edge as the story decodes browse as primary and the segments fetch as secondary.
  - Fix: `m -->|"1· GET /ui/rows<br/>HTTPS · JSON"| cp["control plane<br/>UI rows: candidates -> rank"]`
- **[medium] §4 arrows** — 31.netflix/03-architecture.md, line 25 (edge 4, member → Open Connect)
  - Companion to the above: the byte-fetch edge is the primary path but is drawn thin — `m ==>|"4· GET /titles/{id}/segments<br/>HTTP range"| oc` has plain `-->`. Swap the two decorations so the single `==>` lands on the delivery edge.
  - Fix: `m ==>|"4· GET /titles/{id}/segments<br/>HTTP range"| oc`
- **[medium] prose-consistency** — 31.netflix/03-architecture.md, diagram missing edge member → entitlements+DRM (between lines 25 and 26)
  - The diagram's own Q: caption covers DRM brokerage, and prose playback step 3 is "Device obtains a DRM license bound to its CDM (F9) — this is the money gate". The member node never touches the entitlements+DRM node: the license appears only inside `en`'s label chain (`-> key server · license -> CDM`), so edges alone read as if the license never crosses to the device, and the numbered edges jump 1·, 2·, 4· — step 3 is absent.
  - Fix: `m -->|"3· POST /license<br/>HTTPS · CENC/key"| en`
- **[low] §6 API edges** — 31.netflix/03-architecture.md, line 24 (edge playback-svc → steering)
  - RPC contract edge names only args and transport — `ps -->|"titleId · device class · probes<br/>gRPC · proto"| sd[...]` — but §6's RPC form requires a method name (`-->|"gRPC GetDegree()"|`). Every other RPC edge in the block (AuthorizeSession) has one; this edge does not decode alone.
  - Fix: `ps -->|"gRPC SteerSession(titleId · device probes)<br/>gRPC · proto"| sd["steering<br/>OCA choice -> steered appliance URL<br/>throughput + capacity probes"]`

### study_cases/32.spotify
- **[medium] §5 transfer-vs-transformation** — study_cases/32.spotify/03-architecture.md, mermaid block, edge `pop -->|"GET audio/{track}-{rendition} on miss"| ob`
  - The edge label `pop -->|"GET audio/{track}-{rendition} on miss"| ob` ends with an outcome clause ("on miss"), not the data that crosses. §5 names "miss" explicitly as an outcome that must be rewritten as the payload; §4 already makes the cache-miss fetch a plain sync `-->` OP-arg edge, so "on miss" adds a decode-able but non-conforming outcome word. The parallel `cdn -->|"GET audio/{track}-{rendition}"| ob` shows the conforming form.
  - Fix: `pop -->|"GET audio/{track}-{rendition}"| ob`
- **[medium] §3 labels** — study_cases/32.spotify/03-architecture.md, mermaid block, node declaration `pe[playback edge<br/>regional gateway: CDN vs own POP]`
  - The node label `pe[playback edge<br/>regional gateway: CDN vs own POP]` is unquoted yet contains `:` ("gateway: CDN vs own POP"). §3 requires quoting any label containing `:`. It does not break mermaid parsing here, so it is a spec-violation finding, not a render failure.
  - Fix: `pe["playback edge<br/>regional gateway: CDN vs own POP"]`

### study_cases/33.instagram
- **[medium] prose-consistency** — study_cases/33.instagram/03-architecture.md, mermaid edge `ev -.->|"consume: story.created"| stor`
  - Edge `ev -.->|"consume: story.created"| stor` consumes an event that no node produces. The client's story-post edge is absent (c only shows POST /v1/media originals, GET /v1/story/tray, POST /v1/engage, GET /{rendition-key}), yet prose flow 3 and 02-scale-estimates (~1B story items/day in the upload math) make story posting a first-class post path. A reader cannot decode where story.created originates.
  - Fix: `Add after the `ing -->|"SQL INSERT post row<br/>state = PENDING"| meta` line: `ing -.->|"produce: story.created"| ev``
- **[medium] prose-consistency** — study_cases/33.instagram/03-architecture.md, feed-api node `feed["feed-api<br/>merge pools · celebrity on-read → rank → screenful"]` and its source edges
  - The feed-api node label promises "celebrity on-read -> rank -> screenful" and prose flow 2 lists "on-read merges for followed celebrities", but feed-api has only three source edges (MGET pool:uid, gRPC Rank(), GET /v1/candidates to explore). The author-level recent-posts index named in 04-deep-dives #3 ("merge reads it directly") appears as no node or edge, so the feed read — half of this diagram's stated question — cannot be decoded for celebrity posts.
  - Fix: `Add after `feed -->|"MGET pool:uid"| pool`: `feed -->|"GET celebrity recent-posts index"| meta``
- **[low] prose-consistency** — study_cases/33.instagram/03-architecture.md, numbered client edges 2-5 (user to story svc seen-state write)
  - Client edges cover post, feed read, tray read, engage and CDN bytes, but the per-viewer seen-state write (prose flow 3 "view -> seen-state write", ~7.5B/day in 02) has no edge: only the resulting `produce: story.seen` event is drawn, with its trigger undrawn. Lower impact because the stated question is "on post and on feed read".
  - Fix: `Add `c -.->|"POST /v1/story/view · seen mark"| stor` (async fire-and-forget view mark)`

### study_cases/34.reddit
- **[medium] §5 transfer-vs-transformation** — study_cases/34.reddit/03-architecture.md, read-path diagram, line 102 (cmt → cass edge)
  - Edge label carries an outcome, "on cache miss", not what crosses (§5: edge labels are payload/event/query only; outcomes like "miss" must live in prose or the node label). Verbatim: cmt -->|"CQL prefix scan<br/>on cache miss"| cass
  - Fix: `cmt -->|"CQL prefix scan"| cass`
- **[medium] §6 API edges** — study_cases/34.reddit/03-architecture.md, both diagrams: line 44 (write path) and line 99 (read path), edge → gw
  - Cross-deploy-boundary edge (edge -> gateway) has no contract: "HTTP proxy" is not a METHOD /path and not an RPC method (§6; cf. 09.cdn's "3· GET /objects/{key} keep-alive" pattern). Verbatim: edge -->|"HTTP proxy<br/>HTTPS · JSON"| gw
  - Fix: `line 44: edge -->|"POST /submit<br/>HTTPS · JSON"| gw    — line 99: edge -->|"GET /v1/feed<br/>HTTPS · JSON"| gw    (forward the original client contract)`
- **[medium] prose-consistency** — study_cases/34.reddit/03-architecture.md, read-path diagram, line 110 (k -.-> rc consume edge)
  - The event log node is labeled "votes · content · mod events" and the diagram's question is "how does the event log keep those caches warm?", but the refresher consumes only content.events. The prose (data flows 3–4: "rank caches refresh on a short lag", "removed content drops out of feeds within seconds"; deep dives 3/5: velocity windows fed by vote consumers) requires vote.cast and mod.action to feed the rank/candidate caches, and the read-path diagram draws no consumer for either — a reader sees warm caches driven by content events alone.
  - Fix: `k -.->|"consume: content.events · vote.cast · mod.action"| rc`
- **[low] §6 API edges (store edges OP-arg)** — study_cases/34.reddit/03-architecture.md, write-path diagram, line 53 (cmt -.-> cass)
  - Store edge lacks the OP-arg form §6 requires for store edges; it names the payload, not the operation. Verbatim: cmt -.->|"comment rows<br/>async reconcile"| cass
  - Fix: `cmt -.->|"PUT comment rows<br/>async reconcile"| cass`

### study_cases/35.tiktok
- **[medium] §4/§6 arrows + edge contracts (produce:/consume: must terminate at a log node, per §1 hexagon = topic/log)** — study_cases/35.tiktok/03-architecture.md, ingest plane edge from upl (upload svc) to xc (transcode farm)
  - `upl -.->|"produce: transcode.job"| xc` labels an async-event edge with produce:, but the edge terminates at the transcode farm's worker rectangle instead of a topic/log hexagon. Every other produced event in this same diagram routes through the k hexagon (produce: video.publish, produce: rendition.ready, produce: user.events), so the transcode job queue is the only log in the design that is drawn as invisible; a reader could conclude the upload service calls into the farm directly. The prose (dive 3) explicitly declares it a first-class entity: 'Priority transcode queue ordered by predicted audience' — that priority note is exactly what would live on the missing hexagon.
  - Fix: `Replace `    upl -.->|"produce: transcode.job"| xc` with two lines:
    upl -.->|"produce: transcode.job"| tq{{"📨 transcode queue<br/>priority by predicted audience"}}
    tq -.->|"consume: transcode.job"| xc
and drop `priority by predicted audience` from xc's third label line (it moves to the queue, leaving `xc["transcode farm<br/>per-title ladder"]`).`
- **[medium] §5 transfer vs transformation (edge labels state only what crosses)** — study_cases/35.tiktok/03-architecture.md, edge from xc (transcode farm) to ob (origin blob store)
  - `xc -->|"PUT rung 1 first<br/>prioritized for publish"| ob` — the second line 'prioritized for publish' is a policy/outcome phrase, not data crossing the wire (§5 allows only payload/event/query in edge labels). It also duplicates the immediately following `xc -->|"PUT ladder renditions"| ob`, which carries the same contract (PUT into ob), so the pair reads as two different transfers when it is one segmented upload under one priority policy (dive 3's 'publish low rungs first').
  - Fix: `Delete both xc->ob lines and replace with the single edge
    xc -->|"PUT ladder renditions"| ob
The priority policy is already in xc's node label ('priority by predicted audience') and prose (dive 3), so nothing is lost by removing it from the edge.`

### study_cases/37.ticketmaster
- **[low] §4 arrows** — study_cases/37.ticketmaster/03-architecture.md, inv→cach edge (line 27)
  - `inv -.->|"PUT section aggregates"| cach` — a cache/store operation drawn as a dotted async edge. §4 reserves `-.->` for events/queue/replication and says store operations are `-->`; "PUT section aggregates" is an OP-store write, not an event, so the reader cannot tell if it is a sync store edge or an event and it is neither form cleanly.
  - Fix: `inv -->|"PUT section:aggregates"| cach`
- **[low] syntax** — study_cases/37.ticketmaster/03-architecture.md, line 20 (inv) and lines 22/26 (log)
  - `ord -->|"commit(hold-id)<br/>gRPC · proto"| inv[inventory svc<br/>per-event shard<br/>CAS · TTL holds]` — node `inv` redefined with a full label although already declared on line 16; and `log{{📨 event log<br/>seat + order transitions}}` fully redefined on lines 22 and 26 after line 21. Labels/shapes are byte-identical so it renders correctly (not a parse failure), but it is redundant duplication that can drift per §3.
  - Fix: `ord -->|"commit(hold-id)<br/>gRPC · proto"| inv`
- **[low] syntax** — study_cases/37.ticketmaster/03-architecture.md, line 22 (log redefinition)
  - `inv -.->|"produce: hold.expired"| log{{📨 event log<br/>seat + order transitions}}` — node `log` redefined with its full label although already declared on line 21; identical label and shape so it renders as one node, but the redefinition should be dropped.
  - Fix: `inv -.->|"produce: hold.expired"| log`

### study_cases/38.leetcode
- **[medium] §4 arrows (request direction)** — study_cases/38.leetcode/03-architecture.md, line 21 (edge tc --> sbx)
  - The test-case store access is drawn in the wrong direction: `tc[(test-case store<br/>hidden tests · read-only)] -->|"mount 1 case"| sbx[[untrusted run...]]`. "mount 1 case" is a store read/operation performed as part of the run, and §4 says draw the request direction with the requester's edge pointing INTO the store (like `a -->|"⚡ MGET feed:{uid}"| rd`); as drawn the store pushes data out, contradicting the read semantics (deep dive 1: test data is mounted read-only one case at a time — the run side mounts it).
  - Fix: `sbx -->|"mount 1 case"| tc`
- **[medium] §4 arrows** — study_cases/38.leetcode/03-architecture.md, line 27 (edge lb --> dlq)
  - A gRPC call is drawn as an async edge: `lb -.->|"gRPC PushDeltas()<br/>gRPC · proto"| dlq`. Per §4, `-->` means synchronous call (API calls, RPC) and `-.->` is reserved for events/queue consumption/replication/log shipping; PushDeltas() is a sync RPC the caller waits on, so the dotted arrow mis-decodes as an async event stream. (The redundant second line `gRPC · proto` should also drop, per §6's own example `-->|"gRPC GetDegree()"|` which needs no transport line when the method name carries it.)
  - Fix: `lb -->|"gRPC PushDeltas()"| dlq`

### study_cases/39.e-commerce
- **[medium] §6 API edges** — study_cases/39.e-commerce/03-architecture.md, edge co --> ord (line 32)
  - Cross-boundary RPC edge uses a verb phrase instead of a method name, and the transport is incomplete: `co -->|"create order<br/>gRPC"| ord[order svc<br/>state machine]`. Every sibling RPC edge uses `Reserve()<br/>gRPC · proto` form; this one says `create order` and `gRPC` without `· proto`, so it does not decode alone per §6.
  - Fix: `co -->|"CreateOrder()<br/>gRPC · proto"| ord[order svc<br/>state machine]`
- **[medium] §6 store edges (OP-arg form)** — study_cases/39.e-commerce/03-architecture.md, edge proj --> cach (line 35)
  - Store edge is not in OP-arg form: `proj -->|"PUT read models"| cach`. "read models" is a noun category, not an operation argument; a reader cannot decode the key. The cache edge it mirrors (`⚡ MGET pdp:{sku}`) shows the intended form.
  - Fix: `proj -->|"PUT pdp:{sku}"| cach`
- **[medium] prose-consistency** — study_cases/39.e-commerce/03-architecture.md, edge proj --> cach (line 35) and node idx (line 27)
  - Missing flow: the projection-consumers node includes `search indexer`, and 03 prose Flow 2 says "search indexer updates postings" (deep dive 1 also makes the index a projection over the spine), but the diagram gives idx[(🔍 search index ...)] no producer at all. Within the diagram's own question (how read models come off the spine), the search index appears as a source with no write path — the indexer member of `proj` writes to cach but never to the index it serves (se --> idx).
  - Fix: `proj -->|"PUT postings"| idx`

### study_cases/41.web-crawler
- **[medium] prose-consistency** — study_cases/41.web-crawler/03-architecture.md, diagram node `dd` (line 32: `ex -.->|"produce: content.record"| dd[dedup gate<br/>SimHash vs corpus]`)
  - The dedup gate node `dd[dedup gate<br/>SimHash vs corpus]` is labeled as comparing against a corpus of existing signatures, but it has no edge to any signature store — its only inputs are the `content.record` event and nothing else, so a reader sees the gate emit near-dup flags from nothing. The case's Q: caption explicitly names 'dedup', and prose 03 §3.6 ('SimHash compared against a corpus index of existing signatures') plus deep dive 2b ('the index can be shard- or trie-organized so Hamming ≤ k lookups don't require a linear scan') define a real, separately-deployed signature index the diagram omits. Every other stateful operation in the diagram (seen-set lookup, fetch status, blob store) has its store drawn; this one doesn't.
  - Fix: `Add after the `ex -.->|"produce: content.record"| dd` line: `dd -->|"GET {simhash} k-near"| si[(🗄️ signature index<br/>SimHash corpus · Hamming ≤ k lookup)]``
- **[medium] §5 transfer-vs-transformation** — study_cases/41.web-crawler/03-architecture.md (line 33: `dd -.->|"produce: content.flagged"| st{{📨 stream<br/>partitioned log<br/>at-least-once}}`)
  - The edge `dd -.->|"produce: content.flagged"| st` names the event crossing as `content.flagged`, which reads as if only flagged near-duplicates reach the downstream stream. Prose 03 §3.6 says the opposite: the gate 'does not block handoff, adds a flag' — exact matches are discarded but everything else flows downstream, flagged or not, and the indexer/Indexer consumers expect all content records (FR5, deep dive 5 'consumers dedup by URL+content-hash'). 'flagged' is an outcome annotation, not the payload — per §5 the label should state what crosses (the content record), with the flagging behavior living in the node's label.
  - Fix: ``dd -.->|"produce: content.record"| st{{📨 stream<br/>partitioned log<br/>at-least-once}}` and change the dd label to `dd[dedup gate<br/>SimHash vs corpus<br/>flag near-dups]`

### study_cases/42.linkedin
- **[medium] prose-consistency** — study_cases/42.linkedin/03-architecture.md, node `cn[connection svc<br/>invite state machine]` and its edges (lines 12-13)
  - The connection service is drawn with exactly one outgoing edge: `cn ==>|"atomic edge upsert<br/>accept: both-or-neither"| gs`. The prose puts pending-invite state in its own authoritative store — 03-architecture.md Component table: "Connection service | ... invite rows; adjacency lists in KV store", key data flows: "A->B invite written as `PENDING` (unique pair constraint)", 04-deep-dives.md §1: "invite state in a relational table (small, transactional)" — but that store appears nowhere in the diagram, so the accept txn looks like the only write cn ever makes and invite state looks unpersisted.
  - Fix: `Add node `iv[(🐬 MySQL invites<br/>PENDING->ACCEPTED · unique pair (a,b))]` and edge `cn -->|"1· INSERT invite state=PENDING"| iv` (before the accept edge, so the txn that flips iv to ACCEPTED + writes gs two direction rows is the reader-visible commit).`
- **[medium] §5 transfer-vs-transformation** — study_cases/42.linkedin/03-architecture.md line 13: `cn ==>|"atomic edge upsert<br/>accept: both-or-neither"| gs`
  - The primary-path label is a property of the transaction, not what crosses: §5 ("An edge only moves data. Its label states what crosses") — `cn ==>|"atomic edge upsert<br/>accept: both-or-neither"| gs` is a verb phrase plus an outcome clause, and it is not the §6 OP-arg form the other store edges use.
  - Fix: `cn ==>|"PUT edge (a,b)<br/>2 direction rows · 1 txn"| gs`
- **[low] prose-consistency** — study_cases/42.linkedin/03-architecture.md line 23: `eb -.->|"consume: profile.edits · edge.changes"| jobs` vs jobs label on line 33-34
  - jobs' label claims WVYP (`jobs[stream jobs<br/>invalidators · PYMK · WVYP]`) but its consume edge is `eb -.->|"consume: profile.edits · edge.changes"| jobs` — neither event is a view event, yet per 04-deep-dives.md §5 WVYP aggregation is driven by `profile.views`. As drawn, the view stream reaches only ml and nt, so the jobs node appears to run WVYP on events that never arrive.
  - Fix: `eb -.->|"consume: profile.edits · edge.changes · profile.views"| jobs`

### study_cases/44.payment-system/
- **[medium] §5 transfer-vs-transformation** — 03-architecture.md diagram, edge gw→idem (line 10): `gw -->|"2· GET idem:key · INSERT on miss"|`
  - Edge label `gw -->|"2· GET idem:key · INSERT on miss"|` contains "on miss" — §5 explicitly names `miss` as an outcome that must be rewritten as the data that crosses. The conditional behavior is already decoded by the node label (key -> result/in-flight); the edge should state only the operations and key.
  - Fix: `gw -->|"2· GET idem:key · INSERT idem:key"| idem[(idempotency store<br/>key -> result/in-flight · TTL ~24 h)]`
- **[medium] prose-consistency** — 03-architecture.md diagram, edge pay⇒ga (line 13): `pay ==>|"5· capture intent<br/>gRPC · proto"|` vs edge ga⇒nw (line 14): `==>||"6· Auth request + uniqueness ref"|`
  - Edge 5 says `capture intent` but edge 6 (its destination onto the bank seam) says `Auth request + uniqueness ref` — capture named before the auth request, and the flagship flow in the prose (03-architecture.md Key data flows, deep dives §3) is auth-only at the acquirer during charge; capture is a separate later merchant action within the auth hold window. A careful reader decodes the chain as capture-then-auth, contradicting the prose. The Q: caption asks how the intent gets *authorized* — the adapter call here is the authorize dispatch.
  - Fix: `pay ==>|"5· gRPC Authorize()<br/>gRPC · proto"| ga[gateway adapters<br/>per-rail protocol + credentials<br/>timeouts · uniqueness ref to bank]`
- **[low] §5/§6 store edge form** — 03-architecture.md diagram, edge rec→led (line 20): `rec -->|"SQL scheme-ref+amount match"| led`
  - Edge label `-->|"SQL scheme-ref+amount match"|` states the verb/work ("match") rather than the data crossing; the matching work is already in rec's node label (`match settlement files -> breaks queue`). §6 store edges want the OP/SQL + argument form naming what the query selects.
  - Fix: `rec -->|"SQL postings by scheme-ref + amount"| led`
- **[low] §6 API edges** — 03-architecture.md diagram, edge nw-.->rec (line 19): `nw -.->|"daily settlement file<br/>HTTPS/SFTP · CSV"|`
  - Edge carries payload + transport + format but no METHOD/verb — §6 requires METHOD /path + transport · format (or a named RPC or produce:/consume:) on every edge crossing a deploy boundary. A vendor file drop has no REST method, but the delivery action is unspecified.
  - Fix: `nw -.->|"PUT /settlement/{acquirer}/{date}<br/>SFTP · CSV"| rec[reconciliation<br/>match settlement files -> breaks queue]`

### study_cases/45.typeahead
- **[medium] prose-consistency** — 45.typeahead/03-architecture.md, edge "k -.->"consume: query.logs"| bl" (line 16)
  - The topic's two consume edges use different event names: "t -.->|"produce: query.events"| k" is followed by "k -.->|"consume: query.events"| hb" and "k -.->|"consume: query.logs"| bl". The event named in the 02 estimate sheet is one stream ('800M/day ... fed to counters + batch build'), yet 'query.logs' is consumed without anyone producing it — a reader decodes two topics where the prose has one feed.
  - Fix: `k -.->|"consume: query.events"| bl`

### study_cases/46.google-docs
- **[medium] §5 transfer-vs-transformation** — study_cases/46.google-docs/03-architecture.md, diagram line 15: `ph -.->|"cursor fanout"| ed`
  - Edge label "cursor fanout" is a verb/outcome phrase, not what crosses the edge. §5: an edge label states what crosses (`payload/event/query`), never an outcome or verb — 'cursor fanout' describes the hub's action, so a reader must guess the payload is cursor updates.
  - Fix: `ph -.->|"cursors @ 1–2 Hz"| ed`
- **[medium] §2 icons** — study_cases/46.google-docs/03-architecture.md, diagram line 17: `k -.->|"consume: doc.changed"| nt[notifications<br/>dedupe · throttle per user]`
  - `notifications` is in the §2 emoji table (notifications / push -> 🔔) but the label is bare. House style applies the emoji (42.linkedin: `nt[🔔 notification svc<br/>push + email · throttle]`); the standard says table-covered component types carry the emoji at the start of the label, with the sized name also present.
  - Fix: `k -.->|"consume: doc.changed"| nt[🔔 notifications<br/>dedupe · throttle per user]`
- **[medium] §2 icons** — study_cases/46.google-docs/03-architecture.md, diagram line 13: `cs -->|"GET /acl/{docId}<br/>HTTPS · JSON"| ac[ACL service<br/>per-doc + link-sharing]`
  - `ACL service` is a table-covered component type (auth / identity -> 🔑) but the label is bare; other cases mark this type (32.spotify: `ent["🔑 entitlement…"]`). Emoji must prefix the label while the sized name stays present.
  - Fix: `cs -->|"GET /acl/{docId}<br/>HTTPS · JSON"| ac[🔑 ACL service<br/>per-doc + link-sharing]`

### study_cases/47.rate-limiter/
- **[low] §1 node shapes** — study_cases/47.rate-limiter/03-architecture.md, node `an` in the mermaid block: `k -.->|"consume: decision"| an[usage dashboards<br/>billing rollups]`
  - One rectangle merges two distinct components named separately in the prose — "usage dashboards" and "billing rollups" — while the third consumer (abuse engine) gets its own node; the shape table defines a rectangle as one named deployed unit.
  - Fix: `k -.->|"consume: decision"| an[usage dashboards]
    k -.->|"consume: decision"| br[billing rollups]`

### study_cases/48.google-search
- **[medium] §4 arrows** — 03-architecture.md line 36: `ft -->|"fresh segments"| mr`
  - Store-to-worker edge `ft -->|"fresh segments"| mr` points out of the store with a bare noun label. §4 says draw the request direction only — the bg merger initiates a read from the fresh tier (deep dive 3: 'a background promotion merges fresh segments'); as drawn, the fresh tier appears to push segments, and the label is not the OP-arg form store edges carry.
  - Fix: `    mr -->|"read segments"| ft`
- **[medium] §4 arrows** — 03-architecture.md line 37: `mr[bg merger<br/>fresh segments → merged base segment] -.->|"write merged segments"| bi`
  - Store write drawn as a dotted async edge: `mr -.->|"write merged segments"| bi`. §4 reserves `-.->` for events/queue consumption/replication; a write into a keyed store is a store operation and must be solid `-->` with the op in the label (the same rule that correctly makes ix/write segments and read features solid). A reader could decode the base index as an event log.
  - Fix: `    mr[bg merger<br/>fresh segments -> merged base segment] -->|"write merged segments"| bi`
- **[low] prose-consistency** — 03-architecture.md line 25: `fe -->|"3· gRPC Vertical(q) · proto"| vt[vertical rankers<br/>news · images · video<br/>omit on timeout]`
  - Node label `vt[vertical rankers<br/>news · images · video<br/>omit on timeout]` lists only three verticals, but FR6 in 01-requirements.md names news, images, video, places, shopping as the verticals the SERP composes from; a careful reader could conclude places and shopping don't exist in this system.
  - Fix: `    fe -->|"3· gRPC Vertical(q) · proto"| vt[vertical rankers<br/>news · images · video · places · shopping<br/>omit on timeout]`

### study_cases/49.monitoring-metrics/
- **[medium] §6 edge contracts** — study_cases/49.monitoring-metrics/03-architecture.md, block line 13, edge am→oc
  - Cross-deploy-boundary edge carries no API contract: `am -->|"firing alert payload<br/>PagerDuty · Slack · email"| oc((🧑 on-call))` names the payload and destinations but has neither METHOD /path nor transport · format, so the edge does not decode alone under §6.
  - Fix: `am -->|"POST firing alerts<br/>PagerDuty · Slack · email · JSON"| oc((🧑 on-call))`
- **[medium] §6 edge contracts** — study_cases/49.monitoring-metrics/03-architecture.md, block line 17, edge qy→sw
  - Cross-deploy-boundary edge missing transport/format: `qy -->|"GET old block chunks"| sw` names the operation but sw is a deployed gateway (rectangle), so the store-edge exemption does not apply and §6 requires a named transport; the sibling edge qy->pr already uses gRPC StoreAPI, so this one decodes inconsistently.
  - Fix: `qy -->|"gRPC StoreAPI Series()<br/>old blocks"| sw`

### study_cases/50.stock-exchange
- **[medium] prose-consistency** — study_cases/50.stock-exchange/03-architecture.md, diagram line: `jr -.->|"consume: outbound.events"| od[outbound dispatch<br/>exec reports · drop-copy · MD writer]`
  - The prose "Events out" paragraph (03-architecture.md) says dispatch readers fan outbound events into "execution reports to the member's session, drop-copy to compliance, MD to the multicast feed", but the diagram routes only md.tick to the multicast and SubmitTrade() to the CCP. The od node's own label lists "exec reports · drop-copy · MD writer" yet exec reports have no destination edge, so the fill half of the Q: ("How does an order become a fill...") dead-ends inside od — a reader could wrongly conclude exec reports never leave dispatch, contradicting 01-requirements.md ("Execution reports back to the placing firm").
  - Fix: `Add after the "jr -.->|"consume: outbound.events"| od" line: `od -.->|"produce: exec.report<br/>TCP · FIX-like binary"| mf`  (TCP · FIX-like binary mirrors the existing member-facing session transport; drop-copy can stay prose-only or be folded into the same edge label's second line.)`
- **[low] §6 API edges** — study_cases/50.stock-exchange/03-architecture.md, diagram edge: `sq -->|"3· inbound msg seq N"| jr{{📨 event journal<br/>inbound + outbound events<br/>one sequence space}}`
  - Edge 3 carries no transport/format line, while every other cross-component sync edge in the block names one — edge 2 says `TCP · FIX-like binary`, edge 4 (same host pair) says `in-proc · binary`, su->rp says `TCP · binary`. §6 requires each cross-boundary sync edge to name its transport; the omission is inconsistent with its sibling edge and 02-scale-estimates' shared-host table.
  - Fix: ``sq -->|"3· inbound msg seq N<br/>in-proc · binary"| jr``

