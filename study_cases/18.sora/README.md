# Case 18 — Sora (video generation)

**The system**: a user types a prompt; minutes later they get a multi-second (up to ~20s
at launch, per OpenAI's publicly reported feature set) generated video clip. Behind that:
a GPU fleet where one clip burns *minutes* of accelerator time, a job orchestrator that
resumes half-finished generations, moderation that gates novel content before release,
and delivery of the finished artifact.

**Why it's hard**:

- The cost asymmetry is total. A *served* video is milliseconds of CDN time and pennies of
  bytes; a *generated* clip is minutes of H100-class GPU time worth ~$0.09–0.15 in raw
  compute (estimate — see [02](02-scale-estimates.md)). The system's real load is GPU-seconds, not
  QPS. Every architectural choice is downstream of "generation is ~10^3× more expensive
  than serving."
- Generation is a long, fallible GPU job (minutes), on the same fleet as everything else
  the company runs. Jobs fail mid-flight; a naive design regenerates from scratch and pays
  twice. The pipeline must be a resumable DAG of checkpointed chunks, not one monolithic
  kernel call.
- Every clip is content that has *never existed before*. Classification and takedown
  patterns built for user uploads do not transfer directly: there is no fingerprint to
  match against, so screening has to run on the generated artifact itself, before release,
  at generation throughput.
- Users iterate: they edit a prompt and want the change applied, not a brand-new random
  clip. Edit = regenerate vs edit = patch is a genuine design fork with cost and quality
  consequences.

**Reading order**: [01-requirements](01-requirements.md) →
[02-scale-estimates](02-scale-estimates.md) → [03-architecture](03-architecture.md) →
[04-deep-dives](04-deep-dives.md).

**Patterns taught here**: cost-driven scheduling (queue, priorities, budgets), diffusion
generation as a parallel chunk DAG with checkpoints, at-least-once GPU job execution with
idempotent artifacts, GPU-cost-aware caching/lifecycle, pre-release moderation of novel
content, watermarking/provenance.

**Compare with**: [YouTube](../01.youtube/README.md) — the *other* side of a video
platform (ingest → transcode → serve; there bytes dominate, GPU time is nearly free). Also
the chunked-DAG pattern from [YouTube deep dives](../01.youtube/04-deep-dives.md) and the
retry/idempotency discipline from
[WhatsApp deep dives](../03.whatsapp/04-deep-dives.md) both reappear here, transposed.

**Try it first**: before reading 03, answer: (1) your 500M-user DAU and 1M DAU products
have the same QPS for *this* system — why, and what is the real scaling knob? (2) a GPU
dies at minute 6 of a 10-minute job — what did you save, and what would it cost to have
saved nothing? (3) where in the pipeline does moderation sit, and what does that position
cost you in latency?
