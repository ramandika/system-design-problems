# RPC frameworks & serialization formats across the case catalog

Where gRPC and protobuf appear in the 50 study cases, what role they play, and how deep each
case goes. Companion to [CONCEPTS.md](CONCEPTS.md); see also [KAFKA.md](KAFKA.md) (which owns
the schema-registry boundary from the log side), [BIGTABLE.md](BIGTABLE.md) (RPC-free
partition-aware access) and [POSTGRES.md](POSTGRES.md) (the thinnest of these component docs,
until this one).

Headline: **this is the thinnest component doc in the set, and that is itself the finding.**
gRPC is a one-case technology in this catalog: only [05.discord](05.discord/README.md) designs
it into an architecture, as an *internal* data-services hop. No case studies a serialization
format the way 13 studies Kafka. The closest thing to a genuine wire-format trade-off lives in
[26.google-maps](26.google-maps/README.md) (raster pixels vs vector protobuf payloads) and
[04.deployment-system](04.deployment-system/README.md) ("one wire format" as a rollout
invariant). An explicit REST-vs-gRPC Option A/B appears **nowhere** — the catalog's stance is
implicit and consistent, not argued.

Counts below are **mention-based greps** (distinct from [CONCEPTS.md](CONCEPTS.md) union
counts). Filtered out: ~10 files where "serialization" means concurrency control (one
serialization point, serialized queue — 18, 39, 41, 44, 47, 48, 50) and 24's *client-side
SQLite* schema evolution (local DB migration, not wire schema). Thrift: zero hits anywhere.
Avro: one hit, as a sizing lever (13). Generated-client patterns: zero hits.

- Files with a real hit on gRPC / protobuf / wire format: **10 / 50**
- gRPC designed into the case's own architecture: **1** (05 — the only designed component)
- gRPC discussed as a documented pattern, not built by the case: **1** (03, citing Discord)
- By-reference only (pointing at the same idea): **1** (12)
- Explicit Option A/B where gRPC is the vehicle (trade-off is really service topology): **1** (14)
- Serialization format itself the trade-off object (binary payload vs pixels): **1** (26)
- Schema/wire-format compatibility as the trade-off object: **1** (04)
- Naming gRPC/schema tooling in order to fence it out: **1** (13 — schema registry out of scope)
- Transport shorthand only (a diagram edge label): **2** (08, 22)
- Explicit REST-vs-gRPC decision anywhere: **0**

## Summary table

| Case | Role | Depth |
|---|---|---|
| [05.discord](05.discord/README.md) | The only designed component: gRPC data-services layer between API and wide-column store | Designed component |
| [03.whatsapp](03.whatsapp/README.md) | Cites Discord's gRPC data-services layer as the documented pattern to adopt, with caveats | Pattern discussion |
| [26.google-maps](26.google-maps/README.md) | Vector tiles as a protobuf-like binary payload vs raster pixels — the wire-format trade-off | Deep trade-off |
| [04.deployment-system](04.deployment-system/README.md) | "The whole world runs one wire format" as a rollout invariant across regions | Deep trade-off |
| [14.ml-production-system](14.ml-production-system/README.md) | Option A: separate inference service per model over gRPC vs co-located in-process serving | Trade-off (topology) |
| [13.kafka](13.kafka/README.md) | Schema registry/evolution explicitly out of scope — "the log carries bytes"; Avro as a sizing lever | Boundary |
| [12.bigtable](12.bigtable/README.md) | "A later system built a gRPC data-service layer on the same partition-aware idea" (cross-link to 05) | By-reference |
| [08.chatgpt](08.chatgpt/README.md) | "HTTP/gRPC per turn" — diagram edge for gateway → conversation service | Brief |
| [22.frontend-uber](22.frontend-uber/README.md) | "socket (WS/gRPC-RT, keepalive)" — realtime-edge transport shorthand | Brief |
| [01.youtube](01.youtube/README.md) | "one protobuf-encoded row per entity" — validation detail on YouTube's Bigtable warehouse | Brief |

## Per-case notes

### The one designed component: the internal data-services hop

**[05.discord](05.discord/README.md)** — the only case that puts gRPC in its own component
list ([03-architecture.md](05.discord/03-architecture.md)): a **DATA-SERVICES LAYER (gRPC,
consistency-hash routing per channel/user)** between the REST/RPC API fleet and the
wide-column message store. [04-deep-dives.md](05.discord/04-deep-dives.md) makes it the thing
that absorbs the store choice's costs: "we win bounded partitions and lose a two-level routing
layer and an occasional cross-partition read — the data-services consistent-hash routing shim
(**one gRPC instance per channel shard**, request coalescing of concurrent same-row readers)
absorbs both." The unit of routing is the channel shard, and n readers of the same row become
one DB query.

**[03.whatsapp](03.whatsapp/README.md)** — WhatsApp's actual store is not public, so deep dive
§3 adopts Discord as the documented analogue and names the two transferable details, the first
being "(a) the data-services layer between API and DB — one gRPC endpoint per query type,
consistent-hash-routed so a channel's traffic hits one instance, with request coalescing." The
case's *own* architecture ([03-architecture.md](03.whatsapp/03-architecture.md)) has no such
layer — its hot path is gateway → delivery service → gateway, and its stated priority is
minimizing hops — so this is a pattern citation, not a designed component.

**[12.bigtable](12.bigtable/README.md)** — pure by-reference: MapReduce-style batch scans read
tablets directly, "no central fanout service needed — the tablet map IS the partitioning plan,"
with a pointer to WhatsApp §3 "for how a later system built a gRPC data-service layer on the
same partition-aware idea." The pattern's ancestry is the 2006 paper's client library, which
does partition-aware routing without a server-side RPC tier at all.

### Trade-offs where the format or the framework is the object

**[26.google-maps](26.google-maps/README.md)** — the catalog's only genuine on-the-wire
serialization trade-off ([04-deep-dives.md](26.google-maps/04-deep-dives.md)). **Option A —
raster tiles**: 256×256 PNG/WebP pixels; client just blits, but labels are pixels — the same
tile must be re-rendered per locale and theme, exploding the cache-key space. **Option B —
vector tiles**: "geometry + attribute protobuf/protobuf-like payload (the Mapbox Vector Tile
spec is the public analogue)"; one tile serves every locale and style, and payloads shrink at
high zoom. Chosen: **hybrid**, because at ~10M tile-fetch QPS the argument is the edge hit
ratio, not bytes — vector collapses the cache key from `tile × locale × style × day` toward
`tile × epoch`, and "at 10M QPS the hit ratio is the product." The cost is stated honestly:
renderer complexity, per-platform output drift, client CPU on low-end devices. Binary
serialization wins here for *key cardinality* reasons, not size reasons.

**[04.deployment-system](04.deployment-system/README.md)** — schema-evolution-as-operations
([04-deep-dives.md](04.deployment-system/04-deep-dives.md)). Rolling out the deploy platform
itself, multiregion. **Option A — regions upgrade on their own schedule**: worst case, "a
protocol bug in event *serialization* means region A (v5) writes transition events region B
(v4) rejects → cross-region run state diverges precisely during the rollout." **Option B —
globally serialized waves**: hours to complete, coordinator is a critical dependency. Chosen: B
("we take hours of rollout latency to make 'the whole world runs one wire format' an
invariant"), with parallelism inside each wave. Contract-mismatch during mixed versions — not
wire efficiency — is what actually breaks fleet rollouts.

**[14.ml-production-system](14.ml-production-system/README.md)** — Option A is "a separate
inference service per model over gRPC": independent scaling, per-model languages and hardware,
per-service blast radius; the loss is "one extra network hop on the hot path (tens of ms —
against a 50 ms p99 budget that is enormous)" plus version skew between feature service and
model service, and fleet count growing as O(models). Option B (one co-located binary, many
in-process models) wins; gRPC here is the vehicle for the separation, and the case's lesson is
that a ~ms-class internal RPC hop is not free against a hard p99 budget. At ChatGPT scale
([08.chatgpt](08.chatgpt/04-deep-dives.md), by-reference) the calculus inverts — one enormous
workload justifies the dedicated service.

### The deliberate boundary

**[13.kafka](13.kafka/README.md)** — the schema-registry boundary, drawn from the log side
([01-requirements.md](13.kafka/01-requirements.md)): "schema management (schema registry,
evolution, compatibility rules) — the log carries bytes; schema governance is a separate
service." The broker stays dumb in terms of the payload's structure, not just its durability.
Avro appears once, in [02-scale-estimates.md](13.kafka/02-scale-estimates.md), as a sizing
lever: bumping the average 1 KB message to 4 KB ("Avro/Parquet-adjacent") at 1M msgs/s with 4×
compression yields ~259 TB/day on disk ×3 replicas ≈ 1.8 PB for 7 days — ~3× the 600 TB
baseline, so richer payloads dwarf compression gains. Note the honest asymmetry: 04 treats wire
compatibility as an ops invariant, 13 fence-lines it as someone else's service, and no case
actually designs a schema registry.

### Brief mentions

- **[08.chatgpt](08.chatgpt/README.md)** — architecture diagram edge: client (SSE/WS) →
  gateway, then "HTTP/gRPC per turn" gateway → conversation service. Unlabeled choice; gRPC
  reads as the default internal transport, exactly the pattern 05 makes explicit.
- **[22.frontend-uber](22.frontend-uber/README.md)** — diagram edge: driver apps ↔ backend,
  "socket (WS/gRPC-RT, keepalive)." Transport shorthand on the realtime edge; no framework
  argument anywhere in the case, and the designed contracts are the two socket channels, not
  the RPC plumbing.
- **[01.youtube](01.youtube/README.md)** — "one protobuf-encoded row per entity" in YouTube's
  Bigtable-backed dimensional warehouse (Google Cloud blog, Aug 2023, by the system's own
  engineers). A validation detail pinning a real design: row key = entity, payload =
  protobuf. No trade-off attached.

### Filtered (in context, not RPC/serialization)

- **38.leetcode, 45.typeahead** — SSE-vs-WebSocket-vs-polling transport trade-offs that
  *would* be streaming-RPC trade-offs in a gRPC-first catalog; they never mention gRPC and
  resolve on HTTP/2 + SSE instead. Adjacent, filtered.
- **21.frontend-calendar** — "server contract is taken as given: REST/push sync API." REST is
  a given, not a decision.
- **Discord's own REST/RPC API** — 05 mixes "REST or gateway socket message op" without
  arguing it; no case in the catalog states a REST-vs-gRPC decision.

## The portable pattern

The catalog never argues RPC choice, but what it does use reduces to one shape, cross-linked
to [05.discord](05.discord/README.md) as the reference implementation:

1. **gRPC is for the internal hop only.** It appears between an API/gateway plane and a
   store or scoring service (05's data-services tier, 08's per-turn hop) — never at the
   client edge. Every browser/mobile edge in the catalog is REST, SSE, or WebSocket (05, 08,
   22, 38), and no case argues about it; the edge protocols are chosen by what browsers can
   speak, not by what the framework offers.
2. **The value is partition affinity, not the framework.** 05's gRPC tier earns its place by
   consistent-hash routing so a channel's traffic hits one instance and coalesces concurrent
   readers; 12 points out the same partition-aware idea predates RPC entirely (Bigtable's
   client library, "the tablet map IS the partitioning plan"). If a server-side tier would
   just re-implement client-library routing, skip the hop.
3. **Wire format matters where it collapses cache keys, not where it saves bytes.** 26's
   vector-protobuf tiles win because the payload carries locale/style work to the client,
   shrinking the cache-key space at 10M QPS. A ~2× byte saving on an internal message is
   noise against the arithmetic 13 shows for payload growth (4 KB Avro-adjacent messages →
   ~3× stored bytes).
4. **Schema compatibility is an operations decision, not a framework feature.** 04 buys
   "one wire format" with hours of serialized rollouts; 13 hands the whole problem to a
   separate schema-governance service. Two mechanisms outside the RPC stack; neither case
   assumes the framework solves evolution.
5. **The extra-RPC-hop tax is real.** 14 prices tens of ms against a 50 ms p99 budget and
   picks in-process co-location; 03 refuses hops on its hot path by design. Default to fewer
   hops; add a service-per-model (or the internal RPC tier that implies it) only when scale or
   blast-radius demands it.

### Boundaries: when even gRPC is overkill

- **Client-facing APIs**: browsers can't speak raw gRPC (grpc-web needs a proxy and loses
  streaming in most stacks), and this catalog's edges are all user devices — so the edge is
  REST/SSE/WS across all five frontend cases and every server-driven case, without exception
  or apology.
- **Small internal services**: 14's arithmetic — an extra hop is enormous against a tight p99
  budget — means gRPC service separation is a scaling decision, not a style choice; dozens of
  models, in-process wins; thousands of models or one giant workload, separation wins, and
  only then does the transport matter.
- **High-durability event flow**: 13's boundary holds — the log carries bytes; if schema
  governance is needed at all, it is its own service, and 04 shows the compatibility problem
  can also be bought down with rollout discipline alone.
- **Where it's already the right call**: one partitioned store with many query types behind an
  API fleet (05) — the shape where a schema'd unary RPC per query type plus hash routing pays
  for itself, and the only place in this catalog it does.
