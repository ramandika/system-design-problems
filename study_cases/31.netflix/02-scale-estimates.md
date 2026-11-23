# 31.02 — Scale estimates

Assumptions (label as assumptions; redo the arithmetic if they change):

- **~300M paid households** — publicly reported (Netflix crossed ~300M memberships in early
  2025 reporting, then stopped quarterly subscriber disclosure). Treat as a floor.
- **~2 h watched/day per household** — Netflix has publicly stated roughly this level of
  member viewing. Anchor: → **~600M watch-hours/day** (300M × 2 h).
- **Catalog ~100K hours of content globally** (rough estimate: ~20K titles, majority
  episodic — a series title runs ~45 min/episode × ~6 episodes ≈ ~4.5 h, 20K × ~4.5 h ≈
  ~90K h ≈ ~100K h — this is ~4 orders of magnitude below YouTube's catalog hours).
- **Average delivered bitrate ~3.5 Mbps** — assumption spanning HD/4K mix on modern codecs
  (HEVC/AV1 with per-title optimization), minus the long tail of mobile.
- **Average session ~1.2 h**, concentrated in evening blocks.

### How these compare to public numbers (checked Nov 2026)

- The 300M-membership and 2 h/day figures are public; everything derived from them is our
  arithmetic, not company data.
- Netflix has for years shown up in Sandvine-style reports as one of the largest single
  contributors to global downstream traffic (double-digit % share in several years) —
  consistent with a rough ~1 EB/day egress, though the share is competitive pressure, not a
  published byte count.
- Open Connect is publicly described as ~thousands of appliance sites in ~175+ countries,
  with individual appliances in the tens-of-TB range (Netflix's public appliance specs:
  up to ~120 TB raw on the 2U storage appliance, ~60 TB on the smaller global appliance;
  exact fleet mix is an estimate). Netflix does not publish
  fleet aggregate storage; our inventory math below is an estimate consistent with those
  per-unit numbers.

## Read path (the big one)

**Bytes delivered/day**:

600M h/day × 3600 s/h × 3.5 Mbps ÷ 8 = 600M × 1.575 GB/h ≈ **~945 PB/day ≈ ~0.95 EB/day**

Per household: 2 h × 1.575 GB/h ≈ **~3 GB/day ≈ ~95 GB/month** — plausible for an
HD/4K-heavy household and a good sanity handle: any design that doubles per-title bitrate
doubles the biggest cost line in the business.

**Peak concurrency** (this determines fleet size, not the daily average):

Assume ~25% of daily watch-hours land in the 19:00–22:00 block:
0.25 × 600M h = 150M h ÷ 3 h = **~50M concurrent streams at peak**

Egress at peak: 50M × 3.5 Mbps ≈ **~175 Tbps** sustained for hours. This is why
prepositioning exists (deep dive 2): you cannot rent or burst this in a reactive cache; you
size it during the night.

Playback starts: 600M h ÷ 1.2 h/session = ~500M sessions/day. If ~25% of starts land in
the same evening block: ~125M starts over 3 h ≈ **~40M session starts/hour at peak** — each
pulling metadata, a manifest, and a DRM license.

## Non-video QPS (trivial by count, brutal by latency budget)

- Sessions/day ~500M × ~8 control-plane calls (profile, rows, metadata, manifest, license,
  logging) ≈ 4B requests/day ÷ 86,400 s ≈ **~46K req/s average, ~2–3× at peak**.
- These calls are on the critical path of *everything the member feels* (N1/N3 in 01): each
  one is a p99 latency and dependency-failure story, not a throughput story.

## Ingest (where this case inverts Case 01)

- New content: assume **~100–300 h/day of new masters** across originals, licensed churn,
  and localization variants (≈37–110K h/year at 100–300 h/day: 100×365 ≈ 37K, 300×365 ≈ 110K — against a ~100K h catalog with heavy annual
  churn — an estimate; Netflix adds hundreds of original titles/year plus licensing).
- Mezzanine bytes: 200 h/day × ~60 GB/h (4K master, uncompressed-ish) ≈ **~12 TB/day raw**
  — vs YouTube's ~6.5 **PB**/day ingest. Netflix ingest is ~**3.5 orders of magnitude** lower
  by hours (~3600×) and ~500× by bytes — the point stands, slightly rounded.
- Transcode effort: 200 h/day × ~25 rungs (ladder × codec variants, HDR included) ×
  ~0.3 machine-hours per output-hour (hardware encoders run at ~3× real-time) ≈
  **~1,500 machine-hours/day** —
  a fixed cluster, not a global burst market. The ingest problem is therefore
  **deadline-driven batch quality, not throughput** (deep dive 1).
- Localization is the multiplier people forget: 30+ dub languages × recording, QC, sync,
  packaging per title — human-latency work with a hard release date.

## Storage

Catalog renditions for ~100K h:

- Video ladder (all rungs, one codec): sum of rung bitrates ≈ ~40 Mbps → 40/8 × 3600 s =
  18 GB/h → × ~2.5 codecs (H.264 + HEVC + AV1 co-exist) ≈ **~45 GB/h** → 100K h ≈ **~4.5 PB**
- Dubs: 128 kbps audio × 30 languages ≈ 1.7 GB/h → **~0.2 PB**
- Masters (kept for re-encode/regeneration): 60 GB/h × 100K h ≈ **~6 PB**
- Total ≈ **~10 PB**, ×2–3 replication across regions ≈ **~20–30 PB of catalog storage**.

For perspective: **YouTube stores ~20 PB *per day*** (ingest + rendition copies; [Case 01, 02-scale-estimates](../01.youtube/02-scale-estimates.md));
Netflix's *entire replicated catalog* is about one YouTube day. Storage is a rounding
error; **delivery is the product**.

CDN fleet inventory (estimate, to be checked against deep dive 2): if the fleet holds the
most-viewed 70–80% of demand as prepositioned bytes — say the top ~5K h × ~45 GB/h ≈ ~200 TB
of hot renditions + working margin — that is small enough that each ISP site can hold its
local top titles on one tens-of-TB appliance (per-unit numbers above), and the fleet
aggregate (thousands of sites) is therefore tens of PB. Per-site *storage* is not the
binding constraint; per-site *serving capacity* and fill logistics are (deep dive 2).

## Ratios worth memorizing

- delivered bytes : ingested bytes ≈ 945 PB : 12 TB ≈ **~80,000:1 read:write** (YouTube is
  ~200:1). Netflix is not a write-heavy system that got famous; it is a broadcast network
  wearing the clothes of an app.
- ranking surface: a regional catalog of a few thousand titles (a licensed subset of the
  ~20K-title global catalog) at ~1–2K candidates per row — the ranking
  problem is *which* of a few thousand plausible choices for this member *now*, not
  *finding a needle in 500M items* (see deep dive 4, [Case 15](../15.ml-ranking-system/README.md)).
- storage trivial : egress dominant — flip of any UGC platform; this is the single most
  transferable insight of the case.

## Consistency check

- 600M h/day ÷ 300M households = 2 h/household/day ≈ 0.7 h/day per person in a 3-person
  household —
  plausible for TV time.
- ~50M peak concurrent over ~300M households ≈ 17% of homes streaming simultaneously at
  9 PM — plausible for a Friday.
- 0.95 EB/day is in the same order as publicly observed downstream-traffic shares — sanity,
  not proof.
