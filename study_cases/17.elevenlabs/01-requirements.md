# 01 — Requirements

## Users

- **Voice-agent developers** (API): their software streams text in and must play audio out
  in near-real time. Our SLO is their user's ear.
- **Interactive UI users**: playground / projects UI, previewing and iteratively editing
  an audio artifact.
- **Batch content pipelines**: audiobooks, dubbing of videos, long-form narration — job
  granularity of minutes to hours of output audio.
- **Voice owners / cloning customers**: registering private voices from consented
  recordings, managing who can use them.

## Functional requirements

**F1 — Streaming TTS (interactive path)**. Client sends text over a WebSocket (token
deltas as they are produced, or full text) and receives audio chunks back starting within
a bounded initial latency, then continuously at ≥1× real time. Multiple output formats
(PCM, MP3, Opus) and sample rates. Cancellable mid-utterance (interruption is normal for
voice agents).

**F2 — Batch rendering (asynchronous path)**. Submit a long document/script as a job
(whole-file TTS, dubbing: per-segment translation + synthesis + alignment). Job is
queued, synthed at best effort, output stored, downloadable when done. Status polling and
webhooks. Jobs are resumable — a partial fleet failure resumes, never restarts, a job.

**F3 — Voice registry**. Named voices with metadata (language, style, owner, sharing
scope): (a) curated public library voices, (b) per-customer private voices, (c) shared
within a customer's workspace. Every synthesis request names a voice; the lookup and
authorization on it is on the hot path.

**F4 — Voice cloning / per-voice training**. Two tiers: **instant clone** — a
prompt-based model conditioned on a short consented recording (minutes of audio), no
training job, usable immediately; **professional clone** — a longer recording and a
fine-tuning job that produces per-customer model weights. Both produce registry entries
usable by F1/F2.

**F5 — Consent & abuse controls**. Recorded consent statement (a specific spoken
phrase) attached to every cloned voice; refusal logic at synthesis time (notably
imitating voices whose owner has not consented); audio watermarking; detection of
attempts to clone protected/public-figure voices. see deep-dive 5.

**F6 — Metering**. Usage accounting per character/credit with per-API-key quotas,
self-serve spend limits. Metering must be exact-enough for billing (reconciled, not
best-effort).

## Non-functional requirements

| Property | Target (our design target, an estimate-anchored SLO) |
|---|---|
| First-audio-byte latency (streaming) | p95 ≤ ~500ms from connection establishment to first playable audio; p99 ≤ ~1s |
| Steady-state stream speed | audio produced ≥1.2× real time continuously; no audible gap at any point after the first byte |
| Batch turnaround | job of 60 min output audio completes in ≤ ~30 min at p95 (throughput-SLO, ~2×+ real time end-to-end incl. queueing) |
| Interactive availability | ≥99.9%; explicit goal: batch NEVER degrades interactive latency |
| Batch durability | no job lost: restartable, checkpointed at segment boundaries; outputs retained 30 days hot, then cold |
| Multi-tenant isolation | a customer's voice weights are readable only by that customer's requests; audio and weights never cross tenants |
| Cloning consent integrity | no synthesis through a private voice without the owner's authorization on record; consent records are append-only/audited |

## Scope decisions

**In scope**: streaming + batch synthesis, voice registry, both cloning tiers, dubbing of
long documents, metering, consent/abuse control plane. Both API and product UI (they share
the same synthesis stack — a product requirement that forces the "two workloads, one
fleet" problem).

**Explicitly OUT of scope**:

- **Speech-to-text / transcription**: the case is production of speech, not consumption;
  an ASR stack shares almost zero hot-path components with TTS.
- **Agent orchestration logic** (dialogue policy, LLM turn-taking, telephony trunks): we
  are the synthesis *backend* under someone's agent; their websocket-to-PBX plumbing is
  theirs. This boundary justifies one clean interface: text deltas in, audio out.
- **Video processing / lip-sync rendering**: dubbing here produces an aligned audio track;
  muxing it into video is the customer's pipeline.
- **Music generation**: different model family entirely.
- **Client-side edge inference**: all synthesis runs in our datacenters; we design around
  that (this is what makes the GPU fleet a real capacity problem).

## The central tension, stated once

Requirements F1 and F2 pull the fleet in opposite directions (tail latency vs $/hour of
output), and F4 makes the fleet's inputs *non-uniform* (thousands of distinct per-customer
models, not one shared model). Almost every hard decision in
[03](03-architecture.md)/[04](04-deep-dives.md) is a way of keeping those three from
fighting each other in the same process.
