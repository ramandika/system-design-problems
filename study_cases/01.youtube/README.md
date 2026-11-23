# Case 01 — YouTube

**The system**: users upload video; billions of people watch it. That's the whole product,
and everything hard follows from those two sides having wildly different scale (millions of
uploads/day vs billions of views/day).

**Why it's hard**:

- Video is the worst-case payload: 100–1000× the bytes of any web page, so bandwidth and
  storage costs dominate every other decision.
- A video can't be served as-is: it must be transcoded into multiple resolutions/bitrates,
  which is expensive compute and can take longer than the video itself.
- One popular video = thundering herd on the same bytes millions of times → you must not
  serve viewers from origin, ever, if you can help it.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Reality-checked (Nov 2026)**: the scale assumptions in 02 are annotated against public
sources (1B h/day = 2017 YouTube-published floor; ~1.4 EB/day delivered = 2021 Sandvine
-derived estimate; upload-hours & storage unverifiable), and 04 adds validated real-world
details: the two-pass popularity-driven transcode pipeline, the Argos transcoder ASIC,
YouTube's tiered CDN shape, and its Bigtable/Vitess metadata stores.

**Patterns taught here** (see [case index](../README.md)):
ingest/pipeline design, asynchronous DAG processing, CDN delivery, adaptive bitrate,
hot vs cold storage.

**Try it first**: before reading 03, sketch a design answering: (1) what happens between a
user tapping "publish" and the video being watchable, (2) where do viewers' bytes physically
come from, (3) how does a video play smoothly on a 4 Mbps phone and a 50 Mbps fiber line
from the same source file?
