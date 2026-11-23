# 01 — Requirements

## Functional

**Core**

- **Prompt → clip**: text prompt (optionally plus a reference image) → a video clip,
  ~5–20 seconds by default (publicly reported Sora launch range; treat as a parameter,
  not a fact — the design works for any clip length).
- **Resolution/quality ladder**: the model produces the full-quality artifact; players get
  compressed renditions like any video file (see [02](02-scale-estimates.md) for sizes).
- **Job lifecycle visible to the user**: submitted → queued → generating → screening →
  ready. If generation takes minutes, the queue position and honest ETA are part of the
  product, not a status endpoint.
- **Edit/remix flow**: the user modifies a prompt (or reuses one with a tweak) and expects
  the *same* underlying world with the change applied — not a new unrelated clip. The
  system keeps per-chunk state on disk specifically to make this cheap (deep dive 5).
- **Library**: the user's generations persist; each clip is addressable, shareable
  (link/embed), and downloadable (with provenance watermark).
- **Feed**: published clips are browsable. Curation matters — the feed is the showcase of
  what the model can do, and its worst clip is everyone's screenshot.

**Content pipeline**

- **Prompt-side screening** before submission (cheap text-level checks).
- **Artifact-side screening** before release: every clip passes automated classification;
  borderline ones go to human review; hard-fail ones are blocked and logged. No clip is
  publicly visible without a screening verdict.
- **Provenance**: every published clip carries a visible+invisible watermark and a C2PA-style
  metadata trail (publicly announced approach for this class of systems). The watermark is
  part of the artifact contract — downstream tooling may rely on it.

## Non-functional

- **Latency SLO (generation)**: p50 ~1–5 min, p95 <15 min for a default clip (estimate;
  the honest framing is "minutes, and we never pretend otherwise"). The queue is visible,
  not hidden — users tolerate minutes for a 20-second clip they couldn't make otherwise.
- **Latency SLO (serving)**: library/feed playback normal CDN latencies (p99 <500ms to
  first frame). Serving must NEVER degrade because generation is congested — separate
  planes (deep dive 1).
- **Durability**: a submitted prompt is never silently lost; a job's checkpoints survive
  any single GPU/machine failure; finished artifacts are durably stored and never
  re-generated after a moderation pass has cleared them.
- **Cost governance**: global and per-user GPU budgets; the queue is the pressure valve.
  Abuse resistance: per-user rate limits priced in *expected GPU-minutes*, not requests
  (one 20s 1080p request is not equal to one storyboards request).
- **Availability**: generation is allowed to be best-effort under overload (queue grows);
  serving, library, and feed are must-stay-up. Generation fleet maintenance is routine —
  the design must never require a "quiet period" to roll GPUs.
- **Safety**: CSAM and violent extremist content hard-blocked at prompt stage and
  re-checked at artifact stage; identity/likeness abuse mitigations; audit trail on every
  blocked generation (regulatory + model-improvement feedback).
- **Determinism/versioning**: every job records the exact model version and generation
  parameters; re-runs pin that version even while newer models deploy fleet-wide.

## Scope decisions — explicitly OUT

- **Real-time / streaming generation**: anything interactive (live avatar, game-world
  synthesis) has a completely different latency budget and is out of scope.
- **Long-form (minute-scale+) narrative editing**: the editing suite for arranging many
  clips into a longer piece is out; we produce clips and a library.
- **Audio/music generation**: treated as a separate modality; the clip container is
  audio-ready but generating audio is not this system's job.
- **Model training and eval infrastructure**: assumed to exist; this system consumes
  model versions from a registry and reports quality telemetry back.
- **3D/world simulation outputs, asset exports** (meshes, camera files): the latent
  pipeline may contain the machinery for these as a research matter; the product surface here
  is video clips.
- **User-uploaded video processing** (upscaling, style transfer on footage): a different
  ingest model and a different abuse surface; out.
- **Multi-tenant GPU cloud**: the fleet is our fleet; selling spare capacity is out.
