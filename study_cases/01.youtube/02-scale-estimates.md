# 02 — Scale estimates

Assumptions (label as assumptions; adjust and redo the arithmetic if they change):

- ~500 h of video uploaded per minute → **~720,000 h of video/day** (~263M h/year: 720,000 × 365 ≈ 262.8M)
- ~1 billion hours watched per day
- Average watched session ~20 min (estimate) → 1B h/day ÷ (1/3 h per view) ≈ **3B views/day**.
  (Average upload length is not needed for any number here and is deliberately not assumed:
  an upload-length mean would have to stay >= the per-view watched mean, which public views/
  hours framings make hard to pin down — see the audit note on the units tension below.)
- Assume average ingest bitrate 20 Mbps (mixed smartphone/pro cameras)

### How these assumptions compare to public numbers (checked Nov 2026)

- **1B hours watched/day** — confirmed, but as a **2017 floor**: YouTube publicly announced
  "1 billion hours every day" in Feb 2017 (its VP of Engineering at the time; no newer
  official figure exists). The current number is almost certainly higher (~2.5B users
  publicly reported late 2025) — treat 1B h/day as the arithmetic anchor and the
  delivered/storage numbers below as lower bounds.
- **~1.4 EB/day delivered** — consistent with third-party measurement: YouTube was ~15% of
  global application traffic in 2021 (Sandvine; cited verbatim by the NSDI '23 "HALP"
  paper written with YouTube CDN engineers) and 15% × ~3.4 ZB/yr ≈ 1.4 EB/day. Sandvine's
  2023 report shows the share **declined** (~11.6% by mid-2022) — quote this as a
  2021-era estimate, not a current fact.
- **~500 h/min uploaded** — **unverifiable**: no primary source publishes hours-uploaded;
  the circulating number has no origin (2017-era reports said ~400 h/min). The only
  official stat is **>20M videos uploaded/day**. Note the unit tension: if all 20M videos
  were counted in 720K h/day the average upload would be ~2 min, which is implausibly
  short — so these two framings don't cover the same set; we keep the hours-based one for
  arithmetic and flag the mismatch rather than resolve it with invented numbers.
- **~20 PB/day stored** — pure estimate; no primary source publishes storage volume. Kept
  because it, not YouTube's actual number, decides the cold-tier conclusion.
- **Missing workload: Shorts — ~200B views/day publicly reported by YouTube's CEO
  (Jan 2026; 50B in Feb 2023 → 70B in 2024 → ~200B now, self-reported views = play
  starts, not hours or bytes).** Any model counting only long-form playback undercounts a
  fast-growing vertical stream with its own ladder, cache pattern, and transcode economics.

## Ingest & storage

**Raw upload bytes/day**:

720,000 h/day × 3600 s/h × 20 Mbps ÷ 8 = 720,000 × 3600 × 2.5 MB ≈ **6.5 PB/day raw ingest**

**Stored renditions multiply that.** Say each video is stored at ~5 renditions (240p…1080p+)
plus the original; renditions average roughly 0.4× the source bitrate (estimate — ladder
bitrates fall much faster than source bitrate, though 4K sources add higher rungs), so
5 × 0.4 + 1 (original) ≈ **~3× the source bytes**:

~6.5 PB × 3 ≈ **~20 PB/day stored** (before replication, cache copies, thumbnails)

With 3× replication on stored bytes: 20 PB/day × 365 ≈ 7.3 EB/year raw, × 3 ≈ **~22 EB/year
at rest**. This
number alone decides: you cannot store all of it on SSD; cold-storage tiering is mandatory
(see deep dives).

**Compute**: transcoding at roughly real-time-per-rendition speed (1s of work per 1s of
video per rendition, with modern encoders maybe 0.3× real time): 720,000 h of video ×
5 renditions ≈ 3.6M transcoding-hours/day. Even at 0.3× real-time ≈ 1.1M machine-hours/day.
This is clearly a distributed, autoscaling, spot-capacity-priced workload — not one big box —
and in the real world since ~2018 it has also become a **custom-silicon** workload
(Google's Argos transcoder ASIC; see [deep dive 7](04-deep-dives.md)): 1.1M machine-hours/day
is precisely the kind of fixed-function workload an ASIC kills.

## Read path (the big one)

**Bytes delivered/day** — the dominant cost:

1B h/day watched × 3600 s/h × average delivery bitrate. Average adaptive bitrate is maybe
3 Mbps after ladders + compression → 1e9 × 3600 × 3 Mbps ÷ 8 ≈ **~1.4 EB/day** (1.4 million TB).

At any plausible CDN per-GB pricing this is astronomically the #1 cost line. Consequence:
every design choice on the read path (encoding bitrate, CDN caching, skipping bytes) is a
business decision, not just an engineering one.

**QPS (non-video)**:

- Page views: ~3B views/day (from above) ≈ ~3B watch-page loads × ~10 requests
  (metadata, related videos, comment counts, ...) ≈ 3e10 requests/day ÷ 86,400 s ≈
  ~350K req/s average (estimate — assumes each view is one page load), peaks 5–10×. Large
  for app servers but cache-first reads make it tractable — the *bytes* remain the problem,
  not the QPS.
- This is a common trap: people estimate QPS and think "shard a Postgres", but the real load
  is throughput of bytes via a CDN (~200:1 by volume — see ratios below), which changes the
  architecture entirely.

## Ratios worth memorizing

- bytes delivered : bytes ingested ≈ 1.4 EB : 6.5 PB ≈ **~200:1** — read-heavy in bytes
- transcoding compute ≈ 1–4× real time per rendition set
- views are extremely skewed: top ~1% of videos likely take a majority of watch hours →
  heavy-tail means a small hot set + huge cold tail (drives the storage & cache design)

## Consistency check

- 1B h/day over ~1.5B-ish viewers (2017-era monthly user count, the era when the 1B h/day
  floor was announced) ≈ 40 min/user/day — plausible for watch-heavy MAUs.
- 1B watch hours ÷ 20 min avg view = ~3B views/day — consistent with the ~3B views/day assumed above.
