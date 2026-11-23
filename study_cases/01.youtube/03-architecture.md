# 03 — High-level architecture

Two mostly independent planes: the **content plane** (the bytes) and the **control plane**
(metadata, API, discovery). The content plane is 99.99% of the traffic and almost never
touches your application servers.

## Diagram

```
 UPLOAD PATH (write)
 Creator ──► Upload/Edge POP ──► Ingest Store (object storage)
                    │                  │
                    │                  ▼
                    │           Transcoding Pipeline (async DAG, autoscaled workers)
                    │                  │  produces: renditions (480p..1080p+), thumbnails, captions,
                    │                  │            keyframes/diagnostics, audio-only
                    │                  ▼
                    │           Metadata + Storage Catalog DB  ("video X = these files")
                    │                  │
                    ▼                  ▼
              publish event ──► Search/Rec indexing ──► CDN pre-warm (best-effort)

 WATCH PATH (read)  — 90%+ of design effort
 Viewer ──► Global LB / DNS geo-route
              │
              ├─► Video App servers ──► Metadata DB / Cache (video meta, view counts, comments)
              │        │
              │        └─► picks playback file set + URLs
              │                    │
              ▼                    ▼
          CDN edge POP  ◄──────── signed URLs / HTTP-range segments
              │
              └─► cache miss ──► Mid-tier cache ──► Origin blob store (hot) ──► Cold archive
```

## Components

| Component | Job | Notes |
|---|---|---|
| Upload edge | accept + checksum + chunked resumable upload | terminates the TCP/HTTP close to user for speed |
| Ingest store | durable raw object storage | original file kept (re-encode later); 3× replic. or erasure-coded |
| Transcoding pipeline | raw → renditions | async DAG: split video into chunks (e.g., 2–10 s), transcode chunks in parallel, mux; see deep dives |
| Catalog DB | source of truth: video ↔ file set ↔ availability | strongly consistent; small rows, low QPS |
| Metadata services | video pages, channels, comments, like/view counters | eventually consistent counters (batch/aggregate) |
| Search / Recommendation | text search, "up next" | ML-heavy; can be fully degraded without failing watch |
| CDN | delivers ~all video bytes | the actual workhorse; origin shield to absorb back-to-back misses |
| Blob storage tiers | hot (SSD/HDD, first N days/hours by popularity) → cold/durable | lifecycle driven by the skew in popularity |

## Key data flows

**Publish**: upload → event "v=123 uploaded" → transcoding DAG finishes per-rendition →
catalog update "v=123 playable at set R" → fan out to search/rec index + invalidate/pre-warm.
Everything between upload and catalog update may fail/retry — idempotency keyed by
(video_id, rendition, segment_idx).

**Watch**: viewer hits app → app server calls catalog+metadata (cache-first) → returns
playback manifest with signed CDN URLs + rendition ladder → player downloads 4–10 s media
segments adaptively. The app server never streams video bytes.

## Where the money is

- ~1.4 EB/day through CDN (from estimates) → encoding bitrate policy and CDN hit ratio
  are the two levers that actually change P&L.
- Storage: keep originals (needed for future re-encodes) in cold storage, hot renditions
  sized by popularity curve.
