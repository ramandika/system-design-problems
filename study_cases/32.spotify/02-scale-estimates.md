# 02 — Scale estimates

Assumptions (label as assumptions; adjust and redo the arithmetic if they change):

Units: decimal SI throughout (1 KB = 10^3 B, 1 GB = 10^9 B, 1 PB = 10^15 B) — so a bitrate of
160 kbps ÷ 8 is exactly 20 KB/s. All arithmetic below uses this base.

- **Users**: ~600M monthly actives, ~250M paid subscribers (publicly reported 2024–2025,
  roughly). Assume **200M DAU**, ~55% paid.
- **Catalog**: ~100M tracks (publicly stated "100M+ tracks"), avg 3.5 min (210 s) per track.
  Plus ~6M podcast shows (publicly reported roughly ~6M+), assume ~20M actively stored
  episodes at ~45 min avg.
- **Listening**: avg user listens **~1.5 h/day** → 200M × 1.5 = **300M listening hours/day**.
- **Bitrate tiers** (publicly documented client settings, Ogg Vorbis on mobile/desktop):
  low 96 kbps, normal 160 kbps, high 320 kbps. Web uses AAC (assumed ~128 kbps). Assume
  blended **160 kbps** across all plays.

### How these assumptions compare to public numbers (checked Nov 2026)

- 600–700M MAU / ~250M subs — confirmed range from company reporting through 2025; the exact
  DAU split is my assumption (MAU→DAU conversion for music apps is usually higher than social,
  since listening is habitual).
- 96/160/320 kbps tiers (mobile/desktop, Ogg Vorbis) — publicly documented client quality
  settings. The web player's AAC bitrate is publicly reported up to ~256 kbps at "high", so
  the ~128 kbps web figure I use below is an assumption, not a published setting.
- 10,000-track offline limit per device, premium-only downloads — publicly documented.
- 100M tracks — publicly stated by the company. Podcast episode count/length mix — estimate.
- Listening hours/day: no public per-platform number; 1.5 h/day is a habit-based estimate.
  Everything below scales linearly with it — recompute if you disagree with the anchor.

## Bandwidth (the dominant cost)

**Streaming egress/day**:

300M h/day × 3600 s/h × 160 kbps ÷ 8 = 300M × 3600 × 20 KB/s = **21.6 PB/day**

Peak egress: 21.6 PB/day ÷ 86,400 s ≈ **250 GB/s average, ~500–750 GB/s peak**

Sanity check against siblings: that's ~1/60th of the ~1.4 EB/day scale estimated for YouTube
(../01.youtube/02-scale-estimates.md). The gap decomposes cleanly: audio's blended 160 kbps is
~20× below YouTube's ~3 Mbps delivered bitrate, and YouTube serves ~3× more listening hours/day
(~1B vs ~300M) — ~20 × ~3 ≈ 65×, matching the EB-vs-PB ratio. **This is the single
most important number in the case**: it says bytes are not the core difficulty, latency is.

**Free-tier share** (assumption): ~45% of plays carry ads; ad creative adds a trivial byte volume but a
*latency* coupling (an ad must also start on time) → see 03 data flows.

## Per-track start rates

Track starts/day = listening hours ÷ track length:

300M h/day × (3600 s/h ÷ 210 s/track) ≈ **5.1B track starts/day ≈ 59K/s average**

Peak ×2–3 (evenings, commute) → **~120–180K track starts/s**. Each start costs a license
check + catalog metadata fetch + playback URL issuance — so this QPS, not bytes/s, is what
sizes the control plane. Even at ~10 KB of metadata+auth per start: 5.1B × 10 KB ≈ 51 TB/day
≈ 0.6 GB/s average (~1–2 GB/s at peak) of *metadata* traffic — cacheable, see 10.distributed-cache.

## Storage

**Encoded audio catalog** — 100M tracks × 210 s, per rendition:

| Rendition | Size per track | ×100M tracks |
|---|---|---|
| 96 kbps Ogg | 210 × 12 KB = 2.5 MB | 252 TB |
| 160 kbps Ogg | 210 × 20 KB = 4.2 MB | 420 TB |
| 320 kbps Ogg | 210 × 40 KB = 8.4 MB | 840 TB |
| AAC ~128 kbps (web) | 210 × 16 KB = 3.4 MB | 336 TB |

**~1.8 PB total for the entire music catalog in 4 renditions** — roughly a fifth of
Netflix's ~10 PB full catalog (see ../31.netflix/02-scale-estimates.md). The counterintuitive
takeaway: music streaming is
a **small-storage, high-request-rate** system. Storage choices barely matter for cost;
request fan-out and edge placement matter enormously.

**Podcasts** (estimate): 20M actively stored episodes × 45 min × ~64 kbps mono
= 20M × 2700 s × 8 KB/s = 20M × ~21.6 MB ≈ **430 TB** — a quarter of the music catalog's
encoded bytes, at a fraction of the plays.
Label as estimate; podcast cataloging (which back-catalogue stays stored vs. re-ingested on
demand) decides the real number.

**Metadata**: 100M tracks × ~2 KB structured metadata + credits + rights + popularity =
**~200–400 GB** — small enough that the working set (the top few % of the catalog by
plays) fits in RAM everywhere, which is why search and browse can be edge-served.

**User data**: 200M users × ~500 playlists-followed/saved items × ~100 B ≈ ~10 TB *stored*;
play history appends at the track-start rate (5.1B/day × ~100 B ≈ **~500 GB/day**);
trivially small but **append-only and
write-heavy per user** — a scale problem of writes-per-second-per-key, not bytes.

## Offline sync traffic

Assume ~30M premium users run offline mode; they add/refresh ~50 tracks/user/month
(playlist churn + rotating saves), and each device fetches at its 320 kbps tier:

30M × 50 tracks/month × 8.4 MB = 12.6 PB/month ≈ **420 TB/day ≈ 0.4 PB/day**

About 2% of streaming egress — but it's *bursty* (one user trip = one multi-GB binge) and
it must keep working when the user is in roaming/expensive networks, which is a client-side
scheduling problem more than a server one.

**Client storage per device**, for reference: the 10k-track cap at 320 kbps = 10,000 ×
8.4 MB = **84 GB worst case** (plus a metadata DB of ~10s of MB). Realistic saves
(~1–3k tracks) land at 8–25 GB — comfortable on modern phones, and the cap exists precisely
so worst-case provisioning is predictable.

## Search load

Assume ~25% of DAU searches, ~3 queries/day → 150M queries/day ≈ **1.7K/s average,
~5K/s peak**; each query fans out to typeahead (as-you-type) 5–10 sub-requests. Cheap in
absolute terms; the difficulty is ranking quality and prefix latency, not QPS (deep dive 5).
