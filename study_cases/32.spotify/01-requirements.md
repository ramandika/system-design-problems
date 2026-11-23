# 01 — Requirements

## Functional

Core — must have for v1:

- **Stream**: play any licensed track on demand, with seek, gapless transitions between
  consecutive tracks in a playlist/album, volume normalization, and quality tiers
  (low/normal/high ≈ 96/160/320 kbps).
- **Catalog**: artists, albums, tracks, playlists (curated + user), podcasts/episodes,
  audiobooks. Everything browsable and searchable; "start playing" is one tap from anywhere.
- **Library & playlists**: users create/edit/reorder playlists, save albums/tracks, follow
  artists. A playlist is a mutable ordered list edited from multiple devices concurrently.
- **Offline**: paid tier downloads tracks to device storage and plays them with no network.
  This is the flagship paid feature — it must survive long offline periods.
- **Player state sync**: queue, "currently playing", shuffle/repeat mode, and progress resume
  ("continue listening on your speaker") across devices.
- **Social-adjacent**: follow users, share playlists, collaborative playlists (multiple
  editors), a lightweight activity feed.

Out of scope for v1 (note them, don't design them): payments/subscription billing, royalty
reporting and label-contract reconciliation (see deep dive 6 for the constraint they impose),
lyrics, live audio rooms, video podcasts, upload-by-users (there is no creator side — the
catalog arrives via distributor feeds, which makes ingest a batch/ETL problem, not a scaling
one), music-identification (Shazam-style).

## Non-functional

| Property | Target | Why it forces the design |
|---|---|---|
| Time-to-first-sample | prefetched/cached (the common case — 03's flow 1 serves most playlist-chain tracks from local cache): < 100 ms, with 04 deep dive 1's estimate at tens of ms; cold network fetch: < 300 ms typical, < 1 s SLO cap (04 deep dive 1 budgets 150–300 ms) | audio is a latency product: a tap that doesn't produce sound feels broken. This is the single metric that shapes the player architecture |
| Gapless playback | no audible gap between consecutive tracks | requires the next track's bytes to be in memory before the current one ends — drives prefetch depth and buffering policy, not just the CDN |
| Seek latency | < 500 ms | users scrub constantly; drives range-addressable, chunk-indexed files (same conclusion as video, see ../01.youtube/04-deep-dives.md) |
| Availability | playback: ~4 nines from the *edge*; free tier ads/metadata: ~3 nines tolerable | a playback failure is a support ticket; a stale like-count is not |
| Offline resilience | downloads playable fully offline up to ~30 days (the publicly documented reconnect cadence — beyond it, the device must revalidate on its next connection); licensing revocation: tombstone deployed and enforced on streaming + at next reconnection within ~24–72 h of takedown | dictated by label contracts (offline window) and by legal exposure (takedown SLA) — both are contractual numbers, not engineering taste (see 04 deep dives 2 and 6: streaming enforcement is instantaneous, fully-offline enforcement is by design deferred to reconnection) |
| Sync latency (playlists/queue) | push update to other devices in < 1 s when they're online | collaborative playlists and queue-following are real-time-feeling features |
| Search latency | autocomplete as-you-type < 100 ms p99 served; full ranked results < 300 ms | typing-latency thresholds; drives prefix index design |
| Consistency | **catalog is server-authoritative** (a takedown removes access everywhere, promptly); playlists eventually consistent with a merge policy; play counts/history append-only | the licensed catalog can't be "eventually" removed; but a mutable user list can be merged without dialogs |
| Codec/file format | one audio format family per platform class (e.g., Ogg Vorbis on mobile/desktop, AAC on web where licensing differs) | codec licensing and device decoder support decide the rendition matrix as much as bitrate does |
| Cost | delivered GB per listening hour must be cents-level | audio bitrate is low, so cost is dominated by CDN egress and per-track-start request overhead, not bytes — see 02 |

## The scope decisions that shape everything

1. **There is no user-generated content.** The catalog comes in via label/distributor feeds.
   ingest is therefore a **correctness problem** (matches, metadata quality, licensing flags)
   rather than a throughput problem — the exact inverse of 01.youtube.
2. **The client is trusted to cache aggressively but never trusted on licensing.** The device
   caches metadata and audio locally (SQLite + audio files, publicly documented), but every
   playback right is checked server-side and offline files carry an expiry the client must
   honor. Deep dive 2.
3. **We optimize "the next 3 minutes", not "the next hour".** A session is a chain of short
   tracks. The design wins are per-track start latency and prefetch pipelining, not
   long-tail bitrate ladder efficiency (that's the 31.netflix / 01.youtube problem).
4. **Free tier is a first-class citizen**, not a downgrade: mobile free tier historically
   constrains on-demand behavior (shuffle-first on phones, ads), which changes the read path
   (ad delivery becomes a playback-start dependency) without changing the architecture.
