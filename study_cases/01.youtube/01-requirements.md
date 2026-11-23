# 01 — Requirements

## Functional

Core — must have for v1:

- **Upload**: creators upload video files (any of dozens of formats/containers, durations
  from seconds to many hours). Upload must be resumable — a 10 GB upload over flaky Wi-Fi
  must survive disconnects.
- **Process**: system transforms the uploaded file into a standard playable set.
- **Watch**: viewers stream video with seek (jump anywhere instantly), adjust quality,
  switch devices mid-video (phone → TV).
- **Metadata**: title, description, thumbnail, channel. Video pages must exist for any
  published video.
- **Discovery**: search by text; recommendations; subscriptions feed.

Out of scope for v1 (note them, don't design them): live streaming, comments, monetization,
copyright detection, playlists/analytics dashboards. Saying this is part of the design.

## Non-functional

| Property | Target | Why it forces the design |
|---|---|---|
| Upload latency | minutes, not seconds | transcoding cost dominates; users tolerate async publish |
| Time-to-first-frame (watch) | < 1–2 s | the single most-watched metric; drives CDN edge placement & encoding ladder |
| Seek latency | instant within buffered range, <1s otherwise | drives segment-based encoding (see deep dives) |
| Availability | watch: ~4 nines; search/recs can degrade | watching is the product; discovery can fail briefly without losing users |
| Consistency | eventually consistent for social bits (views, likes, comments counts); strongly consistent for the video ID ↔ playable files mapping | nobody notices a view counter that's 10 min stale; everyone notices a broken video |
| Cost | per-video storage + delivery cost must be tiny | at petabyte/day scale, even fractions of a cent per view are the whole business |
| Duplicates/abuse | spam upload floods must not starve legit uploads | triage before paying compute costs |

## The scope decisions that shape everything

1. **Processing is asynchronous.** "Published" is decoupled from "uploaded" — the system
   promises "your video will be live soon", not "live now". This buys the entire pipeline.
2. **We optimize reads, not writes.** Roughly ~400–500 hours of video uploaded per minute
   (widely circulated figure — no primary source publishes it; see 02 for the audit vs
   YouTube's only official stat, >20M videos/day) vs ~1 billion hours watched per day. Reads
   outnumber
   writes in bytes by orders of magnitude → design for read path efficiency first.
3. **One video file becomes N encoded renditions**, not one. Quality adapts at playback
   time (see deep dives).
