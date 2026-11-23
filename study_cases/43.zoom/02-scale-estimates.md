# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- ~150M meeting participants/day (design point; Zoom publicly reported ~300M daily meeting
  participants at its April 2020 peak — our steady-state number is deliberately half that).
- Mean meeting size ~10 participants (bimodal in reality: median 2–3, tail of 50–1000
  webinars; weighted mean ~10 with the tail included).
  → meetings/day ≈ 150M ÷ 10 = **15M meetings/day**.
- Mean meeting duration ~40 min → 150M × 40 = **6B participant-minutes/day**.
- At any instant, ~50% of participants have video on; screen share active in ~15% of meetings.
- Video simulcast top/middle/low layers ≈ 2.5 Mbps / 600 kbps / 150 kbps; sending all layers
  plus audio ≈ **~3 Mbps upstream per video sender**. Audio-only upstream ≈ 40 kbps mean
  (Opus ~32 kbps + overhead, with silence suppression pulling the average down).
- Per-receiver downstream: a receiver subscribes to ~3 video streams at ~1 Mbps *selected*
  layer each + ~9 co-participants' audio streams (every participant publishes audio always,
  01's rule) ≈ 3 + 0.36 ≈ **~3.4 Mbps mean**.

## Concurrency

- Average concurrent participants = 6B participant-min/day ÷ 1,440 min ≈ **4.2M**.
- Peak (work-hours concentration ≈ 3× average) ≈ **~12M concurrent**.
- Peak video senders = 12M × 50% ≈ 6M.

## Bandwidth (the estimate that shapes the design)

**Ingress (to SFU fleet):**

- Peak: 6M video senders × 3 Mbps = 18 Tbps; 6M audio-only × 0.04 Mbps ≈ 0.24 Tbps
  → **~18 Tbps peak ingress**.
- Average: 2.1M video × 3 + 2.1M × 0.04 ≈ **~6.4 Tbps**.

**Egress (from SFU fleet)**: each published stream is forwarded to the subscribers that
request it; at mean meeting size ~10 with selective subscription (gallery caps / active
speaker), the fleet-wide egress:ingress ratio lands ~2–3×:

- Peak egress ≈ 12M concurrent × 3.4 Mbps ≈ **~40 Tbps**.
- Check: 40 ÷ 18 ≈ 2.2 — consistent with a published 3 Mbps stream being *watched* by ~6 of
  the 9 other participants (10 receivers × 3 subscriptions ÷ ~5 video senders per meeting)
  at an average selected layer of ~1 Mbps, not the naive (N−1 = 9 receivers × full 3 Mbps)
  of full-mesh reception at top quality.
- SFU node egress ~15 Gbps usable → peak fleet ≈ 40 Tbps ÷ 15 Gbps ≈ **~3,000 media nodes**
  at peak; ~3–5K with regional headroom.

The *upload* insight survives every correction: 18 Tbps of peak ingress comes out of
consumer upload lines, which is exactly why the SFU exists (deep dives §1) and why the sender
encodes 3 fixed layers instead of personalizing per receiver.

## Signaling & meeting state

- Joins: 150M participant-joins/day ≈ 1.7K joins/s average, ~5K/s peak — small volume,
  latency-critical; join = auth + meeting admission + relay assignment + state sync.
- In-meeting control events (mute, raise hand, chat) ≈ bulk single-digit K events/s — trivial
  bandwidth, but must re-sync atomically on reconnect (deep dives §5).

## Storage & recording

- 2% of meetings record → 15M × 2% = **300K recordings/day**; mean 40 min at ~1.2 Mbps mixed
  composite ≈ 360 MB each → **~108 TB/day** (call it ~100 TB/day). Append to object storage
  (→ [07.blob-store](../07.blob-store/README.md)); retention per customer policy.
- Meeting metadata (schedule, participants, chat, controls log): ~150M participant events/day
  × ~500 B ≈ **~75 GB/day** — small; a normal transactional store handles it.

## Consistency check

- Meetings sanity: 15M meetings/day ≈ 10.4K meetings started/minute; at 40-min mean,
  meetings end at the rate they start, so a 40-minute rolling window holds ~10.4K × 40 ≈
  **416K live meetings** — consistent with 4.2M concurrent participants (4.2M ÷ 10 ≈ 420K
  live meetings at any instant ✓ round-trips).
- Egress math: at mean meeting size 10, a video sender's stream *could* go to 9 receivers ×
  3 Mbps = 27 Mbps of egress per 3 Mbps ingress — our 2.2× total ratio says receivers take
  top layer rarely and subscribe selectively; both statements are the same fact and both must
  hold in the design (receiver-driven selection is what buys that 10× gap — deep dives §2).
- Recording vs meeting volume: 2% recording share gives 300K recordings × ~40 min ≈ 200K
  recording-hours/day ≈ ~8K recording-hours in flight per wall-clock hour — a modest
  transcode/normalize farm (hundreds of cores), not a fleet-sizing problem (contrast [01.youtube](../01.youtube/README.md)'s 500
  hours/min of *ingest*).
