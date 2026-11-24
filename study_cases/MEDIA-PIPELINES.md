# Media pipelines (transcode ladders & ABR) across the case catalog

Where the video/photo media pipeline — ingest, transcoding DAGs, rendition ladders, ABR playback —
appears in the 50 study cases, what role it plays, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md) (precompute-at-write union, Tier 2, 26% — the ladder is that
union's flagship instance), [OBJECT-STORAGE.md](OBJECT-STORAGE.md) (where renditions live), and
[CDN.md](CDN.md) (where they are served from). Playback-protocol transport lives in
[REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) only where it is interactive; HLS/DASH-style
segmented delivery is covered here.

Headline: **every media case builds the same DAG: one fast analysis pass becomes the coordination
artifact, segments are retryable idempotent units, and the ladder is a cost curve — not a fixed
table.** No case studies "a transcode farm" per se; six cases design one, each rediscovering the
other's trade-off (bitrate quality vs operational simplicity; publish-latency vs ladder
completeness).

- Cases with a designed media-pipeline component: **6 / 50** (01, 18, 23, 31, 33, 35)
- Designed component: 5 ingest/transcode; the client playback stack is fully designed in 1 (23)

## Summary table

| Case | Ingest/transcode | Ladder policy | Publish staging | Client playback |
|---|---|---|---|---|
| [01.youtube](01.youtube/README.md) | THE deep dive: segmented parallel DAG, UIR, second-pass re-encode | Popularity-driven; codec ladder H.264→VP9/AV1 | Publish after mux join gates | ABR + segmented streaming protocol |
| [31.netflix](31.netflix/README.md) | Per-title, per-language jobs with join gates + QC | Per-title *and* per-shot complexity-aware | QC gate before publish | Client ABR (delegated to 23) |
| [33.instagram](33.instagram/README.md) | Worker fleet; renditions content-addressed | Fixed photo ladder + video ABR ladder | "Renditions done" state transition | ABR playback; offline upload queue |
| [35.tiktok](35.tiktok/README.md) | Transcode farm (~26K cores est.), queue-driven | Per-title ladder; separate audio track | **Publish before ladder completes**, first segment primed | Server-hinted rendition negotiation |
| [23.frontend-video-streaming](23.frontend-video-streaming/README.md) | — (server assumed) | Consumes the ladder, doesn't build it | — | The client stack is THE design (ABR controller, segment scheduler, worker demuxer) |
| [18.sora](18.sora/README.md) | Post-encode pool: upscale + H.264/AV1 encode ("exactly YouTube's transcode DAG") | Renditions as final stage of generation | Publish-gated on watermark/C2PA | — |

## Per-case notes

### The reference design

**[01.youtube](01.youtube/README.md)** — [04-deep-dives.md](01.youtube/04-deep-dives.md): the naive
linear encode takes hours; instead **split into N-second segments, transcode independently, mux**
(3600 s video ÷ 6 s chunks = 600 parallel work units per rendition × 5 renditions). The stated
trade-off: segmented encoding loses lookahead across chunk boundaries and with it some compression
quality, mitigated by passing keyframe/hint decisions between workers. Chunk boundaries must land
on keyframes — so ingestion runs a fast analysis pass (scene detection + keyframe map) that becomes
the DAG's coordination artifact, and each `(video, rendition, segment)` is a retryable idempotent
unit. Two cost levers beyond the DAG: a **universal intermediate representation** feeding all
renditions, with a **second, higher-effort re-encode for popular videos** (popularity-driven, not
uniform); and the **codec ladder** H.264 → VP9/AV1, where modern codecs cut ~20–40% bitrate at
3–10× transcode cost — worth it only on the popular tail. Finally, **fixed-function transcode ASICs
** (Argos-class) change the cost constant and force the DAG to be codec-agnostic and the fleet to
be sized in two currencies (software and fixed-function).

### Ladder policy: fixed table vs per-title/shot-aware

**[31.netflix](31.netflix/README.md)** — the explicit version of 01's cost curve: a fixed ladder
wins *operational simplicity* (one manifest shape, one CDN pricing story) and loses bytes on hard
scenes and quality on easy ones; per-title and **shot-aware** encoding (decisions from per-shot
complexity) wins bytes-per-quality at the cost of ladder computation at ingest — publicly
documented Netflix practice. The ingest DAG runs independent per-rendition/per-language jobs with
join gates and a QC gate before publish, and the analysis pass's shot/complexity map is what the
ladder decision consumes — the same "analysis pass as coordination artifact" move as 01. The
prepositioning/OCA appliance story (Open Connect) borders [CDN.md](CDN.md) and is noted there.

### Serving-adjacent pipelines

- **[33.instagram](33.instagram/README.md)** — transcode pipeline produces renditions (resolutions,
  codecs, thumbnails, ABR ladder) written to the blob store **keyed immutably by content hash**
  ([OBJECT-STORAGE.md](OBJECT-STORAGE.md) pattern), then a metadata state transition marks "renditions
  done". A fixed photo ladder plus a video ABR ladder; the client queues uploads offline.
- **[35.tiktok](35.tiktok/README.md)** — a queue-driven transcode farm (~26K cores est.) builds the
  per-title ladder; the distinctive decision is **publishing before the ladder completes** — video
  metadata + first rendition publish, the farm registers the ladder afterwards, with first-segment
  priming so the exploration burst always has renditions ready. Same staged-publish pattern as 33's
  state transition, made explicit as a freshness-vs-complete-ladder trade-off.
- **[18.sora](18.sora/README.md)** — the encode step is the last node of a *generation* DAG; the
  case says its post-encode pool (upscale + H.264/AV1 renditions) is "exactly YouTube's transcode
  DAG" attached to a different source material. Publish is gated on watermark + C2PA provenance
  rather than QC.

### The client half

**[23.frontend-video-streaming](23.frontend-video-streaming/README.md)** — designs the playback stack
servers in 01/31/33/35 assume: an **ABR controller** that is a pure function of (throughput samples,
buffer level, current quality — EWMA estimator + buffer-based switching); a **segment scheduler** as
the only component that touches the network for media; and a **worker demuxer** parsing container
boxes off-thread (MSE/SourceBuffer, native HLS fallback where MSE doesn't exist). The server-side
cases' ladder is this controller's input; the seam between 23 and the ingest cases is the ladder
entry in video metadata, which is why 33/35 make "register ladder" an explicit publish step.

## The portable pattern

1. **Analysis pass first, DAG second.** A fast pre-analysis produces the artifact (keyframe map,
   shot-complexity map) that segments and ladder decisions both consume — one pass coordinates the
   whole pipeline (01, 31).
2. **Segments are the unit of work.** Retryable, idempotent, parallel; joins are explicit gates
   with QC before publish (01, 31, 33).
3. **The ladder is a cost curve, not a table.** Per-title/per-shot decisions trade operating
   simplicity for bytes-per-quality; codec upgrades shift the curve but only get applied to the
   popular tail where transcode cost amortizes (01, 31, 35).
4. **Renditions are immutable, content-addressed artifacts.** Written to the blob store keyed by
   hash; ladder registration is an explicit state transition (33, 35, 01).
5. **Publish ≠ ladder complete.** Every consumer-facing case stages publish earlier than the DAG
   finishes (first rendition, "renditions done" flag, QC-gated full set) — the freshness-vs-completeness
   trade-off, stated and chosen each time (33, 35, 31).
6. **Client ABR is the other half of the contract.** A pure-function controller + single network
   component; server-side ladder changes land as metadata, not protocol changes (23).
