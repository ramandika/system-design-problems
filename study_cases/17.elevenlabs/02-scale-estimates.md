# 02 — Scale estimates

All numbers below are **our own rough estimates** for a system at ElevenLabs-like scale,
not actuals. Anchors from public reporting: ~$100M ARR and ~1M+ registered users were
publicly reported around 2025; we assume the system in this case has grown some margin
beyond that and label everything accordingly. Redo the arithmetic if assumptions change —
the point is the shape, and especially *which* quantity dominates the fleet.

## Assumptions

- ~10M registered users, ~1.5M DAU (estimate; consistent with publicly reported ~1M+ and
  growth margin).
- **Interactive (streaming) audio**: 400K of the DAU (est. ~27%) generate ~8 min of audio
  per day each (playground iteration, voice agents) → 3.2M audio-minutes/day. Round
  **~3M min/day**.
- **Batch (dubbing/audiobooks/projects)**: ~15K batch jobs/day × 30 min output each
  → **~0.45M min/day**.
- Total synthesis: **~3.5M audio-minutes/day** (this is the load-bearing number; the fleet
  is sized to it, not to QPS of tiny requests — see the conclusion).
- Audio format: PCM 44.1kHz/16-bit/mono for raw generation and batch outputs =
  44,100 × 2 bytes ≈ **88.2 KB/s ≈ 5.3 MB/min**. Interactive delivery commonly
  transcoded to ~160 kbps Opus/MP3 ≈ 20 KB/s ≈ **1.2 MB/min**.
- Speaking rate ~150 words/min, ~5.5 chars/word → **~825 chars per audio-minute**.

## The one number that matters: concurrent audio streams

3M min/day × 60 s = 180M audio-seconds/day ÷ 86,400 s =

**~2,100 concurrent audio streams on average**; assume an evening/UTC-day peak of ~3× →
**~6K concurrent streams at peak**.

This — not requests/second — sizes the GPU fleet. A "request" here is minutes long; QPS
would hide that.

## GPU fleet (interactive)

Interactive TTS runs a real-time factor (RTF) model: RTF = 30 means one GPU synthesizes
30 s of audio per second of wall time (an estimate: a modern GPU streams a mid-size TTS
model at roughly tens-of-times real time for a single voice; per-stream concurrency ≈ RTF
since each stream consumes 1× real time).

- GPU-hours of interactive synthesis/day: 180M s ÷ RTF 30 = 6M GPU-seconds ÷ 3600 ≈
  **1,667 GPU-hours/day equivalent of compute** — but the fleet must hold *concurrency*,
  not daily totals: 6K peak streams ÷ 30 streams/GPU ≈ **200 GPUs at peak**, ~70 average,
  with headroom, warm-voice-cache misses, and canaries → **~300–400 GPU interactive pool**
  (estimate).
- Voice switching eats into RTF: a per-customer voice resident on the GPU amortizes its
  load; a cold one pays a load penalty (deep-dive 3). Assume ~10–20% of effective
  capacity lost to model switching — the 1.5–2× headroom above covers it.

## GPU fleet (batch)

- 0.45M min/day = 7,500 audio-hours/day at RTF 30 → 7,500 ÷ 30 = **250 GPU-hours/day** ≈
  only **~10 GPUs running 24/7**.
- Batch is not a capacity problem; it is a *scheduler* problem. Give batch a burst pool of
  ~50 GPUs (≈5× the steady need): a 30-min-output job is only ~1 min of pure synth at
  RTF 30, so with queueing and concatenation jobs turn around in ~10–15 min, well inside
  the 30-min SLO. The interactive pool (~300–400) is ~7× the batch need.

## Throughput — text in, audio out

- **Text ingestion**: 3.5M min × 825 chars ≈ **~2.9B characters/day** ≈ 34K chars/s
  average. Trivial bandwidth (a few MB/s); the cost is inference, not bytes.
- **Audio egress**: interactive 3M min × 1.2 MB ≈ **3.6 TB/day**; batch deliverables as
  WAV 0.45M × 5.3 MB ≈ **2.4 TB/day**. ~6 TB/day total egress — meaningful but CDN-class,
  not exotic (compare ~10 PB/day of WhatsApp media — see
  [WhatsApp estimates](../03.whatsapp/02-scale-estimates.md)).
- **Requests**: ~800K websocket sessions/day (400K users × ~2) ≈ 9 connects/s avg;
  ~5M short HTTP calls/day (previews, non-streaming TTS, registry lookups) ≈ 58 rps avg,
  ~300 rps peak. The control plane could run on a laptop; all real load is streams.
- **Metering writes**: ~2.9B chars/day batched into per-session aggregates → a few hundred
  writes/s. Use streaming aggregation, not per-chunk rows.

## Storage: voice assets and outputs

- Cloned voices: assume 5% of 10M registers create clones → 500K private voices. Instant
 clone = embedding + consented reference recording (~5 MB compressed) → 500K × 5 MB ≈
  **2.5 TB**. Professional clones: 10% of those, ~50K × fine-tuned checkpoint ~500 MB ≈
  **~25 TB**. Voice asset store ≈ **~28 TB** total, small but *latency-critical* on read
  (500 MB checkpoint at 1 Gb/s takes ~4 s — why deep-dive 3 cares about tiering).
- Batch output retention: 2.4 TB/day × 30 days hot ≈ **72 TB**, then downshift to cold
  storage (S3-class archive tier). Interactive audio: not durable by default; assume
  ~1 day of history retention → ~3.6–16 TB/day of ephemeral objects (PCM vs compressed,
  interactive only — batch deliverables are counted separately above).

## Consistency checks

- 3.5M audio-min/day ÷ 1.5M DAU ≈ 2.3 min of generated audio per DAU — plausible for a
  mixed agent/creator population, sensitive to voice-agent adoption; if agents (not
  humans) dominate, sessions become far longer and the concurrent-stream number — not
  DAU — should be re-derived first.
- Fleet arithmetic cross-check: interactive pool 350 GPUs × 30 RTF = 10,500 streams of
  capacity vs ~6K peak demand → ~1.75× headroom. Deliberately not 1.2×: losing the tail
  here is audible to every user at once (deep-dive 4).
- Sanity vs publicly reported ARR: at est. ~$100M ARR and ~2.9B chars/day, blended revenue
  is roughly ~$0.1 per 1K chars — consistent, as an order of magnitude, with publicly
  listed self-serve character prices for high-quality TTS tiers. (Estimate, not pricing.)
