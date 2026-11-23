# 02 — Scale estimates

All numbers are **rough estimates derived from stated assumptions** unless marked
"publicly reported". All byte math uses decimal SI units (1 MB = 10^6 B,
1 TB = 10^12 B, 1 PB = 10^15 B).
Public anchors used: Apple has publicly reported on the order of
**~2B active devices** (roughly, in recent years) and a **~$1T+/yr developer billings
& sales ecosystem** (Apple's stated 2022-ish figure covers billings plus commerce
beyond the store); third-party analysts have estimated App Store consumer spend in the
~$85–100B/yr range. The App Store catalog is publicly reported at ~1.8–2M apps.
Everything below is arithmetic on those anchors plus stated guesses.

## Assumptions

- A1: active devices ~2.0B; **~50% contact the store on a given day** → 1.0B active
  store clients/day.
- A2: catalog ~2M apps; average app ships a new version **every ~28 days**
  → 2M / 28 ≈ **71K new submitted versions/day** (this is the review pipeline load).
- A3: mean installed apps per device ~90 → total app-slots ≈ 2B × 90 = 180B.
  Auto-update is on (default) for ~85% of app-slots, but pickup is opportunistic
  (devices fetch in scheduled background windows), so only a fraction of each
  release's audience pulls the update promptly. The delivery math below assumes
  ~1.4 update events/day per active client → **~1.4B/day**, i.e. ~22% of the
  theoretical instant-pickup ceiling of 180B / 28d ≈ 6.4B updates/day (estimate);
  the rest land late (offline device, rarely-launched app, OS gating) or never.
- A4: delivered payload sizes: mean full package ~**150MB** (games multi-GB skew the
  mean; utility apps are <20MB); deltas cut the update path to a mean of **~40MB**
  (40/150 ≈ 27% of full) for installs that stay on the delta path; ~10% of update
  installs fall back to full download, so mean *delivered* update bytes ≈
  0.9 × 40MB + 0.1 × 150MB ≈ **51MB**.

## Delivery math (the dominant load)

**Delivery events/day.** First installs: analysts' public "downloads" figures for the
App Store are roughly ~8–10B/yr → ~10B/365 ≈ **27M first installs/day**. Updates:
per A3, only a fraction of releases are picked up promptly — take updates at
~**50×** first installs → 27M × 50 ≈ 1.35B ≈ **~1.4B update
installs/day**. Total ≈
**1.4B binary delivery events/day** (estimate). Sanity check the other way: 1B active
clients × ~1.5 install-or-update events/day ≈ 1.5B — consistent order of magnitude
(and far below the 6.4B/day instant-pickup ceiling of A3, as intended).

**Egress/day (client-facing, at CDN edges).**
- Updates: mean delivered 51MB (A4) × 1.4B ≈ **71 PB/day**. Split: delta path
  1.4B × 90% × 40MB ≈ **50 PB/day**; fallback full resends 1.4B × 10% × 150MB ≈
  **21 PB/day** — the fallback share is why delta quality is a first-order cost, not a
  nicety (deep dive 1).
- First installs: 27M × 150MB = **4 PB/day**.
- Total client-facing egress ≈ **~75 PB/day** (estimate; even if delta adoption were
  3× better and payload means halved, it stays in the ~30 PB/day class).

**Origin vs edge.** Popular apps have enormous cacheability: a top-chart game's update
is fetched by tens of millions of devices, so a well-populated POP cache absorbs it.
Estimate **~85–90% byte cache hit** on a mixed (heavy-tail) catalog: origin egress
≈ 75 PB × 12% ≈ **~9 PB/day**. Cache misses cluster in the tail: for a catalog of 2M
apps, a LRU edge cache keyed by (variant, version) holds popular items; the tail is
either pushed proactively to POPs (pre-warm before a known phased rollout) or origin-
served at low cost because tail volume is small.

**Update check-ins (the high-QPS, tiny-payload tier).** 1B clients/day check for
updates, spread over ~12 active hours + overnight batch runs (clients randomize their
check time by design): 1B / 43,200s ≈ **23K QPS baseline**, network-hour peak ~3–5×
→ **~100K QPS**. Each check-in is a few KB (manifest compare against installed
versions) — cheap, but it is a per-day per-device mandatory touch, so it's the
availability-critical tier: if check-ins fail, devices silently age out of updates.

## Storage

- **Binaries.** Keep last N≈4 versions per app for delta base selection and rollback:
  2M apps × 4 × 150MB ≈ **1.2 PB** raw. Universal + device-variant thinning multiplies
  by ~2–3× (variants share assets but differ in slices) → **~3 PB** in the blob store.
  Erasure-coded 1.4× → **~4 PB** physical. Small relative to delivery egress — storage
  is cheap; *delivery* is the budget line.
- **Deltas.** Eagerly generated against the last ~4 retained predecessor versions
  (deep dive 1): ~71K new versions/day × 4 bases × mean delta 40MB, keep ~30 days
  ≈ 71K × 30 × 4 × 40MB ≈ **340 TB**, evicted after the
  predecessor versions age out. (A single-base floor would be ~85 TB; the extra 4×
  buys direct-delta service for devices up to 4 versions behind.) Cheap per day, but
  recomputation on a broken chain is expensive (deep dive 1).
- **Catalog tables.** 2M apps × ~10 localized descriptions × few KB of metadata ≈ tens
  of GB — trivial.
- **Search index.** 2M documents, ~50 tokens/doc across name/keywords/description,
  multiple locales: an in-memory inverted index in the **tens of GB** — the whole
  active index fits on one big-memory fleet (deep dive 2).
- **Receipts/transactions.** Assume **~300M purchase/subscription events/day**
  (estimate anchored on ~$90B/yr spend, mean ticket ~$1, and subscription renewals
  being the bulk). Row ≈ 1KB → 300GB/day appended; keep hot 90 days ≈ **27 TB** hot +
  archival to object storage.

## QPS summary

| Tier | Avg/day | Peak (network hour / launch spike) |
|---|---|---|
| Update check-ins | 23K | ~100K |
| Search queries (assume ~400M/day) | ~4.6K | ~25K |
| Browse/detail views | ~50K | ~200K |
| Download/manifest requests | ~1.4B events spread; bursts at launch | ~500K during hit-update waves |
| Purchase/restore (300M/day) | ~3.5K | ~30K (holiday/gift-card spikes) |
| Review submissions | 71K/day ≈ 0.8/s | ~2/s (Monday developer-time clustering) |

None of the request tiers is individually huge — the store's hardness is bandwidth,
trust, and money, not request QPS.

## Review pipeline sizing

71K versions/day (A2). Assume reviewer hands-on time ~20 min median, 45 min p90 with
big apps; queue must clear ~24h. Reviewer-hours/day ≈ 71K × 0.4h ≈ **28K reviewer-hours**
→ with 7.5 productive hours ≈ **~3,800 reviewer-shift-days/day** (i.e. on the order of
a few thousand active reviewers worldwide — an estimate consistent with the ~24h SLA
Apple publicly describes). Automated pre-screens (static checks, metadata validation,
sandbox crash-detection runs) must cut the human-touch fraction; every automated pass
is a trust decision and belongs in the state machine's audit trail (deep dive 3).

## Event analytics for charts

Impressions + searches + downloads + transactions ≈ 1.4B delivery events + ~400M
searches + ~1B impressions + ~300M purchase events (02 receipts, above) ≈
**~3B events/day ≈ ~35K events/s average** (3.1B / 86,400s ≈ 36K) — a small
Kafka-tier stream; charts are hourly aggregates over (app, storefront, genre). Charts
are a *display* of aggregates, not a real-time system: minutes of lag are invisible.
