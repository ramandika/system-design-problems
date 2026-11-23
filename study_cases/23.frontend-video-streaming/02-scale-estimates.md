# 02 — Scale estimates

All numbers are rough estimates from stated assumptions unless marked "publicly
reported ~X". The point is internal consistency, not precision.

## Audience assumptions

- **DAU = 1B.** YouTube's logged-in audience is publicly reported ~2B+ monthly; we design
  for 1B daily actives (estimate).
- **Average watch time = 30 min/user/day** (rough estimate; mobile-first heavy apps
  publicly report more, but we average over all users).
- **Sessions = 3/user/day** → 3B playback sessions/day, many short.

## Concurrency and request rates

Average concurrent viewers = DAU × hours/day ÷ 24:

```
1e9 users × 0.5 h ÷ 24 h  ≈ 2.1e7 ≈ 20M concurrent (average)
```

Prime-time peak factor ~2× → **~40M concurrent (peak, estimate)**.

Segment cadence: VOD segments of 4s (DASH-typical; HLS default is 6s — see 04 §3 for the
low-latency variant). Each playing client fetches one media segment per segment duration:

```
segment fetch QPS = concurrent ÷ seg_duration
  average:  2.0e7 ÷ 4 s = 5M req/s
  peak:     4.0e7 ÷ 4 s = 10M req/s
```

- **Manifest/init QPS**: one manifest + one init segment per session start. If ~40% of the
  3B daily sessions land in a 4-hour prime window:
  `1.2e9 sessions ÷ (4 × 3600 s) ≈ 83k session-starts/s` → ~83k manifest QPS at peak,
  plus periodic refreshes for live manifests (every ~one segment duration while live).
- **Telemetry**: heartbeat + event stream every 5s while playing →
  `2.0e7 ÷ 5 = 4M events/s average`. Client batches; server design out of scope.
- **Live events**: a record-setting event draws publicly-reported ~millions of concurrent
  viewers (estimates vary by millions); at 1s part cadence (04 §3) that's `1e6 ÷ 1 = 1M
  part-requests/s` for that stream alone — a hot-key problem on the edge, cross-ref
  [Case 09 §5](../09.cdn/04-deep-dives.md).

## Bandwidth and bytes

Delivered bitrate: mixed renditions, weighted average **4 Mbps** (estimate; ladder runs
roughly 240p@0.4 Mbps … 4K@15–20 Mbps, cf. the ladder in
[Case 01 §2](../01.youtube/04-deep-dives.md)).

- **Segment size**: `4 Mbps × 4 s ÷ 8 = 2 MB` per segment at average quality
  (1080p@8 Mbps rung → 4 MB/segment; 4K rung → ~10 MB/4s).
- **Daily egress induced by clients**: watch-hours × bytes/hour:

```
watch-hours/day = 1e9 × 0.5 h                          = 5e8 h
bytes/hour      = 4 Mbps ÷ 8 = 0.5 MB/s × 3600 s       = 1.8 GB/h
egress/day      = 5e8 × 1.8 GB                         ≈ 0.9 EB/day
```

  ~1 exabyte/day flowing from edge to clients (estimate) — this load is *already* CDN
  traffic by design; the client's contribution is to request long-lived, cache-stable URLs
  so the edge hit rate stays high ([Case 09 §2](../09.cdn/04-deep-dives.md)).
- **Per-link headroom**: ABR needs measured link throughput ≥ ~1.8× playback bitrate for a
  confident up-switch (safety factor; see 04 §1) → a 4 Mbps viewer needs an ~7 Mbps link.

## Client memory budget (the side the case actually owns)

Forward buffer target 30s at 4 Mbps, held as compressed segments in MSE SourceBuffers:

```
30 s × 4 Mbps ÷ 8 = 15 MB   (video) + ~1–2 MB audio ≈ 17 MB buffered
```

Decoded frames in flight (compositor/decoder pipeline, YUV420): 1080p frame =
1920×1080×1.5 B ≈ 3.1 MB; ~10 frames pipeline depth ≈ 31 MB.
**Total player memory ≈ 50 MB typical, 100 MB cap** (estimate) — 4K playback and PiP
simultaneously are the cases that breach the cap; the buffer cap (04 §2) is what keeps it
bounded, and MSE `QuotaExceededError` is the enforcement backstop.

Seek: scrubbing past buffered data discards ranges via `remove()`; memory stays bounded by
forward-buffer target regardless of title length (a 10-hour video downloads ~zero extra).

## What this tells the design

1. Segment fetches dominate request volume (5–10M/s at scale) — every extra request
   (retries, parallel previews, manifest churn) is expensive in aggregate; batch and
   back-off (also visible in [Case 19](../19.frontend-chat-app/04-deep-dives.md)
   reconnect/backfill batching, same client-side discipline).
2. Startup requests are small in count (83k QPS) but they're the p50-experience-critical
   path — parallelize manifest + first segment, and let the first segment be
   cache-friendly or the edge absorbs a thundering herd of premiere traffic (Case 09 §3).
3. The client ships no bytes; its whole leverage is *when it asks* — buffer policy decides
   both the user's stall risk and the edge's load profile.
