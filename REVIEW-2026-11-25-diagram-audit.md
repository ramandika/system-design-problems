# Diagram audit pass 2 — 2026-11-25

**Status: report only, no fixes applied.** Fanout audit (workflow wf_2702d0c5-da2): 50/50 cases audited, **285 findings (78 major)** across 53 mermaid blocks. Each agent validated its case's diagrams against `study_cases/DIAGRAMS.md` §1–§8 and verified traceability against the case's 01/02/04 prose. Supersedes nothing: this is a second pass over the tree after the 2026-11-26 fix round (see REVIEW-2026-11-26-diagrams.md); findings here are new or residual.

| case | findings | majors |
|---|---:|---:|
| 01.youtube | 5 | 2 |
| 02.twitter | 1 | 1 |
| 03.whatsapp | 3 | 0 |
| 04.deployment-system | 5 | 1 |
| 05.discord | 6 | 1 |
| 06.apple-app-store | 7 | 1 |
| 07.blob-store | 4 | 1 |
| 08.chatgpt | 6 | 2 |
| 09.cdn | 2 | 1 |
| 10.distributed-cache | 3 | 0 |
| 11.zookeeper | 5 | 1 |
| 12.bigtable | 6 | 1 |
| 13.kafka | 7 | 0 |
| 14.ml-production-system | 5 | 3 |
| 15.ml-ranking-system | 4 | 0 |
| 16.dalle | 5 | 0 |
| 17.elevenlabs | 5 | 2 |
| 18.sora | 10 | 6 |
| 19.frontend-chat-app | 3 | 0 |
| 20.frontend-newsfeed | 7 | 2 |
| 21.frontend-calendar | 8 | 3 |
| 22.frontend-uber | 7 | 3 |
| 23.frontend-video-streaming | 5 | 1 |
| 24.mobile-chat-app | 7 | 3 |
| 25.mobile-newsfeed | 7 | 2 |
| 26.google-maps | 5 | 0 |
| 27.ev-charging-network | 8 | 3 |
| 28.uber | 8 | 3 |
| 29.uber-eats | 7 | 3 |
| 30.facebook-messenger | 7 | 2 |
| 31.netflix | 3 | 0 |
| 32.spotify | 10 | 2 |
| 33.instagram | 7 | 2 |
| 34.reddit | 7 | 3 |
| 35.tiktok | 4 | 0 |
| 36.airbnb | 7 | 0 |
| 37.ticketmaster | 5 | 2 |
| 38.leetcode | 6 | 2 |
| 39.e-commerce | 6 | 2 |
| 40.tinyurl | 5 | 2 |
| 41.web-crawler | 7 | 1 |
| 42.linkedin | 5 | 2 |
| 43.zoom | 6 | 1 |
| 44.payment-system | 7 | 4 |
| 45.typeahead | 4 | 1 |
| 46.google-docs | 4 | 2 |
| 47.rate-limiter | 3 | 0 |
| 48.google-search | 6 | 3 |
| 49.monitoring-metrics | 6 | 0 |
| 50.stock-exchange | 9 | 1 |


## 01.youtube — 5 findings

- **MAJOR · prose-trace** · The transcoder's only inbound edge is a trigger event (`produce: video.uploaded`), so the transfer of the original bytes from the ingest store to the transcoder is never drawn and a reader following the diagram alone cannot decode where the mezzanine input comes from.
  - subject: `ing -.->|"produce: video.uploaded"| tr`
  - evidence: Diagram line: `ing -.->|"produce: video.uploaded"| tr[transcoder<br/>mezzanine → renditions 480p..1080p+]`. Spec §5: "An edge only moves data. Its label states what crosses" and the §5 example draws the transcoder fed by a data edge; spec §8 caption: "How do upload bytes and watch bytes flow". Prose: 03-architecture.md line 34 "Ingest store | durable raw object storage | original file kept (re-encode later)" (i.e. the transcoder re-reads those originals).
  - fix: tr -->|"GET raw originals"| ing
- **MAJOR · prose-trace** · The tiering edge starts from the hot blob store — whose label says it holds renditions — carrying a payload labeled "originals", contradicting the prose that originals are kept by the ingest store and re-encoded from there.
  - subject: `hot -.->|"originals"| cold`
  - evidence: Diagram line: `hot -.->|"originals"| cold[(🪣 cold archive<br/>originals)]`. Prose: 03-architecture.md line 34 (originals live in the ingest store), line 57 "keep originals (needed for future re-encodes) in cold storage, hot renditions sized by popularity curve"; deep dive 4 (04-deep-dives.md line 69) tiers *renditions* hot→cold.
  - fix: ing -.->|"originals"| cold
- **MINOR · §4** · Two edges use the reserved thick arrow, so neither upload nor watch is *the* primary path; the caption itself states two questions (upload bytes and watch bytes), which §8 says to split into a second diagram.
  - subject: `cr ==>|"PUT /upload…"| ue (together with vw ==>|"2· GET segments…"| cdn)`
  - evidence: Diagram line: `cr ==>|"PUT /upload<br/>HTTPS · chunked resumable"| ue[upload-edge<br/>checksum · chunked resumable]` and `vw ==>|"2· GET segments<br/>HTTPS · byte-range"| cdn[/🌍 CDN<br/>media delivery/]`. Spec §4: "`==>` is reserved for the primary *sync* path" (singular); §8: "one diagram, one question… if a second question needs a second diagram (e.g. write path vs read path), draw a second diagram".
  - fix: cr -->|"PUT /upload<br/>HTTPS · chunked resumable"| ue
- **MINOR · §1** · Two `produce:` edges are drawn directly between services/stores with no topic/log node, so the event bus the events cross is missing and the ingest cylinder appears to emit events itself.
  - subject: `ing -.->|"produce: video.uploaded"| tr and tr -.->|"produce: video.updated"| rc`
  - evidence: Diagram line: `ing -.->|"produce: video.uploaded"| tr[transcoder<br/>mezzanine → renditions 480p..1080p+]` (and `tr -.->|"produce: video.updated"| rc[rec indexer…]`). Spec §1: hexagon = "an append-only topic/log you consume from"; §5 example: `T -.->|"produce: video.ready"| K{{📨 Kafka<br/>media events}}`.
  - fix: k{{📨 publish events<br/>upload · video.updated}} (then `ue -.->|"produce: video.uploaded"| k` and `k -.->|"consume: video.uploaded"| tr`, similarly for video.updated)
- **MINOR · prose-trace** · The transcoder label claims a rendition floor of 480p, but the prose ladder and the storage arithmetic start at 240p.
  - subject: `tr[transcoder<br/>mezzanine → renditions 480p..1080p+]`
  - evidence: Diagram line 15: `tr[transcoder<br/>mezzanine → renditions 480p..1080p+]`. Prose: 02-scale-estimates.md line 45 "each video is stored at ~5 renditions (240p…1080p+)"; 04-deep-dives.md line 31 ladder "1080p@6Mbps, 720p@3Mbps, 480p@1.5Mbps, 240p@0.4Mbps".
  - fix: tr[transcoder<br/>mezzanine → renditions 240p..1080p+]

## 02.twitter — 1 findings

- **MAJOR · prose-trace** · The diagram routes celebrity posts into a high-fanout lane for slow-drip timeline inserts, but the case's chosen design says celebrity authors skip fanout entirely and are served only on-read from the celebrity index, which the diagram's own merge edge ("SQL recent-tweets-topN 0 200") already draws — so a reader following the diagram alone decodes the celebrity write path wrong (fanned out AND merged).
  - subject: `a -.->|"produce: post.created<br/>normal · high lane"| q{{📨 fanout queues<br/>normal · high-fanout lane}}`
  - evidence: Diagram 03-architecture.md L12-13: `a -.->|"produce: post.created<br/>normal · high lane"| q{{📨 fanout queues<br/>normal · high-fanout lane}}` and `q -.->|"consume: post.created"| w[fanout workers<br/>drip RPUSH timelines]`. Contradicts 04-deep-dives.md L20-24: "**Chosen: hybrid.** Fanout-on-write for everyone below a follower-count threshold... celebrity authors skip the fanout and their recent tweets are merged into readers' timelines on read", and 02-scale-estimates.md L86-88: "real Twitter's hybrid excluded celebrity followers from fanout entirely". Note: 03-architecture.md L45-50 itself carries the high-lane design ("fanout is enqueued to a slow high-queue for drip insertion"), so the two case docs contradict each other on this central decision; the diagram mirrors 03 and diverges from 04. Also contradicts 01-requirements.md L25 ("Post → visible in follower timelines < 1–2 s typical") unless the on-read merge is the celebrity delivery mechanism, which is exactly the skip-fanout design.
  - fix: a -.->|"produce: post.created"| q{{📨 fanout queues<br/>normal authors only · celebrities skip fanout}} — with the celebrity delivery falling to the existing `s -->|"SQL recent-tweets-topN 0 200"| ci` on-read merge edge (and w's label losing the celebrity-only "drip" wording: `w[fanout workers<br/>RPUSH timelines per follower]`).

## 03.whatsapp — 3 findings

- **MINOR · §6** · The presence edge label mixes the HTTP path form (/v1/presence) with an RPC-style lowercased method (state.update) that is not an HTTP METHOD, so the contract does not match either allowed §6 form.
  - subject: `cg -->|"state.update /v1/presence<br/>gRPC · proto"| p`
  - evidence: 03-architecture.md line 16: `cg -->|"state.update /v1/presence<br/>gRPC · proto"| p`; DIAGRAMS.md §6 line 146 `-->|"METHOD /path<br/>transport · format"|` and line 155 "RPC: method name; each cross-boundary HTTP/RPC edge names its transport"
  - fix: cg -->|"gRPC UpdatePresence()<br/>proto"| p[presence cluster<br/>ephemeral · no durable store]
- **MINOR · §4** · The two outgoing edges of chat gateway whose order the prose fixes (store append assigns the seq number before the delivery call) carry no 1·/2· numbering, so a reader could read deliver-before-append.
  - subject: `cg ==>|"append envelope<br/>seq per conversation"| ms and cg -->|"gRPC deliver(msg)"| dl`
  - evidence: DIAGRAMS.md §4 lines 100-102: "When order matters, number the labels: `-->|"1· GET /v1/feed"|`"; 03-architecture.md lines 39-41: "sender→gateway: write message store (assigns seq number within conversation), then delivery service pushes to the receiver's gateway"
  - fix: cg ==>|"1· append envelope<br/>seq per conversation"| ms[(message store<br/>per-conversation log)] and cg -->|"2· gRPC deliver(msg)"| dl[delivery service<br/>route · park · expire]
- **MINOR · prose-trace** · The ACK → delivered/read-state propagation flow that the prose describes as part of the online 1:1 flow (and 01-requirements makes a core delivery state machine sent→delivered→read) appears nowhere in the diagram.
  - subject: `missing receiver-ACK / delivered-state flow (no edge from cg back through dl carrying acknowledgements)`
  - evidence: 03-architecture.md lines 41-42: "→ receiver ACKs → 'delivered' state propagated to sender (and read receipts when receiver opens)"; 01-requirements.md lines 9-10: "Delivery state machine per message: sent → delivered → read. Read receipts are themselves messages"; DIAGRAMS.md §8 line 184 "Keep diagrams under ~15 nodes" (13 used, room for the edge)
  - fix: cg -.->|"client ack<br/>message id"| dl  (delivery service records delivered; propagation back to sender rides the existing dl -.-> cg session push)

## 04.deployment-system — 5 findings

- **MAJOR · prose-trace** · The CD leg ends at the health evaluator: no edge shows the controller acting on a verdict — promote, routing-weight change, last-known-good write, or address-based rollback — which the prose makes central to the rollout question.
  - subject: `k -.->|"consume: artifact.published"| ctl[deployment controller<br/>rollout state machine · canary]`
  - evidence: Diagram: `ctl -->|"gRPC EvaluateGate()<br/>gRPC · proto"| he[...]` is the last edge on the CD leg (03-architecture.md line 21), with no promote/rollback edge after it. Prose: 03-architecture.md lines 92–99 — "controller promotes to 100% and writes the new last-known-good record. Failure → controller flips routing back... address-based"; component 10 (line 67): "the only component that mutates environment routing weight".
  - fix: ctl -->|"SQL rollouts<br/>routing weight · last-known-good"| db[(control-plane DB<br/>runs · jobs · rollouts)]
- **MINOR · prose-trace** · The diagram makes the control-plane DB itself the producer of run.status, but the prose says status events are emitted by a separate outbox relay — the load-bearing 'DB-rows-are-authority, bus-is-fanout' decision — so a diagram-only reader misses the atomicity mechanism.
  - subject: `db -.->|"produce: run.status"| k`
  - evidence: Diagram: `db -.->|"produce: run.status"| k` (03-architecture.md line 13). Prose: 03-architecture.md lines 39–42 — "Writes happen transactionally with an **outbox row** consumed by a relay into the event bus — status and its event are published atomically".
  - fix: rl[outbox relay<br/>outbox rows → bus events] -.->|"produce: run.status"| k{{📨 event bus<br/>run · job · log · artifact topics}} (fed by db -.->|"outbox rows"| rl; pushes block to 16 nodes, acceptable for a named design decision)
- **MINOR · prose-trace** · The event name `platform.health` appears nowhere in the prose; the coordinator's drawn role (subscribing to a health event) contradicts the prose flow where the coordinator *proposes* platform versions to the same rollout mechanism and each wave's gate runs through the health evaluator machinery.
  - subject: `k -.->|"consume: platform.health"| fro[fleet rollout coordinator<br/>platform waves · self-gated]`
  - evidence: Diagram: `k -.->|"consume: platform.health"| fro[fleet rollout coordinator<br/>platform waves · self-gated]` (03-architecture.md line 23); `grep platform.health` hits nothing outside the diagram. Prose: 03-architecture.md line 102 — "fleet rollout coordinator proposes a platform version to dogfood → waves → regional cohorts"; 04-deep-dives.md lines 235–241 — "Each wave's gate reads the same evaluator machinery as a customer canary".
  - fix: fro -->|"propose: platform rollout"| ctl[deployment controller<br/>rollout state machine · canary]
- **MINOR · §5** · The edge label carries the minting transformation (`OIDC token → short-lived creds`), which belongs inside the owning component's label — where it already is — instead of stating only what crosses the edge in the request direction.
  - subject: `run -->|"gRPC MintCreds()<br/>OIDC token → short-lived creds"| sec`
  - evidence: Diagram: `run -->|"gRPC MintCreds()<br/>OIDC token → short-lived creds"| sec[secrets broker<br/>vault backend · OIDC → creds<br/>TTL ≤ 15 min]` (03-architecture.md line 17). Spec: DIAGRAMS.md lines 127–129 ("An edge only moves data. Its label states *what crosses*"); the `→` chain already lives in sec's label.
  - fix: run -->|"gRPC MintCreds()<br/>gRPC · proto"| sec[secrets broker<br/>vault backend · OIDC → creds<br/>TTL ≤ 15 min]
- **MINOR · §4** · No `==>` marks the primary sync path, so a reader scanning the block cannot see which entry edge starts the commit→artifact data path the case is about.
  - subject: `gh[/git hosts<br/>webhooks/] -->|"POST /webhooks<br/>HTTPS · JSON · signed"| int[event intake<br/>verify · dedupe]`
  - evidence: Diagram: `gh[/git hosts<br/>webhooks/] -->|"POST /webhooks<br/>HTTPS · JSON · signed"| int[event intake<br/>verify · dedupe]` (03-architecture.md line 9) — no `==>` anywhere in the block. Spec: DIAGRAMS.md lines 88–89 (`==>` = "the one data path the case is about") and line 174 ("thick arrow = the primary path").
  - fix: gh[/git hosts<br/>webhooks/] ==>|"POST /webhooks<br/>HTTPS · JSON · signed"| int[event intake<br/>verify · dedupe]

## 05.discord — 6 findings

- **MAJOR · §4/§5** · The client–SFU media link is drawn as a one-way synchronous call, but it is a long-lived bidirectional media session (the SFU also forwards opus frames to the participant), so `-->` decodes as a one-shot sync call and hides the SFU→client media direction.
  - subject: `c -->|"UDP opus · SRTP"| sfu`
  - evidence: Diagram line 22 of study_cases/05.discord/03-architecture.md: `c -->|"UDP opus · SRTP"| sfu`; DIAGRAMS.md §4: `<-->` for long-lived connections (WebSocket, gRPC stream), and `-->` = synchronous call the caller waits on; 03-architecture.md flow 3 step 3: "SFU relays opus frames to other participants" (media also flows SFU→client, which a request-direction-only one-way edge cannot show).
  - fix: c <-->|"UDP opus<br/>SRTP"| sfu
- **MINOR · §5** · The second label line `perms check` states the caller's purpose/outcome, not what crosses the edge, which is the role + override rows.
  - subject: `api -->|"SQL roles+overrides<br/>perms check"| meta`
  - evidence: Diagram line 11 of study_cases/05.discord/03-architecture.md; DIAGRAMS.md §5: "An edge only moves data. Its label states what crosses" and outcomes get rewritten as the data that crosses; 03-architecture.md flow 1 step 2 shows the crossing data: "role + channel overrides, hit-cached from metadata store".
  - fix: api -->|"SQL roles+overrides"| meta
- **MINOR · §4** · The send path has a load-bearing order (publish only after the durable INSERT) but its four edges are unnumbered, so a reader can misorder the produce edge before the store write.
  - subject: `api -.->|"produce: MESSAGE_CREATE"| k (unnumbered send path)`
  - evidence: Diagram lines 10–14 of study_cases/05.discord/03-architecture.md; DIAGRAMS.md §4: "When order matters, number the labels … only when a reader could misorder the edges"; 03-architecture.md flow 1 step 4: "On durable write, the API publishes MESSAGE_CREATE", with the durability NFR in 01-requirements.md ("A message accepted (200-OK) must NEVER be lost").
  - fix: api ==>|"1· POST /channels/ch:{id}/messages<br/>HTTPS · JSON"| api … api -->|"2· gRPC append"| ds; ds -->|"3· INSERT msg:{id}"| m; api -.->|"4· produce: MESSAGE_CREATE"| k
- **MINOR · §6** · The two HTTP attachment edges do not carry the required `transport · format` contract: `objects` is not a body format, and the CDN→origin edge omits the format line entirely.
  - subject: `c -->|"GET attachments/*<br/>HTTPS · objects"| cdn and cdn -->|"GET attachments/*<br/>HTTPS"| o`
  - evidence: Diagram lines 23 and 25 of study_cases/05.discord/03-architecture.md; DIAGRAMS.md §6: HTTP edges use `|"METHOD /path<br/>transport · format"|`.
  - fix: c -->|"GET attachments/*<br/>HTTPS · blob"| cdn  and  cdn -->|"GET attachments/*<br/>HTTPS · blob"| o
- **MINOR · §6** · The RPC method is given in lowercase prose form instead of the spec's `Method()` shape, drifting from the gRPC contract format used elsewhere in the diagram (`gRPC Heartbeat()`, `gRPC JoinVoice()`).
  - subject: `api -->|"gRPC append"| ds`
  - evidence: Diagram line 12 of study_cases/05.discord/03-architecture.md; DIAGRAMS.md §6: RPC edges carry the method name, e.g. `|"gRPC GetDegree()"|`.
  - fix: api -->|"gRPC Append()"| ds
- **MINOR · prose-trace** · The unread-count key is per user only, but the prose defines unread state as aggregated per (user, channel), so the diagram's store schema omits the channel dimension.
  - subject: `fn -->|"INCR unread:{uid}"| meta`
  - evidence: Diagram line 17 of study_cases/05.discord/03-architecture.md: `fn -->|"INCR unread:{uid}"| meta`; study_cases/05.discord/04-deep-dives.md §2: "unread counters are maintained by fanout workers asynchronously, aggregated per (user, channel) in the metadata store" (also 02-scale-estimates.md: "read-state is one row per (user, channel)").
  - fix: fn -->|"INCR unread:{uid}:{ch}"| meta

## 06.apple-app-store — 7 findings

- **MAJOR · §6 + prose-trace** · The release-gate event is drawn as a direct produce: edge from the catalog DB into the search cluster (a cylinder), not into a topic hexagon, and the diagram shows only one subscriber of the fan-out the prose makes central (search indexer, CDN pre-warm, storefront caches), contradicting the single-release-gate stance.
  - subject: `ct -.->|"produce: catalog.changed"| sc[(🔍 search cluster...)]`
  - evidence: Diagram line: `ct -.->|"produce: catalog.changed"| sc[(🔍 search cluster<br/>index rebuilt from approved catalog)]` (03-architecture.md line 14). Prose: study_cases/06.apple-app-store/03-architecture.md lines 74-76 — "catalog row `released` → catalog change event → search indexer, CDN pre-warm, storefront caches. One gate, one flip". Spec: DIAGRAMS.md §1 line 18 (topic = hexagon) and §7 lines 169-170 (produce goes INTO a hexagon, consume comes OUT of it).
  - fix: ct -.->|"produce: catalog.changed"| eb{{📨 catalog change events}} ⏎     eb -.->|"consume: catalog.changed"| sc[(🔍 search cluster<br/>index rebuilt from approved catalog)]  (16 nodes — one over the soft ~15; alternatively relabel the edge `catalog.changed events` and leave the pre-warm subscriber in prose)
- **MINOR · §6** · Sync edge crossing a deploy boundary carries a payload word ('verified binary') plus transport but no METHOD /path or body format, so it does not carry the API contract the spec requires.
  - subject: `gw -->|"2· verified binary<br/>HTTPS"| pipe`
  - evidence: Diagram line: `gw -->|"2· verified binary<br/>HTTPS"| pipe[asset-pipeline...]` (03-architecture.md line 10). Spec: DIAGRAMS.md §6 lines 143-149 — `-->|"METHOD /path<br/>transport · format"|`; "each cross-boundary HTTP/RPC edge names its transport".
  - fix: gw -->|"2· POST /v1/verified-binaries<br/>HTTPS · pkg"| pipe[asset-pipeline<br/>universal → device variants<br/>deltas + re-sign]
- **MINOR · §6** · Store edge gives the operation but no argument, unlike the spec's store-edge format `OP arg`.
  - subject: `dist -->|"GET objects"| bl`
  - evidence: Diagram line: `dist -->|"GET objects"| bl` (03-architecture.md line 18). Spec: DIAGRAMS.md §6 lines 151-152 — stores: "the operation + argument, e.g. `OP arg`".
  - fix: dist -->|"GET object:(variant, version)"| bl
- **MINOR · prose-trace** · The gateway label claims 'virus pre-screen', a function no prose assigns to this component; the gateway's stated job is signature verification, automated pre-screens belong to the review pipeline's state machine, and malware-scanning internals are explicitly out of scope.
  - subject: `gw[ingest-verify-gw<br/>signature check · virus pre-screen]`
  - evidence: Diagram line: `gw[ingest-verify-gw<br/>signature check · virus pre-screen]` (03-architecture.md line 9). Prose: study_cases/06.apple-app-store/03-architecture.md lines 40-42 (gateway = "Verifies the developer signature... Nothing enters the asset pipeline unsigned"); 01-requirements.md lines 84-86 ("Review content intelligence (malware scanning internals...)"); 04-deep-dives.md line 133 (pre-screen is a state-machine transition, not an ingest check).
  - fix: gw[ingest-verify-gw<br/>developer signature check<br/>resumable multi-part intake]
- **MINOR · prose-trace** · The diagram shows no edge from the storefront/check-in tier to the catalog it compares against, so the flow "compares (device, installed) against catalog releases" that the prose makes central to check-in cannot be followed from the diagram alone.
  - subject: `u -->|"4· POST /v1/check-in<br/>HTTPS · JSON"| sf (no read edge from sf to ct)`
  - evidence: Diagram line: `u -->|"4· POST /v1/check-in<br/>HTTPS · JSON"| sf[storefront/check-in tier · delta-manifest...]` (03-architecture.md line 15). Prose: study_cases/06.apple-app-store/03-architecture.md lines 80-82 — "storefront/check-in tier compares (device, installed) against catalog releases → manifest response". Spec: DIAGRAMS.md §4 line 106 — "A `-->` edge into a store is the operation named in its label".
  - fix: sf -->|"SQL released versions<br/>per storefront"| ct
- **MINOR · prose-trace** · The prose money path lists the ledger as an explicit store hop ('IAP service with idempotency key → account ledger transaction → payment processor'), but the diagram folds it into the iap rectangle's label instead of drawing it as the cylinder the spec requires for a keyed store, so the strongly-consistent money core is invisible as a component.
  - subject: `iap[IAP + receipt service<br/>exactly-once ledger<br/>signed offline receipts]`
  - evidence: Diagram line: `iap[IAP + receipt service<br/>exactly-once ledger<br/>signed offline receipts]` (03-architecture.md line 19). Prose: study_cases/06.apple-app-store/03-architecture.md line 86-87 — "Device → IAP service with idempotency key → account ledger transaction → payment processor"; 04-deep-dives.md lines 196-200 ("Single-writer sharded ledger"). Spec: DIAGRAMS.md §1 line 17 (keyed durable store = cylinder).
  - fix: iap -->|"SQL ledger tx<br/>idempotency key"| lg[(ledger<br/>single-writer sharded<br/>append-only)]
- **MINOR · §5** · The → chain in the sf label starts from 'delivery plan', which is the component's output, not its input — the chain does not read input → output as §5 requires.
  - subject: `sf[...<br/>delivery plan → signed URLs · delta chain + full fallback]`
  - evidence: Diagram line: `sf[storefront/check-in tier · delta-manifest<br/>browse · search · check-ins<br/>delivery plan → signed URLs · delta chain + full fallback]` (03-architecture.md line 15). Prose: 03-architecture.md lines 80-84 (check-in and (device, installed) produce the plan). Spec: DIAGRAMS.md §5 lines 126-127 — "Transformations are `in → out` chains", and §5 example line 135 `upload.mov` → transcoder.
  - fix: sf[storefront/check-in tier · manifest<br/>check-in → delivery plan<br/>signed URLs · delta chain + full fallback]

## 07.blob-store — 4 findings

- **MAJOR · §1/§6** · A produce:/consume: event edge terminates directly on a worker rectangle with no topic hexagon, so the reader decodes repair-worker as the queue, while the prose makes repair work queued, idempotent units.
  - subject: `sc -.->|"produce: repair.jobs"| rw[repair-worker<br/>rebuild shard from k survivors]`
  - evidence: Diagram line: `sc -.->|"produce: repair.jobs"| rw[repair-worker<br/>rebuild shard from k survivors]`. Rule: DIAGRAMS.md §1 — hexagon = 'an append-only topic/log you consume from'; §6 — events carry 'produce:/consume: + event name'. Prose: 04-deep-dives.md line 177 — 'Unit of work = (stripe, lost shard id) — idempotent, retriable, **queued**.'
  - fix: rq{{📨 repair queue}} ⏎ sc -.->|"produce: repair.jobs"| rq ⏎ rq -.->|"consume: repair.jobs"| rw
- **MINOR · §4** · Two ==> edges are drawn and the second bundles both PUT and GET into one thick edge, blurring which path is the single primary sync path and whether GET shards is primary or a normal sync read.
  - subject: `c ==>|"PUT /object<br/>HTTPS · signed chunks"| fe ... fe ==>|"PUT/GET shards<br/>RS 8+4 stripes"| dp`
  - evidence: Diagram line: `fe ==>|"PUT/GET shards<br/>RS 8+4 stripes"| dp` (and `c ==>|"PUT /object<br/>HTTPS · signed chunks"| fe`). Rule: DIAGRAMS.md §4 — '`==>` | primary path | the one data path the case is about'; reference example §7 uses one `==>` edge.
  - fix: fe ==>|"PUT shards<br/>RS(8,4) stripes"| dp ⏎ fe -->|"GET ≥k of 12 shards"| dp
- **MINOR · prose-trace** · The GET edge's format claim 'JSON' contradicts the prose, which defines GET as streaming, seekable byte ranges, not a JSON body.
  - subject: `c -->|"GET /object<br/>HTTPS · JSON"| fe`
  - evidence: Diagram line: `c -->|"GET /object<br/>HTTPS · JSON"| fe`. Prose: 01-requirements.md lines 10–11 — '**GET object** / **GET byte range**: read whole object or arbitrary range; streaming, seekable downloads.'
  - fix: c -->|"GET /object<br/>HTTPS · byte stream"| fe
- **MINOR · §4** · The PUT ordering the prose calls load-bearing (data durable first, then index commit) is not numbered in the diagram, so a reader cannot recover the required order from the edges alone.
  - subject: `fe -->|"GET + write index:bucket-key"| mt ... fe ==>|"PUT/GET shards<br/>RS 8+4 stripes"| dp`
  - evidence: Diagram lines: `fe -->|"GET + write index:bucket-key"| mt` and `fe ==>|"PUT/GET shards<br/>RS 8+4 stripes"| dp`. Rule: DIAGRAMS.md §4 — 'When order matters, number the labels: `-->|"1· GET /v1/feed"|`'. Prose: 03-architecture.md lines 43–45 — 'Order matters: data first, then index. A crash between means an orphaned stripe, not a visible-but-missing object.'
  - fix: fe ==>|"1· PUT shards<br/>RS(8,4) stripes"| dp ⏎ fe -->|"GET / 2· write index record:bucket-key"| mt

## 08.chatgpt — 6 findings

- **MAJOR · prose-trace** · The diagram draws the prompt/prefix cache as an external keyed store that prefill and decode read/write KV from, but the prose rejects any separate KV store: KV lives resident on the GPU workers and reuse happens by affinity routing, not get/put against a cache tier.
  - subject: `PF -->|"KV get/put prefix id"| PC and DC -->|"KV get/put prefix id"| PC into PC[(Prompt/prefix cache<br/>hot KV replicas)]`
  - evidence: Diagram lines 72-73: `PF -->|"KV get/put prefix id"| PC[(Prompt/prefix cache<br/>hot KV replicas)]` and `DC -->|"KV get/put prefix id"| PC`; 02-scale-estimates.md:84-85 "KV cache is **not** stored: it is recomputed or reused within a request's lifetime and discarded (the only durable persistence is the text)"; 04-deep-dives.md DD3 (lines 181-188): "Chosen: C ... keep hot KV resident on GPU (paged, LRU) and route follow-up turns to the workers that already hold the prefix" — the durable-KV-store option (B) is explicitly rejected (~300 PB, transfer eats the TTFT budget).
  - fix: Delete the PC node and both `KV get/put prefix id` edges, and express reuse as routing in the scheduler: `SCH -->|"affinity tbl: prefix id → worker set"| PF`
- **MAJOR · prose-trace** · The SA node's own label claims "write final turn" but the diagram has no edge from SA to the conversation store, so the central durability flow (persisting the completed response, a MUST-NOT-LOSE requirement) is missing from the request-path diagram.
  - subject: `SA[Stream assembly + persistence<br/>tokens → gateway · write final turn]`
  - evidence: Diagram lines 74-77 show no SA→DS edge; 03-architecture.md:103-104 "Stream assembly & persistence: forwards tokens to the gateway immediately; on completion (or user stop) writes the final turn"; 04-deep-dives.md:155 sequence `IP-)DS: completed turn text (async write)`; 01-requirements.md:33-34 "a completed response, once saved, is never lost".
  - fix: `SA -.->|"KV put turn · conv id"| DS` — async write of the completed turn back to the existing DS cylinder
- **MINOR · §6** · The edge crossing the SA→GW deploy boundary names the wrong transport: SSE/WS is the gateway↔client leg (declared on the C<-->GW session edge), while this internal hop carries no contract/transport of its own.
  - subject: `SA -->|"token stream<br/>SSE/WS · JSON"| GW`
  - evidence: Diagram line 75: `SA -->|"token stream<br/>SSE/WS · JSON"| GW`; DIAGRAMS.md §6: "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto`) so every edge decodes alone."; the SSE/WS transport belongs to the client session edge at line 62.
  - fix: `SA -->|"token deltas<br/>gRPC stream · proto"| GW`
- **MINOR · §6** · An event edge crossing a deploy boundary must carry the produce:/consume: prefix plus event name; the label only describes the payload as "turn.completed event".
  - subject: `SA -.->|"turn.completed event"| AM`
  - evidence: Diagram line 76: `SA -.->|"turn.completed event"| AM`; DIAGRAMS.md §6: "Events: `produce:` / `consume:` + event name — the event name is the API."
  - fix: `SA -.->|"produce: turn.completed"| AM`
- **MINOR · §5** · The label states the outcome of the admission check ("admitted") instead of only the data that crosses the edge.
  - subject: `RL -->|"gRPC ScreenPrompt()<br/>admitted prompt"| MOD`
  - evidence: Diagram line 66: `RL -->|"gRPC ScreenPrompt()<br/>admitted prompt"| MOD`; DIAGRAMS.md §5: "An outcome or trigger is not a payload: `miss`, `hit 95%`, `push config` get rewritten as the data that crosses."
  - fix: `RL -->|"gRPC ScreenPrompt()<br/>prompt tokens · tier"| MOD`
- **MINOR · prose-trace** · The label claims KV crosses from the scheduler to the decode pool, but per the prose the KV transfer happens prefill→decode (which the diagram already draws as `PF -->|"gRPC KV-transfer()"| DC`); the scheduler only places the request.
  - subject: `SCH -->|"gRPC Decode()<br/>KV · prompt"| DC`
  - evidence: Diagram line 70: `SCH -->|"gRPC Decode()<br/>KV · prompt"| DC`; 03-architecture.md:94-96 "Places each admitted request on a prefill worker, transfers KV to a decode worker" and data flow 1 (03-architecture.md:110): "KV handed to decode pool" — the KV crossing is the PF→DC edge at line 71.
  - fix: `SCH -->|"gRPC Decode()<br/>request · prefix id"| DC`

## 09.cdn — 2 findings

- **MAJOR · prose-trace** · The shield label claims "1 per region", which contradicts the prose (03: "A small number (1–3) of larger POPs per region") and deep dive 3's chosen design of two active/active shields per region.
  - subject: `sh[Regional shield POP<br/>1 per region<br/>L3 cache + collapse]`
  - evidence: Diagram line: sh[Regional shield POP<br/>1 per region<br/>L3 cache + collapse] — vs study_cases/09.cdn/03-architecture.md:38-39 "A small number (1–3) of larger POPs per region designated as second-tier caches" and study_cases/09.cdn/04-deep-dives.md:135-136 "we run shared active/active shields (two per region, each covers all edges; the mapping just flips)".
  - fix: sh[Regional shield POP<br/>1–3 per region · active/active<br/>L3 cache + collapse]
- **MINOR · §6** · The origin-facing edge label says "GET object" instead of METHOD /path, so it does not carry the contract format the spec requires (the same request is labeled /objects/{key} on every other edge).
  - subject: `lb -->|"GET object<br/>origin HTTPS"| o[/Customer origins<br/>S3 · blob · app/]`
  - evidence: Diagram line: lb -->|"GET object<br/>origin HTTPS"| o[/Customer origins<br/>S3 · blob · app/] — vs DIAGRAMS.md §6 line 154: "HTTP: |"GET /v1/feed<br/>HTTPS · JSON"| — method + path, then transport and body format." (the sibling edges correctly use /objects/{key}).
  - fix: lb -->|"GET /objects/{key}<br/>origin HTTPS"| o[/Customer origins<br/>S3 · blob · app/]

## 10.distributed-cache — 3 findings

- **MINOR · §6** · An async event edge uses the trigger-style prefix "push:" instead of the defined event contract produce:/consume: + event name.
  - subject: `fc -.->|"push: slot.map.update<br/>JSON"| cl`
  - evidence: Diagram line 17: fc -.->|"push: slot.map.update<br/>JSON"| cl — 03-architecture.md:18. Spec: DIAGRAMS.md §4 line 110 ('a primary async path stays -.-> ... and carries produce:/consume:'), §6 line 149 ('produce: event.name (or consume:)'), §5 lines 128-129 ('push config' is an outcome, rewritten as the data that crosses).
  - fix: fc -.->|"produce: slot.map.update<br/>JSON"| cl
- **MINOR · §6** · A store edge label names the operations (GET · SET) and the routing (by slot) but not the argument (the key), unlike the spec's store-edge form OP arg.
  - subject: `cl -->|"2· GET · SET by slot"| sh`
  - evidence: Diagram line 12: cl -->|"2· GET · SET by slot"| sh — 03-architecture.md:12. Spec: DIAGRAMS.md §6 lines 151-152 — "stores: the operation + argument, e.g. OP arg". Prose backing: 03-architecture.md Data flow 1 ('L2 GET by slot') and flow 2 ('SETs the value with TTL'); F2 hashing in /home/pranamulia/system-design/study_cases/10.distributed-cache/01-requirements.md:14-16.
  - fix: cl -->|"2· GET key · SET key by slot"| sh
- **MINOR · prose-trace** · The label claims the fallback store is "read only on outage", a use the prose never establishes — requirements say it is not read on request and losing a session forces re-login, and the prose only hedges that it is not consulted in steady state.
  - subject: `fs[(session fallback store<br/>durable · replicated<br/>read only on outage)]`
  - evidence: Diagram line 15: fs[(session fallback store<br/>durable · replicated<br/>read only on outage)] — 03-architecture.md:15. Prose: /home/pranamulia/system-design/study_cases/10.distributed-cache/01-requirements.md:18 ('durable fallback exists but is not read on request'); /home/pranamulia/system-design/study_cases/10.distributed-cache/03-architecture.md:69 ('the read path never consults it in steady state'); 04-deep-dives.md:71 ('sessions are not reconstructible in the hot path').
  - fix: fs[(session fallback store<br/>durable · replicated<br/>receive-only from hot path)]

## 11.zookeeper — 5 findings

- **MAJOR · prose-trace** · The caption asks how reads behave, and reads are the case's dominant workload (95%, 02-scale-estimates), but the diagram contains no read edge from the client to any server, so the read half of the stated question is unanswerable by following edges.
  - subject: `read path — caption half "how do reads ... behave" answered only by node labels, no read edge`
  - evidence: Caption 03-architecture.md:5: "Q: How does a write reach a ZAB quorum, and how do reads, sessions and one-shot watches behave on the servers?" — every edge in the block (lines 17-25) carries writes/protocol/watch traffic only ("1· write /path", "2· forward write", "3· PROPOSAL zxid", "4· fsync proposal", "5· COMMIT zxid", "COMMIT zxid", "watch event · one-shot", "snapshot tree", "session · TCP"); no read edge exists. Prose says reads are a first-class path: 03-architecture.md:57 "Any server answers any read straight from its in-memory tree, no disk, no leader round trip"; 02-scale-estimates.md:29 "~95% reads, ~5% writes"; deep dive 2 is entirely about the read path.
  - fix: c -->|"read /path"| f    (unnumbered reads added for follower and observer, or split a second read-path flowchart per §8: "c -->|\"2· read /path\"| f" with reads served from memory)
- **MINOR · prose-trace** · The observer's label claims it applies commits, but the only edge into it delivers "COMMIT zxid" — no proposal/txn content ever crosses, so a reader following the diagram alone cannot see how the observer obtains the mutation it applies.
  - subject: `o[🗝️ Observer<br/>applies commits<br/>serves reads · no vote]`
  - evidence: Diagram line 10: "o[🗝️ Observer<br/>applies commits<br/>serves reads · no vote]"; line 23: "l -.->|\"COMMIT zxid\"| o" is o's only inbound edge. Prose 03-architecture.md:34: "Observers (no vote) | apply committed txns, serve reads" — applying a txn requires the mutation to reach the observer.
  - fix: l -.->|"PROPOSAL · COMMIT zxid"| o
- **MINOR · prose-trace** · The prose makes the leader's own fsync central (every voting server fsyncs every proposal; the leader's log adds a 5th fsync), but the diagram only shows the follower writing the txn log.
  - subject: `Leader → txn log edge (missing)`
  - evidence: Diagram has no l→tl edge (lines 17-25); only line 21: "f -->|\"4· fsync proposal\"| tl". Prose 02-scale-estimates.md:42-44: "each write is one proposal broadcast to 4 followers plus an fsync barrier per follower (leader's own log adds a 5th fsync)" and 02:46-47: "every one of the 5 voting servers fsyncs every proposal"; 03-architecture.md:80: "leader fsyncs own log ──┐".
  - fix: l -->|"fsync own log"| tl    (added alongside the follower's line 21)
- **MINOR · §5** · The edge label states an action ("forward write"), not what crosses the edge, which is the write request itself.
  - subject: `f -->|"2· forward write"| l`
  - evidence: Diagram line 19: "f -->|\"2· forward write\"| l"; DIAGRAMS.md §5 (lines 124-126): "An edge only moves data. Its label states what crosses: payload, event name, or query." The payload is the same write from line 17 ("1· write /path").
  - fix: f -->|"2· write /path"| l
- **MINOR · §4** · The session edge's label mixes in "ephemerals", which is session-owned state in the session table, not data crossing the connection, and the label names no payload for the session (pings).
  - subject: `c <-->|"session · TCP<br/>ping/timeout · ephemerals"| f`
  - evidence: Diagram line 18: "c <-->|\"session · TCP<br/>ping/timeout · ephemerals\"| f"; DIAGRAMS.md §5 (line 128-129): an outcome/attribute is not a payload — "ephemerals" is not what crosses; §4 (line 103): "<--> marks a connection/session, never a pull/push API pair"; the prose's actual cross-traffic is pings (02-scale-estimates.md:98 "~200 pings/s/ensemble") with writes and watch events already drawn as separate edges (lines 17, 24) — correct use of <-->, just mislabeled.
  - fix: c <-->|"PING · TCP"| f

## 12.bigtable — 6 findings

- **MAJOR · §4** · A response is drawn as a second edge with async (dotted) semantics, and the direction reverses the prose: the repush of tablet location info is the tablet server's answer to a client that contacted it with a stale cache entry, not an event the server pushes.
  - subject: `ts -.->|"3· repush tablet map"| c`
  - evidence: Diagram line 18: `ts -.->|"3· repush tablet map"| c`; DIAGRAMS.md §4 lines 98-99: "Draw the request direction only. Don't draw the response as a second edge" and line 93: `-.->` = "asynchronous — the caller does not wait"; 03-architecture.md lines 50-53: "if the server is gone, walk up the range and ask the nearest server; tablet servers answer 'not mine, and your range is now at X'" and lines 46-47: "client → tablet server (stale cache entry → repush → retry)".
  - fix: Delete the line `ts -.->|"3· repush tablet map"| c` and carry the mechanism in the client label instead: `c((🧑 clients<br/>embedded library<br/>cached tablet map · stale → repush → retry))`
- **MINOR · §6** · The edge crosses the client→tablet-server deploy boundary but carries no transport · format line (and 'GET/PUT' mimics HTTP rather than the tablet-server RPC).
  - subject: `c ==>|"2· GET/PUT row-key"| ts`
  - evidence: Diagram line 17; DIAGRAMS.md §6 lines 143-156: "Every edge crossing a deploy boundary carries the contract in this order … METHOD /path<br/>transport · format … each cross-boundary HTTP/RPC edge names its transport"; 01-requirements.md lines 54-57 confirm the client talks directly to tablet servers with "no coordinator in the request path".
  - fix: c ==>|"2· RPC Read/Write row-key<br/>tablet RPC · proto"| ts
- **MINOR · §5** · The label qualifies the read with an outcome/condition ('on miss/stale') instead of stating only what crosses the edge; the miss/stale trigger belongs in prose or the client node's label.
  - subject: `c -->|"1· read root pointer<br/>on miss/stale"| ch`
  - evidence: Diagram line 16; DIAGRAMS.md §5 line 128: "An outcome or trigger is not a payload: miss, hit 95%, push config get rewritten as the data that crosses".
  - fix: c -->|"1· read root-tablet pointer"| ch
- **MINOR · §5** · 'hold' describes a persistent state, not the operation that crosses the edge; the crossing action is the periodic lock renewal that keeps the server serving.
  - subject: `ts -->|"hold liveness lock"| ch`
  - evidence: Diagram line 22; DIAGRAMS.md §5 lines 128-129 (outcomes/triggers are rewritten as the data that crosses) and §6 store edges = "the operation + argument"; 04-deep-dives.md lines 101-102 and 135-138 describe the per-server liveness lock held/renewed while serving.
  - fix: ts -->|"renew liveness lock"| ch
- **MINOR · §6** · A store edge must name the operation plus argument, but this label names only the payload and leaves the op implicit.
  - subject: `ts -.->|"SSTable files"| fs`
  - evidence: Diagram line 20; DIAGRAMS.md §6 lines 151-153: "stores: the operation + argument, e.g. `OP arg`".
  - fix: ts -.->|"flush SSTable files"| fs
- **MINOR · §6** · Control-plane edges crossing the control-plane boundary carry content but no contract form (no RPC method + transport, no produce:/consume: event name) — same for the reverse edge `m -.->|"tablet assignment · move orders"| ts`.
  - subject: `ts -.->|"load gossip · split notify"| m`
  - evidence: Diagram lines 23-24; DIAGRAMS.md §6 lines 143-157: every boundary-crossing edge carries "METHOD /path + transport · format; RPC method + transport; produce:/consume: + event name".
  - fix: ts -.->|"gossip TabletLoad · SplitDone<br/>RPC · proto"| m  (and symmetrically: m -.->|"RPC LoadTablet / MoveOrder<br/>gossip push"| ts)

## 13.kafka — 7 findings

- **MINOR · §2** · The controller quorum is a coordination/metadata component, which the §2 emoji table defines as 🗝️, but the node is drawn bare while the Kafka topics carry 📨.
  - subject: `ctl[controller quorum<br/>KRaft-style · 3-5 nodes<br/>assignment · leader election]`
  - evidence: Diagram line: `ctl[controller quorum<br/>KRaft-style · 3-5 nodes<br/>assignment · leader election]` — no emoji; DIAGRAMS.md §2 table row "| coordination (ZK, etcd) | 🗝️ |"; 03-architecture.md Components table: "Controller/quorum ... a metadata service the rest depends on".
  - fix: `ctl[🗝️ controller quorum<br/>KRaft-style · 3-5 nodes<br/>assignment · leader election]`
- **MINOR · §4** · The same semantic (dotted consume from topic) is drawn in two opposite directions: topic→consumer for cg but consumer→topic for og, so a reader decodes og's edge as a push/command from the groups into the topic.
  - subject: `og -.->|"consume: orders · clickstream<br/>independent cursor"| t`
  - evidence: Diagram line: `og -.->"|consume: orders · clickstream<br/>independent cursor"| t` vs same block `t -.->|"consume: orders<br/>fetch · sendfile"| cg`; DIAGRAMS.md §7 reference line: `k -.->|"consume: post.fanned"| a`
  - fix: `t -.->|"consume: orders · clickstream<br/>independent cursor"| og`
- **MINOR · §5** · "on error" is a trigger condition, not data crossing the edge (§5), and the edge targets the topic-partitions node, which a reader decodes as "the topic serves cluster metadata" instead of the metadata/quorum plane.
  - subject: `pr -->|"metadata request<br/>on error"| t`
  - evidence: Diagram line: `pr -->|"metadata request<br/>on error"| t`; DIAGRAMS.md §5: "An outcome or trigger is not a payload: `miss`, `hit 95%`, `push config` get rewritten as the data that crosses"; 03-architecture.md: "producers refresh metadata on error and retry" (metadata is cluster metadata, not per-topic data).
  - fix: `pr -->|"metadata.refresh request<br/>TCP · Kafka wire protocol"| ctl`
- **MINOR · prose-trace** · The warehouse group matches the prose, but "fraud" and "alerting" consumer groups are diagram-only inventions; the case prose defines only streaming-consumer services and warehouse batch loads as consumers.
  - subject: `og((🧑 other groups<br/>warehouse · fraud · alerting))`
  - evidence: Diagram line: `og((🧑 other groups<br/>warehouse · fraud · alerting))`; 01-requirements.md lines 3–5: "clickstream, order events, change-data-capture, metrics — feeding both streaming consumers (services reacting in real time) and batch consumers (data warehouse loads)"; no fraud/alerting group appears anywhere in the case files.
  - fix: `og((🧑 other groups<br/>warehouse · streaming services))`
- **MINOR · §6** · The edge describes operations ("join", "commit") rather than the RPC method names that §6 requires as the contract for a cross-boundary RPC edge with a transport line.
  - subject: `cg -->|"join · commit offset<br/>TCP · Kafka wire protocol"| gcw`
  - evidence: Diagram line: `cg -->|"join · commit offset<br/>TCP · Kafka wire protocol"| gcw`; DIAGRAMS.md §6: "RPC: method name; each cross-boundary HTTP/RPC edge names its transport".
  - fix: `cg -->|"JoinGroup · OffsetCommit<br/>TCP · Kafka wire protocol"| gcw`
- **MINOR · §5** · "fetch · sendfile" describes how the broker serves the read (mechanism), not what crosses the edge, which §5 says must be the payload/event/query.
  - subject: `t -.->|"consume: orders<br/>fetch · sendfile"| cg`
  - evidence: Diagram line: `t -.->|"consume: orders<br/>fetch · sendfile"| cg`; DIAGRAMS.md §5: "An edge only moves data. Its label states what crosses: payload, event name, or query." (sendfile mechanism belongs in the prose/broker node, deep dives §6).
  - fix: `t -.->|"consume: orders<br/>byte ranges via sendfile"| cg`
- **MINOR · prose-trace** · The label attaches the P0..P119 (120-partition) range to both named topics, while the prose gives 120 partitions only to the single hottest of 500 topics, so a reader infers 120 partitions per topic.
  - subject: `t{{📨 topic partitions<br/>orders · clickstream · P0..P119<br/>leader + ISR followers}}`
  - evidence: Diagram line: `t{{📨 topic partitions<br/>orders · clickstream · P0..P119<br/>leader + ISR followers}}`; 02-scale-estimates.md: "500 topics; hottest topic carries ~10% of traffic; it has **120 partitions**"
  - fix: `t{{📨 topic partitions<br/>500 topics · hot topic P0..P119<br/>leader + ISR followers}}`

## 14.ml-production-system — 5 findings

- **MAJOR · prose-trace** · The diagram draws a cross-deploy gRPC hop between feature service and model server, but the case's central serving trade-off is that scoring is in-process in one co-located binary with no separate inference-service hop.
  - subject: `fs -->|"3· gRPC Score()<br/>proto · packed vectors"| ms`
  - evidence: Diagram line "fs -->|"3· gRPC Score()<br/>proto · packed vectors"| ms" vs 03-architecture.md lines 56-58 ("the only network hops are the feature fetches ... the model itself runs in-process, no separate 'inference service' hop (trade-off in deep dive 4)") and 04-deep-dives.md lines 166-176 (Option 'B with isolation' wins: 'one co-located serving binary, many in-process models' beats 'separate inference service per model over gRPC').
  - fix: Make the model server a subroutine stage of the serving binary and drop the transport line: "ms[[model server<br/>N versions in-process<br/>champion · challenger · shadow]]" with edge "fs -->|"3· Score()"| ms" (unlabeled transport; per §6 the Score() call is a function call inside one deploy unit)
- **MAJOR · prose-trace** · The edge direction is reversed: the diagram shows the offline feature store feeding the OLAP warehouse, but the prose says the warehouse's daily batch tables feed the offline store's batch side.
  - subject: `offl -.->|"SQL daily batch tables"| olap`
  - evidence: Diagram line "offl -.->|"SQL daily batch tables"| olap" vs 03-architecture.md lines 63-64 ("The warehouse's daily batch tables feed the offline store's batch side (deep dive 2)") and 04-deep-dives.md lines 82-85 ("warehouse = source of truth for batch-computed features (rebuilt daily)").
  - fix: "olap -.->|"SQL daily batch tables"| offl"
- **MAJOR · prose-trace** · The event log never feeds the online feature store, so the diagram shows the online KV cluster (o) never being populated — the diagram alone cannot explain where online features come from, contradicting the 'one spine, three readers' claim and the streaming-freshness requirement.
  - subject: `ev{{📨 event log<br/>actions · raw context}}`
  - evidence: Diagram has ev consuming only into offl and bud, with no edge into o; vs 03-architecture.md line 79 ("One spine, three readers: the event stream feeds features, training and monitoring") and 01-requirements.md line 36 ("streaming features < ~5 min stale"), supported by 02-scale-estimates.md line 77 ("streaming feature updates ... ≈ 58K/s").
  - fix: "ev -.->|"consume: feature updates"| o" (streaming aggregates are what populate the online KV, per 04-deep-dives.md deep dive 2)
- **MINOR · §6** · A solid edge into a store (reg is a cylinder) must be labeled as operation + argument, not as a verb/outcome phrase; what crosses is the pointer lookup, not an 'artifact load'.
  - subject: `ms -->|"load production artifact"| reg`
  - evidence: Diagram line "ms -->|"load production artifact"| reg" vs DIAGRAMS.md §6 line 151 ("stores: the operation + argument, e.g. `OP arg` or `SQL <table/purpose>`") and §5 line 128 (outcomes rewritten as the data that crosses).
  - fix: "ms -->|"SQL production pointer"| reg" (pointer flip read; artifact bytes load locally per 03-architecture.md line 45 "artifact store pointer")
- **MINOR · §6** · The store edge states the payload but omits the operation, so it does not decode as a keyed-store write of the form OP arg.
  - subject: `bud -->|"vN candidate + receipts"| reg`
  - evidence: Diagram line "bud -->|"vN candidate + receipts"| reg" vs DIAGRAMS.md §6 line 151 ("stores: the operation + argument, e.g. `OP arg`").
  - fix: "bud -->|"put vN + eval receipts"| reg" (register step per 01-requirements.md line 15 "register the artifact; run the promotion workflow")

## 15.ml-ranking-system — 4 findings

- **MINOR · prose-trace** · The label implies the serving ANN fleet uses HNSW shards, but deep dive 2 explicitly chose sharded IVF/brute-force as the primary and admitted HNSW only for per-user "more like this" rails, which are not the request path drawn here.
  - subject: `ann[two-tower ANN fleet<br/>HNSW / IVF shards]`
  - evidence: Diagram line: "ann[two-tower ANN fleet<br/>HNSW / IVF shards]"; /home/pranamulia/system-design/study_cases/15.ml-ranking-system/04-deep-dives.md lines 75-77: "Chosen: two-tower + sharded IVF/quantized brute-force as the primary ... HNSW admitted at the edge cases where incremental insert dominates (per-user 'more like this item' rails)".
  - fix: ann[two-tower ANN fleet<br/>sharded IVF / brute-force]
- **MINOR · prose-trace** · The label claims the fresh source only holds items under 15 minutes old, but deep dive 3 says the fresh index holds minutes-to-hours-old items and only hands off the prior once items are hours old; 01 uses <15 min as an eligibility threshold, not the index contents.
  - subject: `frsh[trending + fresh source<br/>items under 15 min]`
  - evidence: Diagram line: "frsh[trending + fresh source<br/>items under 15 min]"; /home/pranamulia/system-design/study_cases/15.ml-ranking-system/04-deep-dives.md line 142-143: "a tiny recency-ordered stream-backed index holding minutes-to-hours-old items"; 01-requirements.md line 11: "an item published < 15 min ago is eligible to rank".
  - fix: frsh[trending + fresh source<br/>minutes-to-hours-old items]
- **MINOR · §6** · This cross-deploy-boundary RPC edge names only the transport and payload but no RPC method name, which §6 requires for every cross-boundary RPC edge.
  - subject: `mg -->|"2· ~2,000 cands<br/>gRPC · proto"| fa`
  - evidence: Diagram line: "mg -->|"2· ~2,000 cands<br/>gRPC · proto"| fa"; /home/pranamulia/system-design/study_cases/DIAGRAMS.md §6 (lines 155-156): "RPC: method name; each cross-boundary HTTP/RPC edge names its transport"; contrast the compliant "fa -->|"3· gRPC Score()<br/>packed vectors · proto"| rk" in the same block.
  - fix: mg -->|"2· gRPC Candidates()<br/>~2,000 cands · proto"| fa
- **MINOR · §6** · This cross-boundary RPC edge names only the transport and payload but no RPC method name, which §6 requires for every cross-boundary RPC edge.
  - subject: `rk -->|"predicted scores<br/>gRPC · proto"| rr`
  - evidence: Diagram line: "rk -->|"predicted scores<br/>gRPC · proto"| rr"; /home/pranamulia/system-design/study_cases/DIAGRAMS.md §6 (lines 155-156): "RPC: method name; each cross-boundary HTTP/RPC edge names its transport"; the same block's "u ==>|"1· POST /rank<br/>HTTPS · JSON"| ap" and "fa -->|"3· gRPC Score()<br/>packed vectors · proto"| rk" carry method names.
  - fix: rk -->|"gRPC Scores()<br/>predicted scores · proto"| rr

## 16.dalle — 5 findings

- **MINOR · §6** · The worker-to-human-review edge crosses a deploy boundary but its label is a bare payload description, not the produce:/consume: contract §6 requires.
  - subject: `wk -.->|"borderline image"| rv[human-review<br/>borderline holds]`
  - evidence: Diagram line: `wk -.->|"borderline image"| rv[human-review<br/>borderline holds]` — spec study_cases/DIAGRAMS.md §6 lines 142-149 (every edge crossing a deploy boundary carries METHOD /path + transport, or produce:/consume: + event name); 04-deep-dives.md §4 lines 128-130 ("a small borderline band routes to human review")
  - fix: wk -.->|"produce: image.review · borderline band"| rv[human-review<br/>borderline holds]
- **MINOR · §6** · The webhook push to the client crosses a deploy boundary but 'webhook:' is not a §6 contract form; an HTTP callback should carry method + path with transport/format.
  - subject: `js -.->|"webhook: image.done<br/>at-least-once · client dedup"| u`
  - evidence: Diagram line: `js -.->|"webhook: image.done<br/>at-least-once · client dedup"| u` — spec study_cases/DIAGRAMS.md §6 lines 146,154 (HTTP contract = method + path, then transport and body format); 04-deep-dives.md lines 20-24 (polling is chosen baseline, webhooks are an API-user upgrade)
  - fix: js -.->|"POST /client webhook<br/>image.done · at-least-once dedup"| u
- **MINOR · prose-trace** · The blob-store label claims a count of '3 derived sizes' that no case file states or derives; prose only says 'derived sizes' / '~110 KB derived thumbs'.
  - subject: `rb[(🪣 result blob store<br/>full-res + 3 derived sizes<br/>immutable · TTL)]`
  - evidence: Diagram line: `rb[(🪣 result blob store<br/>full-res + 3 derived sizes<br/>immutable · TTL)]` — 02-scale-estimates.md line 48 ("~110 KB derived thumbs", no count); 03-architecture.md line 32 ("full-res + derived sizes, immutable, TTL'd") — no "3" anywhere in the case prose
  - fix: rb[(🪣 result blob store<br/>full-res + derived sizes<br/>immutable · TTL)]
- **MINOR · prose-trace** · The diagram shows CDN only pulling from origin on miss, while the prose says derived sizes are proactively pushed to CDN on completion rather than pulled on first miss — a reader following the diagram alone decodes pull-only pre-generation.
  - subject: `cdn -->|"GET object"| rb`
  - evidence: Diagram lines: `u -->|"GET object via signed URL<br/>HTTPS · JPEG"| cdn[/🌍 CDN/]` and `cdn -->|"GET object"| rb` — 04-deep-dives.md §5 lines 149-150 ("derived sizes are proactively pushed to CDN on job completion (creation is the known access event) rather than pulling on first miss")
  - fix: rb -.->|"warm: derived sizes"| cdn[/🌍 CDN/]   (added alongside the existing pull-through edge)
- **MINOR · prose-trace** · Only the webhook delivery path is drawn, though the prose names polling the universal baseline and webhooks the upgrade — a reader of the diagram alone infers push-only completion.
  - subject: `js -.->|"webhook: image.done"| u (webhook-only completion path)`
  - evidence: Diagram line: `js -.->|"webhook: image.done<br/>at-least-once · client dedup"| u` — 04-deep-dives.md lines 20-24 ("Poll = status reads from a cheap cached read model... Chosen: polling as the universal baseline, webhooks as an API-user upgrade — both emit from the same state machine")
  - fix: js -.->|"poll · push: image.done webhook<br/>at-least-once · client dedup"| u

## 17.elevenlabs — 5 findings

- **MAJOR · prose-trace** · The diagram draws one GPU pool serving both interactive and batch workloads (and the caption asserts "share one GPU pool"), but the prose explicitly rejects the one-pool design in favor of separate schedulers/pools on shared hardware with batch preemptible at segment boundaries.
  - subject: `gp[🧠 gpu-inference-pool … interactive · batch · LRU voice cache] and q -.->|"consume: synth.segment"| gp`
  - evidence: 03-architecture.md line 11: `gp[🧠 gpu-inference-pool<br/>TTS → PCM chunks<br/>interactive · batch · LRU voice cache]`; line 16: `q -.->|"consume: synth.segment"| gp`; line 5 caption: `Q: How do interactive streaming and batch synthesis share one GPU pool — and what does every synthesis check before it spends GPU-seconds?`; vs 03-architecture.md line 52: "segment jobs are enqueued to the **batch pool** only"; 04-deep-dives.md lines 166-175 (Option C: separate schedulers, batch pool of ~50 GPUs, interactive always wins); 01-requirements.md line 54 ("batch NEVER degrades interactive latency").
  - fix: sc ==>|"gRPC InferSegment()<br/>stream_id · voice_id"| gp[🧠 gpu-inference-pool<br/>TTS → PCM chunks<br/>interactive · LRU voice cache] and change line 16 to: q -.->|"consume: synth.segment"| bp[🧠 batch-pool<br/>preemptible at segment boundaries] (and reword the Q: caption to "...share one GPU fleet")
- **MAJOR · prose-trace** · The pre-GPU-spend authorization/registry check is drawn as performed by the GPU pool after the ==> dispatch edge, but the prose puts the registry lookup and synthesis-time refusal in the stream controller before GPU dispatch — so the caption's second question is decoded backwards by the diagram.
  - subject: `gp -->|"SQL voice lookup"| vr`
  - evidence: 03-architecture.md line 12: `gp -->|"SQL voice lookup"| vr` following line 11's `sc ==>|"infer segment…"| gp`; vs 03-architecture.md lines 65-67: "voice_id must resolve to the caller or a shared scope workspace, via the registry … This check is cheap and sits in front of every GPU-seconds purchase"; line 62: "(controller reads a replica)"; 04-deep-dives.md lines 207-208: "refused in the controller *before* GPU dispatch".
  - fix: sc -->|"SQL voice lookup · refusal check"| vr  (delete the gp -->|"SQL voice lookup"| vr edge)
- **MINOR · §6** · The cross-boundary edge carries only a payload noun ("text deltas") with no METHOD/path or RPC method name and no transport · format line, so it does not decode alone per §6.
  - subject: `wsg -->|"text deltas"| sc`
  - evidence: 03-architecture.md line 10: `wsg -->|"text deltas"| sc`; vs DIAGRAMS.md §6: "Every edge crossing a deploy boundary carries the contract in this order" with HTTP/RPC examples, and "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto`)".
  - fix: wsg -->|"gRPC DeliverDelta()<br/>text deltas · proto"| sc
- **MINOR · §6** · The primary-path cross-boundary edge's label is a description ("infer segment") plus arguments, not a contract of the form METHOD /path or RPC Method() with a named transport.
  - subject: `sc ==>|"infer segment<br/>stream_id · voice_id"| gp`
  - evidence: 03-architecture.md line 11: `sc ==>|"infer segment<br/>stream_id · voice_id"| gp`; vs DIAGRAMS.md §6: `-->|"gRPC GetDegree()"|` — RPC edges carry a method name plus transport.
  - fix: sc ==>|"gRPC InferSegment()<br/>stream_id · voice_id"| gp
- **MINOR · §6** · The edge label's second line is `HTTPS · origin pull`, where the format slot holds a mechanism word instead of a body format, so the label does not match the §6 contract shape.
  - subject: `cdn -->|"GET /batch-objects/*<br/>HTTPS · origin pull"| ob`
  - evidence: 03-architecture.md line 18: `cdn[/🌍 CDN<br/>vendor/] -->|"GET /batch-objects/*<br/>HTTPS · origin pull"| ob`; vs DIAGRAMS.md §6: `-->|"METHOD /path<br/>transport · format"|`; §5: "An outcome or trigger is not a payload … `push config` get rewritten as the data that crosses".
  - fix: cdn[/🌍 CDN<br/>vendor/] -->|"GET /batch-objects/*<br/>HTTPS · bytes"| ob

## 18.sora — 10 findings

- **MAJOR · §4** · A `==>` edge is drawn from the orchestrator to the GPU fleet although generation is a multi-minute asynchronous job the orchestrator does not wait on, and together with the u ==> edge the diagram carries two thick primary arrows where the spec reserves ==> for the single primary sync path.
  - subject: `jo ==>|"chunk jobs"| mf`
  - evidence: Diagram line 13: `jo ==>|"chunk jobs"| mf[🧠 main-gpu-fleet<br/>chunked latent generation]`; 03-architecture.md line 9: `u((🧑 client)) ==>|"POST /generations<br/>HTTPS · JSON"| edge[...]` (the second ==>). Spec §4: `==> | primary path | the one data path the case is about`. Prose: 04-deep-dives.md line 71-72 "The orchestrator owns state; GPUs are stateless executors that pull work." and 03-architecture.md line 81 "any GPU or worker death is invisible to correctness — the orchestrator re-runs unfinished chunk-jobs". 02-scale-estimates.md line 25-27: even the submission API is "one modest API pod", so the GPU dispatch is not a sync call the user-side path waits on.
  - fix: jo -.->|"produce: job.chunks<br/>gRPC LeaseChunkJob() · GPUs pull"| mf[main-gpu-fleet<br/>chunked latent generation]  (keep u ==>|"1· POST /generations<br/>HTTPS · JSON"| edge as the diagram's single ==> primary path)
- **MAJOR · §6** · The api-edge → prompt-intake call crosses a deploy boundary but its label states only the payload, with no METHOD /path + transport · format contract as §6 requires for every cross-boundary edge.
  - subject: `edge -->|"validated prompt"| pin`
  - evidence: Diagram line 9: `edge -->|"validated prompt"| pin[prompt-intake<br/>text safety classifiers<br/>prompt-hash dedupe]`; spec §6 line 143-149: "Every edge crossing a deploy boundary carries the contract in this order ... -->|"METHOD /path<br/>transport · format"|".
  - fix: edge -->|"POST /intake<br/>HTTPS · JSON (prompt + params)"| pin[prompt-intake<br/>text safety classifiers<br/>prompt-hash dedupe]
- **MAJOR · §6** · The prompt-intake → job-orchestrator call crosses a deploy boundary but carries no contract (method/path and transport · format), only the payload noun "shaped job".
  - subject: `pin -->|"shaped job"| jo`
  - evidence: Diagram line 10: `pin -->|"shaped job"| jo[job-orchestrator<br/>job DAG · priorities · budgets<br/>checkpoints · credits]`; spec §6 line 154-156: "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto")".
  - fix: pin -->|"POST /v1/jobs<br/>HTTPS · JSON (validatable params)"| jo[job-orchestrator<br/>job DAG · priorities · budgets<br/>checkpoints · credits]
- **MAJOR · §6** · The post-assembly worker is shown with no edge to the metadata DB, although the prose makes job/artifact state transitions (including marking chunks `assembled`/`reused` and writing rendition locations) metadata-DB records, so the diagram under-represents a store the prose treats as strongly consistent for job/artifact state.
  - subject: `pp -->|"read latents · write renditions"| ar`
  - evidence: Diagram line 17: `pp[post-assembly<br/>stitch → upscale → encode renditions]` — no edge connects pp to the metadata DB. Prose: 03-architecture.md lines 49-51 "**Metadata DB**: job records (id, user, prompt, model version, params), artifact records (location, verdict, watermark id, TTLs), credit balances. Strongly consistent for job/artifact state"; lines 55-73 flows 1 and 6 write and read job/artifact state; §1 (flow 1) "writes the job, charges the user's credits, returns job id ~instantly" and 04-deep-dives.md line 41 "treat 'verdict exists' as an invariant, checked at publish".
  - fix: pp -->|"SQL artifacts · job=assembled"| mdb[(metadata DB<br/>jobs · assets · verdicts<br/>credit balances)]
- **MAJOR · §6** · The post-assembly → screening-svc call crosses a deploy boundary with no API contract on the edge.
  - subject: `pp -->|"final artifact"| srv`
  - evidence: Diagram line 18: `pp -->|"final artifact"| srv[screening-svc<br/>artifact + latent classifiers<br/>human review queue]`; spec §6 line 146: `-->|"METHOD /path<br/>transport · format"|`.
  - fix: pp -->|"POST /screen<br/>HTTPS · JSON (artifact id)"| srv[screening-svc<br/>artifact + latent classifiers<br/>human review queue]
- **MAJOR · §6** · The screening-svc → publish-and-watermark call crosses a deploy boundary with no contract, and the label word "cleared" reads as an outcome rather than a contract per §5.
  - subject: `srv -->|"cleared artifact"| pub`
  - evidence: Diagram line 20: `srv -->|"cleared artifact"| pub[publish-and-watermark<br/>watermark embed · C2PA manifest]`; spec §6 line 148: `-->|"produce: event.name"|` / RPC or HTTP contract for cross-boundary edges; 03-architecture.md line 48: "Position matters: *before* publish, no exceptions."
  - fix: srv -->|"POST /publish<br/>HTTPS · JSON (artifact id, verdict=clear)"| pub[publish-and-watermark<br/>watermark embed · C2PA manifest]
- **MINOR · §6** · The publish → CDN edge carries a transport but no METHOD /path, so it does not decode as a complete cross-boundary contract the way §6 requires.
  - subject: `pub -->|"renditions<br/>HTTPS"| cdn`
  - evidence: Diagram line 21: `pub -->|"renditions<br/>HTTPS"| cdn[/🌍 CDN/]`; spec §6 line 146 requires `METHOD /path<br/>transport · format` (transport alone is not a contract); prose 03-architecture.md lines 72-73: "renditions pushed to the CDN, user notified".
  - fix: pub -->|"PUT renditions<br/>HTTPS"| cdn[/🌍 CDN/]
- **MINOR · §5** · The second line of the store-edge label, "chunk 2 on 1", states an ordering/conditionality outcome rather than the data that crosses the edge, which §5 disallows in an edge label.
  - subject: `mf -->|"GET boundary latents<br/>chunk 2 on 1"| ar`
  - evidence: Diagram line 15: `mf -->|"GET boundary latents<br/>chunk 2 on 1"|`; spec §5 line 128-129: "An outcome or trigger is not a payload: `miss`, `hit 95%`, `push config` get rewritten as the data that crosses".
  - fix: mf -->|"GET boundary latents"| ar
- **MINOR · §2** · The 🧠 emoji is defined in the §2 table for an ML model / LLM node, but it is applied to the GPU compute fleet (a deployed execution unit), using a table emoji for a component type the table does not cover instead of leaving the node bare.
  - subject: `mf[🧠 main-gpu-fleet]`
  - evidence: Diagram line 13: `mf[🧠 main-gpu-fleet<br/>chunked latent generation]`; spec §2 line 56-57 table row "ML model / LLM | 🧠" and lines 63-65: "No emoji exists for a plain service — leave it bare. Don't invent a new one per diagram."
  - fix: mf[main-gpu-fleet<br/>chunked latent generation]
- **MINOR · prose-trace** · The prose's job DAG has three GPU pools (draft, main, post) and the happy-path timeline makes the draft pass step 2 before any main-pool chunking, but the diagram draws only the main pool, so a reader following the diagram sees the DAG start directly with the expensive chunked generation and misses the cheap draft pass.
  - subject: `mf[🧠 main-gpu-fleet<br/>chunked latent generation] (only GPU pool drawn)`
  - evidence: Prose: 03-architecture.md lines 34-38 "**GPU fleet, three pools**: - *Draft pool*: low-cost passes ... to kill bad prompts early ..." and lines 61-63 flow 2 "Draft pass on the draft pool (~15–30s): either the user sees an early preview and can cancel/adjust ... or it's skipped; 04-deep-dives.md line 79-80: "**Draft passes on the draft pool** cheaply reject prompts that produce garbage before the main pool is touched."
  - fix: jo -.->|"produce: draft.pass<br/>~1/10 of a job"| dp[draft-gpu-fleet<br/>low-res early preview pass]

## 19.frontend-chat-app — 3 findings

- **MINOR · prose-trace** · The ui-state label claims a "drafts" state that no prose file defines or mentions.
  - subject: `st[ui-state<br/>message window · optimistic index<br/>typing · presence · drafts]`
  - evidence: Diagram line: `st[ui-state<br/>message window · optimistic index<br/>typing · presence · drafts]` | Prose: 03-architecture.md:29 defines ephemeral state as "typing, presence" only; "drafts" appears nowhere in 19.frontend-chat-app/{README,01-requirements,02-scale-estimates,03,04} (only the composer/compose activity is mentioned, never a drafts store).
  - fix: st[ui-state<br/>message window · optimistic index<br/>typing · presence]
- **MINOR · §4** · The single store edge names only the outbox PUT, but the caption's second question depends on live-event and backfill writes to the messages sub-store crossing the same edge, whose operation is named nowhere in the diagram.
  - subject: `se -->|"1· PUT outbox:{client_msg_id}"| db[(local store · IndexedDB / SQLite<br/>messages by conv,seq · outbox<br/>read markers · media cache LRU)]`
  - evidence: Diagram line: `se -->|"1· PUT outbox:{client_msg_id}"| db[(local store · IndexedDB / SQLite<br/>messages by conv,seq · outbox<br/>read markers · media cache LRU)]` | Prose: 03-architecture.md:56-58 receive path "writes to store in one batched transaction"; caption 03-architecture.md:7 "how do live events, backfill, and media reach the same store through one writer"; §4: "A `-->` edge into a store is the operation named in its label".
  - fix: se -->|"1· PUT outbox:{client_msg_id}<br/>PUT messages batch (recv · backfill)"| db[(local store · IndexedDB / SQLite<br/>messages by conv,seq · outbox<br/>read markers · media cache LRU)]
- **MINOR · prose-trace** · The edge numbered 3· (send-path step 3, the outbox drain POST) also carries "last_seq per conv", which belongs to the prose's distinct reconnect/backfill sync request, so the number ties the backfill flow to the wrong prose step.
  - subject: `se -->|"3· POST /batch_sync<br/>HTTPS · JSON · last_seq per conv"| srv`
  - evidence: Diagram line: `se -->|"3· POST /batch_sync<br/>HTTPS · JSON · last_seq per conv"| srv` | Prose: 03-architecture.md:51 "Background drain loop POSTs it" (send step 3) vs 03-architecture.md:61-62 and 04-deep-dives.md:126 "on reconnect: one batched `sync(conversations -> last_seq)`" — two flows the prose distinguishes but the edge merges under the send-path number.
  - fix: se -->|"3· POST /batch_sync (drain · reconnect backfill)<br/>HTTPS · JSON · last_seq per conv"| srv

## 20.frontend-newsfeed — 7 findings

- **MAJOR · prose-trace** · The cold-start flow that delivers page 1 and its cursor token into the feed store is not drawn, so a reader following the diagram alone cannot see where the pagination cursor originates.
  - subject: `fs[(feed store<br/>append-only paged log · page 1 inline-seeded<br/>entry_id dedup · stable anchors)]`
  - evidence: Diagram line 18: `fs[(feed store<br/>append-only paged log · page 1 inline-seeded<br/>entry_id dedup · stable anchors)]` — no edge from `ssr` to `fs` exists anywhere in the block. Prose: 03-architecture.md:66-67 "inline payload seeds the feed store (page 1)", 03-architecture.md:33-34 "plus an inline JSON payload (page 1 + cursor token + variant flags)".
  - fix: Add after line 16: `ssr -.->|"inline payload<br/>page-1 entries · cursor token"| fs`
- **MAJOR · prose-trace** · The fetched page's arrival in the store (the append+dedup step of the scroll path) is never drawn, leaving the prefetch fetch and the winder's page reads unconnected.
  - subject: `pf -->|"GET page N+1<br/>HTTPS · last-page anchor cursor"| tla`
  - evidence: Diagram lines 19-20 draw only `win -->|"scroll position · velocity"| pf` and `pf -->|"GET page N+1<br/>HTTPS · last-page anchor cursor"| tla`; no edge carries the fetched page into `fs`. Prose: 03-architecture.md:73-74 "fetch page N+1 with the *last page's anchor token* (dive 4) → store appends page, dedups"; 04-deep-dives.md:17-19 "the store appends page N+1 keyed by the anchor token of page N".
  - fix: Add: `pf -->|"append: page N+1 · next cursor token"| fs`
- **MINOR · §6** · A cross-boundary HTTP edge carries a method+path but omits the transport · format line the spec requires.
  - subject: `ssr -->|"GET /variants/sticky"| ab[ab-decider<br/>assignment before HTML render]`
  - evidence: Diagram line 16: `ssr -->|"GET /variants/sticky"| ab[ab-decider<br/>assignment before HTML render]` — has method+path but no second transport · format line; DIAGRAMS.md:155 "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto")"; the prose makes them separate (03-architecture.md:246 "couples the SSR hot path to the experiment service").
  - fix: `ssr -->|"GET /variants/sticky<br/>HTTPS · JSON"| ab[ab-decider<br/>assignment before HTML render]`
- **MINOR · §6** · The API edge label has method but no path — "page N+1" describes the argument, not a `/path`.
  - subject: `pf -->|"GET page N+1<br/>HTTPS · last-page anchor cursor"| tla`
  - evidence: Diagram line 20: `pf -->|"GET page N+1<br/>HTTPS · last-page anchor cursor"| tla[...]`; DIAGRAMS.md:146 `-->|"METHOD /path<br/>transport · format"|` and :154 "method + path, then transport and body format" (cf. the compliant `pf` sibling in §7: `1· GET /v1/feed`).
  - fix: `pf -->|"GET /v1/feed/next<br/>HTTPS · after:{anchor}"| tla["timeline-api<br/>Case 02: pre-ordered entries<br/>~15 items / page"]`
- **MINOR · §6** · The new-stories fetch edge has method but no path — "items above top anchor" is a description standing where `/path` belongs.
  - subject: `isl -->|"POST /v1/like<br/>HTTPS · JSON"| tla alongside -->|"GET items above top anchor<br/>HTTPS · JSON"|`
  - evidence: Diagram line 25: `isl[...] -->|"GET items above top anchor<br/>HTTPS · JSON"| tla`; DIAGRAMS.md:146 defines the edge label as `METHOD /path<br/>transport · format`.
  - fix: `isl[...] -->|"GET /v1/feed/above-top<br/>HTTPS · JSON"| tla`
- **MINOR · §5** · The edge into the store leads with the action performed on the data ("hydrate") instead of the operation+argument of the store access.
  - subject: `isl -->|"hydrate entry_id list"| fs`
  - evidence: Diagram line 21: `isl[...] -->|"hydrate entry_id list"| fs`; DIAGRAMS.md:125-127 "An edge only moves data. Its label states *what crosses*: payload, event name, or query... not `-->|"aggregates popularity"|`"; DIAGRAMS.md:105 "A `-->` edge into a store is the operation named in its label" — cf. compliant store edge on line 18 `-->|"read page by anchor"|`.
  - fix: `isl[...] -->|"read entries by entry_id"| fs`
- **MINOR · §4** · The store-write edge is labeled with a mode adjective ("optimistic") instead of the store operation and its argument, so the write is only implied, not named.
  - subject: `isl -->|"optimistic like · comment"| fs`
  - evidence: Diagram line 23: `isl -->|"optimistic like · comment"| fs`; DIAGRAMS.md:105-106 "read-modify-write is one edge labeled e.g. `read+write seen-set`"; DIAGRAMS.md:128-129 outcome/trigger wording gets rewritten as the data that crosses.
  - fix: `isl -->|"write like · comment state (optimistic)"| fs`

## 21.frontend-calendar — 8 findings

- **MAJOR · prose-trace** · The change hub is drawn as a writer to the local store, but the prose defines it as the only notify path and never has it write to the store; flow 4's notify even originates from the store side.
  - subject: `hub --> ls`
  - evidence: Diagram line: `hub --> ls` (03-architecture.md:26). Prose: study_cases/21.frontend-calendar/03-architecture.md:59-61 "**Change hub** — the only notify path. UI/state subscribers subscribe to *keys* ... an edit invalidates exactly the affected derived state" and :87 "change hub notifies affected keys" — notify flows store/changes → hub → subscribers (sel), never hub → store.
  - fix: ls -->|"changed keys · versioned"| hub
- **MAJOR · prose-trace** · The diagram omits the central optimistic write edge: the UI's immediate write of the edited event row (plus outbox append) to the local store, which is what makes the store the only source of truth.
  - subject: `local store writers (no UI → ls edge)`
  - evidence: Diagram lines 23-36 contain no edge from ui (or any edit-source) into ls with a write contract; the only ls writers drawn are `hub --> ls` and `sync -->|"append outbox op<br/>read+write deltas+token"| ls`. Prose: study_cases/21.frontend-calendar/03-architecture.md:74-77 "UI writes to the store immediately (optimistic, the event is 'real' the moment the dialog closes) + appends the mutation to the outbox"; invariant I2 at :99-100.
  - fix: ui -->|"optimistic event row + outbox op<br/>one SQLite tx"| ls
- **MAJOR · prose-trace** · The node's label claims the channel carries 'invalidation hints only', yet the diagram routes a payload-bearing reminder push through the same node, contradicting both itself and the prose model where reminder payloads are display-sufficient.
  - subject: `pch["Push channel<br/>FCM/APNs/WS · invalidation hints only"]`
  - evidence: Diagram lines: `pch["Push channel<br/>FCM/APNs/WS · invalidation hints only"]` (:20) vs `pch -.->|"reminder.push<br/>push · JSON"| bell` (:35). Prose: study_cases/21.frontend-calendar/03-architecture.md:55-56 "A push channel carries only 'invalidate, pull now' hints (never payloads)" vs 04-deep-dives.md:66-68 "reminder payloads must be small (id + summary + fire time)" and 03-architecture.md:92 "a reminder for an event the client has never seen ... is displayed from the push payload alone".
  - fix: pch["Push channel<br/>FCM/APNs/WS · sync hints · reminder payloads"]
- **MINOR · prose-trace** · The sync engine's store edge claims it appends outbox ops, but the prose attributes the outbox append to the UI-side edit flow and scopes the sync engine to 'owns the conversation with the server, nothing else'; its store work is applying deltas and persisting the token.
  - subject: `sync -->|"append outbox op<br/>read+write deltas+token"| ls`
  - evidence: Diagram line: `sync -->|"append outbox op<br/>read+write deltas+token"| ls` (03-architecture.md:30). Prose: study_cases/21.frontend-calendar/03-architecture.md:75 (UI appends the mutation to the outbox) and :48-49 (sync engine "owns the *conversation with the server*, nothing else").
  - fix: sync -->|"apply deltas + persist token<br/>same SQLite tx"| ls
- **MINOR · §4** · No edge uses `==>`; the case's primary sync path (outbox push with client op IDs, dive 1 'THE dive') is drawn as a plain `-->`, so the thick-arrow primary-path marker the standard reserves is missing.
  - subject: `sync -->|"1· push(ops)<br/>HTTPS · JSON — client op IDs"| api`
  - evidence: Diagram line: `sync -->|"1· push(ops)<br/>HTTPS · JSON — client op IDs"| api` uses `-->` (03-architecture.md:32). Spec: DIAGRAMS.md:108-110 "`==>` is reserved for the primary *sync* path"; reference example DIAGRAMS.md:166 draws the primary as `u((🧑 user)) ==>|...| a[feed-api...]`; compliant sibling study_cases/19.frontend-chat-app/03-architecture.md:11-12 marks its core path with `==>`.
  - fix: sync ==>|"1· push(ops)<br/>HTTPS · JSON · client op IDs"| api
- **MINOR · §6** · Cross-boundary sync-API edges carry verb-style names (pull(token), push(ops)) with no METHOD /path, which does not match the spec's HTTP contract shape; adjacent compliant cases use method+path form.
  - subject: `sync -->|"2· pull(token)<br/>HTTPS · JSON"| api`
  - evidence: Diagram lines: `sync -->|"2· pull(token)<br/>HTTPS · JSON"| api` and `sync -->|"1· push(ops)<br/>HTTPS · JSON — client op IDs"| api` (03-architecture.md:31-32). Spec: DIAGRAMS.md:146 `-->|"METHOD /path<br/>transport · format"|` and :154 "method + path, then transport and body format"; compliant sibling study_cases/19.frontend-chat-app/03-architecture.md:22 `-->|"3· POST /batch_sync<br/>HTTPS · JSON · last_seq per conv"|`.
  - fix: sync -->|"2· POST /sync/pull?token<br/>HTTPS · JSON"| api
- **MINOR · §6** · `produce:` is the topic/log contract (spec draws producers into a hexagon), but pch is a push-channel rectangle, not a topic, and the prose defines no event API `reminder.due` — it says the pipeline schedules per-device pushes.
  - subject: `np -.->|"produce: reminder.due"| pch`
  - evidence: Diagram line: `np -.->|"produce: reminder.due"| pch` (03-architecture.md:34); pch node at :20 is a rectangle `pch["Push channel<br/>FCM/APNs/WS · invalidation hints only"]`. Spec: DIAGRAMS.md:18 hexagon = "an append-only topic/log you consume from", :25 "If consumers read it as a log (consume:), draw hexagon"; §6 :149 produce:/consume: is the topic event contract.
  - fix: np -.->|"per-device push schedule"| pch
- **MINOR · prose-trace** · An unlabeled edge from the UI into the notification presenter has no backing flow in the prose; the presenter fires on push receipt or on foreground time passage, not on a UI call.
  - subject: `ui --> bell`
  - evidence: Diagram line: `ui --> bell` (03-architecture.md:29). Prose: study_cases/21.frontend-calendar/04-deep-dives.md:64-66 — "**Notification presenter** — ... Fires display when the device receives the server's reminder push (or when it observes the passage of the time for already-known reminders in foreground)"; no UI→presenter edge is described anywhere in 03/04.
  - fix: (remove the line `ui --> bell`; foreground time-passage trigger stays in the presenter's label/prose)

## 22.frontend-uber — 7 findings

- **MAJOR · prose-trace** · The event-log topic node and its three produce/consume edges are diagram-only inventions: no case file defines a server event log, and 03-architecture.md explicitly declares the doc covers only client-facing contracts of the given backend.
  - subject: `k{{📨 event log<br/>ride events}} with ts -.->|"produce: ride.updated"| k`
  - evidence: Diagram line 39: `ts -.->|"produce: ride.updated"| k` plus node line 29 `k{{📨 event log<br/>ride events}}`; prose 03-architecture.md:107-109 "No server diagram beyond the contract: gateway fanout, matching, snapshot service are given services — this doc defines only their client-facing contracts"; grep for "event log" in the case files finds only 04-deep-dives.md:209 "late event log" (a client-side support log, not a backend topic).
  - fix: Delete `k{{📨 event log<br/>ride events}}` and the three produce/consume edges; the event-delivery contract is already carried by `gw <-->|"WS ride events seq'd<br/>wss · JSON"| rc`.
- **MAJOR · prose-trace** · The ride store's label and the prose declare user intents as one of its two only inputs, but no ui→rs edge carries the intent, so a reader of the diagram alone cannot trace data flow 2 (REQUEST intent → optimistic state).
  - subject: `rs[ride store<br/>state machine browse→trip→done<br/>server events + intents only]`
  - evidence: Diagram line 32-33: `u((🧑 rider)) -->|"taps · gestures"| ui` and `rs --> ui` — no edge from ui to rs; prose 03-architecture.md:81-82 (data flow 2) "**Request**: intent `REQUEST` → optimistic `requesting` state…" and 03-architecture.md:53 "Server events (seq'd) and user intents are the only inputs".
  - fix: ui -->|"RideIntent REQUEST/CANCEL<br/>client UUID"| rs
- **MAJOR · prose-trace** · match.found is produced into the event log but no consumer edge names it, so the central MATCHED flow (F3, data flow 2) dead-ends and a reader cannot trace how the match reaches the rider's realtime client.
  - subject: `ms -.->|"produce: match.found"| k followed by k -.->|"consume: ride.updated"| gw`
  - evidence: Diagram line 40-41: `ms -.->|"produce: match.found"| k` and the only consumer edge `k -.->|"consume: ride.updated"| gw`; prose 03-architecture.md:82 "server event `MATCHED {driver_id, quote, seq}`" and 01-requirements.md F3 "Request & match: rider requests, sees 'matching…' … then the matched driver's car".
  - fix: k -.->|"consume: ride.updated · match.found"| gw
- **MINOR · §5** · "16 ms frame budget" is a performance constraint, not the data that crosses the edge — the label should name the payload (the render target list) and leave the frame budget to the prose.
  - subject: `rs ==>|"render targets<br/>16 ms frame budget"| ren`
  - evidence: Diagram line 34: `rs ==>|"render targets<br/>16 ms frame budget"| ren`; spec DIAGRAMS.md §5 "An edge only moves data. Its label states what crosses: payload, event name, or query." (prose source of the number is 02-scale-estimates.md:101, a render budget, not an edge payload).
  - fix: rs ==>|"render target list<br/>id → position targets"| ren
- **MINOR · §6** · The HTTP contract is "GET tile x,y,z" — a method plus argument instead of method + path as §6 requires for a cross-boundary HTTP edge.
  - subject: `ren -->|"GET tile x,y,z<br/>HTTPS · vector"| cdn`
  - evidence: Diagram line 35: `ren -->|"GET tile x,y,z<br/>HTTPS · vector"| cdn`; spec DIAGRAMS.md §6 "HTTP: |"GET /v1/feed<br/>HTTPS · JSON"| — method + path, then transport and body format."
  - fix: ren -->|"GET /tiles/{z}/{x}/{y}<br/>HTTPS · vector"| cdn
- **MINOR · §4** · "5–20 KB" is the response size annotated on a request-direction edge; the label should state only the request contract.
  - subject: `rs -->|"GET /ride/current<br/>HTTPS · JSON 5–20 KB"| ss`
  - evidence: Diagram line 43: `rs -->|"GET /ride/current<br/>HTTPS · JSON 5–20 KB"| ss`; spec DIAGRAMS.md §4 "Draw the request direction only. Don't draw the response as a second edge" (the 5–20 KB figure is already carried in 02-scale-estimates.md:94 and 03-architecture.md:91 prose).
  - fix: rs -->|"GET /ride/current<br/>HTTPS · JSON"| ss
- **MINOR · §6** · Deep dive 2 chose client-requested cadence (Option B, "re-negotiated with server ack"), but the diagram draws the position channel as server push only, with no client→server subscribe/re-negotiate edge — a reader would decode the rejected Option C (server-driven push).
  - subject: `gw -->|"position batches<br/>wss push · lossy fresh-wins"| rc`
  - evidence: Diagram lines 36-37 show only `gw <-->|"WS ride events seq'd<br/>wss · JSON"| rc` and `gw -->|"position batches<br/>wss push · lossy fresh-wins"| rc`; prose 04-deep-dives.md:78-80 "**Option B — client-requested cadence** (subscribe with phase-specific rate)… subscribe/re-subscribe on transitions" and 04-deep-dives.md:89-90 "cadence rides the same transition events re-negotiated with server ack".
  - fix: rc -->|"subscribe posn · re-neg by phase<br/>wss · JSON"| gw

## 23.frontend-video-streaming — 5 findings

- **MAJOR · §4** · The primary fetch edge points CDN→scheduler while its label is a client-initiated GET, so the request direction reads backwards (the response/delivery direction is drawn, contradicting §4's request-direction rule and both sibling frontend cases where the client's GET points at the CDN).
  - subject: `cdn ==>|"GET /videos/{id}/manifest.m3u8 · /segments/*.m4s…"| sched`
  - evidence: 03-architecture.md:21 `cdn[/🌍 CDN<br/>edge vendor/] ==>|"GET /videos/{id}/manifest.m3u8 · /segments/*.m4s<br/>HTTPS · fMP4"| sched`; DIAGRAMS.md:99 "Draw the request direction only"; precedent 22.frontend-uber/03:36 `ren -->|"GET tile x,y,z<br/>HTTPS · vector"| cdn[/🌍 CDN<br/>map tiles vendor/]`
  - fix: sched ==>|"GET /videos/{id}/manifest.m3u8 · /segments/*.m4s<br/>HTTPS · fMP4"| cdn[/🌍 CDN<br/>edge vendor/]
- **MINOR · §4** · Two edges use the thick arrow although §4 reserves `==>` for the single primary sync path of the diagram.
  - subject: `wrk ==>|"fMP4 fragments…"| mse`
  - evidence: 03-architecture.md:21 (`cdn ==>|…| sched`) and 03-architecture.md:24 (`wrk ==>|"fMP4 fragments<br/>Transferable buffers"| mse`); DIAGRAMS.md:112 "`==>` | primary path | the one data path the case is about"
  - fix: wrk -->|"fMP4 fragments<br/>Transferable buffers"| mse
- **MINOR · §5** · The edge label carries the outcome qualifier "on miss" instead of stating only what crosses the edge, and the path is named loosely ("GET manifest") rather than as METHOD /path per §6.
  - subject: `cdn -->|"GET manifest · /segments/* on miss…"| org`
  - evidence: 03-architecture.md:22 `cdn -->|"GET manifest · /segments/* on miss<br/>HTTPS · fMP4"| org[...]`; DIAGRAMS.md:128 "An outcome or trigger is not a payload: `miss`, `hit 95%`, `push config` get rewritten as the data that crosses"
  - fix: cdn -->|"GET /manifests/{id}.m3u8 · /segments/*<br/>HTTPS · fMP4"| org[origin + manifest service<br/>encoding = Case 01]
- **MINOR · §6** · The only cross-boundary edge out of the browser tab lacks the contract's method+path and body format, naming only the transport.
  - subject: `tel -.->|"beacon batch<br/>sendBeacon · HTTPS"| be`
  - evidence: 03-architecture.md:32 `tel -.->|"beacon batch<br/>sendBeacon · HTTPS"| be[telemetry ingest]`; DIAGRAMS.md:154 "HTTP: `|"GET /v1/feed<br/>HTTPS · JSON"|` — method + path, then transport and body format"; 02-scale-estimates.md:37–38 "heartbeat + event stream every 5s … Client batches; server design out of scope"
  - fix: tel -.->|"POST /v1/playback-events<br/>sendBeacon · JSON"| be[telemetry ingest]
- **MINOR · prose-trace** · No case prose defines a "telemetry ingest" component (01-requirements explicitly puts the analytics backend out of scope), so the target of the beacon edge is a diagram-only component; the prose only implies a collector must exist as the sink.
  - subject: `be[telemetry ingest]`
  - evidence: 03-architecture.md:32 `be[telemetry ingest]` vs 01-requirements.md:75–77 "Analytics backend (aggregation, dashboards): we emit events; we don't build the collector" and 02-scale-estimates.md:38 "server design out of scope"
  - fix: be[telemetry ingest<br/>client event sink · collector out of scope]

## 24.mobile-chat-app — 7 findings

- **MAJOR · prose-trace** · The sync engine only fetches from gwsvc but has no edge into the store, so fetched messages never reach SQLite and the store→UI delta edge has no input — the diagram cannot answer the sync half of its own Q: caption.
  - subject: `sync -->|"GET gaps since cursor<br/>HTTPS · encrypted"| gws`
  - evidence: Diagram 03-architecture.md line 30: `sync -->|"GET gaps since cursor<br/>HTTPS · encrypted"| gws` with no other edge touching `sync`; vs 03-architecture.md lines 70-73: "sync engine finds sequence gap vs cursor → gap-fetch + decrypt → write rows → store emits delta → UI" and 04-deep-dives.md lines 239-241: "compare per-conversation sequence cursors with the server; fetch gaps".
  - fix: sync -->|"SQL write gap msgs + cursors"| store
- **MAJOR · prose-trace** · The upload manager never persists per-chunk ACKs to the local store, so the resumability mechanism the caption's 'upload durability-first' depends on is absent from the diagram.
  - subject: `up -->|"HEAD hash · POST chunks<br/>HTTPS · byte-range"| blob`
  - evidence: The `up` node has only two edges, both to blob: 03-architecture.md line 27 `up -->|"HEAD hash · POST chunks<br/>HTTPS · byte-range"| blob[/🪣 blob store<br/>vendor/]`; vs 03-architecture.md line 77: "persist per-chunk ACK in ledger → on any failure, next retry resumes from the last ACKed chunk" and 04-deep-dives.md lines 68-69: "Resume ledger persisted in the local store (not memory): upload state survives process death mid-upload".
  - fix: up -->|"SQL resume-ledger ACKs"| store
- **MAJOR · §1** · The blob store is drawn as an external third-party vendor (parallelogram), but §1 classifies key-addressed object stores like S3 as cylinders, and the prose lists the blob store as part of the case's own given server fleet, not a vendor — a reader decodes it as someone else's system.
  - subject: `blob[/🪣 blob store<br/>vendor/]`
  - evidence: Diagram 03-architecture.md line 27: `blob[/🪣 blob store<br/>vendor/]`; DIAGRAMS.md §1 cylinder row: "a keyed store you query by key (durable or cache) — Postgres, Redis, S3, Cassandra" and §5 example `S[(🪣 S3<br/>video objects)]`; 03-architecture.md line 94: "push service, blob store (case 07)" listed under "Server assumptions (given, not designed here)".
  - fix: blob[(🪣 blob store<br/>chunks keyed by content hash)]
- **MINOR · §6** · The edge crosses into the external OS keystore but carries no transport line, so it does not decode alone per §6.
  - subject: `crypto -->|"get · sign keys"| os`
  - evidence: Diagram 03-architecture.md line 24: `crypto -->|"get · sign keys"| os[/🔑 OS keystore<br/>secure enclave/]`; DIAGRAMS.md §6: "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto`) so every edge decodes alone."
  - fix: crypto -->|"get · sign keys<br/>OS keystore API"| os
- **MINOR · §6** · The gateway→push-service edge crosses into the vendor parallelogram but states only the payload, with no METHOD/transport contract.
  - subject: `gws -->|"undelivered msgs"| push`
  - evidence: Diagram 03-architecture.md line 31: `gws -->|"undelivered msgs"| push[/🔔 Push service<br/>APNs · FCM vendor/]`; DIAGRAMS.md §6: "Every edge crossing a deploy boundary carries the contract in this order… HTTP: `|"GET /v1/feed<br/>HTTPS · JSON"|`"; 03-architecture.md line 93-94 treats the push service as a given separate server component.
  - fix: gws -->|"POST deliver-notify<br/>HTTPS · encrypted metadata"| push
- **MINOR · prose-trace** · The controller label claims a 'render queue', which no case prose defines — the prose defines the controller as the send pipeline with queued→sent ACK escalation, and rendering is handled by the UI layer subscribing to store deltas.
  - subject: `ctl[conversation controller<br/>send pipeline · render queue]`
  - evidence: Diagram 03-architecture.md line 11: `ctl[conversation controller<br/>send pipeline · render queue]`; 03-architecture.md lines 40-43: "Conversation controller / send pipeline — appends outgoing messages to a durable send queue… escalates to 'sent' on server ACK"; lines 37-39: "UI subscribes to store deltas, not to socket events" (rendering is the UI layer's job, driven by store deltas).
  - fix: ctl[conversation controller<br/>send pipeline · queued→sent acks]
- **MINOR · §3** · Node IDs exceed the 1–4-letter limit: `store` is 5 letters and `crypto` is 6.
  - subject: `store[(SQLite<br/>messages · send queue · cursors)]`
  - evidence: Diagram 03-architecture.md line 16: `store[(SQLite<br/>messages · send queue · cursors)]` and line 15: `crypto[crypto layer<br/>prekeys · ratchet state]`; DIAGRAMS.md §3: "Node IDs: 1–4 lowercase letters, meaningful, unique within the diagram (`a` = api, `pg` = Postgres, `k` = Kafka)."
  - fix: store --> st[(SQLite<br/>messages · send queue · cursors)]  (and `crypto` --> `cry[crypto layer<br/>prekeys · ratchet state]`)

## 25.mobile-newsfeed — 7 findings

- **MAJOR · prose-trace** · The label claims the feed controller ships bitmaps to the UI, but the prose puts bitmap delivery in the image pipeline, and the diagram never connects the pipeline to the UI at all.
  - subject: `fc ==>|"rows · bitmaps"| ui`
  - evidence: Line 26: `fc ==>|"rows · bitmaps"| ui`. 03-architecture.md line 55-56: image pipeline "→ network fetch → decode at target size off the main thread → hand the bitmap to the view"; 04-deep-dives.md dive 1: pipeline "hand[s] the bitmap to the view". No edge from ip to ui exists in the block (lines 10–35).
  - fix: fc ==>|"rows · stable IDs"| ui
- **MAJOR · prose-trace** · The diagram makes the local DB the actor that asynchronously pushes interactions to the API, but the prose makes the feed controller the one writer of all fetches, and the cross-boundary edge carries no contract.
  - subject: `db -.->|"interaction batch · reconnect"| api`
  - evidence: Line 32: `db -.->|"interaction batch · reconnect"| api`. 03-architecture.md line 48: "All fetches and prefills go through it — one writer" (feed controller); cross-boundary edge without METHOD /path violates DIAGRAMS.md §6; "reconnect" is an outcome word under §5.
  - fix: fc -->|"POST /interactions/batch<br/>HTTPS · JSON"| api
- **MINOR · §4** · Two edges carry the single-primary-path arrow; `==>` is reserved for the one data path the case is about, so the reader cannot tell which path is the primary one.
  - subject: `u ==>|"scroll · open app"| fc`
  - evidence: Line 24: `u ==>|"scroll · open app"| fc` and line 26: `fc ==>|"rows · bitmaps"| ui` — both edges use `==>` in one diagram.
  - fix: u -->|"open app · scroll"| fc
- **MINOR · §6** · The contract line has method + transport but no path, while §6 requires METHOD /path for a cross-boundary HTTP edge; "image variant" is a description, not a path.
  - subject: `ip -->|"GET image variant<br/>HTTPS · JPEG"| cdn`
  - evidence: Line 31: `ip -->|"GET image variant<br/>HTTPS · JPEG"| cdn`; DIAGRAMS.md §6 §7 reference format `-->|"GET /v1/feed<br/>HTTPS · JSON"|`; 01-requirements.md line 11: "CDN-served media keyed by the variants in each item (the client picks the variant)".
  - fix: ip -->|"GET /img/{variant_url}<br/>HTTPS · JPEG"| cdn
- **MINOR · prose-trace** · The cold-start coordinator reads the DB but is never connected to the feed controller, so the restore flow cannot be followed through the diagram.
  - subject: `co (no edge to fc)`
  - evidence: Lines 13–19: the co subgraph block has only `co -->|"read last window · cursor"| db` (line 25) and no edge from co to fc; DIAGRAMS.md §6: "An edge without a label is allowed only between components inside one deploy boundary" (here unillustrated).
  - fix: co --> fc
- **MINOR · §6** · The path rewrites the given server contract's query parameter (`cursor=<opaque>`) to `next_cursor`, which is the response field name, not the request parameter in the given contract.
  - subject: `fc -->|"GET /feed?next_cursor<br/>HTTPS · JSON"| api`
  - evidence: Line 28: `fc -->|"GET /feed?next_cursor<br/>HTTPS · JSON"| api`; 01-requirements.md line 7: "`GET /feed?cursor=<opaque>&n=20`".
  - fix: fc -->|"GET /feed?cursor=<opaque><br/>HTTPS · JSON"| api
- **MINOR · prose-trace** · The store label omits the last scroll position, one of the four things the prose says the local feed DB holds.
  - subject: `db[(local feed DB<br/>rows · cursors · outbox)]`
  - evidence: Line 19: `db[(local feed DB<br/>rows · cursors · outbox)]`; 03-architecture.md line 50: "rows (item metadata, ~400 B each), the fetched pages' cursors, the last scroll position, and the interaction outbox".
  - fix: db[(local feed DB<br/>rows · cursors · position · outbox)]

## 26.google-maps — 5 findings

- **MINOR · §6** · The edge from the edge POPs to the origin tile service crosses a deploy boundary but carries no contract (no METHOD/path, no transport · format); the label states the lookup key and a routing hint instead.
  - subject: `cdn -->|"tile key z,x,y<br/>region shard"| tile`
  - evidence: Diagram line: `cdn -->|"tile key z,x,y<br/>region shard"| tile[tile service<br/>raster + vector origin<br/>keyed by epoch]` — spec DIAGRAMS.md §6: "Every edge crossing a deploy boundary carries the contract in this order: -->|\"METHOD /path<br/>transport · format\"|"; cf. reference example's `T -->|"segments · HTTPS"|` and this diagram's own labeled edge `u ==>|"GET /tile/{z,x,y}<br/>HTTPS · protobuf"| cdn`.
  - fix: cdn -->|"GET /tile/{z,x,y}?epoch=N<br/>HTTPS · protobuf"| tile[tile service<br/>raster + vector origin<br/>keyed by epoch]
- **MINOR · §6** · All three cross-boundary RPC edges name only the payload ("search + S2 bias", "origin · destination", "route legs") and the transport, but no RPC method name, so none decodes alone as an API contract.
  - subject: `gf -->|"search + S2 bias<br/>gRPC · proto"| srch`
  - evidence: Diagram lines: `gf -->|"search + S2 bias<br/>gRPC · proto"| srch[search + geocode<br/>FST prefix + fuzzy per region]`, `gf -->|"origin · destination<br/>gRPC · proto"| dir[...]`, `dir -->|"route legs<br/>gRPC · proto"| eta[ETA service<br/>ML model + speed layer]` — spec §6: "RPC: method name; each cross-boundary HTTP/RPC edge names its transport" (example `-->|"gRPC GetDegree()"|`).
  - fix: gf -->|"gRPC Search(query · S2 bias)<br/>gRPC · proto"| srch[search + geocode<br/>FST prefix + fuzzy per region] (and likewise gRPC GetDirections() and gRPC DecorateLegs() on the dir/eta edges)
- **MINOR · §4** · The edge into the rout store is labeled with a payload noun ("edge weights snapshot"), not the operation the store edge performs, so it can also be misread as dir writing weights into rout.
  - subject: `dir -->|"edge weights snapshot"| rout`
  - evidence: Diagram line: `dir -->|"edge weights snapshot"| rout[(routing graph<br/>packed + CRP hierarchy)]` — spec §4: "A `-->` edge into a store is the operation named in its label" and §5: "An edge only moves data. Its label states what crosses"; prose 03-architecture.md Data flows 3: "CRP query with current weight snapshot".
  - fix: dir -->|"CRP query · pinned weight snapshot"| rout[(routing graph<br/>packed + CRP hierarchy)]
- **MINOR · §5** · The label claims rout reads a "weight snapshot" from the segment-speed store, contradicting the prose, which says spd holds segment speeds and the CRP customization phase recomputes weights from them.
  - subject: `rout -->|"read weight snapshot"| spd`
  - evidence: Diagram lines: `rout -->|"read weight snapshot"| spd[(segment-speed store<br/>RAM · S2 shard · versioned)]` vs prose 04-deep-dives.md (04.3, Customization phase): "given current segment speeds, recompute edge weights and then shortcut weights cell-by-cell" and prose 03-architecture.md: "Segment-speed store — ... the join point between telemetry (write side) and routing/ETA (read side)" — the store holds speeds, not weight snapshots.
  - fix: rout -.->|"consume: segment speeds<br/>re-customization refresh"| spd (or solid: rout -->|"read segment speeds"| spd)
- **MINOR · §5** · The label "publish tiles · new epoch" describes the action/outcome rather than the data crossing the edge, and carries no contract for a cross-deploy-boundary edge.
  - subject: `pipe -->|"publish tiles · new epoch"| tile`
  - evidence: Diagram line: `pipe[map-data pipeline<br/>imagery → graph → tile render<br/>region-sharded · re-customizes rout] -->|"publish tiles · new epoch"| tile` — spec §5: "An outcome or trigger is not a payload: ... `push config` get rewritten as the data that crosses"; §6: every boundary-crossing edge carries the contract.
  - fix: pipe -->|"rendered tiles · epoch id<br/>HTTPS · protobuf"| tile[tile service<br/>raster + vector origin<br/>keyed by epoch]

## 27.ev-charging-network — 8 findings

- **MAJOR · §4** · The charger↔gateway edge is a persistent WebSocket session, but it is drawn as `==>` (and as the second `==>` in the diagram alongside tar==>led), so the session semantics the spec assigns to `<-->` are lost and ==> is duplicated instead of marking one primary sync path.
  - subject: `ch((chargers<br/>15K DC · 85K AC<br/>mid-meter on board)) ==>|...| gw["OCPP gateway fleet..."]`
  - evidence: Diagram line 9: `ch((chargers<br/>15K DC · 85K AC<br/>mid-meter on board)) ==>|"1· TransactionEvent(Started)<br/>OCPP 2.0.1 WS · JSON"| gw["OCPP gateway fleet...""); spec §4 table: `<-->` "long-lived connections (WebSocket, gRPC stream), mutual handshake"; §4 bullet: "`==>` is reserved for the primary *sync* path" (implying the one primary path, per §4 "primary path | the one data path the case is about"); prose 03-architecture.md lines 16-17: "persistent OCPP 2.0.1 WebSocket to the backend".
  - fix: ch <-->|"1· TransactionEvent(Started)<br/>OCPP 2.0.1 WS · JSON"| gw
- **MAJOR · prose-trace** · The OCPI adapter pushes CDRs to partners, but no edge delivers our ledger-side CDRs to the adapter — the outbound CDR flow the caption explicitly asks about is severed, so a reader cannot complete it from the diagram.
  - subject: `oca -.->|"OCPI push: CDRs · occupancy"| em`
  - evidence: Diagram line 24: `oca -.->|"OCPI push: CDRs · occupancy"| em` — the only edges into `oca` are line 22 (`av -.->|"occupancy events"| oca`) and line 23 (`em -->|"POST /ocpi/cdrs..."| oca`); prose 03-architecture.md lines 97-98: "tariff engine emits CDR ... → ledger entry → push CDR to relevant partner queue"; component 7, lines 72-73: "OCPI partner pushes ride the **per-partner queue** (through the OCPI adapter)"; 02-scale-estimates.md line 90: "**~120K CDRs/day** outbound to partners".
  - fix: led -.->|"produce: cdr.issued"| oca
- **MAJOR · prose-trace** · Partner token exchange is drawn as a synchronous pull only, so a reader decodes every roaming authorization as a partner-API round trip; the prose makes the push-to-cache flow primary and the pull a rare miss path.
  - subject: `tok -->|"GET token:{id}<br/>OCPI 2.2.1 · partner API"| em`
  - evidence: Diagram line 13 is the only partner-token edge: `tok -->|"GET token:{id}<br/>OCPI 2.2.1 · partner API"| em` (unqualified pull); prose 03-architecture.md line 51: "Partners push token lists to us over the OCPI adapter (ingress)"; 02-scale-estimates.md lines 87-89: "partner token lists are pushed to us for caching, so the large majority resolve from the local cache; only cache misses send our `Authorize` request *outbound*".
  - fix: em -.->|"PUT /oicg/tokens<br/>OCPI 2.2.1 · JSON"| oca  +  oca -.->|"token list → cache fill"| tok
- **MINOR · §6** · The cache-miss partner lookup is labeled with the Redis store-op syntax `GET token:{id}` instead of an OCPI REST contract (METHOD /path + transport · format), and omits the cache-miss qualifier that 02 makes load-bearing.
  - subject: `tok -->|"GET token:{id}<br/>OCPI 2.2.1 · partner API"| em`
  - evidence: Diagram line 13: `tok -->|"GET token:{id}<br/>OCPI 2.2.1 · partner API"| em`; spec §6: "HTTP: `|"GET /v1/feed<br/>HTTPS · JSON"|` — method + path, then transport and body format".
  - fix: tok -->|"POST /oicg/authorize<br/>OCPI 2.2.1 · JSON (cache miss only)"| em
- **MINOR · §6** · The cross-boundary event edge from availability svc to the OCPI adapter carries no produce:/consume: prefix or contract, only a bare payload noun.
  - subject: `av -.->|"occupancy events"| oca`
  - evidence: Diagram line 22: `av -.->|"occupancy events"| oca`; spec §6: "Events: `produce:` / `consume:` + event name — the event name is the API."
  - fix: av -.->|"produce: occupancy.events"| oca
- **MINOR · §6** · The push to the external partner crosses the deploy boundary as an HTTP REST push but its label names no METHOD /path and no transport line, unlike the OCPI call drawn in the opposite direction.
  - subject: `oca -.->|"OCPI push: CDRs · occupancy"| em`
  - evidence: Diagram line 24: `oca -.->|"OCPI push: CDRs · occupancy"| em`; spec §6 example: `-->|"METHOD /path<br/>transport · format"|`; prose 04-deep-dives.md lines 309-310: "OCPI carries CDRs as individual POSTs with per-CDR ACK".
  - fix: oca -.->|"POST /oicg/cdrs · /locations/status<br/>OCPI 2.2.1 · JSON"| em
- **MINOR · prose-trace** · The command loop is drawn open: device mgmt produces remote commands onto the bus and consumes ACKs, but nothing consumes the commands to deliver them to a charger, leaving a dangling flow whose endpoint the reader cannot decode.
  - subject: `dv -.->|"produce: remote cmds"| bus`
  - evidence: Diagram lines 18-19: `bus -.->|"consume: commands-ack"| dv` and `dv -.->|"produce: remote cmds"| bus` with no consumer edge off the bus for commands; prose 03-architecture.md lines 79-81 (component 8: "OCPP command dispatch with per-device ACK tracking") and line 117: "the gateway is a connection multiplexer".
  - fix: bus -.->|"consume: remote cmds"| gw
- **MINOR · prose-trace** · The node label "eMSP partners" covers an inbound-CDR edge whose counterparty the prose defines as other CPOs (with us acting as the eMSP), so the reader takes partners as the sender of inbound CDRs.
  - subject: `em[/"eMSP partners<br/>roaming · OCPI 2.2.1"/]`
  - evidence: Diagram line 23: `em -->|"POST /ocpi/cdrs<br/>ocpi 2.2.1 · JSON"| oca` against label on line 13 `em[/"eMSP partners<br/>roaming · OCPI 2.2.1"/]`; prose 03-architecture.md line 100: "**Inbound roam** (we are the eMSP): another CPO's CDRs for our drivers arrive at the OCPI adapter's inbound endpoint"; 01-requirements.md line 52: "we are a **CPO, not an eMSP** (in product terms — on the wire we hold both OCPI roles)".
  - fix: em[/"OCPI counterparties<br/>eMSP · CPO roles · 2.2.1"/]

## 28.uber — 8 findings

- **MAJOR · prose-trace** · The call direction is inverted: the matching engine issues the 'drivers near P within R' query against the geo index, but the diagram draws the geo index calling the matching engine.
  - subject: `gi -->|"5· nearby(P, R)"| me`
  - evidence: Diagram line 16: `gi -->|"5· nearby(P, R)"| me`. Prose 03-architecture.md line 52: "**Matching engine** (deep dive 3). Takes a ride request, queries the geo index"; 04-deep-dives.md line 165: "geo-index query at expanding resolution". A reader following the diagram alone concludes the index drives matching.
  - fix: me -->|"5· nearby(P, R)"| gi
- **MAJOR · prose-trace** · The ride request never reaches the matching engine: after 2· POST /rides the only outgoing edge is the quote call, and quoting dead-ends at the surge store (no edge q→me or gw→me), so the diagram's own caption question cannot be traced.
  - subject: `r ==>|"2· POST /rides<br/>HTTPS · JSON"| gw  →  gw -->|"3· QuoteRide()<br/>gRPC · proto"| q`
  - evidence: Diagram lines 10–11 (POST /rides → gw → QuoteRide) are the only edges out of gw toward services on the ride path; q has no other outgoing edge. Prose 03-architecture.md line 81: "**Ride flow (read+write, latency-critical)**: rider request → gateway → matching → geo-index query"; only the position flow (line 77) uses gw→ingest.
  - fix: gw -->|"3· DispatchRide()<br/>gRPC · proto"| me   (add; keep the quote edge unnumbered as the separate quote flow: gw -->|"GetQuote()<br/>gRPC · proto"| q)
- **MAJOR · §4** · Two thick ==> edges are drawn in one diagram, and the offer push is a server-initiated WS push, not a synchronous call the caller waits on, so ==> is doubly misused.
  - subject: `me ==>|"6· offer push<br/>WS · JSON"| d`
  - evidence: Diagram lines 10 and 17 both use ==>: `r ==>|"2· POST /rides…"| gw` and `me ==>|"6· offer push…"| d`. Spec §4: "==> | primary path | the one data path the case is about" and "one-way push (SSE) is `-->`"; §7 uses ==> only for the sync GET /v1/feed.
  - fix: me -->|"6· offer push<br/>WS · JSON"| d
- **MINOR · §6** · A store edge is labeled with an action phrase instead of the operation+argument / SQL <table> contract the spec requires.
  - subject: `tsm -->|"persist transition"| tdb`
  - evidence: Diagram line 20: `tsm -->|"persist transition"| tdb`. Spec §6: "stores: the operation + argument, e.g. `OP arg` or `SQL <table/purpose>`".
  - fix: tsm -->|"SQL trips"| tdb
- **MINOR · §6** · Both edges cross deploy boundaries and carry no contract (no method + transport), unlike every other cross-boundary edge in the diagram.
  - subject: `gi -->|"supply counts"| sda   and   sda -->|"surge tile time-series"| pr`
  - evidence: Diagram lines 23–24: `gi -->|"supply counts"| sda` and `sda -->|"surge tile time-series"| pr`. Spec §6: "Every edge crossing a deploy boundary carries the contract… RPC: method name; each cross-boundary HTTP/RPC edge names its transport"; an unlabeled/uncontracted edge is allowed only within one deploy boundary.
  - fix: gi -->|"gRPC SupplyCounts(tile)"| sda ⏎     sda -->|"gRPC TileRatios()<br/>gRPC · proto"| pr
- **MINOR · §6** · The write edge into the surge tile store uses an action phrase ('write multiplier') instead of an operation+argument label.
  - subject: `pr -->|"write multiplier"| st`
  - evidence: Diagram line 26: `pr -->|"write multiplier"| st`. Spec §6 store-edge rule (`OP arg`); §7 reference uses `-->|"⚡ MGET feed:{uid}"|` for the same store class.
  - fix: pr -->|"SET surge:{tile}"| st
- **MINOR · §6** · A 'produce:' edge terminates on a rectangle service rather than on a topic hexagon, so the reader decodes the trip state machine as a message consumer with no topic; the prose describes a direct handoff.
  - subject: `me -.->|"7· produce: trip.created"| tsm`
  - evidence: Diagram line 19: `me -.->|"7· produce: trip.created"| tsm`. Spec §1: produce:/consume: edges belong to "an append-only topic/log you consume from" (hexagon — el, which already carries tsm's own `produce: trip.events` on line 21); prose 03-architecture.md line 55: matching "Owns reservation of the winning driver and hands the trip to the state machine".
  - fix: me -->|"CreateTrip(trip_id)<br/>gRPC · proto"| tsm
- **MINOR · prose-trace** · The reserve edge is drawn with no trigger: the driver-accept leg of the two-sided loop that the prose makes the precondition of the CAS reserve is absent from the diagram, so reserve looks unconditional after the offer.
  - subject: `me -->|"reserve driver<br/>CAS"| gi`
  - evidence: Diagram line 18 is the only edge into the reserve, preceded only by the one-way offer push (line 17). Prose 03-architecture.md line 82: "offer push → driver accept → reserve driver"; 04-deep-dives.md line 172: "the winner's offer is guarded by a compare-and-swap" — accept precedes reserve.
  - fix: d -->|"8· accept<br/>WS · JSON"| gw   and   gw -->|"AcceptOffer(trip_id)<br/>gRPC · proto"| me

## 29.uber-eats — 7 findings

- **MAJOR · §6/prose-trace** · The only gateway→order-service edge is labeled POST /orders/{id}/transition, so the create flow numbered 1· (POST /orders) never reaches the order service under its real contract; a reader decodes checkout as a transition call.
  - subject: `gw -->|"4· POST /orders/{id}/transition<br/>gRPC · proto"| os`
  - evidence: Diagram line: "gw -->|"4· POST /orders/{id}/transition<br/>gRPC · proto"| os["order svc<br/>state machine · price snapshot"]" — the only gw→os edge. Prose: 03-architecture.md:112 '«POST /orders» (idempotent) → order service validates against price snapshot, authorizes payment, writes CREATED'; 03-architecture.md:54 'Accepts transitions from any party'. Spec: DIAGRAMS.md §6 'every edge crossing a deploy boundary carries the contract'.
  - fix: gw -->|"4· gRPC CreateOrder()<br/>ApplyTransition() · proto"| os["order svc<br/>state machine · price snapshot"]
- **MAJOR · prose-trace** · Dispatch never produces dispatch.* back to the bus and the order service never consumes it, so the assignment result never reaches the state machine — the bus label lists dispatch.* but nothing in the diagram produces or consumes it.
  - subject: `ds["dispatch svc<br/>candidates → score → assign"]`
  - evidence: Diagram has only 'ob -.->|"consume: orders.accepted"| ds', 'ds -->|"gRPC GetEta()<br/>proto"| eta' for dispatch. Prose: 03-architecture.md:66 'commit an assignment, emit «dispatch.*» events'; 03-architecture.md:67-68 'Never writes to order state directly — it emits "course assigned" events that the order service consumes and applies as transitions'; 04-deep-dives.md:77 'The order service is the only writer of courier-assignment transitions, from «dispatch.*» events'.
  - fix: ds -.->|"produce: dispatch.assigned"| ob{{"📨 event bus<br/>orders.* · dispatch.*<br/>location.* · payments.*"}} plus ob -.->|"consume: dispatch.*"| os["order svc<br/>state machine · price snapshot"]
- **MAJOR · prose-trace** · Dispatch's second input — the live courier supply — is missing: ds has no read edge to the supply store, so a follower cannot see where dispatch's candidates come from.
  - subject: `ds["dispatch svc<br/>candidates → score → assign"] → sup[("live supply store")]`
  - evidence: Diagram: ds has no edge to sup; sup only appears in 'lp -->|"upsert courier:{id}"| sup[("live supply store<br/>geo-sharded · in-mem")]'. Prose: 03-architecture.md:62-65 'Two inputs joined in memory: live courier supply (geo-partitioned position store fed by the location pipeline) and unassigned/assignable orders (event bus)'; 04-deep-dives.md:92-94 'prune to a candidate set of ~50 couriers (estimate) by geo-proximity (kd-tree / geohash cells in the supply index)'.
  - fix: ds -->|"geo query couriers near:{cell}"| sup[("live supply store<br/>geo-sharded · in-mem")]
- **MINOR · §4** · The continuous courier-ping stream is numbered 3· inside the order-flow sequence, implying pings happen after merchant accept, which the prose contradicts (pings flow whenever couriers are online).
  - subject: `cu -->|"3· POST /location/pings<br/>HTTPS · JSON"| gw`
  - evidence: Diagram line: 'cu((🧑 courier)) -->|"3· POST /location/pings<br/>HTTPS · JSON"| gw'. Prose: 02-scale-estimates.md:54-58 telemetry is a continuous firehose (~375K pings/s at dinner peak), not an order-flow step. Spec: DIAGRAMS.md §4 'only when a reader could misorder the edges'.
  - fix: cu((🧑 courier)) -->|"POST /location/pings<br/>HTTPS · JSON"| gw
- **MINOR · §6** · The transport · format line reads 'HTTPS · idempotency key' — the second slot names a header requirement instead of the body format, so the edge does not decode its format alone.
  - subject: `c ==>|"1· POST /orders<br/>HTTPS · idempotency key"| gw`
  - evidence: Diagram line: 'c((🧑 customer)) ==>|"1· POST /orders<br/>HTTPS · idempotency key"| gw'. Spec: DIAGRAMS.md §6 '"GET /v1/feed<br/>HTTPS · JSON" — method + path, then transport and body format'.
  - fix: c((🧑 customer)) ==>|"1· POST /orders<br/>HTTPS · JSON · idempotency key"| gw["edge gateway<br/>authn · idempotency · rate limits"]
- **MINOR · prose-trace** · The CDN is drawn only as a purge target while the menu GET terminates at catalog svc, so a reader decodes the 25K QPS read path as gateway→catalog with no edge serving, contradicting the edge-dominated read path.
  - subject: `gw -->|"GET /menus/{id}<br/>HTTPS · JSON"| cat`
  - evidence: Diagram lines: 'gw -->|"GET /menus/{id}<br/>HTTPS · JSON"| cat["catalog svc<br/>versioned menu snapshots"]' and cat serves only 'cat -->|"PURGE /menus/{id}<br/>HTTPS"| cdn'. Prose: 03-architecture.md:46 'Read path is edge-dominated (see 02, ~25K QPS peak, ~95% cacheable)'; 02-scale-estimates.md:36-38 '~25K QPS peak, essentially all cacheable at the edge'.
  - fix: gw -->|"GET /menus/{id}<br/>HTTPS · JSON · CDN-first"| cdn[/"🌍 CDN<br/>menus · photos"/] (with cdn -->|"cache miss"| cat as origin)
- **MINOR · prose-trace** · Checkout's synchronous payment authorization — the hard money gate at order creation — is not represented anywhere in the diagram, though the caption's question ends at settlement.
  - subject: `os["order svc<br/>state machine · price snapshot"]`
  - evidence: Diagram line: 'os["order svc<br/>state machine · price snapshot"]' is terminal on the money path; ledger only gets 'ob -.->|"consume: orders.delivered"| led'. Prose: 03-architecture.md:99 'customer charge authorization happens synchronously at checkout'; 04-deep-dives.md:51-52 'Money-relevant transitions are the only hard gates: delivery authorization, refund on cancel, courier payout on delivery.'
  - fix: os -->|"payment auth<br/>HTTPS · JSON"| psp[/payment PSP/]

## 30.facebook-messenger — 7 findings

- **MAJOR · prose-trace** · The live-push half of the caption question is unconnected: nothing ties the fanout (the component the prose says 'live push[es] or park[s]') to the connection tier, so a reader following the diagram sees K mailbox rows but cannot connect the send to any push.
  - subject: `gt <-->|"wss · live push"| rc`
  - evidence: Diagram line 14 (`gt <-->|"wss · live push"| rc`) is the only push-related edge and it is a bare session; 03-architecture.md lines 49-52: "fanout writes one mailbox entry per recipient, then live-pushes to each recipient's connected devices"; components table line 38: "Fanout service | one canon send → K per-recipient mailbox entries; live push or park"; caption line 5: "...one sent message become K per-recipient mailbox rows plus a live push".
  - fix: fs -.->|"produce: delivery.push"| gt
- **MAJOR · prose-trace** · Draws the preview as a synchronous sender-facing call to an API the prose never defines, encoding the option the case explicitly rejects (inline fetch at send time) and leaving the produced card with no edge back to the chat path.
  - subject: `s -->|"POST /links/preview<br/>HTTPS · JSON"| pv`
  - evidence: 04-deep-dives.md lines 116-122: "preview fetch is async off the hot path (the message text appears immediately; the card populates)" and "Option A — fetch at send time, inline ... We pick B"; 03-architecture.md line 66: "Links carry a reference the *server* materialized via preview fetch".
  - fix: cs -.->|"produce: link.preview"| pv["preview pipeline<br/>URL → title · thumb card"]
- **MINOR · §4** · The number 6· places the media upload after the 5· receiver read, but per the prose the media bytes are uploaded before the send because the chat message carries only a reference to them.
  - subject: `s -->|"6· PUT media bytes"| s3`
  - evidence: Diagram lines 13-17 order edge 5· (receiver read) before edge 6· (media upload); 03-architecture.md line 65: "Media: bytes take the blob-store path; the chat message carries only a reference"; DIAGRAMS.md §4: number labels "only when a reader could misorder the edges" — the side path needs no number.
  - fix: s -->|"PUT media bytes"| s3
- **MINOR · §6** · Mixes an HTTP METHOD+path request line with an RPC transport; §6 wants either "METHOD /path<br/>HTTPS · JSON" or a named RPC method for a gRPC edge.
  - subject: `gt -->|"2· POST /threads/{id}/messages<br/>gRPC · proto"| cs`
  - evidence: DIAGRAMS.md §6 lines 143-156: HTTP form is `|"METHOD /path<br/>transport · format"|` and RPC form is `|"gRPC GetDegree()"|` — one or the other, not a gRPC transport carrying an HTTP path.
  - fix: gt -->|"2· gRPC SendMessage(thread, msg)<br/>proto"| cs
- **MINOR · §6** · A cross-boundary RPC edge names only the payload, not the RPC method the contract requires.
  - subject: `cs -->|"3· canon message copy<br/>gRPC · proto"| fs`
  - evidence: DIAGRAMS.md §6 line 147: `-->|"gRPC GetDegree()"|` — the RPC form is the method name; §6 "RPC: method name; each cross-boundary HTTP/RPC edge names its transport".
  - fix: cs -->|"3· gRPC Fanout(msg)<br/>proto"| fs
- **MINOR · §5** · "live push" is an action, not the data crossing the session edge; the label should name the payload that travels over it.
  - subject: `gt <-->|"wss · live push"| rc`
  - evidence: DIAGRAMS.md §5 lines 127-129: "An outcome or trigger is not a payload: `miss`, `hit 95%`, `push config` get rewritten as the data that crosses"; reference example uses `|"WS feed.delta<br/>wss · JSON"|`.
  - fix: gt <-->|"msg envelope<br/>wss · JSON"| rc
- **MINOR · §6** · An async edge crossing a deploy boundary states the payload but carries no produce:/consume: event contract; the twin edge to si has the same gap.
  - subject: `mb -.->|"mailbox rows · unread deltas"| ib`
  - evidence: Diagram lines 15-17: `mb -.->|"mailbox rows · unread deltas"| ib` and `mb -.->|"mailbox rows<br/>for indexing"| si`; DIAGRAMS.md §6 lines 147-149: async events use `produce:`/`consume:` + event name "— the event name is the API".
  - fix: mb -.->|"consume: mailbox.append"| ib

## 31.netflix — 3 findings

- **MINOR · §5** · The edge label names the outcome of the steering call (the OCA choice steering produces), not the data that crosses the ps→sd edge.
  - subject: `ps -->|"OCA choice"| sd`
  - evidence: Diagram line: ps -->|"OCA choice"| sd — DIAGRAMS.md §5: "An outcome or trigger is not a payload: miss, hit 95%, push config get rewritten as the data that crosses"; the transformation itself is already correctly in sd's label ("OCA choice → steered appliance URL").
  - fix: ps -->|"titleId · device class · probes<br/>gRPC · proto"| sd
- **MINOR · §6** · A cross-deploy-boundary edge carries an action phrase ("session checks") with no payload and no contract (transport · format), so it does not decode alone.
  - subject: `ps -->|"3· session checks"| en`
  - evidence: Diagram line: ps -->|"3· session checks"| en — DIAGRAMS.md §6: "Every edge crossing a deploy boundary carries the contract in this order" (e.g. -->|"gRPC GetDegree()"|, -->|"METHOD /path<br/>transport · format"|); prose 03-architecture.md step 2/F8–F9 names the payload as subscription state, plan limits, geo/age rating.
  - fix: ps -->|"3· gRPC AuthorizeSession(sub · plan · geo)<br/>gRPC · proto"| en
- **MINOR · §6** · This origin-to-appliance fill edge crosses a deploy boundary but names no transport, while the parallel fill edge og→fl on the same path does ("off-peak HTTPS"), so the fill contract is inconsistent across the diagram.
  - subject: `fl -.->|"tonight's title set · renditions"| oc`
  - evidence: Diagram lines: og -->|"fill list · renditions<br/>off-peak HTTPS"| fl  vs  fl -.->|"tonight's title set · renditions"| oc — DIAGRAMS.md §6: every cross-boundary edge carries the contract; prose 03 'Fill' section: "fill servers pull from origins, push to appliances inside ISP networks" (off-peak bulk transfer); max two <br/> lines per label is preserved.
  - fix: fl -.->|"tonight's title set · renditions<br/>off-peak HTTPS"| oc

## 32.spotify — 10 findings

- **MAJOR · prose-trace** · The external CDN node is labeled 'audio segments', implying HLS/DASH segmented streaming, which the case explicitly rejected in favor of one range-addressable file per (track, tier) delivered by progressive download.
  - subject: `cdn[/"🌍 third-party CDN<br/>audio segments"/]`
  - evidence: Diagram line 27: `cdn[/"🌍 third-party CDN<br/>audio segments"/]`; and line 47: `cdn -->|"pull audio segments"| ob`. Contradicts /home/pranamulia/system-design/study_cases/32.spotify/04-deep-dives.md lines 26–33: "We pick A + C ... per (track, tier) there is one range-addressable file", and lines 14–19 where segmented streaming (HLS/DASH) is rejected because the manifest fetch attacks the time-to-first-sample budget.
  - fix: cdn[/"🌍 third-party CDN<br/>audio files · byte range"/]
- **MAJOR · prose-trace** · The client consumes 'rights.changed' from the event bus, but no node produces that event, so the diagram cannot answer the caption's second half (how rights reach the offline client) — the push originates from the catalog/rights side, which is never wired to the bus.
  - subject: `ev -.->|"consume: rights.changed"| u`
  - evidence: Diagram line 43: `ev -.->|"consume: rights.changed"| u`. No producer edge for rights.changed exists (cat's only edge is inbound from etl, line 38). Prose: /home/pranamulia/system-design/study_cases/32.spotify/03-architecture.md lines 83–85 "ingest flips rights flags or takedown → catalog published → ... offline devices get a rights-revalidation push", and /home/pranamulia/system-design/study_cases/32.spotify/04-deep-dives.md lines 227–229 (takedown = tombstone + fast propagation, tombstone list pushed in renewal). The event bus label (line 25) already claims 'rights changes'.
  - fix: cat -.->|"produce: rights.changed"| ev
- **MINOR · §6** · The edge into a keyed store is labeled with an action verb ('pull') plus 'segments' instead of a store operation + argument, and 'segments' repeats the segmentation wording the prose rejected.
  - subject: `cdn -->|"pull audio segments"| ob`
  - evidence: Diagram line 47: `cdn -->|"pull audio segments"| ob` — §6 store-edge requirement ("stores: the operation + argument, e.g. `OP arg`"); §5: an outcome/verb ('pull config' class) must be rewritten as the data that crosses. Prose backing: 04-deep-dives.md line 125 "origin blob store serves renditions".
  - fix: cdn -->|"GET audio/{track}-{rendition}"| ob
- **MINOR · §6** · The ETL→catalog edge crosses a deploy boundary but carries no transport/format line, so it does not decode alone per §6 (it is neither a produce:/consume: event nor METHOD /path + transport · format).
  - subject: `etl -.->|"batch feed: metadata + rights"| cat`
  - evidence: Diagram line 38: `etl -.->|"batch feed: metadata + rights"| cat` — §6 "Every edge crossing a deploy boundary carries the contract in this order ... METHOD /path + transport · format"; this cross-boundary edge names the payload but no transport · format line.
  - fix: etl -.->|"batch feed: metadata + rights<br/>daily file · TSV"| cat
- **MINOR · §6** · The primary-path edge label is 'GET track bundle' — a description, not METHOD /path, so it fails the §6 HTTP contract shape every other numbered edge follows.
  - subject: `u ==>|"1· GET track bundle<br/>HTTPS · JSON"| gw`
  - evidence: Diagram line 31: `u ==>|"1· GET track bundle<br/>HTTPS · JSON"| gw` — §6 HTTP edge format "METHOD /path<br/>transport · format" (reference example line 166: `==>||"1· GET /v1/feed<br/>HTTPS · JSON"|`).
  - fix: u ==>|"1· GET /v1/track-bundle<br/>HTTPS · JSON"| gw
- **MINOR · §6** · The second line of the audio-edge label says 'signed URL' where the contract format belongs — a signed URL is an auth artifact, not a body format, so the edge does not state transport · format as required.
  - subject: `u -->|"5· GET /audio/{id} range<br/>HTTPS · signed URL"| pe`
  - evidence: Diagram line 44: `u -->|"5· GET /audio/{id} range<br/>HTTPS · signed URL"| pe` — §6 format "METHOD /path<br/>transport · format"; 'signed URL' names the auth mechanism, not what crosses.
  - fix: u -->|"5· GET /audio/{id} range<br/>HTTPS · binary"| pe
- **MINOR · §6** · Both playback-edge fetches cross a deploy boundary with transport only — the '· format' half of the §6 two-part contract is missing on each.
  - subject: `pe -->|"GET /audio range<br/>HTTPS"| cdn / pop`
  - evidence: Diagram lines 45–46: `pe -->|"GET /audio range<br/>HTTPS"| cdn` and `pe -->|"GET /audio range<br/>HTTPS"| pop` — §6 "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto")", here only transport with no format.
  - fix: pe -->|"GET /audio range<br/>HTTPS · binary"| cdn  (same for the pop edge)
- **MINOR · prose-trace** · The recommendation service is drawn as the producer of 'play.telemetry' — a flow no case file describes; play events originate with the client/playback, and reco consumes signals for ranking, so a reader following the diagram alone misattributes a core write path.
  - subject: `rec -.->|"produce: play.telemetry"| ev`
  - evidence: Diagram line 39: `rec -.->|"produce: play.telemetry"| ev` — 'play.telemetry' appears nowhere in 01/02/03/04; the closest prose is /home/pranamulia/system-design/study_cases/32.spotify/02-scale-estimates.md lines 87–89: "play history appends at the track-start rate (5.1B/day ...)", i.e. events emitted per play by the client-side, not by the recommendation service.
  - fix: u -.->|"produce: play.telemetry"| ev
- **MINOR · §4** · Continuous numbering across two independent flows implies a false global order: 5· (audio range fetch, which in the track-start flow happens between the gateway call and streaming) is numbered after 4· (a playlist edit from a different flow), so the numbers misinform rather than disambiguate.
  - subject: `"4·" and "5·" edge labels`
  - evidence: Diagram lines 40 and 44: `u -->|"4· POST /playlists/{pid}/ops<br/>HTTPS · JSON"| pl` and `u -->|"5· GET /audio/{id} range<br/>HTTPS · signed URL"| pe`; §4: "number ... only when a reader could misorder the edges"; the flows are separate in 03-architecture.md ("The 300 ms track start" lines 68–77, "Collaborative playlist edit" lines 89–91).
  - fix: u -->|"POST /playlists/{pid}/ops<br/>HTTPS · JSON"| pl   (audio range edge becomes 4·)
- **MINOR · §6** · The POP node is drawn as a dead end: it serves pe but nothing provisions it, while the prose describes POP origin fetches for cold items on miss — a reader cannot see how the last-hop cache ever gets filled.
  - subject: `pop[Spotify edge POP<br/>last-hop audio cache]`
  - evidence: Diagram line 28: `pop[Spotify edge POP<br/>last-hop audio cache]` receives only `pe -->|"GET /audio range<br/>HTTPS"| pop` (line 46); prose: /home/pranamulia/system-design/study_cases/32.spotify/04-deep-dives.md lines 115–117 "Cache-tiering inside a POP: RAM/SSD for the hot head ... origin fetches for cold items", and 03-architecture.md line 102 "Origin blob store sees cache-miss traffic".
  - fix: pop -->|"GET audio/{track}-{rendition} on miss"| ob

## 33.instagram — 7 findings

- **MAJOR · §4** · The arrow direction makes the feature store the caller performing a GET on the ranker, when prose says the ranker reads precomputed features from the feature store.
  - subject: `feat -->|"GET features · candidate key"| rank`
  - evidence: 03-architecture.md:82 `feat -->|"GET features · candidate key"| rank`; prose 03-architecture.md:129-130 "requests scores from the ranker (precomputed features boosted by fresh signals)" and 04-deep-dives.md:199-200 "reuses the *same* ranker, feature store" (store feeds the ranker); DIAGRAMS.md §4: `-->` = synchronous call — the caller waits
  - fix: rank -->|"GET features · candidate key"| feat
- **MAJOR · §6** · The `object.ready` event is produced but never consumed on the diagram, so nothing triggers the transcode pipeline and the ack-first async split (the case's defining decision) decodes as a solid sync chain into transcode.
  - subject: `ing -.->|"produce: object.ready"| ev`
  - evidence: 03-architecture.md:74 `ing -.->|"produce: object.ready"| ev` with no consume edge for any consumer; prose 03-architecture.md:98-100 "stores the original blob, writes the metadata row with state `PENDING`, emits `object-ready`. This is the *synchronous* part of the pipeline; everything downstream is async"; 01-requirements.md:46 "heavy transcoding happens after ack"; 04-deep-dives.md:36-39 "Chosen: B with C for photos. Ack on original stored"; DIAGRAMS.md §6: every cross-boundary edge carries the contract (produce:/consume: + event name)
  - fix: ev -.->|"consume: object.ready"| xc
- **MINOR · prose-trace** · Ingest writes the `PENDING` metadata row per the prose (and its own label), but the diagram draws no ingest→metadata-DB edge, leaving the transcode `SQL UPDATE post row, state → SERVING` UPDATE dangling with no row creation.
  - subject: `xc -->|"SQL UPDATE post row<br/>state → SERVING"| meta`
  - evidence: 03-architecture.md:76 (only write to meta is the transcode UPDATE); prose 03-architecture.md:98 "Media ingest: validates, stores the original blob, writes the metadata row with state `PENDING`" and :125-126 "ingest stores original + `PENDING` row → ack to client"
  - fix: ing -->|"SQL INSERT post row<br/>state = PENDING"| meta
- **MINOR · §4** · No `==>` primary-path arrow is used anywhere in the diagram, although §4 reserves `==>` for the one data path the case is about (here the upload pipeline, the case's stated differentiator from Twitter).
  - subject: `c -->|"1· POST /v1/media originals<br/>HTTPS · JPEG/HEIC"| ing`
  - evidence: 03-architecture.md:72 `c -->|"1· POST /v1/media originals<br/>HTTPS · JPEG/HEIC"| ing`; README.md:9-12 "the write path is a pipeline, not a row insert. Twitter's hardest write-path problem was fanout; ours starts *before* fanout"; DIAGRAMS.md §4: `==>` | primary path | the one data path the case is about
  - fix: c ==>|"1· POST /v1/media originals<br/>HTTPS · JPEG/HEIC"| ing
- **MINOR · prose-trace** · The edge is drawn as explore pushing candidates into feed-api, but the prose says the feed API gathers (pulls) a slice of explore recommendations on read, so feed should be the caller.
  - subject: `exp -->|"POST /v1/feed/candidates<br/>HTTPS · JSON"| feed`
  - evidence: 03-architecture.md:85 `exp -->|"POST /v1/feed/candidates<br/>HTTPS · JSON"| feed`; prose 03-architecture.md:129-130 "feed API gathers the user's pool + on-read merges for followed celebrities + a slice of explore recommendations" (feed initiates)
  - fix: feed -->|"GET /v1/candidates<br/>HTTPS · JSON"| exp
- **MINOR · prose-trace** · The feed-api label and edges show only pool + ranker as candidate sources, omitting the celebrity on-read merge (author recent-posts index) that the prose makes central to the hybrid fanout design and explicitly says is NOT in the pool.
  - subject: `feed["feed-api<br/>merge pools → rank → screenful"]`
  - evidence: 03-architecture.md:67 `feed["feed-api<br/>merge pools → rank → screenful"]`; prose 03-architecture.md:111 "merges pools + celebrity/backfill sources + explore recommendations" and :110 "Celebrity posts are *not* here (04 #3)"; 04-deep-dives.md:148-149 "stale celebrities are merged on read and scored like anyone else"
  - fix: feed["feed-api<br/>merge pools · celebrity on-read → rank → screenful"]
- **MINOR · prose-trace** · Story svc has only the inbound tray-read edge; the story post path (viewer-list fanout on write) and the seen-state/view events from prose flow 3 are undrawn, so `what crosses on story post` is unanswered even though the node is drawn.
  - subject: `stor["story svc<br/>tray · 24h TTL · seen state"] + c -->|"3· GET /v1/story/tray"| stor`
  - evidence: 03-architecture.md:67-68 (stor drawn with only a sync edge from c, no bus edges); prose 03-architecture.md:133-135 "post → viewer list fanned out ... view → seen-state write; expiry → TTL purge"; 04-deep-dives.md:100-101 "seen(author, item, viewer) and the author's view list"; node budget: 15 nodes already at the §8 ~15 limit
  - fix: ev -.->|"consume: story.created"| stor   (and optionally stor -.->|"produce: story.seen"| ev)

## 34.reddit — 7 findings

- **MAJOR · prose-trace** · The caption promises to answer how a *vote* write reaches its system of record, but the diagram contains no vote-api node or ingress edge, so the event log receives vote.cast from nothing — the flow is only half-drawn (04-deep-dives.md's sequence diagram covers it, but this block re-declares its shared nodes and still doesn't show it).
  - subject: `k -.->|"consume: vote.cast"| vcon (block 1, lines 42-59)`
  - evidence: Diagram 03-architecture.md:8 "Q: How does a write (post, comment, vote, mod action) reach its system of record..." and lines 56-57 ("k -.->|"consume: vote.cast"| vcon") have no producing edge; prose 03-architecture.md:157 "**Vote** — GW → vote API → event log (fast ack to the user)" and 04-deep-dives.md:148/155 have the vote-api node and produce edge.
  - fix: Add: gw -->|"gRPC castVote()"| vapi[vote api<br/>ack after publish]  and  vapi -.->|"produce: vote.cast"| k  (or, if the 15-node budget wins, narrow the caption to "(post, comment, mod action)" and leave the vote path to the 04 sequence diagram)
- **MAJOR · §4 (solid = sync, contradicts prose)** · The comment-to-Cassandra edge is solid (synchronous, caller waits), but the prose explicitly says the Cassandra write happens asynchronously after Postgres — wrong arrow semantics.
  - subject: `cmt -->|"CQL upsert comment"| cass (block 1, line 50)`
  - evidence: Diagram 03-architecture.md:50 "cmt -->|"CQL upsert comment"| cass"; prose 03-architecture.md:121-122 "Write-ahead: Postgres first, Cassandra asynchronously reconciled".
  - fix: cmt -.->|"comment rows<br/>async reconcile"| cass
- **MAJOR · prose-trace** · The thread read diagram draws only the Redis tree-cache hit; on a cache miss the prose says the comment service does a Cassandra prefix scan, but Cassandra is entirely absent from this diagram, so the read question is unanswerable for the miss path.
  - subject: `Read-path block (lines 64-106) — cmt -->|"GET tree page"| rd with no miss-path store`
  - evidence: Diagram 03-architecture.md:64-106 has no cass node and only "cmt -->|"GET tree page"| rd" (line 97); prose 03-architecture.md:155-156 "...miss → Cassandra prefix scan → Postgres fallback only for reconciliation"; 04-deep-dives.md:81 "all readers miss and hit Cassandra simultaneously".
  - fix: Add node cass[(🗄️ Cassandra<br/>comment tree per post<br/>materialized path)] and edge: cmt -->|"CQL prefix scan<br/>on cache miss"| cass
- **MINOR · §6 / §5** · The edge-to-gateway label states a description, not a contract — no method/path and no transport·format line, and "routed request" is not a payload/event/query (same edge in block 2, line 95).
  - subject: `edge -->|"routed request"| gw`
  - evidence: Diagram 03-architecture.md:43 "edge -->|"routed request"| gw"; spec study_cases/DIAGRAMS.md §6 "Every edge crossing a deploy boundary carries the contract in this order".
  - fix: edge -->|"HTTP proxy<br/>HTTPS · JSON"| gw
- **MINOR · prose-trace** · The event-log topic label claims an "impressions" event family that no case file mentions — a diagram-only invention (both blocks re-declare this node).
  - subject: `k{{📨 event log<br/>votes · content · impressions · mod events}}`
  - evidence: Diagram 03-architecture.md:29 and 83 "k{{📨 event log<br/>votes · content · impressions · mod events}}"; grep shows "impression" appears only in these two diagram lines, not in 01/02/03/04 prose.
  - fix: k{{📨 event log<br/>votes · content · mod events}}
- **MINOR · prose-trace** · The caption asks "who consumes the events it fans out?" but content.events is produced by post and cmt with no consuming edge anywhere in block 1 (the ranking refiller is only in block 2), and the spam classifier consumes vote.cast while deep dive 5 says content events feed the ML classifiers.
  - subject: `k -.->|"consume: vote.cast"| spam (block 1, line 57) with unmatched producers post/-.-> content.events (line 53)`
  - evidence: Diagram 03-architecture.md:52-57 (producers post/cmt/mod; consumers only "consume: vote.cast"); caption line 8; prose 04-deep-dives.md:410-411 "the same content events feeds ML classifiers for spam/scam/botness" and 03-architecture.md:132 "rank refreshers".
  - fix: k -.->|"consume:<br/>content.events · vote.cast"| spam
- **MINOR · prose-trace** · The feed assembler's subscription source is missing: the prose says the feed assembler reads the user's subscribed communities, but in the diagram only the gateway calls the user service and feed never touches usr.
  - subject: `gw -->|"gRPC prefs()"| usr (block 2, line 99) with no feed→usr edge`
  - evidence: Diagram 03-architecture.md:99-100 (gw → usr prefs; feed → rd only, no feed→usr edge); prose 03-architecture.md:163 "**Render home feed** — feed assembler reads the user's subscribed communities".
  - fix: feed -->|"gRPC getSubs()"| usr

## 35.tiktok — 4 findings

- **MINOR · §4** · Two edges use the thick primary-path arrow, but the spec reserves ==> for the one primary sync path.
  - subject: `u ==>|"2· GET segments<br/>HTTPS · bytes"| cdn`
  - evidence: Diagram lines 30 and 35: `u ==>|"1· GET /feed?page<br/>HTTPS · JSON"| gw` and `u ==>|"2· GET segments<br/>HTTPS · bytes"| cdn`; DIAGRAMS.md §4: "`==>` | primary path | the one data path the case is about" and "`==>` is reserved for the primary *sync* path"; reference example §7 uses exactly one `==>`.
  - fix: u -->|"2· GET segments<br/>HTTPS · bytes"| cdn   (keep 1· GET /feed as the only thick arrow)
- **MINOR · §4** · The telemetry POST is drawn as async dotted, but the collector waits for a durable fast ack, which is a synchronous call by the spec's arrow definitions.
  - subject: `u -.->|"3· POST /v1/events<br/>HTTPS · JSON"| col`
  - evidence: Diagram line 37: `u -.->|"3· POST /v1/events<br/>HTTPS · JSON"| col`; DIAGRAMS.md §4: "`-.->` | asynchronous — the caller does not wait"; 03-architecture.md line 20: `col[edge collectors<br/>durable accept · ack fast]`; 04-deep-dives.md line 161: "edge collectors acknowledge durably and fast".
  - fix: u -->|"3· POST /v1/events<br/>HTTPS · JSON"| col
- **MINOR · §2** · 🐘, ⚡ and 🗄️ imply Postgres, Redis and a wide-row store, technologies the case prose never names for these stores.
  - subject: `vd[("🐘 video meta<br/>title · ladder · perms")]`
  - evidence: Diagram lines 17, 18, 23; DIAGRAMS.md §2 table maps 🐘→Postgres (line 43), ⚡→Redis (line 45), 🗄️→wide-row store Cassandra/Bigtable (line 48); 03-architecture.md lines 98–100 name them only as "video metadata service, sharded approximate counters, item-feature cache, embedding indexes" with no storage engine named anywhere in 01/02/03/04.
  - fix: vd[(video meta<br/>title · ladder · perms)]   (equally drop ⚡/🗄️ from cn and lake, or have the prose name the engine)
- **MINOR · prose-trace** · The edge claims the upload service writes a transcode rung into the blob store, but the prose says upload stores only the original and the lowest rung appears at publish, produced by the transcoding side.
  - subject: `upl -->|"PUT rung 1"| ob`
  - evidence: Diagram line 44: `upl -->|"PUT rung 1"| ob`; 03-architecture.md lines 81-82: "*Upload service*: chunked, resumable, idempotent upload sessions (mobile networks drop); stores the original in the blob store and enqueues transcoding"; 03-architecture.md line 108: "*publish* (1st rendition + meta + index upsert)"; 04-deep-dives.md line 100: "a full per-title ladder takes minutes of compute" (so the upload service cannot produce a transcode rung).
  - fix: xc -->|"PUT rung 1 first<br/>prioritized for publish"| ob   (replacing the upl -->|"PUT rung 1"| ob edge)

## 36.airbnb — 7 findings

- **MINOR · §1** · Two consume edges terminate directly on store cylinders (the calendar DB and the search index), but the prose names rectangle services as the consumers (search index updater, calendar-vs-bookings reconciliation job), so the diagram decodes as the stores themselves consuming events — and '· reconcile' on the calendar edge further blurs which write path is authoritative, the very question the caption poses.
  - subject: `eb -.->|"consume: booking.confirmed · reconcile"| cal`
  - evidence: Diagram line: `eb -.->|"consume: booking.confirmed · reconcile"| cal` (03-architecture.md line 44; same pattern on the es edge, line 43). Spec: DIAGRAMS.md §1 — cylinder = 'a keyed store you query by key', not an event consumer. Prose: 03-architecture.md lines 80–81 — 'Feeds the search index updater and the calendar-vs-bookings reconciliation job.'
  - fix: `eb -.->|"consume: booking.confirmed"| cal` (and on the es edge: `eb -.->|"consume: listing.edited · calendar.snapshot"| es`), reading the target cylinder as 'its updater' per prose — or, if a node slot is freed, name the consumer rectangle instead.
- **MINOR · §5** · The consume label appends '· reconcile', a purpose/outcome word, to the event name; per §5 an edge label states only the event that crosses.
  - subject: `consume: booking.confirmed · reconcile`
  - evidence: Diagram line 44: `eb -.->|"consume: booking.confirmed · reconcile"| cal`. Spec: DIAGRAMS.md §5 — 'An edge only moves data. Its label states what crosses: payload, event name, or query. An outcome or trigger is not a payload.'
  - fix: `eb -.->|"consume: booking.confirmed"| cal`
- **MINOR · §6** · The primary-path edge from guest to gateway is the only guest-side HTTP edge without the 'transport · format' line, so it does not decode alone; its path segment 'hold' is a body qualifier, not a path.
  - subject: `g ==>|"2· POST /bookings hold"| gw`
  - evidence: Diagram line 27: `g ==>|"2· POST /bookings hold"| gw` vs line 26: `g -->|"1· GET /search?q<br/>HTTPS · JSON"| gw`. Spec: DIAGRAMS.md §6 — 'HTTP: |"GET /v1/feed<br/>HTTPS · JSON"| — method + path, then transport and body format' and 'each cross-boundary HTTP/RPC edge names its transport … so every edge decodes alone.'
  - fix: `g ==>|"2· POST /bookings (hold)<br/>HTTPS · JSON"| gw`
- **MINOR · §6** · The host-to-gateway crossing edge is labeled 'PUT calendar block' — operation+argument store-edge style, not the METHOD /path + transport contract §6 demands for cross-boundary HTTP.
  - subject: `h -->|"PUT calendar block"| gw`
  - evidence: Diagram line 37: `h -->|"PUT calendar block"| gw` vs line 30: `g -->|"GET /img/{hash}<br/>HTTPS · JPEG"| cdn`. Spec: DIAGRAMS.md §6 — 'method + path, then transport and body format' on edges crossing a deploy boundary; gw is a rectangle (deployed unit), not a cylinder, so the store-edge exception of §6 ('no transport line, the node shape already says it's a store') does not apply.
  - fix: `h -->|"PUT /v1/calendar/blocks<br/>HTTPS · JSON"| gw`
- **MINOR · §6** · The gateway-to-calendar-svc edge carries a bare 'PUT block' with no transport/format line, unlike its sibling gateway edge 'gRPC Search()<br/>gRPC · proto', so the contract does not decode from the diagram alone.
  - subject: `gw -->|"PUT block"| c`
  - evidence: Diagram line 38: `gw -->|"PUT block"| c` vs line 28: `gw -->|"gRPC Search()<br/>gRPC · proto"| ss`. Spec: DIAGRAMS.md §6 — 'unlabeled edges … allowed only between components inside one deploy boundary'; the gateway and calendar svc are separate deployed units, so 'PUT block' alone under-supplies the contract.
  - fix: `gw -->|"POST /v1/calendar/blocks<br/>gRPC · proto"| c`
- **MINOR · §8** · The diagram declares 16 nodes, one over the §8 budget of under ~15, while peripheral-to-the-caption detail (search-stage internals, actor split) could be pushed to prose.
  - subject: `16 node declarations (g, h, gw, ss, es, ls, img, cdn, cal, bk, led, psp, po, c, rk, eb)`
  - evidence: Node declarations, 03-architecture.md lines 9–24 (count = 16), against 'flowchart LR' with all edges on one diagram. Spec: DIAGRAMS.md §8 — 'Keep diagrams under ~15 nodes; push detail into the prose instead.'
  - fix: `u((🧑 guest·host))` replacing g and h, with edges `u ==>|…| gw`, `u -->|"GET /search?q …"| gw`, `u -->|"PUT /v1/calendar/blocks (host) …"| gw`, `u -->|"GET /img/{hash} …"| cdn`
- **MINOR · prose-trace** · The 'edge / API gateway' node is a diagram-only invention: no case file defines an API gateway, rate limiting, or edge authn as a component; the traceability rule says every node must name a component the prose defines.
  - subject: `gw[edge / API gateway<br/>rate limit · authn]`
  - evidence: Diagram line 11: `gw[edge / API gateway<br/>rate limit · authn]` — grep of 36.airbnb/*.md finds 'gateway/rate limit/authn' nowhere outside the diagram (01-requirements.md, 03-architecture.md components list, 04-deep-dives.md all silent).
  - fix: Either add a one-line component bullet to 03-architecture.md's Components list (e.g. '**API gateway** — edge authn/rate-limit for search, booking and calendar APIs') or drop the node and route the edges straight to the target services.

## 37.ticketmaster — 5 findings

- **MAJOR · prose-trace** · The diagram never shows the order service committing the hold back to the inventory shard (HELD → SOLD), so a reader following it cannot see how a checkout converts into a sold seat.
  - subject: `fe ==>|"5· POST /hold<br/>HTTPS · JSON"| inv[...]`
  - evidence: Diagram line: `fe ==>|"5· POST /hold<br/>HTTPS · JSON"| inv[inventory svc<br/>per-event shard<br/>CAS · TTL holds]` — no edge returns from ord to inv. Prose: /home/pranamulia/system-design/study_cases/37.ticketmaster/03-architecture.md line 104 'Checkout authorizes payment → order service commits the hold → capture → ticket delivery'; 04-deep-dives.md dive 4 'The seat→order atomicity: commit of a hold to SOLD is a single CAS on the inventory shard keyed by hold_id, and the order write + capture proceed only after it.'
  - fix: ord -->|"commit(hold-id)<br/>gRPC · proto"| inv[inventory svc<br/>per-event shard<br/>CAS · TTL holds]
- **MAJOR · prose-trace** · The reaper consumes 'hold.expired' from the log, but no node in the diagram ever produces a hold.expired event into the log, so the release loop is unsourceable.
  - subject: `log -.->|"consume: hold.expired"| reap[reaper<br/>release TTL holds]`
  - evidence: Diagram lines: `inv -.->|"7· produce: seat.held"| log{{📨 event log<br/>seat + order transitions}}` (only producer besides ord) and `log -.->|"consume: hold.expired"| reap[reaper<br/>release TTL holds]`. Prose: 04-deep-dives.md line 105 'The reaper: a scanner over HELD seats on TTL expiry, emitting HELD→AVAILABLE transitions to the event log; inventory rebuild replays them.'
  - fix: inv -.->|"produce: hold.expired"| log{{📨 event log<br/>seat + order transitions}}
- **MINOR · prose-trace** · The reaper's own output (the HELD→AVAILABLE transition it emits) is not streamed to the event log, although the prose makes that event the thing inventory rebuild replays.
  - subject: `reap[reaper<br/>release TTL holds]`
  - evidence: Diagram lines for reap: only `log -.->|"consume: hold.expired"| reap[reaper<br/>release TTL holds]` and `reap -->|"release(hold-id)<br/>gRPC · proto"| inv`. Prose: /home/pranamulia/system-design/study_cases/37.ticketmaster/04-deep-dives.md line 105 'The reaper: a scanner over HELD seats on TTL expiry, emitting HELD→AVAILABLE transitions to the event log; inventory rebuild replays them.'
  - fix: reap -.->|"produce: seat.released"| log{{📨 event log<br/>seat + order transitions}}
- **MINOR · §4** · Three edges use ==> although the spec reserves the thick arrow for the single primary path, so 'primary' no longer single-handedly selects the case's data path.
  - subject: `wr ==>|"3· GET /admit<br/>HTTPS · signed token"| fe[sale frontend<br/>seat map UI · purchase caps] (and fe ==>|"5· POST /hold..."| inv)`
  - evidence: Diagram lines 13 and 16: `wr ==>|"3· GET /admit<br/>HTTPS · signed token"| fe[...]` and `fe ==>|"5· POST /hold<br/>HTTPS · JSON"| inv[...]`, alongside line 9 `u((🧑 fans<br/>~2M at on-sale)) ==>|"1· GET /sale<br/>HTTPS · HTML"| edge[...]`. Spec: /home/pranamulia/system-design/study_cases/DIAGRAMS.md line 108 "`==>` is reserved for the primary *sync* path" and §8 'one diagram, one question'.
  - fix: fe -->|"5· POST /hold<br/>HTTPS · JSON"| inv[inventory svc<br/>per-event shard<br/>CAS · TTL holds]
- **MINOR · §4** · The label is numbered 7·, placing the seat.held event after checkout (6·), but the seat.held transition streams at the hold operation itself (concurrent with 5·), so the numbering misorders the flow.
  - subject: `inv -.->|"7· produce: seat.held"| log{{📨 event log<br/>seat + order transitions}}`
  - evidence: Diagram line 20: `inv -.->|"7· produce: seat.held"| log{{📨 event log<br/>seat + order transitions}}`. Prose: /home/pranamulia/system-design/study_cases/37.ticketmaster/03-architecture.md ('inventory svc ... All mutations stream to the event log' — the hold mutation, flow step 3, precedes checkout, flow step 4); 02-scale-estimates.md item 3 places seat transitions at the hold attempt (~15–20 successful holds/s at the contention point).
  - fix: inv -.->|"produce: seat.held"| log{{📨 event log<br/>seat + order transitions}}

## 38.leetcode — 6 findings

- **MAJOR · prose-trace** · The verdict never reaches the delivery hub (or the API read model) in the diagram, so the caption's core question — how a submission becomes a verdict on the user's stream and who consumes the verdict log — cannot be answered by following the diagram: dlq only receives leaderboard rank deltas, not verdicts.
  - subject: `jd{{📨 verdict log<br/>partitioned event stream}} → dlq[delivery hub] edge missing`
  - evidence: Diagram (03-architecture.md lines 23–26): jd -.->|"consume: verdict"| lb / jd -.->|"consume: verdict"| ac / lb -.->|"submission-id · rank deltas"| dlq — no jd→dlq edge. Prose: 03-architecture.md lines 71–74 "Both result delivery and downstream (leaderboard, anti-cheat) consume from it"; data flows lines 87–89 "verdict event on the log → delivery hub pushes over the client's SSE conn"; 04-deep-dives.md lines 88–90 (vl-)hub and vl-)api consume steps).
  - fix: Add after the jd consume edges: jd -.->|"consume: verdict"| dlq
- **MAJOR · §4** · The edge points from the delivery hub to the user but carries the client's own request (GET), inverting §4's request-direction rule — a reader decodes the hub as issuing GETs to users; the client-initiated subscription actually flows user → hub.
  - subject: `dlq[delivery hub<br/>SSE/WS sessions per submission] -->|"GET /submissions/{sid}/stream<br/>SSE · JSON"| w`
  - evidence: Diagram 03-architecture.md line 14: dlq[delivery hub<br/>SSE/WS sessions per submission] -->|"GET /submissions/{sid}/stream<br/>SSE · JSON"| w. Spec: DIAGRAMS.md §4 "Draw the request direction only" (line 98) and §4 line 104 "one-way push (SSE) is -->" (push content, not the GET, is what would cross hub→user).
  - fix: w -->|"GET /submissions/{sid}/stream<br/>HTTPS · SSE"| dlq
- **MINOR · §6** · This edge crosses a deploy boundary (leaderboard service → delivery hub) but its label states only the payload with no contract (no METHOD/RPC form, no transport · format), so the edge does not decode alone.
  - subject: `lb[leaderboard<br/>exact per-contest counters] -.->|"submission-id · rank deltas"| dlq`
  - evidence: Diagram 03-architecture.md line 26: lb -.->|"submission-id · rank deltas"| dlq. Spec: DIAGRAMS.md §6 "Every edge crossing a deploy boundary carries the contract" with forms -->|"METHOD /path<br/>transport · format"| or -->|"gRPC GetDegree()"| (lines 143–149).
  - fix: lb -.->|"gRPC PushDeltas()<br/>gRPC · proto"| dlq
- **MINOR · §4** · No edge uses ==>, so the sync path the case is about (submit → verdict) is unmarked and a reader decoding per the spec's convention cannot identify the primary path.
  - subject: `w((🧑 users<br/>submit · watch)) -->|"POST /submissions<br/>HTTPS · JSON"| gw (and all other edges)`
  - evidence: All 14 edges in 03-architecture.md lines 13–26 are --> or -.->; spec: DIAGRAMS.md §4 table "==> | primary path | the one data path the case is about" and §7 "thick arrow = the primary path".
  - fix: w((🧑 users<br/>submit · watch)) ==>|"POST /submissions<br/>HTTPS · JSON"| gw
- **MINOR · prose-trace** · The prose names the read-model path (client that disconnects/reconnects reads the verdict from the API row, which the API projects from the verdict log), but the diagram's only verdict-log consumers are leaderboard and anti-cheat, leaving the caption's "who consumes the verdict log?" under-answered; peripheral enough that the ≤15-node rule tolerates prose-only, hence minor.
  - subject: `jd{{📨 verdict log}} → api[submission-api] consume edge missing (read-model projection / poll fallback)`
  - evidence: Diagram 03-architecture.md lines 23–26 (consumers lb, ac only). Prose: 03-architecture.md lines 75–76 "A client that disconnects falls back to polling the submission row (idempotent)" and lines 88–89 "client that reconnected reads the verdict from the API (read model)"; 04-deep-dives.md lines 88–93 (vl-)api step 7, GET /submissions/{id} fallback steps 9–10).
  - fix: jd -.->|"consume: verdict"| api
- **MINOR · prose-trace** · The gateway node and its auth claim are diagram-only inventions: no case file defines a gateway on the submit path, and prose component 1 assigns user validation to the submission-api, not an edge gateway.
  - subject: `gw[gateway<br/>auth · routing]`
  - evidence: Diagram 03-architecture.md line 13 (gw between users and api). Prose: 01-requirements.md and 03-architecture.md Components section (lines 49–82) define no gateway; the only gateway mention is 02-scale-estimates.md lines 99–100, about SSE/delivery gateways, not the submit path; 03 line 52 "Validates the user" = submission API's job.
  - fix: w((🧑 users<br/>submit · watch)) ==>|"POST /submissions<br/>HTTPS · JSON"| api[submission-api<br/>validate · durable row · enqueue]

## 39.e-commerce — 6 findings

- **MAJOR · §4** · The thick arrow `==>` is reserved for the single primary sync path, but the diagram uses it three times (browse entry, checkout entry, and the internal reserve RPC), so a reader following the diagram alone cannot tell which path is primary.
  - subject: `==> edges: `==>"1· GET /pdp…"` (cli→cdn), `==>"3· POST /checkout…"` (gw→co), `==>"reserve items<br/>gRPC"` (co→inv)`
  - evidence: Diagram line: `cli((🧑 clients<br/>web · mobile · API)) ==>|"1· GET /pdp<br/>HTTPS · JSON"| cdn[/🌍 CDN<br/>media · page fragments/]`, `gw ==>|"3· POST /checkout<br/>HTTPS · JSON"| co[...]`, `co ==>|"reserve items<br/>gRPC"| inv[...]`. Spec: DIAGRAMS.md §4 (L92): "`==>` | primary path | the one data path the case is about"
  - fix: `cli -->|"1· GET /pdp<br/>HTTPS · JSON"| cdn[/🌍 CDN<br/>media · page fragments/]` and `co -->|"Reserve()<br/>gRPC · proto"| inv[inventory svc<br/>all-or-nothing ATP + reservations<br/>sole reservation writer]`, keeping `gw ==>|"3· POST /checkout<br/>HTTPS · JSON"| co` as the single thick arrow (the checkout→spine path the caption's question is about)
- **MAJOR · §6 (prose-trace)** · The edge routes all three projection consumers under the contract `consume: ORDER_PLACED`, but the prose says the search indexer and availability projection are driven by `sku.updated` (Flow 2) and `inventory.committed` (Flow 5); only the fulfillment workflow consumes ORDER_PLACED.
  - subject: ``sp -.->|"consume: ORDER_PLACED"| proj` with proj label "search indexer · availability<br/>fulfillment workflow"`
  - evidence: Diagram line: `sp -.->|"consume: ORDER_PLACED"| proj[projection consumers<br/>search indexer · availability<br/>fulfillment workflow]`. Prose: 03-architecture.md L49–51 "emits `sku.updated` to the spine → search indexer updates postings, availability projection updates badges" and L90–91 "availability projection decrements display stock from `inventory.committed` events". Spec: §6 "Events: `produce:` / `consume:` + event name — the event name is the API."
  - fix: `sp -.->|"consume: sku.updated · inventory.committed · ORDER_PLACED"| proj[projection consumers<br/>search indexer · availability<br/>fulfillment workflow]`
- **MINOR · §6** · The edge crossing into an external vendor names neither a method nor a body format — `authorize` is a description, not a contract item, and `HTTPS` omits the format half of `transport · format`.
  - subject: ``co -->|"authorize<br/>HTTPS"| psp``
  - evidence: Diagram line: `co -->|"authorize<br/>HTTPS"| psp[/payment PSP adapter/]`. Spec: DIAGRAMS.md §6 (L154–156): "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto`) so every edge decodes alone" and "RPC: method name".
  - fix: `co -->|"Authorize()<br/>HTTPS · JSON"| psp[/payment PSP<br/>vendor/]` (label fix per the separate parallelogram finding)
- **MINOR · §6** · Same contract drift: `redeem promo` is a description rather than a method name, and the transport line omits the format (`gRPC` without `· proto`).
  - subject: ``co -->|"redeem promo<br/>gRPC"| pmo``
  - evidence: Diagram line: `co -->|"redeem promo<br/>gRPC"| pmo[promotions svc<br/>atomic coupon redemption]`. Spec: DIAGRAMS.md §6 (L148, L156): RPC edges carry method name (`-->|"gRPC GetDegree()"|`) plus `gRPC · proto`.
  - fix: `co -->|"Redeem()<br/>gRPC · proto"| pmo[promotions svc<br/>atomic coupon redemption]`
- **MINOR · §6** · Same contract drift on the inventory edge: `reserve items` is a description, transport line omits `· proto`; this edge is also wrongly on a `==>` thick arrow (see the §4 finding).
  - subject: ``co ==>|"reserve items<br/>gRPC"| inv``
  - evidence: Diagram line: `co ==>|"reserve items<br/>gRPC"| inv[inventory svc<br/>all-or-nothing ATP + reservations<br/>sole reservation writer]`. Spec: DIAGRAMS.md §6 (L148, L156); §4 limits `==>` to the one primary path.
  - fix: `co -->|"Reserve()<br/>gRPC · proto"| inv[inventory svc<br/>all-or-nothing ATP + reservations<br/>sole reservation writer]`
- **MINOR · §1** · The parallelogram is defined as the external third-party vendor viewed whole, but the label names our adapter wrapper around it, blurring which side owns retries/idempotency propagation.
  - subject: ``psp[/payment PSP adapter/]``
  - evidence: Diagram line: `psp[/payment PSP adapter/]`. Spec: DIAGRAMS.md §1 (L20): "parallelogram | external third-party vendor you call out to | payment PSP"; §7 reference draws vendors as `CDN[/🌍 CDN<br/>vendor/]`. Prose: 04-deep-dives.md L232 "The PSP adapter propagates the same key" (adapter = our component).
  - fix: `psp[/payment PSP<br/>Authorize() via checkout adapter/]`

## 40.tinyurl — 5 findings

- **MAJOR · prose-trace** · The diagram puts a synchronous gRPC GetVerdict() call to the abuse scanner on the redirect hot path, but the prose routes status through a cached field on the record, never a live call from the redirect service.
  - subject: `rd -->|"GetVerdict(code)<br/>gRPC · proto"| ab`
  - evidence: Diagram line 23: `rd -->|"GetVerdict(code)<br/>gRPC · proto"| ab`. Prose: /home/pranamulia/system-design/study_cases/40.tinyurl/03-architecture.md lines 54-56 "verdicts ... feed a `status` field on the record; redirects consult the cached status"; /home/pranamulia/system-design/study_cases/40.tinyurl/04-deep-dives.md lines 198-200 "verdict → `status=blocked` in KV → cache invalidation → redirect serves an interstitial ... The status field rides in every cached record, so takedown is a single-field update, not a rewrite." The per-redirect gRPC scanner call also contradicts the p99 < ~10 ms hot-path budget (01-requirements.md line 27).
  - fix: Delete the line `rd -->|"GetVerdict(code)<br/>gRPC · proto"| ab` (the existing `ab -->|"PUT status"| kv` already shows where verdicts land) and let status ride in the cached record by changing line 18 to `ca[(⚡ shared cache<br/>code → URL · status<br/>click-counter cache)]`.
- **MAJOR · §1** · The Bloom filter is drawn as an independently deployed cylinder store that both api and rd call over edges, but the prose places it in every redirect node's in-process memory (and a Bloom filter is a probabilistic membership stage, not a keyed store).
  - subject: `bl[(Bloom filter<br/>all known codes)]`
  - evidence: Diagram lines 19 and 22: `api -->|"insert code"| bl` / `rd -->|"might-contain code"| bl` with `bl[(Bloom filter<br/>all known codes)]`. Prose: /home/pranamulia/system-design/study_cases/40.tinyurl/04-deep-dives.md lines 71-72 "a **Bloom filter of all known codes** in every redirect node's memory"; 03-architecture.md line 45 "On a KV miss it consults a Bloom filter of known codes" (an in-memory lookup of the redirect service). DIAGRAMS.md §1: cylinder = "a keyed store you query by key"; subroutine `[[...]]` = "a stage inside one component ... not a deployed unit".
  - fix: Delete lines 19 and 22 (`api -->|"insert code"| bl` and `rd -->|"might-contain code"| bl`) and fold the filter into the redirect node's label: `rd[redirect svc<br/>in-proc LRU → cache → KV<br/>Bloom(known codes) → garbage-code 404]`.
- **MINOR · §6** · Both redirect HTTP edges carry only `HTTPS` and are missing the second transport · format line the spec requires for HTTP edges crossing a deploy boundary.
  - subject: `u ==>|"1· GET /aB3xK9<br/>HTTPS"| cdn`
  - evidence: Diagram lines 12-13: `u ==>|"1· GET /aB3xK9<br/>HTTPS"| cdn` and `cdn -->|"2· GET /aB3xK9<br/>HTTPS"| rd`. Spec: DIAGRAMS.md §6, HTTP example `|"GET /v1/feed<br/>HTTPS · JSON"|` — method + path, then transport and body format.
  - fix: `u ==>|"1· GET /aB3xK9<br/>HTTPS · 302"| cdn` and, likewise, `cdn -->|"2· GET /aB3xK9<br/>HTTPS · 302"| rd`.
- **MINOR · §5** · "write-through" is cache-mechanism description, not the data crossing the edge; §5 says edge labels state only the payload/event/query, and the write-through mode is already in the Components prose.
  - subject: `api -->|"PUT code → URL<br/>write-through"| kv`
  - evidence: Diagram lines 17-18: `api -->|"PUT code → URL<br/>write-through"| kv` and `api -->|"PUT code → URL<br/>write-through"| ca`. Spec: DIAGRAMS.md §5 — the edge label states the payload; an outcome word ("push config") is rewritten as the data that crosses ("purge list"); §6 store edges are `OP arg`.
  - fix: `api -->|"PUT code → URL"| kv` and `api -->|"PUT code → URL"| ca` (the read-your-own-writes mode stays in the Components prose).
- **MINOR · prose-trace** · ca's label advertises a click-counter cache that no edge uses, while a diagram-only `cnt` store receives a per-event solid INCR — the prose keeps per-link counters in the shared cache with batched periodic flush, never per-click store writes.
  - subject: `ag -->|"INCR clicks:{code}"| cnt`
  - evidence: Diagram lines 18, 26-27: `ca[(⚡ shared cache<br/>code → URL<br/>click-counter cache)]` receives no counter edge; `ag -->|"INCR clicks:{code}"| cnt` into `cnt[(click counters<br/>per-link · daily buckets)]`. Prose: /home/pranamulia/system-design/study_cases/40.tinyurl/04-deep-dives.md lines 163-166 "counters are maintained in the shared cache with periodic flush ... cache-side merging must be per-node, flushed in batch, never per-click"; 03-architecture.md lines 50-53 "stream aggregators maintain per-link counters and time-bucketed rollups".
  - fix: `ag -->|"BATCH PUT rollups:{code}"| cnt` — label the flush target as the rollup store and either drop the dangling "click-counter cache" from ca's label or draw the flush `ag -.->|"INCR clicks:{code}"| ca` into the shared cache instead.

## 41.web-crawler — 7 findings

- **MAJOR · prose-trace** · The diagram has no blob store for fetched pages, so a reader following it alone concludes the raw HTML is only streamed to extraction and never stored, contradicting FR7's replay-from-blobs contract and the 150–200 TB/day stored-blob arithmetic.
  - subject: `fe -.->|"fetched blob"| ex`
  - evidence: Diagram line: `fe -.->|"fetched blob"| ex` (41.web-crawler/03-architecture.md:24) — the only cylinders in the block are robots cache, seen-set and the URL/content table. Prose: 41.web-crawler/01-requirements.md:31-32 (FR7 "per-record replay from stored blobs"); 41.web-crawler/02-scale-estimates.md:138-141 (§8 "150 TB/day stored blobs" — orders of magnitude beyond the URL table); 41.web-crawler/04-deep-dives.md:327-329 ("the blob is the recovery point … the blob store retains raw HTML for the corpus TTL"); README.md:39-40 ("a corpus of compressed page blobs").
  - fix: Add `bs[(🪣 S3<br/>page blobs · corpus TTL)]` plus `fe -->|"PUT raw HTML"| bs` and `bs -->|"blob ref"| ex`
- **MINOR · prose-trace** · The seen-set is only written (SADD by the frontier) and never read in the diagram, so the URL-level lookup that the prose places at discovery — the gate that skips already-seen URLs before enqueueing, i.e. the 10× discovery→fetch drop — is invisible to a reader following the diagram alone.
  - subject: `fr -->|"SADD seen {url-hash}"| seen`
  - evidence: Diagram line: `fr -->|"SADD seen {url-hash}"| seen[(seen-set<br/>hash-level dedup)]` (41.web-crawler/03-architecture.md:20) — the only seen-set edge is a frontier write. Prose: 41.web-crawler/03-architecture.md:105-107 (§4 flow 1: "Discovery: link extracted → canonicalized → seen-set lookup (skip if seen in-flight or very recently fetched) → front queue by priority"); 01-requirements.md:26-27 (FR5 "never enqueue the same canonical URL twice concurrently").
  - fix: Add `in -->|"SISMEMBER {hash}"| seen[(🗄️ seen-set<br/>hash-level dedup)]` (before the intake → frontier edge)
- **MINOR · §6** · Several edges crossing deploy boundaries carry only a payload description and none of the §6 contract forms (METHOD /path + transport · format, RPC method, produce:/consume: + event name).
  - subject: `in -->|"url · priority"| fr`
  - evidence: Diagram lines (41.web-crawler/03-architecture.md:19,21,24,27,28). Spec: study_cases/DIAGRAMS.md:143-149 (§6 "Every edge crossing a deploy boundary carries the contract") and 155-157 ("each cross-boundary HTTP/RPC edge names its transport").
  - fix: `in -.->|"produce: url.discovered"| fr` (event name as the contract; and give the other edges RPC/produce contracts, e.g. `fr ==>|"gRPC DispatchWorkUnit()"| fe`)
- **MINOR · prose-trace** · The diagram shows the fetcher reading the robots cache but no edge fetching robots.txt from the crawled sites, so the robots-refill path (the ~1,000 robots fetches/s and the mandatory serialized miss step) is undrawable from the diagram alone.
  - subject: `fe -->|"GET robots.txt"| rb`
  - evidence: Diagram lines 22 (`fe -->|"GET robots.txt"| rb`) and 23 (`fe -->|"GET /{page-url}<br/>HTTP · HTML"| ext`); no edge targets ext from rb. Prose: 41.web-crawler/03-architecture.md:76-77 ("Cache miss = serialize the fetch of robots.txt before any page fetch on that host"); 02-scale-estimates.md:103-106 ("~75 M / 86400 ≈ ~1,000 robots fetches/s … a mandatory serialized step").
  - fix: Add `rb -->|"GET /robots.txt<br/>HTTP · text"| ext[/crawled sites<br/>robots.txt · pages/]`
- **MINOR · §2** · The seen-set is drawn with a Redis-style SADD operation and no wide-row-store emoji, while the prose defines it as the same bigtable-style wide-row KV as the URL table (which does carry 🗄️) — a reader decodes it as Redis.
  - subject: `fr -->|"SADD seen {url-hash}"| seen[(seen-set<br/>hash-level dedup)]`
  - evidence: Diagram line: `seen[(seen-set<br/>hash-level dedup)]` with edge `fr -->|"SADD seen {url-hash}"|` (41.web-crawler/03-architecture.md:20) vs `ut[(🗄️ URL/content table …)]` (line 25). Prose: 41.web-crawler/02-scale-estimates.md:88-90 (§4: seen-set is "a key-value workload with range scans by host — the 12.bigtable access pattern"); ~8 TB of hashes (02-scale-estimates.md:86). Spec: study_cases/DIAGRAMS.md:49 (🗄️ wide-row store).
  - fix: `seen[(🗄️ seen-set<br/>hash-level dedup)]` with `fr -->|"PUT {hash}"| seen`
- **MINOR · prose-trace** · The fetcher label claims "politeness budgets" as the fetcher's own concern, but the prose places politeness state and budgets in the frontier/scheduler and warns that fetchers self-throttling is the classic politeness bug.
  - subject: `fe[fetcher fleet<br/>politeness budgets<br/>retry-after honored]`
  - evidence: Diagram line: `fe[fetcher fleet<br/>politeness budgets<br/>retry-after honored]` (41.web-crawler/03-architecture.md:21). Prose: 41.web-crawler/04-deep-dives.md:235-238 ("This is where the politeness state has to live: **in the frontier/scheduler, not in the fetchers.** Fetchers that self-throttle without knowing about the other fetchers … are the classic 'we all thought we were being polite' bug."); 03-architecture.md:21 edge `fr ==>|"work unit per host"| fe`.
  - fix: `fe[fetcher fleet<br/>timeouts · size caps · redirect hops<br/>retry-after honored]`
- **MINOR · prose-trace** · The single edge routes both content and link events through the dedup gate, implying link records pass content near-dup filtering, but the prose has link records landing on the stream directly with links separately feeding back into intake.
  - subject: `dd -.->|"produce: content · link events"| st{{📨 stream<br/>partitioned log<br/>at-least-once}}`
  - evidence: Diagram line: `dd -.->|"produce: content · link events"| st{{📨 stream…}}` (41.web-crawler/03-architecture.md:29). Prose: 41.web-crawler/03-architecture.md:95-98 (§3.6: "content records and link records land in topics the indexer and link-graph builder consume") and line 93 (dedup gate "does not block handoff, adds a flag"); §4 flow 3 (line 111-113) routes only content records through the gate.
  - fix: `ex -.->|"produce: link.records"| st` alongside `dd -.->|"produce: content.flagged"| st`

## 42.linkedin — 5 findings

- **MAJOR · §6** · The notification service consumes events named "invites" and "WVYP" that no node on the bus produces, so the event API does not decode — a reader cannot trace what feeds the notification path.
  - subject: `eb -.->|"consume: invites · WVYP"| nt[🔔 notification svc<br/>push + email · throttle]`
  - evidence: Diagram line 26: eb -.->|"consume: invites · WVYP"| nt[🔔 notification svc<br/>push + email · throttle]; the only produce edges (lines 19-21) are profile.views (pr), feed.engagement (fd) and edge.changes (cn). Spec DIAGRAMS.md line 157: "Events: produce: / consume: + event name — the event name is the API." Prose: 04-deep-dives.md dive 2 lines 68-69 "Notifications are fanout from a *state transition event*" and dive 5 lines 172-176 (WVYP notification derives from profile view events).
  - fix: eb -.->|"consume: edge.changes · profile.views"| nt[🔔 notification svc<br/>push + email · throttle]
- **MAJOR · prose-trace** · The DEL of the profile cache is shown as flowing from the stream jobs that consume edge.changes, but the prose says profile-cache invalidation is driven by profile edits, and no profile-edit event is produced anywhere in the diagram.
  - subject: `jobs -->|"DEL profile:{mid}"| pc`
  - evidence: Diagram lines 21-24: cn -.->|"produce: edge.changes"| eb ... eb -.->|"consume: edge.changes"| jobs[stream jobs<br/>invalidators · PYMK · WVYP] ... jobs -->|"DEL profile:{mid}"| pc. Prose: 03-architecture.md line 35 "Profile cache | hot profiles in Redis, cache-aside with edit-driven invalidation"; 04-deep-dives.md line 98 "invalidation stays simple (edit → purge one key)".
  - fix: pr -.->|"produce: profile.edits"| eb{{📨 event bus<br/>edge · profile · feed events}}  (and relabel the jobs edge to eb -.->|"consume: profile.edits · edge.changes"| jobs)
- **MINOR · §2** · The 📨 emoji marks the bus as Kafka per the icon table, but the label never names Kafka, so the emoji carries that meaning alone.
  - subject: `eb{{📨 event bus<br/>edge · profile · feed events}}`
  - evidence: Diagram line 19: eb{{📨 event bus<br/>edge · profile · feed events}} vs components table 03-architecture.md line 41 "Event bus (Kafka)"; spec DIAGRAMS.md lines 36-37: "the sized label text (`Postgres`, `Kafka`) must also be present, because emoji never carries meaning on its own."
  - fix: eb{{📨 Kafka event bus<br/>edge · profile · feed events}}
- **MINOR · §2** · The ⚡ icon means Redis per the icon table, but the label says only "profile cache", so the Redis fact rides on the emoji alone.
  - subject: `pc[(⚡ profile cache<br/>raw doc · cache-aside)]`
  - evidence: Diagram line 10: pr -->|"⚡ MGET profile:{mid}"| pc[(⚡ profile cache<br/>raw doc · cache-aside)]; components table 03-architecture.md line 35 "hot profiles in Redis"; spec DIAGRAMS.md lines 36-37.
  - fix: pc[(⚡ Redis profile cache<br/>raw doc · cache-aside)]
- **MINOR · §6** · A cross-boundary RPC edge mixes an HTTP verb ("GET") with a gRPC transport and no path or method name, drifting from the spec's RPC contract format.
  - subject: `pr -->|"2· GET degree<br/>gRPC"| g`
  - evidence: Diagram line 11: pr -->|"2· GET degree<br/>gRPC"| g[graph svc<br/>adjacency · degree<br/>path-through-whom]; spec DIAGRAMS.md lines 147 and 155-156: RPC edges carry the method name plus transport, e.g. -->|"gRPC GetDegree()"|.
  - fix: pr -->|"2· gRPC GetDegree()"| g[graph svc<br/>adjacency · degree<br/>path-through-whom]

## 43.zoom — 6 findings

- **MAJOR · prose-trace** · The diagram shows the meeting service pushing roster/controls to the client after reconnect, but the prose defines state resync as the client making one idempotent "get full state" call keyed by session token.
  - subject: `sg-)c: roster · controls re-sync`
  - evidence: Diagram line 66: `    sg-)c: roster · controls re-sync`; contradicts /home/pranamulia/system-design/study_cases/43.zoom/04-deep-dives.md lines 128-130: "State resync: roster, mute states, controls, chat catch-up from the meeting service — one idempotent 'get full state' call keyed by session token" and 03-architecture.md line 46 "roster/controls re-sync".
  - fix: c->>sg: get full state · session token  (then)  sg-->>c: roster · controls re-sync
- **MINOR · prose-trace** · The label claims the notification service handles invites and "starting soon" notices, but no case file mentions invites or starting-soon notifications anywhere (03-architecture.md line 31 defines the control plane only as "meetings/roster SQL, notifications, metrics/QoE telemetry").
  - subject: `nt[🔔 notification svc<br/>invites · starting soon]`
  - evidence: Diagram line 17: `    sg -.->|"produce: meeting.events"| nt[🔔 notification svc<br/>invites · starting soon]`; no occurrence of "invite" or "starting soon" in 43.zoom/01-requirements.md, 03-architecture.md prose, or 04-deep-dives.md.
  - fix: nt[🔔 notification svc<br/>meeting.events → push]
- **MINOR · §1** · A produce: edge terminates on a notification-service rectangle with no topic/log node, so a reader decoding alone cannot see what the event travels through — a produce:/consume: edge per §1/§6 belongs on a hexagon topic.
  - subject: `sg -.->|"produce: meeting.events"| nt[🔔 notification svc …]`
  - evidence: Diagram line 17: `    sg -.->|"produce: meeting.events"| nt[🔔 notification svc<br/>invites · starting soon]`; DIAGRAMS.md §1: "`N{{label}}` | hexagon | an append-only topic/log you consume from | Kafka topic" and §6: "Events: `produce:` / `consume:` + event name — the event name is the API".
  - fix: sg -.->|"produce: meeting.events"| k{{📨 Kafka<br/>meeting events}}  (plus)  k -.->|"consume: meeting.events"| nt[🔔 notification svc]
- **MINOR · §6** · This edge crosses a deploy boundary (relay allocator → SFU fleet) but names neither an RPC method nor a transport; the sibling edge (edge 2) correctly carries "gRPC · proto".
  - subject: `ra -->|"3· relay assignment<br/>SFU address · token"| sfu`
  - evidence: Diagram lines 10-11: `    ra -->|"3· relay assignment<br/>SFU address · token"| sfu`; DIAGRAMS.md §6: "RPC: … each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto`) so every edge decodes alone."
  - fix: ra -->|"3· gRPC AssignRelay()<br/>SFU address · token"| sfu
- **MINOR · §6** · The push edge crosses a deploy boundary (notification service → user device) but carries only a payload phrase, no contract or transport line.
  - subject: `nt -->|"push notification<br/>invites · starting soon"| inv((🧑 invited users))`
  - evidence: Diagram line 18: `    nt -->|"push notification<br/>invites · starting soon"| inv((🧑 invited users))`; DIAGRAMS.md §6: "Every edge crossing a deploy boundary carries the contract in this order" (METHOD/path + transport · format, or event name).
  - fix: nt -->|"push: meeting.notify<br/>APNs · FCM"| inv((🧑 invited users))
- **MINOR · §4** · Media resumption is drawn as a response arrow with no preceding request, so it decodes as a reply to nothing; per §4/§8 a sequence diagram encodes request/response pairs.
  - subject: `sfu-->>c: media flows again`
  - evidence: Diagram line 67: `    sfu-->>c: media flows again` follows `    c->>sfu: PLI keyframe request` / `    sfu-->>c: keyframe` which already consumed that request; §8: "`->>` sync call, `-->>` response" (the client's subscribe step, the request media answers, is not drawn).
  - fix: c->>sfu: re-subscribe layers · PLI keyframe  (then)  sfu-->>c: media flows again

## 44.payment-system — 7 findings

- **MAJOR · prose-trace** · The idempotency gate is attributed to the payments service, but the prose assigns key interception, GET and INSERT to the gateway, before intent creation and before the risk check, so edge 4 is also out of prose order.
  - subject: `pay -->|"4· INSERT idem:key"| idem`
  - evidence: Diagram line: `pay -->|"4· INSERT idem:key"| idem`; prose: 04-deep-dives.md:24-27 'Gateway intercepts the key → GET idempotency[key] … Miss? INSERT with a unique index on key' and 03-architecture.md:40-41 'gateway validates + intercepts idempotency key → payments service: intent created (or replay/join) → risk check inline'.
  - fix: `gw -->|"GET idem:key · INSERT on miss"| idem` (move the edge to gw; number it before the gw→pay edge if numbering is kept)
- **MAJOR · §6** · An edge crossing to an external vendor carries only payload words and no contract (no method/RPC name and no transport · format), so it does not decode alone.
  - subject: `pay ==>|"5· auth request · uniqueness ref"| nw`
  - evidence: Diagram line: `pay ==>|"5· auth request · uniqueness ref"| nw`; spec: DIAGRAMS.md §6 'Every edge crossing a deploy boundary carries the contract in this order' (METHOD /path + transport · format, or RPC method + transport).
  - fix: `pay ==>|"5· Auth request (amount, token)<br/>per-rail HTTPS/ISO 8583"| nw`
- **MAJOR · prose-trace** · The diagram omits the gateway adapters, which the prose calls the distributed-system seam (per-rail protocol, credentials, timeouts, bank-side uniqueness reference), while the caption's question explicitly asks what sits at the seam to the banks — pay is drawn calling card networks directly, so the caption's question is not answerable from the diagram.
  - subject: `gateway adapters (component-table row) — absent node between pay and nw`
  - evidence: Diagram: `pay ==>|"5· auth request · uniqueness ref"| nw` with no adapter node; prose: 03-architecture.md component table 'Gateway adapters | per-acquirer/per-rail protocol + credentials + timeouts | the *distributed-system seam*'; deep-dives §3 'The adapter sends a uniqueness reference (our payment ID) that the acquirer de-dupes on retry'; caption 03-architecture.md:5 '…what sits at the seam to the banks?'.
  - fix: Add `ga[gateway adapters<br/>per-rail protocol + credentials<br/>timeouts · uniqueness ref to bank]` and route `pay ==>|"5· capture intent<br/>gRPC · proto"| ga` then `ga ==>|"6· Auth request + uniqueness ref<br/>per-rail HTTPS/ISO 8583"| nw`
- **MAJOR · §5/§6** · The label states an outcome/trigger ('funded sweeps') instead of what crosses the edge, and the cross-boundary store edge lacks a contract (the prose defines the real query as a ledger view).
  - subject: `led -->|"funded sweeps"| po`
  - evidence: Diagram line: `led -->|"funded sweeps"| po`; spec: DIAGRAMS.md §5 'An outcome or trigger is not a payload: miss, hit 95%, push config get rewritten as the data that crosses' and §6 'stores: the operation + argument, e.g. OP arg or SQL <table/purpose>'; prose: 04-deep-dives.md:93 'payout eligibility is a ledger view'.
  - fix: `led -->|"SQL payout-eligibility view"| po`
- **MINOR · §6** · The internal edge mixes HTTP verb+path style with a gRPC transport; RPC edges per the spec use a method-name form, and /v1/intents with a POST verb is invented for this internal hop.
  - subject: `gw ==>|"2· POST /v1/intents<br/>gRPC · proto"| pay`
  - evidence: Diagram line: `gw ==>|"2· POST /v1/intents<br/>gRPC · proto"| pay`; spec: DIAGRAMS.md §6 'RPC: method name' with example `-->|"gRPC GetDegree()"|` (the METHOD /path form is the HTTP rule).
  - fix: `gw ==>|"2· gRPC CreateIntent()<br/>proto"| pay`
- **MINOR · prose-trace** · The label claims 'rate limits', which no case file defines as a designed gateway function.
  - subject: `gw[api-gateway<br/>TLS · rate limits · idempotency-key interception]`
  - evidence: Diagram line 9 `gw[api-gateway<br/>TLS · rate limits · idempotency-key interception]`; 01-requirements.md has no rate limiting; 03-architecture.md component table 'API gateway | auth, rate limit, **intercepts idempotency keys**' — rate limit appears nowhere in requirements or deep dives as a designed function.
  - fix: `gw[api-gateway<br/>TLS termination · idempotency-key interception]`
- **MINOR · prose-trace** · One external node merges card networks with acquirers, but the prose treats acquirers as a distinct party (adapter-to-acquirer auth, per-acquirer daily settlement files), so the settlement-file edge is attributed to the wrong external party.
  - subject: `nw[/card networks<br/>per-rail · per-acquirer/]`
  - evidence: Diagram line: `nw[/card networks<br/>per-rail · per-acquirer/]` feeding `nw -.->|"daily settlement file…"| rec`; prose: 04-deep-dives.md:93 'acquirers sweep daily' and :118 '~daily per acquirer, ~100s of files/day' — settlement files arrive from acquirers, not card networks.
  - fix: `nw[/acquirers + card networks<br/>per-rail · per-acquirer/]` (one external-system node whose label covers both parties the prose names)

## 45.typeahead — 4 findings

- **MAJOR · prose-trace** · The string cache is a diagram-only component: no prose defines a separate string-cache tier, and the case prose places the hydration string pool inside the index nodes' RAM.
  - subject: `sc[(⚡ string cache<br/>term id → string)]`
  - evidence: Diagram line: `t -->|"GET strings:{id}"| sc[(⚡ string cache<br/>term id → string)]` (03-architecture.md L13); 02-scale-estimates.md L34-36 counts the "~6 GB string pool" into the ~10–30 GB per full index replica, and 03-architecture.md L23-32 (components table) and L36-39 ("RAM top-k read → overlay merge … → hydrate term ids → strings") define no string-cache component or extra cache hop.
  - fix: Delete line 13 and fold hydration into the index node: `ix[(🔍 index nodes<br/>top-k per prefix · string-pool hydration<br/>double-buffered segments)]`
- **MINOR · prose-trace** · The store-edge label has no key argument and its second line "prefix-hash shard" is routing metadata, not what crosses the edge.
  - subject: `t -->|"3· top-k point lookup<br/>prefix-hash shard"| ix`
  - evidence: Diagram line: `t -->|"3· top-k point lookup<br/>prefix-hash shard"| ix` (03-architecture.md L12); spec §4/§6: store edges are `OP arg` (DIAGRAMS.md L151: e.g. `MGET feed:{uid}`) and §5: the label states only what crosses.
  - fix: `t -->|"3· GET top-k:{pfx}"| ix` (move the shard scheme into ix's label, e.g. `sharded by prefix hash`, per 02-scale-estimates.md L38)
- **MINOR · §4** · Numbering is incomplete: edges 2· and 3· mark the miss waterfall, but the hot-overlay read and hydrate read that the prose places in the same ordered sequence are unnumbered, so a reader cannot place them relative to 3·.
  - subject: `t -->|"gRPC ReadTrending(pfx)"| hb`
  - evidence: Diagram lines 11-14: `t -->|"2· MGET resp:{pfx}"|` and `t -->|"3· top-k point lookup…"|` are numbered while `t -->|"gRPC ReadTrending(pfx)"| hb` (L14) is not; 03-architecture.md L36-38 orders the request as index read → overlay merge with hot layer → hydrate; spec §4 (DIAGRAMS.md L100-102): number labels when a reader could misorder the edges.
  - fix: `t -->|"4· gRPC ReadTrending(pfx)"| hb` (and `5·` on the hydrate edge if any string edge remains)
- **MINOR · §5** · "double-buffer swap" describes an outcome/mechanism, not the data crossing the edge.
  - subject: `bl -->|"PUT top-k segment files<br/>double-buffer swap"| ix`
  - evidence: Diagram line: `bl -->|"PUT top-k segment files<br/>double-buffer swap"| ix` (03-architecture.md L18); spec §5 (DIAGRAMS.md L128-129): "An outcome or trigger is not a payload … get rewritten as the data that crosses"; the swap is already claimed by ix's label "double-buffered segments".
  - fix: `bl -->|"PUT top-k segment files"| ix`

## 46.google-docs — 4 findings

- **MAJOR · prose-trace** · The caption question is 'how does an edit go to every collaborator's screen', but the only label on the bidirectional channel names the upstream op and the author's ack — the broadcast of new_op @ server_rev to subscribers (the core of the op loop in the prose) is not decodable from the diagram.
  - subject: `ed <-->|"1· WS op.append @ rev N + ack\n wss · JSON"| cs`
  - evidence: Diagram (03-architecture.md:9): `ed((🧑 editors<br/>local mirror · op queue<br/>transform engine · incremental layout)) <-->|"1· WS op.append @ rev N + ack<br/>wss · JSON"| cs[collab service...]` vs caption (03-architecture.md:5) 'How does an edit go from a keystroke to every collaborator's screen' and prose flow (03-architecture.md:38-41) 'broadcast `new_op @ server_rev` to subscribers' (also 04-deep-dives.md §2 step 2-3: 'sequence → store durable → broadcast').
  - fix: ed((🧑 editors<br/>local mirror · op queue<br/>transform engine · incremental layout)) <-->|"1· op.append @ rev N · broadcast new_op @ rev M + ack<br/>wss · JSON"| cs[collab service<br/>per-doc sequencer<br/>transform → broadcast → compact]
- **MAJOR · prose-trace** · Presence contradicts the prose route and is a dead end: the prose has clients pushing cursor updates over a separate channel (the op stream carries none) and the hub fanning out to participants, but the diagram routes presence through the collab service's single WS channel and draws no fanout back to editors/readers.
  - subject: `cs -.->|"produce: presence.cursor"| ph[presence hub<br/>ephemeral cursors · rate-limited]`
  - evidence: Diagram (03-architecture.md:14, 9): `cs -.->|"produce: presence.cursor"| ph[...]`, only channel to editors is the op ` <-->|"1· WS op.append..."| ` vs 04-deep-dives.md §5 (~137-143): 'ephemeral pub/sub, TTL'd (~3–10 s liveness), rate-limited (a client may send cursor updates at ~1–2 Hz; the *op* stream carries no presence)... a 200-viewer doc pushes ~200–400 msg/s outbound', and 01-requirements.md (~55-57) scope decision 3: 'Presence, cursors, comments-notifications are separate channels with separate guarantees' — the caption Q (03-architecture.md:5) explicitly asks 'with presence kept separate'.
  - fix: ed((🧑 editors<br/>local mirror · op queue<br/>transform engine · incremental layout)) -.->|"presence.cursor @ 1–2 Hz"| ph[presence hub<br/>ephemeral cursors · TTL ~3–10 s · rate-limited] and ph -.->|"cursor fanout"| ed — replacing the cs→ph edge
- **MINOR · §4** · Notification delivery is drawn as a synchronous solid arrow into the actor population and the label names transports rather than the payload that crosses the edge.
  - subject: `nt -->|"email · push"| rd`
  - evidence: Diagram (03-architecture.md:18): `nt -->|"email · push"| rd` vs DIAGRAMS.md §4 table: `-->` = 'synchronous call — the caller waits', `-.->` = 'asynchronous — the caller does not wait' (events), and §5: an edge label 'states *what crosses*: payload, event name, or query'.
  - fix: nt -.->|"notify: doc.updated<br/>email · push"| rd
- **MINOR · §4** · Number labels 1· and 2· are used across two independent flows (edit loop vs doc open) where the reader cannot misorder the edges, though the spec reserves numbering for possible misordering.
  - subject: `1· WS op.append... / 2· GET /open`
  - evidence: Diagram (03-architecture.md:9-10): labels `"1· WS op.append @ rev N + ack..."` and `"2· GET /open\nHTTPS · JSON"` vs DIAGRAMS.md §4: 'numbering... only when a reader could misorder the edges' — the WS op flow and the GET /open flow are unrelated sequences.
  - fix: ed((...)) <-->|"1·"→"" — drop the digits: `<-->|"op.append @ rev N · broadcast new_op @ rev M<br/>wss · JSON"|` and `-->|"GET /open<br/>HTTPS · JSON"|` (or keep numbering only if both labels belong to one documented flow)

## 47.rate-limiter — 3 findings

- **MINOR · §6** · The HTTP edge crossing the client→gateway deploy boundary states transport · format but not METHOD /path, so the contract line is incomplete.
  - subject: `cl((🧑 clients<br/>key · IP · route)) ==>|"1· API request<br/>HTTPS · JSON"| gw`
  - evidence: Diagram line: `cl((🧑 clients<br/>key · IP · route)) ==>|"1· API request<br/>HTTPS · JSON"| gw[...]` — spec §6: 'HTTP: `|"GET /v1/feed<br/>HTTPS · JSON"|` — method + path, then transport and body format.' (reference example in §7 uses `1· GET /v1/feed`).
  - fix: cl((🧑 clients<br/>key · IP · route)) ==>|"1· POST /v1/charges<br/>HTTPS · JSON"| gw[gateway node ×100<br/>TLS · auth · rules snapshot<br/>local lease buckets · 429 + Retry-After]
- **MINOR · §5** · The edge label carries the fetch-failure behavior ('pin LKG') which does not cross the edge; only the data moved (the snapshot blob for revision N) belongs there.
  - subject: `gw -->|"GET rules revision N<br/>snapshot blob · pin LKG"| cf`
  - evidence: Diagram line: `gw -->|"GET rules revision N<br/>snapshot blob · pin LKG"| cf[(rule config store<br/>tier defaults · per-key overrides<br/>revisioned snapshots)]` — spec §5: 'An edge only moves data. Its label states what crosses: payload, event name, or query. … An outcome or trigger is not a payload … get rewritten as the data that crosses' (the pin-LKG-on-fetch-failure behavior is prose: 03-architecture.md 'pin last-known-good on fetch failure (deep dive 6)').
  - fix: gw -->|"GET rules revision N<br/>snapshot blob"| cf[(rule config store<br/>tier defaults · per-key overrides<br/>revisioned snapshots)]
- **MINOR · §6** · The producer emits event 'decision' but both consumers are labeled 'consume: decisions' (plural), so the event name — which §6 says is the API — does not match across the topic.
  - subject: `gw -.->|"produce: decision<br/>admit · 429"| k ... k -.->|"consume: decisions"| an / ae`
  - evidence: Diagram lines: `gw -.->|"produce: decision<br/>admit · 429"| k{{📨 decision log<br/>sampled under backpressure}}` vs `k -.->|"consume: decisions"| an[usage dashboards<br/>billing rollups]` and `k -.->|"consume: decisions"| ae[...]` — spec §6: 'Events: `produce:` / `consume:` + event name — the event name is the API.'
  - fix: k -.->|"consume: decision"| an[usage dashboards<br/>billing rollups]   (and likewise `k -.->|"consume: decision"| ae[...]`)

## 48.google-search — 6 findings

- **MAJOR · §6/§5** · The fanout edge crosses the FE-to-shard-machine deploy boundary but carries no contract (no RPC method or transport), and its label names orchestration mode ('broadcast top-K', 'parallel fanout') instead of what crosses (the parsed query).
  - subject: `fe -->|"3· broadcast top-K<br/>parallel fanout"| bi[(🔍 base index ...)`
  - evidence: Diagram line: `fe -->|"3· broadcast top-K<br/>parallel fanout"| bi[(🔍 base index ...)]`; DIAGRAMS.md §6: 'HTTP: method + path, then transport and body format... each cross-boundary HTTP/RPC edge names its transport' and §5: 'An edge only moves data. Its label states what crosses: payload, event name, or query.'
  - fix: fe -->|"3· gRPC Search(q) · proto"| bi[(🔍 base index<br/>~2,000 shards · postings<br/>impact-ordered)]
- **MAJOR · §6/§5** · Same contract gap as the base-index fanout: a cross-boundary edge labeled only with an orchestration note ('same merge pool'), not a method+transport contract and not the query that crosses.
  - subject: `fe -->|"3· broadcast top-K<br/>same merge pool"| ft[(🔍 fresh tier ...)`
  - evidence: Diagram line: `fe -->|"3· broadcast top-K<br/>same merge pool"| ft[(🔍 fresh tier ...)]`; DIAGRAMS.md §6 contract rule and §5 'what crosses' rule; 03-architecture.md §4.3 line 117: 'fresh-tier replicas are queried in the same broadcast'
  - fix: fe -->|"3· gRPC Search(q) · proto"| ft[(🔍 fresh tier<br/>minutes-old segments)]
- **MAJOR · prose-trace/§5** · The heavy-ranker label claims the ~5K→top-200 funnel step, but the prose puts that reduction in the coarse ranker ('batch a re-scored top ~200'), so the diagram alone decodes the neural cross-encoder as running over ~5K candidates — the exact 25x cost error the case's NFR5 argument warns about.
  - subject: `hv[heavy ranker<br/>~5K → top 200<br/>neural cross-encoder re-score]`
  - evidence: Diagram lines: `mg[merge + coarse ranker<br/>~2,000 top-K → ~5K]`, `hv[heavy ranker<br/>~5K → top 200<br/>neural cross-encoder re-score]`, `mg -->|"~5K candidates<br/>gRPC · proto"| hv`; 03-architecture.md §3.5 lines 82-87: 'combine ≤2,000 per-shard top-Ks into ~5 k candidates, score with a cheap model... batch a re-scored top ~200 and run the expensive neural model'; 04-deep-dives.md §5 lines 279-283 prices heavy-at-5000 as 25x cost.
  - fix: mg[merge + coarse ranker<br/>~2,000 top-K → ~5K → top 200] -->|"gRPC Rank(top 200)<br/>proto"| hv[heavy ranker<br/>top 200 → top 10<br/>neural cross-encoder re-score]
- **MINOR · prose-trace** · The query path omits the vertical fanout that the prose lists in the same broadcast as base shards and the fresh tier (FR6 makes parallel vertical fan-out + degradation in scope); the composer nod only says 'verticals' with no lane to merge.
  - subject: `vertical fanout (FR6) absent from the fanout edges`
  - evidence: Diagram: no verticals edge — `sc[SERP composer<br/>snippets highlighted · verticals]` only; 03-architecture.md §4.2 line 113: 'broadcast to 2,000 base shards + fresh tier + verticals'; 01-requirements.md FR6 line 46-48: 'the architecture must fan out to verticals in parallel with the web-index retrieval and merge, and degrade the SERP gracefully'.
  - fix: fe -->|"3· gRPC Vertical(q) · proto"| vt[vertical rankers<br/>news · images · video<br/>omit on timeout]  (and sc merges its results alongside the web lane)
- **MINOR · §6** · Shard candidates cross from the shard fleet into the merge service with no transport named, while the very next edge on the same data path (mg→hv) names one — the retrieval edges don't decode alone as their own boundary rules require.
  - subject: `bi -->|"per-shard top-K"| mg[...]  and  ft -->|"candidates"| mg[...]`
  - evidence: `bi -->|"per-shard top-K"| mg`, `ft -->|"candidates"| mg` vs DIAGRAMS.md §6 lines 151-156 ('each cross-boundary HTTP/RPC edge names its transport'; compare `mg -->|"~5K candidates<br/>gRPC · proto"| hv` on the next hop)
  - fix: bi -->|"per-shard top-K<br/>gRPC · proto"| mg[merge + coarse ranker<br/>~2,000 top-K → ~5K → top 200]   (same transport line on the ft edge)
- **MINOR · §6** · The promotion edge into the base-index store states only the payload ('merged segments') without the operation, while store edges are specified as operation + argument.
  - subject: `mr[bg merger ...] -.->|"merged segments"| bi`
  - evidence: Diagram line: `mr[bg merger<br/>fresh segments → merged base segment] -.->|"merged segments"| bi`; DIAGRAMS.md §6: 'stores: the operation + argument, e.g. OP arg or SQL <table/purpose>' (compare `ix -->|"write segments"| ft`, which names the operation)
  - fix: mr[bg merger<br/>fresh segments → merged base segment] -.->|"write merged segments"| bi[(🔍 base index<br/>~2,000 shards · postings<br/>impact-ordered)]

## 49.monitoring-metrics — 6 findings

- **MINOR · §6** · The scrape contract label is three lines (§6 allows one or two) and prefixes the word "scrape" before the HTTP METHOD, so the label does not follow the METHOD /path + transport · format contract.
  - subject: `pr ==>|"1· scrape GET /metrics<br/>HTTP · gzip<br/>fixed 15 s"| ex`
  - evidence: Diagram line: `pr ==>|"1· scrape GET /metrics<br/>HTTP · gzip<br/>fixed 15 s"| ex[...]` (03-architecture.md line 9). Spec: DIAGRAMS.md §6 — "carries the contract in this order, one or two lines" with example `-->|"METHOD /path<br/>transport · format"|`.
  - fix: `pr ==>|"1· GET /metrics<br/>HTTP · gzip · 15 s cadence"| ex[node + app exporters<br/>3,000 hosts · /metrics<br/>dumb · no state]`
- **MINOR · §6** · The cross-boundary Grafana→querier HTTP edge gives no METHOD or path, so the edge does not decode to a contract on its own.
  - subject: `gr -->|"2· PromQL query<br/>HTTP · instant · range"| qy`
  - evidence: Diagram line: `gr[Grafana<br/>dashboards · 200 boards] -->|"2· PromQL query<br/>HTTP · instant · range"| qy[...]` (03-architecture.md line 14). Spec: DIAGRAMS.md §6 — "HTTP: `|"GET /v1/feed<br/>HTTPS · JSON"|` — method + path, then transport and body format."
  - fix: `gr[Grafana<br/>dashboards · 200 boards] -->|"2· POST /api/v1/query<br/>HTTP · JSON"| qy[querier · PromQL engine<br/>postings → fetch chunks → eval]`
- **MINOR · §6** · The RPC edge names the transport and API service (gRPC StoreAPI) but no method name, which §6 requires for RPC edges.
  - subject: `qy -->|"gRPC StoreAPI<br/>read head + local blocks"| pr`
  - evidence: Diagram line: `qy -->|"gRPC StoreAPI<br/>read head + local blocks"| pr` (03-architecture.md line 15). Spec: DIAGRAMS.md §6 — "RPC: method name; each cross-boundary HTTP/RPC edge names its transport"; example `-->|"gRPC GetDegree()"|`.
  - fix: `qy -->|"gRPC StoreAPI Series()<br/>read head + local blocks"| pr`
- **MINOR · prose-trace** · The diagram attributes the block upload to Prometheus itself and has the querier read old blocks straight from object storage, while the prose says a remote-write sidecar uploads blocks and a store gateway serves old-block reads.
  - subject: `pr -->|"PUT 2 h blocks"| lt`
  - evidence: Diagram line: `pr -->|"PUT 2 h blocks"| lt[(🪣 long-term store<br/>object storage · compactor<br/>13 months · downsampled)]` (03-architecture.md line 13). Prose: 03-architecture.md line 32 "Remote-write sidecar / store gateway | uploads compacted blocks to object storage; serves reads of blocks older than local" and lines 62–65 "sidecar uploads each finalized block (2 h+) to object storage... The querier answers queries older than the local window by reading block metadata + chunks from the store gateway."
  - fix: `sw[store sidecar + gateway<br/>upload blocks · serve old-block reads] -->|"PUT 2 h blocks"| lt[(🪣 long-term store<br/>object storage · compactor<br/>13 months · downsampled)]` (and re-point `qy -->|"GET old block chunks"| sw`)
- **MINOR · prose-trace** · The drawgateway node is drawn but the batch-job push that puts data into it is omitted, so a reader following the diagram alone cannot see how the pushgateway receives anything (its only edge is the later scrape).
  - subject: `pg[pushgateway<br/>short-lived batch jobs]`
  - evidence: Diagram line: `pr -->|"GET /metrics<br/>HTTP · text"| pg[pushgateway<br/>short-lived batch jobs]` (03-architecture.md line 10) — no edge into pg exists. Prose: 01-requirements.md lines 21–23 "short-lived batch jobs that cannot be scraped on a cadence push to an intermediary that *is* scrapeable"; 03-architecture.md line 24 "Pushgateway | holds metrics of short-lived batch jobs so they can be scraped".
  - fix: `bj((🧑 batch jobs)) -->|"POST /metrics/job/:job<br/>HTTP · text"| pg[pushgateway<br/>short-lived batch jobs]`
- **MINOR · §3** · Two components inside one node label are joined with `+` instead of the `·` separator the spec mandates (same pattern in ex's "node + app exporters").
  - subject: `qy[querier + PromQL engine<br/>postings → fetch chunks → eval]`
  - evidence: Diagram lines: `qy[querier + PromQL engine<br/>postings → fetch chunks → eval]` and `ex[node + app exporters<br/>3,000 hosts · /metrics<br/>dumb · no state]` (03-architecture.md lines 14, 9). Spec: DIAGRAMS.md §3 — "components inside a label separated by `·`".
  - fix: `ex[node · app exporters<br/>3,000 hosts · /metrics<br/>dumb · no state]`

## 50.stock-exchange — 9 findings

- **MAJOR · §6** · The contract line mixes an HTTP convention (METHOD /path) with a gRPC transport line, so the edge decodes as neither valid HTTP nor valid gRPC.
  - subject: `POST /trades<br/>gRPC · proto<br/>exact-once IDs`
  - evidence: Diagram line 20: `od -->|"POST /trades<br/>gRPC · proto<br/>exact-once IDs"| cc[/clearing link<br/>CCP handoff/]`; DIAGRAMS.md §6: "HTTP: `|"GET /v1/feed<br/>HTTPS · JSON"|` — method + path, then transport and body format. RPC: method name".
  - fix: od -->|"gRPC SubmitTrade()<br/>proto · trade record"| cc[/clearing link<br/>CCP handoff/]
- **MINOR · §5** · The edge label carries a design rule ('journal before dispatch') and a description of the target ('append-only log') instead of only stating what crosses the edge.
  - subject: `sq -->|"3· journal before dispatch: inbound msg seq N<br/>append-only log"| jr`
  - evidence: Diagram line 11: `sq -->|"3· journal before dispatch: inbound msg seq N<br/>append-only log"| jr{{📨 event journal...}}`; DIAGRAMS.md §5: "An edge only moves data. Its label states what crosses: payload, event name, or query."; 03-architecture.md line 38-40 confirms the payload is the journaled order.
  - fix: sq -->|"3· inbound msg seq N"| jr{{📨 event journal<br/>inbound + outbound events<br/>one sequence space}}
- **MINOR · §6** · The produce: label lists event classes but never names the event, unlike the correct pattern used two lines later (produce: md.tick).
  - subject: `me -.->|"produce: acks · fills · depth"| jr`
  - evidence: Diagram line 13: `me -.->|"produce: acks · fills · depth"| jr`; DIAGRAMS.md §4: "`==>` is reserved... produce:/consume: plus..." and §6: "`produce: event.name` (or consume:)"; reference §7 uses `produce: post.fanned`.
  - fix: me -.->|"produce: outbound.events<br/>acks · fills · depth"| jr
- **MINOR · §6** · The consume: label uses a generic category ('outbound events') rather than an event name, so the edge lacks a decodable contract.
  - subject: `jr -.->|"consume: outbound events"| od`
  - evidence: Diagram line 15: `jr -.->|"consume: outbound events"| od[outbound dispatch<br/>exec reports · drop-copy · MD writer]`; DIAGRAMS.md §6: "Events: `produce:` / `consume:` + event name — the event name is the API."
  - fix: jr -.->|"consume: outbound.events"| od[outbound dispatch<br/>exec reports · drop-copy · MD writer]
- **MINOR · §6** · The transport line gives transport · subscriber behavior ('seq-gap detect') instead of transport · format, and gap detection is a subscriber-side concern, not what crosses the edge.
  - subject: `md -.->|"consume: md.tick<br/>UDP · seq-gap detect"| su`
  - evidence: Diagram line 17: `md -.->|"consume: md.tick<br/>UDP · seq-gap detect"| su`; DIAGRAMS.md §6: "each cross-boundary HTTP/RPC edge names its transport (`HTTPS · JSON`, `gRPC · proto`)"; 02-scale-estimates.md line 61: "MD message ≈ ~64 B (fixed-width binary, ITCH-style, est.)".
  - fix: md -.->|"consume: md.tick<br/>UDP · 64 B binary"| su
- **MINOR · §6** · The cross-boundary contract has no format line and crams transport into the request phrase, so it does not decode alone.
  - subject: `su -->|"TCP GET snapshot + seq range"| rp`
  - evidence: Diagram line 18: `su -->|"TCP GET snapshot + seq range"| rp[replay servers<br/>book snapshots · seq replay]`; DIAGRAMS.md §6: "HTTP: `METHOD /path<br/>transport · format`... each cross-boundary HTTP/RPC edge names its transport"; 04-deep-dives.md §5: "snapshot (current book image per symbol) + TCP replay servers".
  - fix: su -->|"GET snapshot + seq range<br/>TCP · binary"| rp[replay servers<br/>book snapshots · seq replay]
- **MINOR · prose-trace** · The label claim 'one per symbol' reads as one shard per symbol, contradicting the prose design where symbols are grouped into ~30-40 shard groups (~200-270 symbols per core) with single-writer per symbol.
  - subject: `me[matching engine shards<br/>one per symbol · price-level books<br/>deterministic state machines]`
  - evidence: Diagram line 12: `me[matching engine shards<br/>one per symbol · price-level books<br/>deterministic state machines]`; 04-deep-dives.md §2 (Option B, chosen): "symbols grouped into shard groups, each group a single-threaded deterministic event loop pinned to a core", "~8,000 symbols ÷ ~30-40 groups ≈ ~200-270 symbols per core"; 03-architecture.md component table says "single-writer per symbol".
  - fix: me[matching engine shards<br/>shard groups · single-writer per symbol<br/>deterministic state machines]
- **MINOR · prose-trace** · 'Retail aggregators' appears nowhere in the case prose; the defined subscriber population is firms (colocated HFT) and vendor networks, roughly 500 direct-feed firms.
  - subject: `su((🧑 MD subscribers<br/>firms · vendors<br/>retail aggregators))`
  - evidence: Diagram line 17: `su((🧑 MD subscribers<br/>firms · vendors<br/>retail aggregators))`; 04-deep-dives.md §5: "Subscribers: hundreds of firms taking the direct feed (02-scale-estimates: ~500), from colocated HFT to vendor networks"; 02-scale-estimates.md line 66: "~500 direct-feed subscriber firms (est.)".
  - fix: su((🧑 MD subscribers<br/>firms · vendor networks<br/>~500 direct feed))
- **MINOR · §3** · The gateway label lists its sequential pipeline stages with · separators instead of the → chain the spec requires for in-component transformations (auth happens before risk checks).
  - subject: `gw[order gateways<br/>auth · risk checks<br/>stable clOrdID]`
  - evidence: Diagram line 9: `gw[order gateways<br/>auth · risk checks<br/>stable clOrdID]`; DIAGRAMS.md §3: "Transformations that happen inside a component are a `→` chain in the label body"; §7 reference: `a[feed-api<br/>auth → rank → merge]`.
  - fix: gw[order gateways<br/>auth → risk checks<br/>stable clOrdID]

## Verification summary

Cases with zero majors: 03.whatsapp, 10.distributed-cache, 13.kafka, 15.ml-ranking-system, 16.dalle, 19.frontend-chat-app, 26.google-maps, 31.netflix, 35.tiktok, 36.airbnb, 47.rate-limiter, 49.monitoring-metrics.
