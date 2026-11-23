# Case 32 — Spotify

**The system**: stream music to ~600M+ monthly users (publicly reported through 2025, with
~250M subscribers — roughly 3 in 5 of them on the free tier), plus podcasts and audiobooks. Two things make it unlike every
video case in this corpus:

1. **The content is licensed, not owned.** A track exists on the platform only as long as
   rights deals say it does — by region, by track, sometimes by device tier. Takedowns and
   region gates are ordinary Tuesday events, not incidents.
2. **One track is played millions of times, in 3–4 minute bursts, and the user notices a slow
   start within a second.** The hard problem is not bytes-per-second (audio is 96–320 kbps,
   roughly 20–40× below video's ~3–6 Mbps delivered bitrate — see 02) — it's *time to first sample* on every single track transition,
   hundreds of times per user per day, plus playlists that must keep working offline.

**Why it's hard**:

- The catalog's *stored* size is trivially small (~1.8 PB for ~100M tracks in all bitrate
  renditions — see 02), which means the design problem is **delivery and latency**, not
  storage. This inverts the YouTube/Netflix economics and is worth internalizing.
- Everything user-visible is a *list of small items synced across devices*: playlists,
  queues, play history, offline saves. Conflict-free sync of a mutable list that a user edits
  on a phone while the desktop edits it too is a genuinely hard consistency problem.
- Offline playback must survive airplane mode for weeks while the licensed catalog changes
  underneath it — a license expiring on Thursday must pull content off devices without any
  user action, enforced the moment the device reconnects (streaming is denied instantly;
  see deep dives 2 and 6 for the two channels and their contractually asymmetric windows).
- Search spans tracks, artists, albums, playlists, podcasts and audiobooks with heavy
  personalization and prefix-as-you-type behavior, over ~100M items — and search results must
  respect the user's licensed region.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Reality-checked (Nov 2026)**: 02 anchors on publicly reported figures (MAU ~600M+/~250M+
subscribers; ~100M tracks; client bitrate tiers 96/160/320 kbps; the 10k-offline-tracks
device limit) and labels everything else as assumption. 04 adds real-world texture: the
client-side SQLite cache and prefetching, and the fact that Spotify mixes its own POPs with
third-party CDNs — described qualitatively, not with invented percentages.

**Patterns taught here** (see [case index](../README.md)): latency-first delivery of tiny
files, offline-first sync with server-authoritative licensing, region-gated catalogs,
CRDT-flavored playlist sync, multi-entity search ranking.

**Sibling cases**: [01.youtube](../01.youtube/README.md) (adaptive delivery at video scale),
[31.netflix](../31.netflix/README.md) (licensed catalog + recommendation, written in
parallel), [09.cdn](../09.cdn/README.md) (edge placement), [10.distributed-cache](../10.distributed-cache/README.md)
(metadata hot read path), [07.blob-store](../07.blob-store/README.md) (origin object storage).

**Try it first**: before reading 03, sketch a design answering: (1) how does the very first
sample of every track arrive in <100–300 ms of the tap, given the bytes come from a CDN
hundreds of ms away, (2) what happens on a device that's been offline for two weeks when the
license for half of its downloads expired, (3) how do two devices editing the same playlist
converge without a "merge conflict" dialog — music apps never show one.
