# Diagram spec review 2026-11-26

**Status: fixes applied 2026-11-26 — 242/247 findings fixed, 9 skipped (all documented as prose-contradicting or self-negating); all 53 resulting mermaid blocks re-render PASS.** This file is now the audit record; the 9 skips are listed in the fixer run notes (journal: wf_f3919b7a-fa5). Fixers applied judgment calls per case — see notes there for the 20+ cases with long judgment-call notes.

## 32.spotify — violations (6 findings)
- **§5** · diagram 1, edge cdn --> ob: Label starts with the outcome word 'miss:' — §5 explicitly says 'miss' is an outcome, not a payload, and must be rewritten as the data that crosses. The edge label should state only what moves: audio segments.
  fix:     cdn -->|"pull audio segments<br/>HTTPS"| ob
- **§4** · diagram 1, edge etl --> cat ("batch: metadata + rights"): Solid --> marks a synchronous call the caller waits for, but a label-feed ingest job is asynchronous batch movement (same family as replication/log shipping). Draw it dotted.
  fix:     etl -.->|"batch feed: metadata + rights"| cat
- **§5** · diagram 1, edges ev -.-> u ("consume: playlist.op · push merge" and "consume: rights.changed · push revalidate"): The trailing '· push merge' / '· push revalidate' name what the client does with the event, not what crosses the edge. Per §5 the event bus→client edge states the consumed event only; the merge/revalidate is a transformation inside the client and belongs in u's label.
  fix:     ev -.->|"consume: playlist.op"| u
    ev -.->|"consume: rights.changed"| u
and add 'op → local merge · rights revalidate' to u's label body
- **§6** · diagram 1, edges gw→cat (2·), gw→ent (3·), gw→iss, gw→sr, gw→ps, gw→rec, u→pl (4·), u→pe (5·), pe→cdn: These cross-deploy-boundary edges lack the §6 contract: several have no path and no transport/format line at all ('GET row + rights flags', '3· GET verdict', 'mint signed URL', 'GET player state', 'GET homeline', 'forward range GET'); '5· GET /audio range' has a path but its second line is 'signed URL' (a credential attribute), not transport/format; 'GET /search typeahead' lacks the transport line.
  fix: e.g.gw -->|"2· GET /catalog/{id}<br/>HTTPS · JSON"| cat
gw -->|"gRPC Evaluate(T,U,R,K)<br/>gRPC · proto"| ent
gw -->|"POST /signed-urls<br/>HTTPS · JSON"| iss
gw -->|"GET /search?q<br/>HTTPS · JSON"| sr
gw -->|"GET /player-state<br/>HTTPS · JSON"| ps
gw -->|"GET /homeline<br/>HTTPS · JSON"| rec
u -->|"4· POST /playlists/{pid}/ops<br/>HTTPS · JSON"| pl
pe -->|"GET /audio range<br/>HTTPS"| cdn
- **§4** · diagram 1, edges u ==> gw (1·) and u ==> pe (5·): Two thick arrows in one diagram. §4 reserves ==> for the one primary data path the case is about; the reference pattern is a single ==>. The numbered labels already carry the ordering, so the second thick arrow is redundant; the audio fetch is an ordinary sync call.
  fix: Keep ==> only on u ==>|"1·..."| gw; change the audio fetch to u -->|"5· GET /audio/{id} range<br/>HTTPS · signed URL"| pe
- **§1 / semantics** · diagram 1, node cdn[/"🌍 CDN + POPs<br/>third-party vendors + Spotify edge"/]: Parallelogram is defined as an external third-party vendor, but the node's own label says 'third-party vendors + Spotify edge' — the Spotify-operated POPs are deployed units owned by this system, not vendor infrastructure, so they are drawn with the wrong shape (or should be a separate rectangle so 'CDN vs own POP' reads correctly).
  fix:     cdn[/"🌍 third-party CDN<br/>audio segments"/]  and  pop[Spotify edge POP<br/>last-hop audio cache] with pe -->|"GET /audio range<br/>HTTPS"| pop

## 21.frontend-calendar — violations (6 findings)
- **§4 (arrows)** · diagram 1, edge ls --> sync ("applied deltas · sync tokens"): Edge is drawn store→sync (the delivery/return direction) with label 'applied deltas · sync tokens', duplicating the existing sync-->ls read/write edge. Per §4 the request direction only is drawn and response edges should not appear; in prose the sync engine applies deltas and persists the token to the store (I3), not the reverse.
  fix: Drop the edge (the sync-->ls read/write edge already covers it), or if a read-back is intended, draw it as: sync -->|"read sync token"| ls
- **semantics** · diagram 1, missing edges np --> pch and pch --> bell: The diagram's own caption asks how a reminder reaches the client, but the reminder path is not drawn: no edge from the notification pipeline to the push channel, and no deliver-to-client edge into the notification presenter. Flow 5 in the prose says server pipeline → per-device push → client presenter.
  fix: Add: np -.->|"produce: reminder.due"| pch and pch -.->|"reminder.push<br/>push · JSON"| bell (bell already exists in the client subgraph).
- **§5 (transfer vs transformation / response content on request edge)** · diagram 1, edge sync -->|"2· pull(token)<br/>HTTPS · JSON — deltas + new token"| api: The request-edge label carries the response payload ('deltas + new token'); the edge label should state only what crosses on the request (the token), with payloads belonging to a sequence diagram.
  fix: sync -->|"2· pull(token)<br/>HTTPS · JSON"| api
- **§6 (store edge form)** · diagram 1, edge sync -->|"append ops · read/write rows"| ls: Store access edge label 'append ops · read/write rows' is not in OP-arg form (e.g. 'PUT outbox:{opid}' or 'SQL <table/purpose>'); it describes categories of work rather than an operation + argument.
  fix: sync -->|"PUT outbox ops · read/write rows"| ls or a split pair like "append outbox row" / "SQL events · tokens"
- **§8 (flowchart LR for the request path)** · diagram 1, flowchart TB: Diagram is flowchart TB; the spec fixes the 03-architecture request-path diagram as flowchart LR. Cosmetic — arrow decoding is unaffected.
  fix: flowchart LR (reorder subgraphs, Client first, to keep it readable left→right)
- **§4/§5 (request direction)** · diagram 1, edge exp --> sel: Edge shows exp delivering occurrences to selectors — the response direction; the request is the selector asking the expansion cache to expand a window (prose flow 1: 'selectors expand the requested range'). Within one deploy unit it is decoded as a data dependency, so this is a minor direction miss.
  fix: Replace with sel -->|"expand window a–b"| exp (keep ls --> exp as the row feed), and either drop exp --> sel or move the apply/notify sequence into a sequence diagram.

## 01.youtube — violations (6 findings)
- **§4 / §6 / semantics** · mermaid diagram, edge tr -.->|"produce: video.playable"| cat: Edge into a keyed store drawn dotted with produce: event vocabulary. Catalog DB is a cylinder (§1), not a topic, so 'produce:' has no topic to produce into; the prose says 'catalog update "v=123 playable at set R"' — this is a keyed catalog write, not an event.
  fix: tr -->|"write video:{id} file set"| cat  (or route through a topic: tr -.->|"produce: video.playable"| k{{📨 video events}})
- **§5** · mermaid diagram, edge hot -.->|"lifecycle move: originals"| cold: Label leads with an outcome/trigger word ('lifecycle move') instead of the data that crosses. Per §5 an edge label states what crosses — payload/event/query — and trigger phrasing gets rewritten as the data.
  fix: hot -.->|"originals"| cold (or "originals · lifecycle tier-down" stating only what crosses)
- **semantics** · mermaid diagram, edge cat -.->|"produce: video.updated"| rc: The catalog DB (a cylinder) is the source of a 'produce: video.updated' event. A keyed store does not produce events; prose says the fanout to search/rec follows the catalog update ('fan out to search/rec index + invalidate/pre-warm') with a service, not the DB, as producer.
  fix: Move the emit to tr (e.g. tr -.->|"produce: video.updated"| rc), or add a 📨 hexagon topic between them per §1.
- **§5 / §3** · mermaid diagram, node rc["🧠 search & rec\nindex · pre-warm hint"] as the target of cat -.->|"produce: video.updated"| rc: Label reads as an outcome ('index', 'pre-warm hint') rather than the data crossing. The transcoder does not write the rec index; the index update is work done inside the search/rec component after the event arrives, per the prose flow.
  fix: Tr[transcoder] -->|"renditions"| or -->|"PUT renditions"| Rc[rec indexer] with indexing as Rc's transformation — keeps tr's edge a pure transfer.
- **§6** · mermaid diagram, edge cr ==>|"PUT /upload\nchunked · resumable"| ue: Transport line says 'chunked · resumable' but omits HTTPS; every other cross-boundary HTTP edge carries 'HTTPS · ...', and the chunked transfer encoding rides on HTTPS. Cosmetic alone, but it breaks the one-formula contract.
  fix: cr ==>|"PUT /upload\nHTTPS · chunked resumable"| ue
- **§2** · mermaid diagram, node rc["🧠 search & rec"]: 🧠 in §2 means 'ML model / LLM'; the node is the search/recommendation *service* (deployed unit), not a model. Table has no entry for a rec/search service (🔍 is a search index), so reusing 🧠 for the whole service stretches the emoji's meaning. The table rule says missing types get added to the table, not repurposed.
  fix: Add R[(🔍 search index\nvideo text · rec vectors)] as a separate 🔍 cylinder from the 🧠 rec service, matching the prose pairing.

## 04.deployment-system — violations (6 findings)
- **§4 arrows / semantics** · high-level flowchart, edge ui -.->|"consume: run.* events"| k: The consume edge is drawn ui -->/-.-> k, i.e. the arrowhead says events flow from the api gateway into the bus. The prose says the opposite: the event bus is 'used for fanout to UI' and data flow 4 says 'bus --> UI/audit'. Every other consumer edge in the diagram points bus --> consumer (k -.-> res, k -.-> dag, k -.-> ctl); this one is reversed and reads as the gateway producing run.* events.
  fix: k{{📨 event bus<br/>run · job · log · artifact topics}} -.->|"consume: run.* events"| ui[api gateway<br/>UI · history · audit]
- **semantics (task step 4)** · high-level flowchart vs prose component 12 'Fleet rollout coordinator' and 'Platform self-deploy' data flow: Prose component 12 and the 'Platform self-deploy' data flow (waves over the platform fleet, global pause state intake can consult) do not appear anywhere in the diagram. It is its own named component and flow, not a detail folded into another node, and the diagram sits at 14 nodes so it fits within the ~15-node cap.
  fix: Add: fro[fleet rollout coordinator<br/>platform waves · self-gated] -.->|"consume: platform.health"| k  (and optionally fro -->|"pause query"| int), keeping total nodes at 15
- **§5 transfer vs transformation** · high-level flowchart, edge dag -->|"enqueue job"| sch: Edge label is an imperative/outcome phrase, not what crosses. The data moving is the claim-request; 'enqueue' names the action. Same class the spec calls out ('push config' --> 'purge list').
  fix: dag -->|"claim request<br/>job:{id} · pool"| sch
- **§6 / §5** · high-level flowchart, edge db -.->|"outbox relay: status events"| k: Event-edge labels must be produce:/consume: + event name. 'outbox relay:' names the mechanism, not the event API; the payload (status events) is named but the contract form is off-pattern from every other bus edge in the diagram.
  fix: db -.->|"produce: run.status"| k
- **§6 API edges** · high-level flowchart, edges dag-->sch, run-->sch, run-->sec, ctl-->he (and gh-->int second line): Four cross-boundary edges carry no method/RPC name or transport line, so none decodes alone: 'enqueue job' (dag --> sch), 'long-poll claim<br/>lease + fence' (run --> sch), 'OIDC token' (run --> sec), 'candidate metrics query' (ctl --> he). The spec requires method+path plus transport · format, or RPC method + transport. Also 'HTTPS · signed' on gh --> int names transport but 'signed' is a property, not a body format (JSON missing).
  fix: dag -->|"gRPC EnqueueJob()<br/>gRPC · proto"| sch; run -->|"gRPC Claim()<br/>gRPC long-poll"| sch; run -->|"gRPC MintCreds()<br/>OIDC token --> short-lived creds"| sec; ctl -->|"gRPC EvaluateGate()<br/>gRPC · proto"| he
- **§6 store access** · high-level flowchart, edge he -->|"candidate vs baseline query"| tsdb[(📈 app TSDB...)]: Label is a descriptive sentence, not the required OP arg form (or SQL <table> form). 'query' is implied by the store shape; the operation + argument should be explicit.
  fix: he -->|"query slo:{service, window}"| tsdb[(📈 app TSDB<br/>SLO metrics)]

## 07.blob-store — violations (4 findings)
- **§5 transfer vs transformation** · diagram 1, edge sc --> dp: The edge label mixes what crosses (shards) with the scrubber's own action ('verify checksums') — a §5 outcome/transformation word on a data edge, the same pattern as the forbidden 'aggregates popularity'. Verification is the node's job and is already in its label ('read + verify stripes').
  fix: sc -->|"GET shards + stored checksums"| dp  (or simply sc -->|"GET shards"| dp, keeping 'read + verify stripes' in the node label)
- **§6 API edges** · diagram 1, edge c --> fe (GET): Cross-boundary client edge carries method+path but no transport/format line; the parallel PUT edge correctly has 'HTTPS · signed chunks', so this edge does not decode alone.
  fix: c -->|"2· GET /object<br/>HTTPS · JSON"| fe
- **§4 arrows (numbering)** · diagram 1, edges c ==> fe (1·), c --> fe (2·), fe --> mt (3·): 1· PUT and 2· GET are alternative client operations numbered as if sequential, and numbering continues with 3· on fe --> mt while the data edge fe ==> dp — which the prose says must come first ('Order matters: data first, then index') — carries no number. A reader could decode index-before-data ordering that the prose explicitly rules out.
  fix: Either drop the numbers, or number per flow following the prose order: c ==>|"1· PUT /object…"| fe, fe ==>|"2· PUT shards…"| dp, fe -->|"3· read+write index"| mt on the PUT chain (GET flow unnumbered or its own count).
- **§6 API edges / §4 store access form** · diagram 1, edges fe --> mt and mp --> mt: Store access edges must carry OP + argument ('OP arg', e.g. 'GET index:{bucket}/{key}') or 'SQL <table>' form; 'GET + commit index records' and 'PUT part + commit records' name generic operations with no argument or table, so the edges don't decode without the prose.
  fix: fe -->|"GET + write index:bucket-key"| mt and mp -->|"write part index + commit"| mt

## 08.chatgpt — violations (6 findings)
- **§4 arrows / §8 semantics (fidelity to prose)** · mermaid flowchart (03-architecture.md), client↔gateway edges C((🧑 clients)) → GW and DC/SA → GW: The token stream never reaches the client: the flowchart ends with SA -->|"token stream<br/>SSE/WS · JSON"| GW, with no edge back to C and no <--> session edge, although the gateway holds a long-lived SSE/WS connection that carries the user turn in and the token stream out (prose: "terminates ~100M persistent client connections... streams tokens back to clients"). The diagram's own Q: caption asks "...to the token stream back?" yet the back leg is absent; §4 makes <--> the marker for exactly this long-lived session.
  fix: Either make the session edge bidirectional, e.g. C <-->|"1· user turn · SSE token stream<br/>SSE/WS · JSON"| GW (dropping the extra SA -->|"token stream"| GW line or keeping it as the internal leg), or add GW -->|"SSE token deltas<br/>SSE/WS · JSON"| C so the stream reaches the client.
- **§4 arrows (request direction only)** · mermaid flowchart, edge REG[...] -->|"gRPC GetVersionMap()"| SCH: The edge is drawn REG -->|"gRPC GetVersionMap()"| SCH, i.e. from the data source to the consumer, but GetVersionMap() is a getter the scheduler calls on the registry (deep dive 5: "registry state maps model+version → checkpoint distribution... the scheduler pins each request to a version at admission"; "schedule can't run without a version map"). §4 says draw the request direction only; as drawn, a reader decodes the registry calling a getter on the scheduler.
  fix: SCH -->|"gRPC GetVersionMap()"| REG[Model registry + canary router<br/>version → routing pct] — swap edge endpoints, keep the node declaration.
- **§2 icons (wrong emoji semantics)** · mermaid flowchart, edges PF -->|"⚡ KV get/put prefix id"| PC and DC -->|"⚡ KV get/put prefix id"| PC: The icon tables map ⚡ to Redis only; a missing component type must be left bare, not borrowed. The prompt/prefix cache is a paged in-memory GPU KV cache (deep dive 3), not Redis, so ⚡ tells a reader decoding by the tables that these edges hit Redis — wrong emoji semantics, and the ⚡ sits in edge labels while the PC node itself is bare, so the emoji does not annotate any sized name.
  fix: Drop the emoji from both edges: PF -->|"KV get/put prefix id"| PC and DC -->|"KV get/put prefix id"| PC (or, if the cache were actually Redis-backed, that would need saying in prose — it is not).
- **§5 transfer vs transformation / §6 API edges (minor)** · mermaid flowchart, edge GW -->|"2· load + append turns<br/>gRPC · proto"| CS: "load + append turns" is an operation description, not what crosses (the user turn), and is not an RPC method name; the file's own sequence diagram in 04-deep-dives.md uses "gRPC AppendTurn(user msg)" for this hop. Reading the turn is the conversation service's job internally (the CS→DS edge already carries "KV get+put turns").
  fix: GW -->|"2· gRPC AppendTurn()<br/>turn text · conv id"| CS
- **§4 arrows / §6 API edges (minor)** · mermaid flowchart, edge SA -.->|"produce: turn.completed"| AM: The async moderation edge is labeled "produce: turn.completed" but points directly at the consumer; there is no topic hexagon to produce onto (prose and deep dives name no event bus for this flow). Per §4/§6, produce: implies a topic endpoint — a reader following the standard looks for a hexagon and finds none.
  fix: Insert a topic node: K{{📨 Kafka<br/>policy events}}, then SA -->|"produce: turn.completed"| K and K -.->|"consume: turn.completed"| AM; or if no broker is intended, relabel as SA -.->|"turn.completed event"| AM.
- **§6 API edges (minor, missing transport line)** · mermaid flowchart, edge DC -->|"generated tokens"| SA: This cross-boundary edge names only the payload; unlike every other RPC edge in the diagram it carries no method name or transport/format line (§6 requires method + transport, or at minimum the transport, so each edge decodes alone).
  fix: DC -->|"gRPC StreamTokens()<br/>token deltas · proto"| SA

## 10.distributed-cache — violations (6 findings)
- **§3/§5 transfer vs transformation** · diagram 1 (high-level flowchart), edge cl --> sh: The second label line 'hash key → shard' is a transformation performed inside the client library (routing), not data crossing the edge; §3 says → chains belong in the owning component's label body and §5 says an edge label states only what crosses.
  fix: cl[client library / proxy<br/>slot map · hash key → shard · coalescer<br/>hit-rate metrics]  and  cl -->|"2· GET · SET by slot"| sh
- **§5 (minor: label wording)** · diagram 1, edge svc ==> cl: 'coalesced' is a mode/outcome word, not what crosses; per §5 the label should state the payload only. Crossing the svc→cl hop with no transport qualifier also sits oddly with the prose ('Either link in-process or run as sidecar'), where in-process would mean this is a call inside one deploy unit.
  fix: svc ==>|"1· GET key"| cl — coalescing is already on cl's label as 'coalescer'; drop 'coalesced' from the edge.
- **§6 API edges (minor: missing transport line)** · diagram 1, edge fc --> sh: 'failover ops' names neither an OP arg nor a method+path/transport; §6 requires every cross-boundary edge to carry a contract.
  fix: fc -->|"gRPC PromoteReplica()<br/>gRPC · proto"| sh (heartbeats/publish belong in fc's label, which already has them).
- **§6 API edges (minor: missing transport line)** · diagram 1, edge fc -.-> cl: Cross-boundary async push carries only a payload name, no transport/format line; the controller 'publishes slot-map updates' per prose, which fits a push/event contract.
  fix: fc -.->|"push: slot.map.update<br/>JSON"| cl (or 'slot-map update · JSON push').
- **§5 (minor: label wording)** · diagram 1, edge svc --> odb: 'on miss' is an outcome/condition word in an edge label; §5 lists 'miss' explicitly as an outcome that must be rewritten as the data crossing. The data crossing is the SQL SELECT.
  fix: svc -->|"3· SQL SELECT key"| odb — keep miss/fallback context in the prose data flows.
- **§8 (minor)** · file-level (no sequence diagram present): The case's core ordering/failure-mode interactions (failover promotion within 15 s, async ack-at-primary losing acked object writes vs sync-acked session writes surviving, write-behind persistence order) live only in prose; the whole file has one flowchart and no sequence diagram.
  fix: Add a sequence diagram: actor svc → cl →|"->> GET"| sh primary; alt primary fails — fc promotes replica, cl refreshes slot map, in-flight fail fast; note async-ack object-write loss vs acked session-write survival.

## 11.zookeeper — violations (3 findings)
- **§1 node shapes** · diagram 1, node tl: The ZAB transaction log is drawn as a keyed cylinder, but cylinder is reserved for stores accessed by keyed read/put. The only access in the diagram is 'f -->|"4· fsync proposal"| tl' — an append with no keyed read; nothing consumes it by key. Per §1 the generalized Kafka rule makes append-only logs hexagons ("cylinder only if every access is keyed read/put").
  fix: tl{{txn log<br/>zxid → mutation<br/>fsync before ACK}}
- **§6 API edges** · diagram 1, edge c -->|"1· write /path"| f and edge c <-->|"session<br/>ping/timeout · ephemerals"| f: Both client-to-ensemble edges cross a deploy boundary but neither names its transport. '1· write /path' gives the verb + path but no transport/format line, and the session edge names only what the session governs. §6 says every cross-boundary HTTP/RPC edge names its transport so it decodes alone.
  fix: c -->|"1· write /path<br/>TCP · session |" f   and   c <-->|"session · TCP<br/>ping/timeout · ephemerals"| f
- **§5 transfer vs transformation** · diagram 1, edge l -->|"5· COMMIT after quorum"| f: 'after quorum' is an outcome/condition, not what crosses the edge; §5 requires the label to state the data crossing. The parallel observer edge already uses the correct pure-payload form 'COMMIT zxid', so the two commit edges are inconsistent. (Quorum math already lives in the Leader label.)
  fix: l -->|"5· COMMIT zxid"| f

## 14.ml-production-system — violations (6 findings)
- **§4 + §6** · Diagram 1 (03-architecture.md), edge offl --> bud: The edge is drawn store→trainer with a payload label ("historical features"), implying the offline store initiates a synchronous push. Prose (Key data flows, cold path) says "training reads" — reads are drawn caller→store with the operation named, per §4 and the reference example (a -->|"SQL seen-set"| pg). It is also not in OP-arg / SQL <purpose> form.
  fix: bud -->|"SQL training partitions · point-in-time"| offl  (caller = trainer, operation + argument form)
- **§5** · Diagram 1 (03-architecture.md), edge mon -.-> bud: The label carries an outcome/trigger word as a directive: "retrain:". An edge label may only state what crosses (payload / event name); the retrain decision is mon's node-label job ("retrain trigger / page") and per §5 triggers get rewritten as the data that crosses or carry the produce:/consume: event form.
  fix: mon -.->|"produce: drift.metrics"| bud   (or plain payload: mon -.->|"drift metrics"| bud if it is a direct async call)
- **§6** · Diagram 1 (03-architecture.md), edge fs --> ms: Cross-boundary RPC edge names only the payload ("packed vectors") and the transport ("gRPC") but not the contract — §6 requires the RPC method name (like "gRPC GetDegree()") so the edge decodes alone.
  fix: fs -->|"3· gRPC Score()"| ms  with the transport + payload lines folded into one edge per §6, e.g. -->|"3· gRPC Score()<br/>proto · packed vectors"|
- **§6** · Diagram 1 (03-architecture.md), edge olap -.-> offl: Cross-boundary edge carrying only a payload description ("daily batch tables"), outside all three contract forms (METHOD+path, RPC method+transport, produce:/consume: + event). Reads also run the wrong direction relative to the standard's store-read convention.
  fix: offl -.->|"SQL daily batch tables"| olap   (offline store pulls, operation + argument)
- **semantics** · Diagram 1 (03-architecture.md), node olap[(OLAP warehouse)]: The OLAP warehouse node has no counterpart in this file's own prose — the Components table and Key data flows never mention a warehouse; it is only defined in 04-deep-dives.md dive 2 ("batch tables land daily (warehouse)"). A diagram node should correspond to the components/data-flows stated in 03.
  fix: Name it in 03's prose (add a Components row or a sentence in Training cold path, e.g. "warehouse daily batch tables feed the offline store, deep dive 2") so the node has provenance in the file that draws it.
- **semantics** · Diagram 1 (03-architecture.md), node bud[training + trainer]: The node merges two components the prose lists separately (Training-set builder and Trainer) under the name "training + trainer"; the label name matches neither prose component exactly, and the compression is invisible to the reader without the table. Acceptable as a deliberate merge, but the name should match what it stands for.
  fix: If both are kept as separate nodes: bud[set builder<br/>point-in-time join → offline eval] and tr[trainer<br/>training → vN artifact], wired bud -.-> tr — or fold builder+trainer into one "training pipeline" label; just keep the name aligned with one of the prose components.

## 20.frontend-newsfeed — violations (10 findings)
- **semantics** · diagram 1 (flowchart LR), node hyd + edge isl -->|"GET hydration payload<br/>HTTPS · entry_id list"| hyd: Diagram invents a deployed 'hydration-svc' (entry_id → payload) that does not exist in the file's Component list or Data flows. Prose says hydration is client-side island scheduling (dive 5: on-visible/on-interaction/on-idle; payloads come from the SSR inline payload / feed store; 01-requirements.md: 'timeline API that returns ordered feed entries (IDs + hydrated payloads)'). Contradicts the prose.
  fix: Delete the hyd node and its two edges; have isl hydrate from fs: isl -->|"hydrate entry_id list"| fs. If per-entry hydration payloads are really fetched over the wire, target tla: isl -->|"GET /timeline/hydration<br/>HTTPS · entry_id list"| tla
- **§4** · diagram 1, edge ssr -->|"HTML stream + page 1 payload<br/>HTTPS · HTML + JSON"| fs: This is the response leg of the u ==> ssr request already drawn (spec: draw request direction only, never the response as a second edge). As drawn it reads as a server-side push into a client store over HTTPS.
  fix: Drop the edge; state the seeding in fs's label or in the u ==> ssr edge label (e.g. ==>|"1· GET /feed<br/>HTTPS · HTML + inline page-1 JSON"|).
- **§6** · diagram 1, edge u ==>|"open /feed"| ssr: Primary cross-boundary edge lacks the contract form: 'open /feed' is not method+path and has no transport/format line.
  fix: ==> |"1· GET /feed<br/>HTTPS · HTML"|
- **§6** · diagram 1, edge ssr -->|"sticky variant"| ab: Cross-boundary edge carries no method+path or RPC method; 'sticky variant' describes the decision, not the contract.
  fix: -->|"GET /variants/sticky"| or, if both sit in one edge deploy boundary, leave the edge unlabeled.
- **§6** · diagram 1, edge u -->|"GET shell JS · media"| cdn: Has a method but no path and no transport/format line.
  fix: -->|"GET /shell.js<br/>HTTPS · JS"| with method+path and the transport/format line.
- **§5** · diagram 1, edge win -->|"window near last item"| pf: Edge label is a trigger/state condition ('window near last item'), not the data crossing the edge.
  fix: Rewrite as the data that crosses: -->|"scroll position · velocity"|.
- **§6 (store access label)** · diagram 1, edge win -->|"read window · mount anchors"| fs: Store edge label is not in OP arg (or SQL <table>) form; 'read window · mount anchors' mixes the read op with an outcome (mounting is winder's work).
  fix: win -->|"read page by anchor"| fs.
- **§6** · diagram 1, edge tel -.->|"exposure + interaction beacons"| rank: Cross-boundary async edge names the payload but carries no contract: no method+path/transport (beacons are not a Kafka-style topic, so produce:/consume: does not apply).
  fix: -->|"POST /beacons<br/>HTTPS · JSON"|
- **§3 (quoting)** · diagram 1, node tla: Node label contains ':' ('Case 02: pre-ordered entries') in unquoted tla[...] form.
  fix: Rephrase without the colon ('Case 02 — pre-ordered entries') or quote the label: tla["timeline-api<br/>Case 02: pre-ordered entries<br/>~15 items / page"].
- **semantics** · diagram 1, edge isl -->|"optimistic like · comment"| fs: Prose interaction path says 'tap like → optimistic store mutation → background POST → reconcile', but no background POST edge to the backend exists in the diagram — the server reconciliation side is missing, consistent with hyd being a stand-in for the timeline API.
  fix: Drop the dedicated node; route the interaction POST to tla (isl -->|"POST /v1/like<br/>HTTPS · JSON"| tla), keeping the optimistic write edge into fs.

## 26.google-maps — violations (6 findings)
- **§4** · Diagram 1 (high-level flowchart), edge agg -.->|"write segment speed"| spd: The telemetry write back into the segment-speed store cylinder is drawn as a dotted (async) edge, but the spec mandates store access edges be solid --> never dotted, carrying OP-arg form; a reader decoding the arrowhead sees an event hop instead of a store write.
  fix: agg -->|"write speed:{segment-id}"| spd
- **§4** · Diagram 1 (high-level flowchart), edge spd -.->|"new edge weights every 30–60 s"| rout: The weight-refresh edge into the rout cylinder is dotted and its label ("new edge weights every 30–60 s") is not OP-arg form — it states a frequency, not the read/write operation crossing. Store access must be -->; the refresh itself belongs in the producing component's → chain or the pipeline node per §5.
  fix: rout -->|"read weight snapshot"| spd, with the re-customization trigger carried in pipe's label (pipe[map-data pipeline<br/>imagery → graph → tile render · re-customizes rout])
- **§6** · Diagram 1 (high-level flowchart), edge gf --> srch[search + geocode ...]: The edge between two deployed units (geo-frontend → search + geocode) carries no label. Unlabeled edges are allowed only inside one deploy boundary, and no subgraph boundary is drawn here, so the reader cannot decode the call.
  fix: gf -->|"search + S2 bias<br/>gRPC · proto"| srch
- **§6** · Diagram 1 (high-level flowchart), edge pipe -.->|"map-edit events"| rout: The event edge into the routing-graph store lacks the required produce:/consume: prefix plus event name; "map-edit events" alone does not decode as an event contract.
  fix: pipe -.->|"produce: map-edit"| rout
- **§6** · Diagram 1 (high-level flowchart), edges u ==>|"1· GET tile z,x,y"| cdn, u -->|"2· search query"| gf, u -->|"3· GET directions origin→dest"| gf, gf -->|"origin · destination"| dir, dir -->|"route legs"| eta: Cross-boundary request edges carry method (or payload) only, with no transport · format line: tile leg lacks the transport line entirely, search/directions/legs edges state payloads with no contract at all.
  fix: e.g. u ==>|"1· GET /tile/{z,x,y}<br/>HTTPS · protobuf"| cdn; u -->|"GET /v1/search<br/>HTTPS · JSON"| gf; dir -->|"legs list<br/>gRPC · proto"| eta
- **§4** · Diagram 1 (high-level flowchart), labels "1· GET tile z,x-y", "2· search query", "3· GET directions origin→dest": The three serving paths (tiles, search, directions) are independent; the spec numbers labels only when a reader could misorder edges, so 1·/2·/3· here imply an ordering that does not exist. Cosmetic; format itself (digit + ·, no space) is correct.
  fix: Drop the 1·/2·/3· digits or keep them only if the paths gain sequence-dependent edges

## 30.facebook-messenger — violations (6 findings)
- **§4 (semantics vs prose)** · diagram 1, edge mb -->|"thread list · unread"| ib: Drawn as a solid sync edge store→service, but the prose says 'The mail inbox and unread counters are updated async from the mailbox rows — never counting on read'. Sync solid changes how the reader decodes the delivery path, and if it is meant as the inbox read path the direction is reversed (inbox reads from the store).
  fix: mb -.->|"mailbox rows · unread deltas"| ib
- **§4/§6** · diagram 1, edge pv -.->|"store variants"| s3: Store access edge drawn dotted with an action label ('store variants' is a verb, not what crosses). Store edges must be --> and carry OP arg form.
  fix: pv -->|"PUT blob variants"| s3
- **§5/§6** · diagram 1, edge gt -->|"2· enqueue + thread seq"| cs: Label describes the work done at the target (enqueue, seq assigned) instead of the payload that crosses, and a cross-boundary edge carries no method/transport contract at all.
  fix: gt -->|"2· POST /threads/{id}/messages<br/>gRPC · proto"| cs
- **§5** · diagram 1, edge cs -->|"3· canon append"| fs: 'canon append' is the transformation name, not the payload crossing to fanout svc; cross-boundary edge also lacks method/transport.
  fix: cs -->|"3· canon message copy<br/>gRPC · proto"| fs
- **§6 (minor)** · diagram 1, edge s -->|"shared link<br/>HTTPS"| pv: Cross-boundary edge has transport but no method+path contract.
  fix: s -->|"POST /links/preview<br/>HTTPS · JSON"| pv
- **§5/§6 (minor)** · diagram 1, edges s3 -->|"media · HTTPS"| cdn and cdn -->|"serve bytes"| rc: 'serve' is outcome wording and no method is named on either cross-boundary edge; cdn→rc also draws delivery rather than the request direction.
  fix: s3 -->|"media bytes<br/>HTTPS"| cdn ; cdn -->|"media bytes<br/>HTTPS"| rc

## 35.tiktok — violations (5 findings)
- **semantics** · Diagram 1 (serve funnel + ingest/observe): edges `upl -->|"transcode job"| xc` and `xc -.->|"produce: video.publish"| k`: The diagram routes upload --> transcode farm and has the transcode farm produce the video.publish event, which makes publish wait on the full ladder. The file's own prose says 'publish does not wait for it (dive 3)' and 04-deep-dives.md Dive 3 explicitly picks Option B: staged publish — metadata + lowest acceptable rung published immediately after the moderation gate, with renditions enriched as they land. In the drawn graph the publish hook can never fire before transcode completes, contradicting both prose and deep dive.
  fix: upl -->|"4· POST /upload/sessions<br/>HTTPS · chunk bytes"| uplModeration... — concretely: upl -->|"PUT original"| ob; upl -->|"PUT rung 1"| ob; upl -.->|"produce: video.publish"| k; and change the transcode-side edge to xc -.->|"produce: rendition.ready"| k with k -.->|"consume: rendition.ready"| pb (pb enriches renditions as they land).
- **§5** · Diagram 1: edge `cdn -->|"miss: pull segment"| ob`: Edge label leads with the outcome word 'miss:' — DIAGRAMS.md §5 explicitly names 'miss' as an outcome/trigger, not a payload; the label must state only the data crossing (the GET of the segment).
  fix: cdn -->|"GET segment:{id}"| ob  (or -->|"GET segment bytes"|)
- **§6** · Diagram 1: edge `u -.->|"3· POST events ~50/POST"| col`: Numbered label missing its contract: method+path form is incomplete ('events' is not a path) and there is no transport · format line, though this edge crosses the client/deploy boundary.
  fix: u -.->|"3· POST /v1/events<br/>HTTPS · JSON"| col
- **§6** · Diagram 1: edge `u -->|"4· POST /upload chunks<br/>resumable"| upl`: Cross-boundary upload edge carries 'resumable' as a second line, which is not a transport/format per §6 (method+path present, transport · format absent).
  fix: u -->|"4· POST /upload/sessions<br/>HTTPS · chunk bytes"| upl
- **§6** · Diagram 1: edge `upl -->|"transcode job"| xc`: Label 'transcode job' is not a §6 contract (no method+path+transport, no RPC+transport, no produce:/consume:) even though it crosses components of the ingest plane; prose describes it as enqueueing onto a queue-driven farm, so it is an event, not a sync call.
  fix: upl -.->|"produce: transcode.job"| xc  (queue-driven workers per prose; the enqueue itself, if kept sync, is upl -->|"gRPC EnqueueTranscode()<br/>gRPC · proto"| xc — but drawn enqueue should be dotted into the farm's queue)

## 36.airbnb — violations (10 findings)
- **§5 transfer vs transformation** · diagram 1, edge cdn -->|"miss · GET by content hash"| img: Label leads with outcome word "miss" — an outcome/trigger, not the data that crosses; the edge label must state the request/payload only
  fix: cdn -->|"GET img/{content_hash}<br/>HTTPS · JPEG"| img
- **semantics** · diagram 1, nodes gw/cal/eb edges gw -->|"PUT day grid"| cal and cal -.->|"produce: calendar.snapshot"| eb: Prose defines a Calendar service as a deployed component that commits all writes first and publishes snapshots, but the diagram omits it and reassigns its behavior: gateway writes the truth store directly (gw -->|"PUT day grid"| cal) and the DB itself produces events (cal -.->|"produce: calendar.snapshot"| eb), making the store a producer
  fix: gw -->|"PUT block"| c[calendar svc<br/>day grid writes · snapshot publisher]
c -->|"write day grid"| cal
c -.->|"produce: calendar.snapshot"| eb
- **§1 shapes + semantics** · diagram 1, node psp[/"PSP adapters…"/]: Node is drawn as an external vendor (parallelogram) but labeled "PSP adapters", which the prose lists as our own component ("Money ledger + PSP adapters"); the external system is the PSP vendors, and a vendor parallelogram's internals/adapters should not be labeled as ours
  fix: psp [/"external PSP vendors<br/>card auth · payouts"/]
- **§5/§6 store edges** · diagram 1, edge led -->|"release escrow · payout batch"| po: Edge label mixes an operation/outcome ("release escrow") with a payload ("payout batch"), and the edge leaves a store initiating work; stores answer queries — the payout service should read escrow entries from the ledger
  fix: po -->|"SQL escrow entries · payout batch"| led
- **§6 API edges** · diagram 1, edge gw -->|"query geo + dates"| ss: Cross-boundary service edge carries no contract — no method+path, no RPC method, no transport/format
  fix: gw -->|"Search(query)"<br/>… gRPC · proto|  i.e. gw -->|"gRPC Search()<br/>gRPC · proto"| ss
- **§6 API edges (store read form)** · diagram 1, edge ss -->|"geo-cell postings + bitmaps"| es: Store access edge states the returned data, not the operation+argument (OP arg or SQL <table> form)
  fix: ss -->|"SQL postings<br/>cell set · date range"| es
- **§6 API edges** · diagram 1, edge gw -->|"3· POST hold"| bk: Cross-boundary edge has no path or transport/body format line
  fix: b -->|"POST /v1/bookings<br/>gRPC · proto"| bk
- **§6 API edges** · diagram 1, edge g -->|"GET listing photos"| cdn: Client-to-CDN edge lacks path and transport/format line
  fix: "GET /img/{hash}<br/>HTTPS · JPEG"
- **§6 API edges** · diagram 1, edges bk -->|"authorize attempt"| psp and po -->|"payout attempt"| psp: Cross-boundary RPC edges to the external vendor name no method and no transport
  fix: authorizer inside adapters: bk -->|"HTTPS · card auth"| psp and po -->|"HTTPS · payout"| psp
- **§4 primary path** · diagram 1, edges g ==>|"1· …"| gw and g ==>|"2· …"| gw: Two thick-arrow edges from actor to gateway — ==> is reserved for the one primary sync path; with two thick edges the thickness no longer discriminates the primary path
  fix: one ==> to gateway (booking path), or ==> only on the correctness-critical edge with the search path as -->

## 38.leetcode — violations (1 findings)
- **§4/§6 (also semantic fidelity to the data-flows prose)** · mermaid diagram, edge `w -->|"SSE verdicts, ranks"| dlq` (line 14 of the file) together with edge `dlq -->|"verdict · rank events"| gw` (line 15): The client-facing delivery of the case's primary output path is drawn backwards and without its contract. The edge `w -->|"SSE verdicts, ranks"| dlq` points user→hub while the label names the payload that flows hub→user, so read literally the user pushes SSE verdicts to the delivery hub — a response leg, contradicting prose ("delivery hub pushes over the client's SSE conn"). Per §4 one-way SSE push draws as --> in the push direction; per §6 an edge crossing a deploy boundary must carry method+path plus transport·format, which "SSE verdicts, ranks" lacks. The companion edge `dlq -->|"verdict · rank events"| gw` (line 15) also disagrees with it about where the client's SSE session terminates (dlq directly, vs gw in the middle), so the two delivery edges cannot both be true as drawn.
  fix: Point the push in the push direction and close the contract where the session lives: replace the w-->dlq edge with `dlq -->|"GET /submissions/{sid}/stream<br/>SSE · JSON"| w` and delete the `dlq -->|"verdict · rank events"| gw` edge (the SSE stream then terminates at the hub, with gw used only on the submit path `w --> gw --> api`). If the hub is instead meant to push through the gateway, make the session legs consistent: `dlq -->|"GET /submissions/{sid}/stream<br/>SSE · JSON"| gw` and draw the client's subscription on `w --> gw`, never w directly to dlq.

## 39.e-commerce — violations (5 findings)
- **4 (Arrows)** · diagram 1 (single flowchart LR block), edge proj -.->|"PUT read models"| cach: Store access edge drawn dotted: proj -.->|"PUT read models"| cach is a projection consumer writing into the KV store; the spec says store access edges are --> (never dotted) and carry OP arg form. The OP arg form (PUT read models) is correct, but the dotted async arrow encodes it as an async event edge. The async part is already carried by the preceding sp -.->|"consume: ORDER_PLACED"| proj edge; the write into the store must be solid (reference example draws even produce as -->).
  fix: proj -->|"PUT read models"| cach
- **3 (Labels)** · diagram 1, node co: Label contains an unquoted colon: 'co[checkout orchestrator<br/>saga: reserve → authorize → promo → commit]'. The spec requires quoting any label containing '&', '<', '>' or ':'.
  fix: co["checkout orchestrator<br/>saga: reserve → authorize → promo → commit"]
- **6 (API edges) — minor** · diagram 1, edge cdn -->|"2· origin fetch<br/>HTTPS · JSON"| gw: 'origin fetch' is a descriptive verb phrase, not an API contract: the other cross-boundary edges carry method+path or RPC name ('GET /pdp', 'POST /checkout', 'reserve items'), this one does not name the method/path the CDN uses to fetch from the gateway, so the edge does not decode alone.
  fix: cdn -->|"2· GET /pdp<br/>HTTPS · JSON"| gw
- **4/6 — minor** · diagram 1, edge co ==>|"reserve items<br/>gRPC · all-or-nothing"| inv: The transport slot 'gRPC · all-or-nothing' embeds a semantic qualifier ('all-or-nothing') where the spec wants transport · format; also the third ==> thick arrow on a sub-step dilutes the spec's 'the one data path the case is about' (cli==>cdn and gw==>co already mark the two plane primaries). Cosmetic.
  fix: co ==>|"reserve items<br/>gRPC"| inv, and note 'all-or-nothing' in inv's label or prose
- **6 — minor** · diagram 1, edges gw -->|"GET /search<br/>gRPC · proto"| se and gw -->|"GET PDP<br/>gRPC · proto"| cat: 'GET /search<br/>gRPC · proto' mixes an HTTP method+path with a gRPC transport (gRPC has no HTTP paths); 'GET PDP' is likewise neither a METHOD /path nor a standard RPC method name. Cosmetic wording miss; transport is named so the edge still decodes.
  fix: cat["catalog svc<br/>fragments · facets"] and se edge label "GET /search<br/>HTTP · JSON", or drop the path and keep an RPC name like "Search()"

## 40.tinyurl — violations (7 findings)
- **§4** · diagram 1 (High level), edge api -.->|"insert code"| bl: The Bloom-filter insert is a blocking step of the create sequence (prose data flow: "KV put (+ write through cache, insert into Bloom filter) → respond with code"), but it is drawn as an asynchronous dotted edge, telling the reader the caller does not wait for it.
  fix: api -->|"insert code"| bl
- **§4** · diagram 1 (High level), edge ab -.->|"PUT status"| kv: Store access edge (KV cylinder) drawn dotted. If the verdict write is genuinely fire-and-forget, the spec-conform alternative is a produce:/consume: edge into a hexagon topic, not a dotted direct store write.
  fix: ab -->|"PUT status"| kv
- **semantics** · diagram 1 (High level), create path around api/kv/ca: Prose says the create path ends with a write-through into the cache layer before responding, but the diagram has no api → ca edge; ca is reachable only from rd on the redirect path. The "write-through" note on the api → kv edge is the only trace of it.
  fix: api -->|"PUT code → URL<br/>write-through"| ca
- **§6** · diagram 1 (High level), edge cdn -->|"2· origin redirect<br/>HTTPS"| rd: Cross-boundary HTTP edge has the transport line (HTTPS) but no method + path; "origin redirect" is a description, not a contract.
  fix: cdn -->|"2· GET /aB3xK9<br/>HTTPS"| rd
- **§6** · diagram 1 (High level), edges api -->|"allocate code<br/>gRPC"| al and rd -->|"GET verdict<br/>gRPC"| ab: Both cross-boundary RPC edges carry a transport line but no RPC method name; "allocate code" and "GET verdict" are descriptive, not the method that is the API.
  fix: api -->|"AllocateCode()<br/>gRPC · proto"| al and rd -->|"GetVerdict(code)<br/>gRPC · proto"| ab
- **§4** · diagram 1 (High level), labels 3· and 4·: Numbering implies a single chronological order 1→2→3→4 across two independent branches: 3· is the create path (POST /shorten) while 4· is the click event that in fact follows 1·/2· on the redirect path. Numbering is for edges a reader could misorder within one flow; as drawn it misorders the create vs redirect flows.
  fix: Keep 1· and 2· on the redirect path, continue as 3· on the rd -.->|"3· produce: click"| k edge, and drop the number from the POST /shorten edge (or restart numbering per branch).
- **§6** · diagram 1 (High level), edge u ==>|"1· GET /aB3xK9<br/>HTTPS · 302"| cdn: Cosmetic: 302 is the response status (a response detail the spec says not to draw) rather than a request transport/format token.
  fix: u ==>|"1· GET /aB3xK9<br/>HTTPS"| cdn

## 41.web-crawler — violations (7 findings)
- **semantics** · mermaid diagram, edge fe -->|"fetched blob"| ex: The fetcher→extraction edge is drawn as a synchronous call (caller waits), but the file's own failure prose says "buffered stream between fetcher and extraction; fetchers continue polite fetching while extraction catches up" — a blocking --> contradicts that. A reader decoding §4 semantics concludes fetchers stall on extraction capacity.
  fix: fe -.->|"fetched blob"| ex — change the arrow to dotted (async), matching the buffered handoff the prose describes.
- **§3 labels (node IDs)** · mermaid diagram, node intake[intake ...]: Node ID "intake" is 6 letters; IDs must be 1–4 lowercase letters. ID is scaffolding so this doesn't change decoding, but it fails the mechanical rule.
  fix: in[intake<br/>normalize → canonicalize<br/>→ canonical hash]
- **§5 transfer vs transformation** · mermaid diagram, edge intake -->|"enqueue by priority"| fr: Label states the operation/outcome ("enqueue by priority"), not the data crossing the edge; §5 requires the label to name the payload/event/query.
  fix: intake -->|"url · priority"| fr (state the payload; "enqueue by priority" is the operation, not what crosses).
- **§4 arrows (store access)** · mermaid diagram, edge fr -->|"hash check · insert"| seen: Store-access edge must carry OP + argument form, and "hash check" narrates the response half of a read-modify-write; the edge label should name the operation on the key (e.g. SADD/SET with the key), not describe probing.
  fix: fr -->|"SADD seen {url-hash}"| seen (or an equivalent single OP-arg store verb like "SET url-hash 1").
- **§4 arrows (store access)** · mermaid diagram, edge ex -->|"write record"| ut: Store-access edge label "write record" is not OP-arg / SQL form; parallel edge from fe already models this correctly as "PUT fetch status".
  fix: ex -->|"PUT content record"| ut
- **§6 API edges** · mermaid diagram, edge fe -->|"HTTP GET page"| ext: Cross-boundary edge to the external vendor names GET + HTTP but not a path or body format; §6 requires method+path plus transport/format so the edge decodes alone.
  fix: fe -->|"GET /{page-url}<br/>HTTP · HTML"| ext
- **§3 labels (format)** · mermaid diagram, node cons[indexer · link-graph builder<br/>· freshness scheduler]: List formatting slip: the second <br/> line opens with a stray " · " before the third component, so the intra-label multi-component list reads as two fragments rather than one list (cosmetic).
  fix: cons[indexer · link-graph builder<br/>· freshness scheduler] → cons[indexer · link-graph builder<br/>freshness scheduler]

## 42.linkedin — violations (4 findings)
- **§4 Arrows** · diagram, edge `jobs -.->|"PUT aggregates"| ds`: Store access edge is drawn dotted. Edge labels must be `-->` (never dotted); asyncness of the rebuild is already carried by the dotted consume: edges into jobs. Compare the compliant `jobs -->|"DEL profile:{mid}"| pc` directly above it.
  fix: jobs -->|"PUT aggregates"| ds
- **§4 Arrows** · diagram, edges `u ==>|"1· GET /in/{mid}..."| pr` and `cn ==>|"atomic edge upsert..."| gs`: Two thick arrows in one diagram. The spec reserves `==>` for "the one data path the case is about"; with both the profile-read path and the connection-write path thick, primary-path semantics is ambiguous for a reader.
  fix: Demote one: u ==>|"1· GET /in/{mid}<br/>HTTPS · JSON"| pr and pr -->|"2· GET degree<br/>gRPC"| g (drop thick), keep cn ==>|"1· POST /connect + edge upsert"| path thick; or drop both to --> .
- **§6 API edges** · diagram, edge `eb -.->|"consume: events"| ml`: consume: carries no event name — "events" is a category, not an API. The producer edges name profile.views/feed.engagement/edge.changes, so the consumer edge should name what it consumes.
  fix: eb -.->|"consume: profile.views · feed.engagement"| ml
- **§6 API edges** · diagram, edge `nt -->|"push · email"| u`: Crossing a deploy boundary to the member but carries only transports (push · email), no method/payload — the edge cannot decode alone like the other member-facing edges do.
  fix: nt -->|"invites · WVYP digest<br/>push · email"| u

## 44.payment-system — violations (8 findings)
- **semantics** · diagram 1 (flowchart), edges `nw -->|"PAN by token"| tv` and node `tv`: The edge "PAN by token" from the card-networks node into the tokenization vault contradicts the file's own design. The prose components table and 04-deep-dives.md §6 say: card capture happens client-side, the vault returns a token, and every platform component downstream (payments, ledger, risk, webhooks) sees only tokens — the auth request is built by our gateway adapter at the PCI seam, and 03-architecture.md's flag flow never places the vault as the networks' PAN source. Drawing nw → tv makes the external rail pull raw PANs from our vault, which is neither in the prose data flows nor consistent with the tokenization option chosen. The label also states the result ("PAN by token") instead of an operation, against §4's store-edge OP/SQL form.
  fix: Re-anchor the vault lookup to the auth-side caller: `pay -->|"GET pan:<token>"| tv` (or from an explicit adapter node), and remove `nw --> tv`. If a card-network token resolution hop is intended, it belongs to the network's side and should not touch our isolated vault node.
- **§1 node shapes / parallelogram = external vendor, internals never drawn** · diagram 1 (flowchart), node `nw[/card networks<br/>per-rail · per-acquirer<br/>gateway adapters/]`: Gateway adapters are our own deployed code (prose: per-acquirer/per-rail protocol + credentials + timeouts, "the distributed-system seam"), but the diagram folds them into the external vendor node's label as a third line. Per §1 a parallelogram is the whole external system whose internals are never drawn — the seam component is drawn as vendor internals instead of an internal unit, so a reader decodes the adapters as belonging to the card network.
  fix: Trim the vendor node to `nw[/card networks<br/>per-rail · per-acquirer/]` and show the adapters where they live — e.g. extend `pay`'s label with `· gateway adapters`, or add `ad[[adapter:<rail>]<br/>protocol + credentials · uniqueness ref]` between pay and nw.
- **§5 transfer vs transformation** · diagram 1 (flowchart), edge `pay ==>|"5· auth request · idempotent retry"| nw`: The edge label carries a behavior/outcome word ("idempotent retry") plus "auth request"; §5 says edge labels state only what crosses (payload/event). The payload per the prose's own flag flow is the auth-only request plus the uniqueness key to the bank — the retry is what the adapter does with it, not data crossing.
  fix: Rewrite to what crosses: `pay ==>|"5· auth request · uniqueness ref"| nw`
- **§6 API edges** · diagram 1 (flowchart), edge `gw ==>|"2· create/confirm intent"| pay`: Cross-boundary edge (gateway → payments service) carries "create/confirm intent" — no METHOD /path and no transport line, required by §6 for any edge crossing a deploy boundary.
  fix: e.g. `gw ==>|"2· POST /v1/intents<br/>gRPC · proto"| pay` (name the real method/path from the API design)
- **§6 API edges** · diagram 1 (flowchart), edge `pay -->|"3· risk pre-check"| rk`: Cross-boundary edge (payments → risk service) carries "risk pre-check" — operation wording only, no RPC method or transport line as §6 requires.
  fix: e.g. `pay -->|"3· gRPC PreCheck()<br/>gRPC · proto"| rk`
- **§6 API edges** · diagram 1 (flowchart), edge `wk -->|"signed webhook<br/>HTTPS · JSON"| m`: Cross-boundary push carries transport + format but no METHOD /path line, which §6 requires in «METHOD /path<br/>transport · format» order.
  fix: Keep the transport line and add a path, e.g. `wk -->|"POST /webhooks/charge<br/>signed · HTTPS · JSON"| m`
- **§4 arrows (store access edges)** · diagram 1 (flowchart), edge `pay -->|"6· postings"| led`: Store access edge label "6· postings" is neither an OP arg (like «4· INSERT idem:key») nor the «SQL <table/purpose>» form (the rec→led edge nearby does use `SQL scheme-ref+amount match`).
  fix: e.g. `pay -->|"6· INSERT postings"| led` and `rec -->|"SQL <postings> scheme-ref+amount "| led`
- **§4 arrows (sync vs async)** · diagram 1 (flowchart), edge `nw -->|"daily settlement files"| rec`: Solid `-->` encodes a synchronous call where the caller waits, but acquirer settlement files are an async push/drop the vendor does not wait on; §4 maps non-waiting delivery to `-.->`.
  fix: Draw the delivery as `nw -.->|"daily settlement file<br/>HTTPS/SFTP · CSV"| rec`

## 46.google-docs — violations (4 findings)
- **§4 arrows / store access** · Diagram 1 (edit/open flow), edge cs -.-> ds: Store access edge drawn dotted (-.->). Spec §4: store access edges are --> (never dotted); dotted reads as an async event, and the edge targets the same doc-store node as the solid 'CAS append ops' edge, giving mixed store decoding. Also 'every ~1K ops' on the second line is a trigger/frequency note, not what crosses.
  fix: cs -->|"put snapshot @ rev N<br/>snapshot + op batch"| ds  -- and move "every ~1K ops" into ds's label notes or the prose
- **§6 API edges (minor)** · Diagram 1, edge rd --> cs: Missing transport line and second line carries return payload ('snapshot + recent ops' is what comes back), not transport·format as §6 requires.
  fix: rd -->|"GET /open<br/>HTTPS · JSON"| cs
- **§6 API edges (minor)** · Diagram 1, edge cs --> ac: Cross-boundary edge lacks a contract: no method+path or RPC method + transport. 'ACL check' is a query type and 'live revocation' is an outcome word, not data crossing the edge.
  fix: cs -->|"GET /acl/{docId}<br/>HTTPS · JSON"| ac
- **§5 transfer vs transformation (minor)** · Diagram 1, edge cs -.-> ph: Second line 'TTL · lossy' states channel properties/behavior, not payload; not in produce:/consume: form for an event edge.
  fix: cs -.->|"produce: presence.cursor"| ph  -- and keep 'TTL · lossy' only in the ph node label/prose

## 47.rate-limiter — violations (5 findings)
- **§5 transfer vs transformation** · High-level diagram, edge an -.->|"greylist IPs as new rules"| cf: Edge label leads with the action/outcome 'greylist IPs as new rules' instead of the data that crosses. Per prose, the data is the new rule revisions (greylisted IPs); 'greylist' is the transformation the config publisher performs, not the payload.
  fix: an -.->|"produce: greylist.rules<br/>flagged IPs"| cf
- **semantics** · High-level diagram, edge an -.-> cf: The greylisting feedback edge originates from the dashboards/billing node, but the prose attributes greylisting to the abuse engine, listed as a separate consumer of the decision log. The abuse engine is not drawn anywhere, so the diagram contradicts the prose's consumer list.
  fix: Add e.g. ae[abuse engine<br/>429 streaks → greylist rule drafts] consuming decisions, with the cf edge sourced from it; or relabel the an node to include the abuse engine (e.g. 'abuse engine · dashboards') so the edge source matches the prose.
- **§6 API edges** · High-level diagram, edge gw -.->|"lease refresh deltas<br/>batched ~500 ms"| qa: Cross-boundary async call to the quota authority carries only a payload phrase; it names neither an RPC method + transport nor an OP-arg form, so the edge does not decode its contract alone.
  fix: e.g. gw -.->|"gRPC LeaseRefresh()<br/>delta batch · ~500 ms"| qa
- **§4/§6 store edges** · High-level diagram, edge ad -->|"rule writes"| cf: Store access edge label is not in OP-arg form ('rule writes' names a category, not operation + argument).
  fix: ad -->|"PUT rules · overrides"| cf
- **semantics** · High-level diagram, node ad[admin API · dashboard]: The admin API (rule-write path) node has no counterpart in the file's Components or Data flows sections; the prose never names what writes rules into the config store.
  fix: Name the rule-write/admin path in 03-architecture.md prose (Components list), or drop the node and route the rule-write edge from a circle actor.

## 48.google-search — violations (3 findings)
- **§4 (Arrows)** · diagram 1 (flowchart LR), edge sc -.->|"write results:{q}"| qc: Store-access edge drawn as dotted -.->, but §4 requires store access edges to be --> (never dotted), and the prose's query flow orders the cache-write inside the request path ("compose → cache-write → respond"), so async semantics are also wrong.
  fix: sc -->|"write results:{q}"| qc
- **§5 (Transfer vs transformation)** · diagram 1 (flowchart LR), edge ft -.->|"bg merger promotes segments"| bi: Edge label names the actor ("bg merger") and uses an outcome verb ("promotes") instead of stating what crosses (merged segments). Prose names the background merger as an independently deployed job, so per §5 it should be a node, not an edge-label phrase.
  fix: mr[bg merger<br/>fresh segments → merged base segment]
ft -->|"fresh segments"| mr
mr -.->|"merged segments"| bi
- **§6 (API edges)** · diagram 1 (flowchart LR), edge mg -->|"~5K candidates"| hv: Cross-service edge from the coarse ranker to the heavy ranker carries no transport/format line and the diagram draws no subgraph marking a shared deploy boundary, so the edge does not decode alone as §6 requires.
  fix: hv[heavy ranker<br/>~5K → top 200<br/>neural cross-encoder re-score] with edge mg -->|"~5K candidates<br/>gRPC · proto"| hv — or wrap fe/mg/sc in a subgraph titled "front-end deploy unit" so the edge is internal to one boundary

## 49.monitoring-metrics — violations (6 findings)
- **§1 node shapes / §2 icons** · diagram 1 (flowchart LR), node gr: Grafana dashboards is drawn as an actor circle with the user emoji, but it is a named deployed unit (a client service issuing PromQL), not a human/actor or an unnamed client population. The circle + 🧑 tells the reader a person is issuing the query.
  fix: gr[Grafana<br/>dashboards · 200 boards]  ==>|"2· PromQL query<br/>HTTP · instant · range"| qy[...]
- **§4 arrows (store access)** · diagram 1 (flowchart LR), edge pr -.-> lt: Store access edge is drawn dotted, but dotted is reserved for async events/queue consumption and store access edges are always --> (the spec's own §5 example shows a solid edge into S3). The reader decodes 'PUT 2 h blocks' as an async event rather than a store write.
  fix: rw[remote-write sidecar<br/>uploads compacted blocks] -->|"PUT 2 h blocks"| lt[(🪣 long-term store<br/>object storage · 13 months · downsampled)] — drawing the prose's sidecar as its own rectangle also restores the missing component; alternatively keep pr as source but change the edge to -->|"PUT 2 h blocks"| lt
- **§5 transfer vs transformation / §6 API edges** · diagram 1 (flowchart LR), edge am --> oc: Label 'page · slack · email' lists routing channels (where it goes), not the data crossing the boundary; a cross-boundary notification edge also carries no method/payload contract as §6 requires.
  fix: am -->|"firing alert payload<br/>PagerDuty · Slack · email"| oc((🧑 on-call))
- **§6 API edges** · diagram 1 (flowchart LR), edge pr ==> ex: The transport/format line carries cadence ('fixed 15 s') instead of a transport; no HTTP is named, so the edge doesn't decode its transport on its own.
  fix: pr ==>|"1· scrape GET /metrics<br/>HTTP · gzip<br/>fixed 15 s"| ex[...] (two <br/> lines stay within the limit)
- **§6 API edges** · diagram 1 (flowchart LR), edge pr --> pg: Cross-boundary HTTP edge has method+path but no transport/format line (e.g. 'HTTP · text').
  fix: pr -->|"GET /metrics<br/>HTTP · text"| pg[pushgateway<br/>short-lived batch jobs]
- **§4 arrows (==>) / §8 one diagram one question** · diagram 1 (flowchart LR), edges pr ==> ex and gr ==> qy: Two thick primary-path edges exist in one diagram (scrape path pr==>ex and read path gr==>qy); ==> is reserved for the one primary sync path, and §8 says write-path vs read-path questions get separate diagrams.
  fix: Keep the scrape path as pr ==> ex; draw pr -->|"gRPC StoreAPI<br/>read head + local blocks"| pr→qy solid, or split the read path into its own flowchart LR per §8

## 33.instagram — violations (4 findings)
- **semantics** · mermaid block: edge `stor -->|"active story ring<br/>HTTPS · JSON"| feed`: The edge 'active story ring → feed-api' has no counterpart in the file's prose. The Components section says the Feed API 'merges pools + celebrity/backfill sources + explore recommendations' (stories absent) and Data flow 2 repeats exactly that list; the story tray is already served directly to the client by the service (edge 3· GET story tray from stor). The diagram's own story-svc node and prose only define tray assembly inside the story service, so a reader decoding the diagram concludes feed-api merges story data that the design never gives it.
  fix: Remove the edge `stor -->|"active story ring<br/>HTTPS · JSON"| feed` entirely (the tray is already served to the client by the story svc), or keep it only with prose support and a read-direction label such as `feed -->|"GET /v1/stories/ring<br/>HTTPS · JSON"| stor` plus a sentence in the Feed API bullet explaining why the merge includes the ring.
- **§8** · mermaid block: node count: The diagram has 16 nodes (c, gw, ing, orig, xc, meta, rend, ev, pool, eng, feat, rank, feed, stor, exp, cdn), exceeding the ~15-node cap. `gw` (api gateway) is also not in the prose Components list, making it the natural one to fold.
  fix: Drop the `gw` node (gateway stays prose-only, as in the file's own Components list) and point `1· POST /v1/media` at `ing`, or fold two small services into one node to get to 15; alternatively split read path vs post path into two diagrams per §8.
- **§6** · mermaid block: edge `gw -->|"forward originals<br/>internal HTTPS"| ing`: Cross-deploy-boundary edge labeled 'forward originals<br/>internal HTTPS' — no METHOD /path and no RPC name, so it does not decode as a replayable contract (payload + transport only).
  fix: `gw -->|"POST /v1/media<br/>internal HTTPS · JPEG/HEIC"| ing` — method + path + transport + format, so the edge decodes alone.
- **§6** · mermaid block: edges `c -->|"3· GET story tray…"| stor`, `c -->|"4· POST engage events…"| eng`, `c -->|"5· GET media bytes…"| cdn`, `exp -->|"explore candidates…"| feed`, `stor -->|"active story ring…"| feed`: Labels name method + a described target, not a path: 'GET story tray', 'POST engage events', 'GET media bytes'; likewise 'explore candidates<br/>HTTPS · JSON' and 'active story ring<br/>HTTPS · JSON' carry payload + transport but no method/path or RPC name, so they are not standalone contracts.
  fix: Rewrite as method+path forms, e.g. `3· GET /v1/story/tray<br/>HTTPS · JSON`, `4· POST /v1/engage<br/>HTTPS · JSON`, `5· GET /{rendition-key}<br/>HTTPS · bytes` (the immutable content-addressed URL per 04 #5), and for the server-to-server pushes give the actual contract, e.g. `exp -->|"POST /v1/feed/candidates<br/>HTTPS · JSON"| feed`.

## 17.elevenlabs — minor (8 findings)
- **§5 (also §4 store-edge form)** · diagram 1 (sole mermaid block), edge gp --> va (voice-asset store): Label carries the trigger/outcome word 'on miss', which §5 explicitly lists as an outcome to rewrite as the data that crosses; it is also not the OP-arg form required for store access edges.
  fix: gp -->|"LOAD voice_weights:{voice_id}"| va  (the miss/caching note is already carried in gp's label as 'LRU voice cache')
- **§6** · diagram 1, edge ca --> vr (voice registry): Store access edge carries only a noun ('voice registry entry') with no operation and no argument form.
  fix: ca -->|"UPSERT voice:{voice_id}"| vr
- **§6** · diagram 1, edge ob --> cdn[/🌍 CDN/]: No method+path on a cross-boundary vendor edge; second line 'origin' is not a format. Direction is drawn as the store pushing to the CDN, but a CDN's origin interaction is a pull — the request direction is CDN → origin.
  fix: cdn -->|"GET /batch-objects/*<br/>HTTPS · origin pull"| ob
- **§5** · diagram 1, edge wsg --> sc: Second label line 'per-stream routing' describes controller behavior, not what crosses; the edge already names its payload ('text deltas'). This edge is inside one deploy boundary, where a bare payload label (or none) is the spec style.
  fix: wsg -->|"text deltas"| sc
- **semantics** · diagram 1, edge ca --> va, vs prose 'Voice registration (cloning)': Attributes clone-weight/checkpoint writes to the consent-and-abuse service. The file's own voice-registration flow places the professional-clone checkpoint write after the training job queued on the batch pool (a flow absent from the diagram); the consent service verifies, refuses, and watermarks — it does not produce model weights. Instant-clone embedding is the only weight write this edge can legitimately cover.
  fix: ca -->|"append attestation"| va  and add a clone/training path (e.g. rect 'clone-trainer' on the batch pool) -->|"checkpoint weights"| va, or relabel precisely 'append attestation · instant-clone embedding'
- **§3** · diagram 1, node cdn[/🌍 CDN/]: Label has no what-it-does line; the spec's label format is 'emoji Name<br/>what it does<br/>notes' and the spec's own CDN example adds a descriptor line.
  fix: cdn[/🌍 CDN<br/>vendor/] or cdn[/🌍 CDN<br/>batch deliverables · downloads/]
- **§6** · diagram 1, edge u <--> wsg (WS session): Second line lists payload direction but omits the transport/format token the reference example carries ('wss · JSON').
  fix: u <-->|"WS /v1/tts<br/>wss · JSON → PCM"| wsg
- **§6** · diagram 1, edge u --> ca (POST /clone): Second line states the payload ('samples + consent phrase') instead of a transport/format line; missing 'HTTPS · multipart'.
  fix: u -->|"POST /clone<br/>HTTPS · multipart"| ca

## 03.whatsapp — minor (2 findings)
- **§6 API edges** · diagram 1 (flowchart LR), edge s --> up (upload edge): Edge label "PUT blob" has a method but no path and no transport/format line; every cross-boundary API edge must decode alone per §6 (compare the correctly labeled r -->|"GET blob:{hash}<br/>HTTPS · ciphertext"| two lines below).
  fix: s -->|"PUT /v1/media/blobs<br/>HTTPS · chunked"| up[upload edge<br/>chunked resumable]
- **§6 API edges** · diagram 1 (flowchart LR), edge cg --> p (presence cluster): The label states what crosses ("online · typing state") but is not an API contract: no method+path and no transport/format line, even though chat gateway and presence cluster are separate deployed units. If they are treated as one deploy boundary, the labeled edge is fine as-is — confirm which.
  fix: cg -->|"state.update /v1/presence<br/>gRPC · proto"| p

## 02.twitter — minor (3 findings)
- **§4/§6 store edge label form** · diagram 1, edge s --> ci: Edge label "recent tweets top-N" does not use the required store-access forms (OP arg, e.g. `MGET key` / `SQL <table/purpose>`); it reads as an outcome description rather than an operation on the keyed store.
  fix: s -->|"SQL recent-tweets-topN 0 200"| ci
- **§6 API edge contract** · diagram 1, edge r ==> s (primary read path): "GET timeline" gives method + resource but not the slash-path form the spec requires (`METHOD /path`), unlike the reference `GET /v1/feed`.
  fix: r((🧑 reader)) ==>|"GET /timeline<br/>HTTPS · JSON"| s[...]
- **§4 primary-path arrow** · diagram 1, edges u ==> a and r ==> s: Two thick arrows in one diagram (u ==> a for the post path and r ==> s for the read path); §4 reserves ==> for "the one data path the case is about", and two primaries dilute the shape encoding the spec relies on for decoding.
  fix: Demote the read edge to -->|"GET /timeline<br/>HTTPS · JSON"| and add `primary read path` to s's label, or split the diagram into a write-path and a read-path flowchart per §8.

## 05.discord — minor (3 findings)
- **§6 (store edges: OP arg or SQL <table> form)** · mermaid diagram, edge api -->|"roles · overrides"| meta: Edge into the metadata store carries a free-form payload label ('roles · overrides') rather than the required OP-arg or SQL <table/purpose> form that sibling store edges in the same diagram use (INSERT msg:{id}, INCR unread:{uid}).
  fix: api -->|"SQL roles+overrides<br/>perms check"| meta
- **§6 (API edges) / §5 (label states what crosses)** · mermaid diagram, edge g -->|"heartbeat"| p: Cross-deploy-boundary edge (gateway fleet to presence cluster) with no contract: not METHOD+path+transport, not RPC method+transport, not produce:/consume:. 'heartbeat' is the raw data crossing; it lacks the transport/method line §6 requires on every boundary-crossing edge.
  fix: g -->|"gRPC Heartbeat()<br/>userID"| p
- **§4 (draw request direction only) / §5** · mermaid diagram, edge o -->|"origin pull: attachments<br/>HTTPS"| cdn: Request direction only (§4): the origin fetch is the CDN's request, so the arrow should point cdn --> o (the GET is initiated by the CDN); the current o --> cdn compensates with a mechanism-style label ('origin pull:') instead of stating what crosses, e.g. GET attachments/*.
  fix: cdn -->|"GET attachments/*<br/>HTTPS"| o

## 09.cdn — minor (3 findings)
- **§3 Labels (node ID format)** · flowchart diagram: nodes U, EP, SH, LB, O, CP, LP: Node IDs are uppercase: U, EP, SH, LB, O, CP, LP. Spec §3 requires IDs of 1-4 lowercase letters (reference example uses u, a, pg, rd, w, k). Cosmetic - the labels remain the truth - but a mechanical miss.
  fix: u((🧑 clients<br/>worldwide)) ==>|"1· GET /objects/{key}<br/>HTTPS · anycast"| ep[🌍 Edge POP<br/>...
- **§6 API edges** · flowchart diagram: edges CP --> EP and CP --> SH: Edges CP --> EP and CP --> SH cross deploy boundaries but carry only a payload ("signed config · purge list") with no contract line. §6 requires method+path plus transport, RPC method + transport, or produce:/consume: + event name for cross-boundary edges. Also the prose describes this as a push fan-out (async), so -.-> with produce: fits the spec's async rule.
  fix: CP[Control plane<br/>config · certs · purge] -.->|"produce: config.update · signed config · purge list"| EP
- **§6 API edges** · flowchart diagram: edge U ==> EP label "1· GET object", likewise "3· GET object" on SH --> LB: Numbered GET edges use "GET object" without a path form. §6 requires METHOD /path (e.g. GET /objects/{key}). The resource name is generic here, so this is cosmetic, but the spec form is method + slash-path.
  fix: ==>"1· GET /objects/{key}<br/>HTTPS · anycast" (and 2· "forward key lookup /objects/{key}")

## 12.bigtable — minor (2 findings)
- **§5** · diagram 1 (flowchart LR), edge m -.->|"assign · move tablets"| ts: Edge label states the master's action (trigger words) instead of the data that crosses. Per §5 an outcome/trigger like "push config" is rewritten as the data ("purge list"); here the payload is the assignment/move orders themselves.
  fix: m -.->|"tablet assignment · move orders"| ts
- **§4** · diagram 1 (flowchart LR), edge ts -->|"append commit log<br/>flush SSTables"| fs: The label bundles two operations on one solid (sync) edge, but the prose states MemTable flush is background work ("MemTable crosses its size threshold..."); a reader decoding --> as 'the caller waits' would think the flush blocks the write path.
  fix: ts -->|"append commit log"| fs
ts -.->|"SSTable files"| fs

## 13.kafka — minor (4 findings)
- **§6 API edges** · diagram 1, edge cg --> gcw (|"join · commit offset"|): Edge crosses a deploy boundary (client -> Kafka cluster) but the label names only the operations; §6 requires the contract to include transport/format (e.g. a second line naming the Kafka wire protocol), or the edge must be unlabeled-internal, which this is not.
  fix: cg -->|"join · commit offset<br/>TCP · Kafka wire protocol"| gcw
- **§4 arrows** · diagram 1, edge gcw --> off (|"read+write offset"|): The target is a hexagon (internal topic), but the label uses the §4 keyed-store read+write form, which reads as KV access and contradicts the append-only topic semantics of the hexagon shape.
  fix: gcw -.->|"produce: offset commit"| off
- **§5 transfer vs transformation** · diagram 1, edge pr --> t (|"metadata refresh<br/>on error"|): "metadata refresh" names a trigger action, not the data crossing the edge; §5 requires the label to state what crosses.
  fix: pr -->|"metadata request<br/>on error"| t
- **§5 transfer vs transformation** · diagram 1, edge ctl -.-> t (|"assignment metadata<br/>on leader election"|): Payload is stated, but the second line is a trigger clause ("on leader election"), not data crossing the edge; §5 wants trigger words rewritten as data.
  fix: ctl -.->|"assignment metadata<br/>leadership + ISR"| t

## 15.ml-ranking-system — minor (2 findings)
- **§6 API edges** · high-level diagram: edges ap-->ann/gph/frsh, ann/gph/frsh-->mg, mg-->fa, rk-->rr, fa-->rk: Edges crossing component boundaries (API -> retrieval sources "candidate query", sources -> merger "candidate ids", merger -> feature assembly "2· ~2,000 cands", ranker -> re-rank "predicted scores") state only the payload; none carries method+path plus transport/format or an RPC method name. The fa->rk edge does name gRPC but omits the method. Minor because §5 is satisfied (labels state what crosses) and the reader still decodes the data path.
  fix: ap -->|"gRPC Retrieve()<br/>gRPC · proto"| ann  and  rk -->|"gRPC Score()<br/>gRPC · proto"| rr (or, if the whole serving path is one deploy boundary, draw that boundary explicitly so unlabeled internal edges are legal)
- **§2 Icons** · high-level diagram: node rk: Heavy ranker is the DNN multi-objective model, an ML model per the file's own Components section, but the label starts bare instead of with the table's ML-model emoji 🧠.
  fix: rk["🧠 heavy ranker<br/>DNN multi-objective<br/>P(watch) · P(skip) · P(like)"]

## 16.dalle — minor (5 findings)
- **§2 icons** · Diagram 1 (flowchart LR), node wk: 🧠 is the emoji for an ML model/LLM, but gpu-worker is a deployed worker service (no emoji exists for a plain service; §2 says leave it bare). Cosmetic only — the name carries the meaning, so decoding is unaffected.
  fix: wk[gpu-worker<br/>diffusion → NSFW screen<br/>exact-request cache]  -- or keep 🧠 only by drawing the model as its own node
- **§4 arrows (numbering)** · Diagram 1, edges js --> jdb and js -->|"produce: image.job"| q: Prose states the job row is 'written BEFORE queueing', so the order of js -> jdb vs js -> q is load-bearing and a reader could misorder the two unlabeled edges from js.
  fix: js -->|"1· SQL upsert job row"| jdb
js -->|"2· produce: image.job<br/>by tier"| q
- **§6 API edges** · Diagram 1, edge sch -->|"lease batch"| wk: Cross-boundary sync edge without a contract: neither METHOD /path + transport, RPC method + transport, nor produce:/consume:. 'lease batch' reads as an operation, not what crosses.
  fix: sch -->|"gRPC LeaseJobs()"| wk
- **§5 transfer vs transformation** · Diagram 1, edge cdn -->|"GET object<br/>on miss"| rb: 'on miss' is an outcome/condition qualifier on the edge; §5 says an edge label states only the data that crosses. The payload is the GET.
  fix: cdn -->|"GET object"| rb
- **§3/§6 labels** · Diagram 1, edge u -->|"GET signed asset URL<br/>HTTPS · images"| cdn: 'GET signed asset URL' can be misread as the CDN issuing the URL, and the format token 'images' is not a body format (§6 wants transport · format).
  fix: u -->|"GET object via signed URL<br/>HTTPS · JPEG"| cdn

## 18.sora — minor (2 findings)
- **§4/§6 store access edge form** · high-level diagram, edge jo -->|"job · artifact · verdict rows"| mdb: Metadata-DB access edge label is a noun list ("job · artifact · verdict rows"), not the required store-access form: OP + argument or SQL <table/purpose> (compare spec's "🐘 SQL seen-set", "⚡ MGET feed:{uid}"). No operation (upsert/SQL) is named, so the edge doesn't decode as an access operation on its own.
  fix: jo -->|"upsert jobs · credits"| mdb
- **§4/§6 store access edge form** · high-level diagram, edge srv -->|"verdict"| mdb: Metadata-DB access edge label is the bare payload "verdict" with no operation; store access edges must carry OP arg form or SQL <table/purpose>.
  fix: srv -->|"write verdict"| mdb  (or -->|"SQL verdicts"| mdb)

## 19.frontend-chat-app — minor (7 findings)
- **§4/§6** · diagram 1, edge se ==> db: The store edge is drawn as ==> (thick) and its label 'write outbox first · client_msg_id durable' is design rationale, not the OP arg contract form stores edges must carry (cf. '⚡ MGET feed:{uid}' in the reference example). Spec §6 says store access edges are -->; ==> is the primary-path arrow.
  fix: se -->|"PUT outbox:{client_msg_id}"| db
- **§4** · diagram 1, edges u ==> v and se ==> db: ==> appears on u→v and se→db but not on the middle hop v→se, so 'the one data path the case is about' is two disconnected thick segments with a thin link between them. One path should be uniformly primary.
  fix: Draw the whole send chain thick: u ==> v ==> se (v --> se becomes ==>); or keep ==: on the request path only and use plain --> for the outbox write.
- **§6/§5** · diagram 1, edge se --> srv (HTTPS, label 'sync last_seq per conv<br/>HTTPS · JSON — gap fetch + merge'): No METHOD /path (spec form: 'GET /v1/feed<br/>HTTPS · JSON'); trailing '— gap fetch + merge' is client-side transformation wording on an edge (outcome, not payload); transport line 'HTTPS · JSON' is present.
  fix: se -->|"POST /batch_sync<br/>HTTPS · JSON · last_seq per conv"| srv
- **§5** · diagram 1, edge se --> st (label 'apply at seq · optimistic entry'): 'apply at seq · optimistic entry' states the transformation ('apply at') rather than only what crosses; the payload + seq is the data.
  fix: se -->|"PUT message · seq<br/>batched tx"| st  (or fold into se's label as 'apply at seq → render' and label the edge 'ordered message · optimistic entry')
- **§4** · diagram 1, edges out of se (db write, st apply, srv sync): The case's ordering (outbox write → optimistic entry → drain) is the point of the diagram, but the three sibling edges out of se carry no 1·/2·/3· numbers, so a reader can only guess the order from geometry.
  fix: se -->|"POST /sync?since={last_seq}<br/>HTTPS · JSON"| srv — or number the branch se -->|"1· write outbox"| db, |"2· apply optimistic"| st, |"3· drain"| srv
- **§3** · diagram 1, node cdn: Label is emoji+name only with no 'what it does' line; the spec's own example draws it as [/🌍 CDN<br/>vendor/]. Cosmetic — emoji is not alone (name text 'CDN' is present), so not a §2 semantic miss.
  fix: cdn[/🌍 CDN<br/>vendor/] or add the what-line, e.g. cdn[/🌍 CDN<br/>media objects<br/>HTTP · bytes/]
- **§6** · diagram 1, edge se --> cdn (label 'GET media by content hash<br/>HTTPS · priority=viewport'): 'GET media by content hash' has method+argument but no format token (transport 'HTTPS' and the priority param only); a cross-boundary vendor edge should decode alone.
  fix: se -->|"GET /media/{content_hash}<br/>HTTPS · bytes · priority=viewport"| cdn

## 23.frontend-video-streaming — minor (3 findings)
- **§6 API edges** · main flowchart, edge cdn ==> sched: Edge label "GET manifest · segment" names the objects but not the path, so the contract is only half-formed; a single GET cannot fetch manifest and segment anyway.
  fix: cdn -->|"GET /videos/{id}/manifest.m3u8 · /segments/*.m4s<br/>HTTPS · fMP4"| sched
- **§6 API edges** · main flowchart, edge org --> cdn: Cross-boundary edge (origin service to CDN vendor, two deploy units) carries a payload-only label "manifest + segment bytes" with no method/path or transport/format, and it is drawn in the data direction rather than the request direction.
  fix: cdn -->|"GET /segments/* on miss<br/>HTTPS · fMP4"| org (or, if kept as a fill flow, add the contract: org -->|"PUT /segments/*<br/>HTTPS · fMP4"| cdn)
- **semantics** · main flowchart, component bm (buffer meter): Buffer meter's edge to the scheduler is missing: the prose says the meter "exposes forward buffer seconds and the stall margin to ABR and scheduler" and that buffer seconds are the shared currency of ABR, scheduler and state machine, but the diagram routes buffer level only to ABR.
  fix: add bm -->|"buffer level secs · stall margin"| sched alongside the bm --> abr edge

## 24.mobile-chat-app — minor (5 findings)
- **§3** · Diagram 1 (flowchart LR), node gws: The node merges two components joined with '+'; §3 says intra-label multi-component lists are separated by '·'
  fix: gws[gwsvc<br/>socket gateway · delivery service<br/>per-conversation sequence]
- **§3** · Diagram 1 (flowchart LR), node sync label: 'gap detect · fill · dedup' is a sequential pipeline (F2: find gap → fetch → dedup), so it is a transformation that should be a '→' chain, not a '·' list
  fix: sync[sync engine<br/>gap detect → fill → dedup]
- **§4** · Diagram 1 (flowchart LR), edge store -->|"row deltas"| ui: Store-to-UI delta notification is an async subscription event, drawn as a solid sync arrow (caller-waits); §4 reserves --> for sync calls, -.-> for async
  fix: store -.->|"row deltas"| ui (dotted async delta event)
- **§5** · Diagram 1 (flowchart LR), edge push -->|"wake · new seq range"| conn: 'wake' is an outcome/trigger word, not data crossing; §5 says rewrite as the payload
  fix: push -.->|"notification<br/>seq range N"| conn (and it is an async wake, so -.->)
- **semantics** · Diagram 1 (flowchart LR), edge u ==>|"compose · send"| ctl: F1 states the send path is 'UI → controller', but the primary edge goes user → controller directly, skipping the UI layer node
  fix: u ==>|"2· send"| ui ... or relabel the edge as the composed message payload entering ctl and note the UI hop in prose

## 25.mobile-newsfeed — minor (3 findings)
- **DIAGRAMS.md §4** · mermaid block 1, edge fc ==>|"SQL rows · outbox append"| db: Edge label "SQL rows · outbox append" is a store access (reads rows, appends outbox) but drawn with →'. Per §4 store access edges are --> (the reference example draws every store read as -->, and →' is reserved for the one primary sync path — u ==> fc already carries it here, and thick arrows into db and ui dilute it).
  fix: fc -->|"SQL rows · outbox append"| db
- **DIAGRAMS.md §5** · mermaid block 1, edge fc ==>|"bind row · bitmap"| ui: Label uses the action verb "bind"; an edge may only name what crosses, and the reader wants the data (rows, bitmaps).
  fix: fc ==>|"rows · bitmaps"| ui
- **DIAGRAMS.md §5** · mermaid block 1, edge db -.->|"flush interaction batch<br/>on reconnect"| api: Trigger/outcome wording: "flush" (operation) and "on reconnect" (trigger/context) describe an outcome, not the data crossing — the §5 rule rewrites triggers as data. Secondary semantic nuance: the edge shows the local DB (a cylinder, normally a store endpoint) initiating a network call, while the prose says all network fetches go through the feed controller ("one writer").
  fix: db -.->|"interaction batch · reconnect"| api  (or move the edge to fc: fc -.->|"interaction outbox·reconnect flush"| api)

## 27.ev-charging-network — minor (6 findings)
- **§4** · high-level diagram, edge bus -.->"write raw events<br/>18 mo retention"| raw: Edge into a cylinder (keyed store) is dotted with a description label; §4 says store access edges are --> (never dotted) and carry OP arg or SQL <table> form, and '18 mo retention' is a policy note, not what crosses.
  fix: bus -->|"append raw_events/{charger}/{session}"| raw (dotted -> solid, OP arg form); move "18 mo retention" into the node label: raw[("raw event store<br/>dispute re-run input · 18 mo retention")]
- **§6** · high-level diagram, edge gw -->"3· idToken lookup"| tok: Cross-boundary edge carries a prose description instead of a contract; §6 requires method+path plus transport/format, or RPC method + transport.
  fix: gw -->|"gRPC Authorize(idToken)"| tok — or -->|"POST /v1/authorize<br/>gRPC · proto"|
- **§5** · high-level diagram, edge tok -->"GET token:{id}<br/>OCPI 2.2.1 · budgeted"| em: 'budgeted' is a policy/trigger annotation, not the data crossing (§5: rewrite outcomes/triggers as the data); the transport line names a spec version rather than transport · format.
  fix: tok -->|"GET token:{id}<br/>OCPI 2.2.1 · partner API"| em — 'budgeted' moves into prose/tok's label
- **§6** · high-level diagram, edge led -->"captures · refunds"| psp: Cross-boundary edge to an external vendor names operations but no method+path and no transport line; per the scoring rubric a missing transport line is cosmetic.
  fix: led -->|"POST /payments/capture<br/>HTTPS · JSON"| psp
- **§6** · high-level diagram, edge em -->"inbound roam CDRs"| oca: Inbound partner push names the payload only; §6 requires method+path plus transport/format (or produce:/consume: form for events).
  fix: em -->|"POST /ocpi/cdrs<br/>ocpi 2.2.1 · JSON"| oca — or name the OCPI method, e.g. em -->|"ocpi POST /cdrs<br/>HTTPS · JSON"| oca
- **§5** · high-level diagram, edge av -.->"per-partner occupancy queue"| oca: Label names the channel/mechanism (the queue) instead of the payload that crosses; per §5 an edge label states what moves, not the routing policy.
  fix: av -.->|"occupancy events"| oca — the per-partner queue belongs in the prose or av's label, not the edge

## 28.uber — minor (4 findings)
- **§6 API edges** · flowchart, edge gw --> q ("3· quote request"): Edge crosses between two separately deployed services (edge gateway, quoting) but carries no contract: no method+path, no RPC name, no transport/format line, violating the §6 requirement that cross-boundary edges decode alone
  fix: gw -->|"3· quote<br/>gRPC QuoteRide() · proto"| q  (or HTTP: "POST /quotes<br/>gRPC · JSON")
- **§6 API edges** · flowchart, edge gw --> ing ("4· pos batches"): Cross-service edge labeled only with the payload name; no transport/format (e.g. gRPC · proto) or produce:/consume: form, so the edge does not decode without the prose
  fix: gw -.->|"produce: driver.pos.raw"| ing  (or a sync -->|"POST /pos<br/>gRPC · JSON"| if the call is synchronous)
- **§6 API edges (also §5 wording)** · flowchart, edge me -.-> tsm ("7· trip handoff"): Cross-boundary async edge with neither produce:/consume: nor an RPC method; "trip handoff" names an outcome/trigger, not the payload crossing (§5 wants the data: the created trip event)
  fix: me -.->|"produce: trip.created"| tsm
- **semantics** · flowchart, edge sda --> st ("write multiplier") vs pricing svc (pr): The damping/hysteresis control-loop math lives in the pricing svc label, yet the tile-store write edge is drawn from the aggregator (sda), whose prose job is only windowed counts; the prose itself is split ("aggregator → pricing service updates tiles" vs "its output is the surge tile time-series"), and the diagram picks the latter while keeping damping+hysteresis on pr, leaving the multiplier computation attributed to the wrong node
  fix: Replace with pr -->|"write multiplier"| st, or relabel sda consistently as the control-loop job (windowed counts → damping · hysteresis) so the math and the write stay in one node

## 29.uber-eats — minor (10 findings)
- **§6** · diagram 1, edge me -->|"2· accept"| gw: Cross-boundary API edge with no method+path/transport contract; the label names the transition/outcome ('accept') rather than the payload crossing.
  fix: me -->|"2· POST /orders/{id}/accept<br/>HTTPS · JSON"| gw
- **§6** · diagram 1, edge cu -->|"3· pos pings"| gw: Cross-boundary API edge labeled with a noun phrase ('pos pings'), missing the method+path+transport contract.
  fix: cu -->|"3· POST /location/pings<br/>HTTPS · JSON"| gw
- **§6** · diagram 1, edge gw -->|"4· validated transition"| os: Gateway→order-service edge crosses a deploy boundary with no contract (no method/RPC name, no transport).
  fix: gw -->|"4· POST /orders/{id}/transition<br/>gRPC · proto"| os
- **§6** · diagram 1, edge gw -->|"5· cart reads"| cat: Gateway→catalog-service edge has no HTTP method+path/transport contract.
  fix: gw -->|"5· GET /menus/{id}<br/>HTTPS · JSON"| cat
- **§6** · diagram 1, edge cat -->|"PURGE menu:{id}"| cdn: The CDN is a parallelogram vendor, not a keyed store, so the bare OP-arg store-edge form doesn't apply here; the purge edge carries no transport line.
  fix: cat -->|"PURGE /menus/{id}<br/>HTTPS"| cdn
- **§4/§6** · diagram 1, edge ds -->|"prep + travel ETA"| eta: No RPC contract on a cross-boundary edge, and the label describes the response payload ETA svc sends back instead of the request that crosses in the drawn direction.
  fix: ds -->|"gRPC GetEta()<br/>proto"| eta
- **§6** · diagram 1, edge gw -->|"courier pings"| lp: Gateway→location-pipeline edge has no method+path/transport contract.
  fix: gw -->|"POST /location/pings<br/>HTTPS · JSON"| lp
- **§4** · diagram 1, edge lp -.->|"consume: location.pos"| tr: The edge is drawn producer→consumer but labeled consume: — the spec draws consume: edges from the topic to the consumer (§7), and the flow is already covered by lp's produce: edge into ob, making this one redundant and direction-inconsistent.
  fix: ob -.->|"consume: location.pos"| tr (and drop the redundant lp -.-> tr edge)
- **§3** · diagram 1, edge gw -->|"5· cart reads"| cat: Numbered as 5· (after 4· validated transition), but per the file's own flow cart reads precede the order placement (step 1: 'customer builds cart against cached catalog'); the number tells the reader a false order.
  fix: renumber the cart-reads edge before the POST /orders edge (e.g. 1· GET /menus) or drop its number
- **§8/semantics** · diagram 1, nodes tr <--> c / tr <--> cu: Prose says per-order rooms are 'multiplexed over the three parties' sockets' (customer, courier, merchant), but the diagram draws WS sessions only to customer and courier.
  fix: me <-->|"WS order events<br/>wss · JSON"| tr

## 31.netflix — minor (5 findings)
- **§5 transfer vs transformation** · High-level mermaid diagram, edge fl -.->|"push to appliances"| oc: Edge label 'push to appliances' states an action/trigger, not the data crossing, exactly the 'push config'-style outcome label the spec says to rewrite as data.
  fix: fl -.->|"tonight's title set · renditions"| oc["🌍 Open Connect<br/>OCA fleet · HTTP range serving"]
- **§5 transfer vs transformation** · High-level mermaid diagram, edge og -->|"off-peak fill pull"| fl: Edge label 'off-peak fill pull' names the operation, not what crosses the edge; it does not state the payload the spec requires.
  fix: og -->|"fill list · renditions<br/>off-peak HTTPS"| fl["fill<br/>forecast-popular titles"]
- **§6 API edges** · High-level mermaid diagram, edge m -->|"2· choose title"| ps: '2· choose title' has no method+path or transport line although it crosses the device→cloud deploy boundary; the sibling m→cp edge carries the full contract, so this edge alone does not decode as an API call.
  fix: m -->|"2· POST /playback<br/>HTTPS · JSON"| ps["playback svc<br/>title → manifest<br/>codec/HDR/DRM negotiation"]
- **§6 API edges** · High-level mermaid diagram, edge m ==>|"4· GET chunks<br/>HTTP range · no control plane"| oc: 'GET chunks' has no path; the second line mixes a real transport ('HTTP range') with a note ('no control plane') that belongs in the node label or prose, not the contract slot.
  fix: m ==>|"4· GET /titles/{id}/segments<br/>HTTP range"| oc
- **§6 API edges** · High-level mermaid diagram, edge m -->|"1· GET rows<br/>HTTPS · JSON"| cp: 'GET rows' is method plus a noun, not the METHOD /path form; no path is given.
  fix: m -->|"1· GET /ui/rows<br/>HTTPS · JSON"| cp["control plane<br/>UI rows: candidates → rank"]

## 34.reddit — minor (5 findings)
- **semantics** · diagram 1 (write path) edges post -.->|"produce: vote.cast"| k and cmt -.->|"produce: vote.cast"| k vs diagram 2 (read path) edge k -.->|"consume: content.events"| rankc: The event contract does not connect between the two diagrams, and content-write publication is never drawn: post service publishes to the event log for feed caches per prose flow 1 ("Submit a post → Postgres + event log. Feed caches pick it up asynchronously"), but the only content-service produces in the write diagram are 'produce: vote.cast'; the read diagram's rank cache consumes 'content.events', which no producer in either diagram emits. The log's label 'votes · impressions · mod events' also does not name content.events.
  fix: Rename consumers' contract or producers' to match, e.g. in write path: post -.->|"produce: content.events"| k and cmt -.->|"produce: content.events"| k (vote.cast only from the vote path), so the read path's consume: content.events has a producer on the other diagram.
- **§3 labels** · diagram 1 node votev; diagram 2 nodes rankc and redis: Node IDs are 5 lowercase letters; §3 caps IDs at 1–4 lowercase letters.
  fix: votev[vote consumer ...], rankc[rank cache refill] → rc[rank cache refill], redis[(⚡ Redis ...)] → rd[(⚡ Redis ...)]
- **§6 API edges** · diagram 1 (write path) edge w -->|"HTTPS · JSON"| edge; diagram 2 (read path) same edge: Edge crosses a deploy boundary but carries only transport+format, no method+path. This also applies to diagram 2's identical edge.
  fix: w ==>|"1· GET /v1/write<br/>HTTPS · JSON"| edge (or POST /submit) and w ==>|"1· GET /v1/feed<br/>HTTPS · JSON"| edge — add the method+path line; optionally use ==> as the primary sync path.
- **§1 node shapes / subgraphs** · diagram 1 and diagram 2: subgraph Client containing only w, subgraph Edge containing only edge: Not a rule violation, an editorial check: the two Client/Edge subgraphs wrap a single node each (edge has two) purely for layout tidying, which §1 discourages; they are defensible as tier boundaries since Client, Edge, Services, Stores are the case's real tiers. No change required.
  fix: @{direction LR} is the mermaid-native fix; a DIAGRAMS.md amendment to allow it would apply beyond this case.
- **§8 one diagram, one question** · diagram 2 (read path) edge cmt -->|"GET tree page"| redis vs prose data flow 2: Noted as a gap, not a violation: the miss fallback to Cassandra (and reconciliation read) the prose describes is not drawn — cmt has only the Redis read. §8 permits pushing detail to prose, so this is acceptable; a sequence diagram is the spec's suggested home for ordering/failure modes.
  fix: If the miss path needs showing, add cmt -->|"CQL prefix scan miss"| cass and a sequence diagram in 04-deep-dives.md for miss ordering; otherwise the flowchart is acceptable under one-diagram-one-question.

## 43.zoom — minor (7 findings)
- **§6** · diagram 1, edge sg --> ra: cross-boundary call labeled "2· pickSFU()" names the RPC method but omits the required transport line (§6: every cross-boundary RPC edge names its transport)
  fix: sg -->|"2· pickSFU()<br/>gRPC · proto"| ra
- **§5** · diagram 1, edge ra --> sfu: edge label "3· assign relay" states the action, not the data that crosses; §5 requires edge labels to name only what moves (the relay assignment payload)
  fix: ra -->|"3· relay assignment<br/>SFU address · token"| sfu
- **§3** · diagram 1, edge u <--> sfu: edge label "SRTP media session<br/>subscribe: layer per receiver" contains an unquoted ':' — §3 requires labels containing ':' to be quoted or the colon removed
  fix: u <-->|"SRTP media session<br/>subscribe · per-receiver layer"| sfu
- **§6** · diagram 1, edge sg -.-> nt: async event edge lacks the required produce:/consume: + event name contract form (plain "meeting events")
  fix: sg -.->|"produce: meeting.events"| nt
- **§2** · diagram 1, node db: the Postgres emoji is used on a generic "SQL" name; the emoji table maps 🐘 to Postgres specifically, so either name the store or the emoji silently claims an unmapped store type
  fix: db[(🐘 Postgres<br/>meetings · roster)]
- **semantics** · diagram 1, edge nt --> u: notifications (invites · starting soon) target users before they join, but the push edge points at u ((🧑 participants)) — the in-meeting population; a reader decodes invites as going to people already in the call
  fix: nt -->|"push notification<br/>invites · starting soon"| inv((🧑 invited users))
- **§8** · file level, no diagram: the reconnect path (ICE failover, keyframe request to resume, roster re-sync) is a failure-mode interaction with ordering that §8 says to draw as a sequence diagram; it lives only in prose
  fix: sequenceDiagram; P->>actor participant: dead path detected; alt signaling session valid / relay same; participant -) participant: new ICE to same relay; else reassigned; participant -) participant: new ICE to reassigned SFU; end; participant -) participant: keyframe request to resume video

## 45.typeahead — minor (3 findings)
- **§5 (and §4 store-edge OP form)** · main flowchart, edge bl[batch build] --> ix[index nodes]: Edge label "swap top-k segments" carries the action/outcome word "swap" instead of the payload that crosses (the segment files). Per §5, an edge label states only what crosses; the trigger word must be rewritten as data (same pattern as the spec's "push config" -> "purge list"). It is also a store-edge label that is not in OP-arg form (§4), since ix is a cylinder. Payload "top-k segments" is visible in the label so the diagram is not misread, hence minor.
  fix: bl -->|"PUT top-k segment files<br/>double-buffer swap"| ix
- **§6 API edges** · main flowchart, edge u((user)) --> cdn[/CDN/]: Cross-boundary edge (client to external CDN) carries a payload phrase ("hot prefix · static top-N") but no method+path+transport contract as §6 requires. Missing transport line is classified minor.
  fix: u -->|"GET /snapshot/{pfx}<br/>HTTPS · JSON"| cdn[/🌍 CDN<br/>hot-prefix snapshots/]
- **§6 API edges** · main flowchart, edge t[typeahead service] --> hb[hot layer]: Cross-boundary edge into a service carries an operation noun ("trending overlay read") and no RPC method/transport line per §6; the payload (trending entries for the prefix) is not stated. Minor: direction and purpose still decode unambiguously.
  fix: t -->|"gRPC ReadTrending(pfx)"| hb[hot layer<br/>streaming counts · trending boost<br/>velocity guardrails]

## 50.stock-exchange — minor (3 findings)
- **§5/§6** · Diagram 1 (flowchart LR), edge sq --> jr: Second line 'journal before dispatch' is a sequencing directive, not what crosses and not a transport/format line — the payload ('inbound msg seq N') is correct, but the extra clause reads as an edge outcome rather than data crossing (§5: label states only what crosses; §6: second line is transport · format)
  fix: sq -->|"3· journal before dispatch: inbound msg seq N<br/>append-only log"| jr — put the ordering constraint into the message description, not as the second line
- **§6** · Diagram 1, edge sq --> me (label "4· dispatch seq N"): Sequencer to matching engine shards are separate deployed units (sequencer has its own active/standby), yet the edge label is a bare payload with no transport line; per example, companion edge gw --> sq carries 'TCP · FIX-like binary'
  fix: sq -->|"4· dispatch seq N<br/>in-proc · binary"| me — add a transport/format line, or keep unquoted payload and add the transport the reader needs to decode it
- **§6** · Diagram 1, edge od --> cc (label "trade records<br/>exact-once IDs"): Cross-boundary edge to an external vendor (parallelogram = CCP) carries only payload wording; no method+path or RPC method + transport, so the edge does not decode alone per §6
  fix: od -->|"POST /trades<br/>gRPC · proto<br/>exact-once IDs"| cc — or RPC form: "SubmitTrades() gRPC · proto"; keep the payload wording inside the method line

## 06.apple-app-store — minor (8 findings)
- **§6** · mermaid diagram, edge dev --> gw ("1· upload binary<br/>resumable HTTPS"): Cross-boundary edge carries only the payload ('upload binary') plus transport 'resumable HTTPS', but no METHOD /path contract as required by §6 for edges crossing a deploy boundary.
  fix: dev -->|"1· POST /v1/submissions<br/>HTTPS · resumable upload"| gw
- **§6** · mermaid diagram, edge u --> sf ("4· update check-in<br/>HTTPS · device profile"): Boundary-crossing edge lacks method+path; 'update check-in' is a payload/operation name, not METHOD /path. Transport is present.
  fix: u -->|"4· POST /v1/check-in<br/>HTTPS · JSON"| sf
- **§6** · mermaid diagram, edge u ==> cdn ("5· GET variant + delta bytes<br/>HTTPS · range requests"): Primary-path edge has method (GET) but no path; 'variant + delta bytes' is the payload. Transport line present.
  fix: u ==>|"5· GET /v1/objects/{key}<br/>HTTPS · range requests"| cdn
- **§6** · mermaid diagram, edge u --> iap ("6· POST purchase<br/>HTTPS · idempotency key"): Edge has method (POST) but no path; 'purchase' is the payload. Transport present.
  fix: u -->|"6· POST /v1/purchases<br/>HTTPS · JSON · idempotency key"| iap
- **§6** · mermaid diagram, edge cdn --> dist ("origin fetch · signed URLs"): Cross-boundary edge carries mechanism words ('origin fetch', 'signed URLs') instead of a method+path or produce:/consume: contract; signed URLs is auth detail, not the API contract.
  fix: cdn -->|"GET /v1/objects/{key}<br/>HTTPS · stream"| dist
- **§6** · mermaid diagram, edge pipe -.-> rv ("submit: package ref"): Async handoff uses invented prefix 'submit:'; the spec's event-vocabulary is produce:/consume: + event name.
  fix: pipe -.->|"produce: app.submitted"| rv
- **semantics** · mermaid diagram, node sf (storefront-api) vs component list item 4 (Delta/Manifest Service): Component 4 'Delta/Manifest Service' from the component table is not drawn as its own node; its job (delivery plan → signed URLs, delta chain + full fallback) is folded into the storefront-api label, which itself is not in the component table (prose calls it 'storefront/check-in tier'). The function survives the merge so no reader is misled, but node-to-component mapping is one-to-many.
  fix: Either draw it as its own rectangle (d[delta-manifest-svc<br/>device+version → delta chain<br/>fallback URL in response]) between sf and cdn, or rename the diagram node to the prose's tier name and note the consolidation in prose.
- **§6** · mermaid diagram, edge iap --> psp ("POST charge<br/>HTTPS · tokenized card"): Cross-boundary edge to the external PSP has method (POST) but no path.
  fix: iap -->|"POST /v1/charges<br/>HTTPS · JSON · tokenized card"| psp

## 22.frontend-uber — minor (2 findings)
- **§6** · high-level diagram, edge gw -->|"position batches<br/>lossy fresh-wins"| rc: This edge crosses a deploy boundary (backend subgraph into app subgraph) but names no transport; the sibling ride-events edge quotes 'WS ride events seq'd<br/>wss · JSON', while this one's second line ('lossy fresh-wins') is delivery semantics, not a transport/format line.
  fix: gw -->|"position batches<br/>wss/UDP-push · JSON<br/>lossy fresh-wins"| rc — or keep two <br/> max by folding lossy into an existing line, e.g. -->|"position batches<br/>wss push · lossy fresh-wins"|
- **§6** · high-level diagram, edge d((🧑 drivers)) -->|"GPS ping every 3 s<br/>gRPC-RT"| gw: Non-standard transport notation 'gRPC-RT'; the spec's RPC form is method/operation plus a transport line like 'gRPC · proto'. A reader cannot decode what '-RT' means from the diagram alone.
  fix: d((🧑 drivers)) -->|"GPS ping every 3 s<br/>gRPC · proto"| gw

## 37.ticketmaster — minor (2 findings)
- **§6** · High-level flowchart, edge reap -->|"release(hold-id)"| inv: Cross-boundary RPC edge to inventory names the method ('release(hold-id)') but not its transport; §6 requires every cross-boundary edge to decode alone with its transport named (every other service-to-service edge in this diagram carries one, e.g. 'HTTPS · JSON').
  fix: reap -->|"release(hold-id)<br/>gRPC · proto"| inv
- **§6 / §5** · High-level flowchart, edge psp -.->|"webhook: authorized"| ord: The PSP webhook edge uses neither accepted form: it is not method+path+transport (HTTP form) nor produce:/consume:+event name (event form), and 'authorized' is an outcome word rather than the data crossing (§5) — the payload is the authorization result message. Also inconsistent with the produce:/consume: style used on the log edges.
  fix: psp -.->|"POST /webhook<br/>HTTPS · JSON"| ord
