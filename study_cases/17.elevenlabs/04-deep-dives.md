# 04 — Deep dives

## 1. Streaming synthesis with sub-second first audio (chunking vs full render)

The user's ear is the deadline: after first audio arrives, later audio must never
catch up late. The synthesis model autoregressively produces a waveform; the question is
*how much text turns into how much audio, when*.

**Option A — full render** (synthesize the whole utterance, ship one file):
- wins: best quality/coherence — the model conditions on the full text; prosody across
  the whole passage is consistent; trivial transport (one HTTP response); trivial
  metering and retries.
- loses: first audio ≈ full render time (seconds for a paragraph — unusable for any
  interactive use); cancels throw away 100% of the GPU work; text must be complete
  before starting (a voice agent fed by an LLM cannot start until the LLM finishes).

**Option B — incrementally render at fixed token/audio-window boundaries** (e.g., every
N tokens or every ~1s of audio, regardless of punctuation):
- wins: absolute minimum first-byte latency; steady, predictable GPU feed.
- loses: synthesis fragments lack prosodic context → unnatural pitch/rhythm breaks at
  chunk boundaries (audible); sentence across a boundary says "wonder-" then "ful" with
  a seam; more, smaller inference calls → worse GPU utilization (more per-call overhead,
  worse batching).

**Option C (chosen) — sentence/clause-boundary chunking with a small coalescing buffer**:
hold incoming text deltas until a boundary is detected (sentence end, comma, or a
character cap as an escape hatch for run-on input), then synthesize that chunk with
*slightly* extended context (previous chunk's last few words conditioning this one).
- wins: prosody is consistent within and (mostly) across chunks; first-byte = time to
  detect the first boundary plus one chunk render — well under our ~500ms p95 for
  mid-sentence starts because the cap forces a chunk even inside a long clause; partial
  work is only wasted if a cancel arrives mid-chunk; chunks batch well on the GPU.
- loses: adds a small initial latency for boundary detection vs Option B; requires a
  per-language boundary detector on the hot path (more places to be wrong, e.g., decimals,
  abbreviations, CJK where "sentence" is not whitespace-findable); quality is a bit below
  full render — acceptable, measured vs Option A by subjective listening tests.

- **Trade-off stated**: we spend a few hundred ms of first-byte latency (A → C) to buy
  prosody quality, and accept slightly worse per-chunk quality to avoid the multi-second
  gaps of Option A. A voice agent's users forgive a slightly flat sentence; nobody
  forgives a two-second silence mid-answer.
- **Failure modes**: boundary detector mis-splits ("Dr." → chunk break mid-name) →
  prosody glitch; mitigation: abbreviation-aware tokenizer plus the character cap as a
  safety net. Boundary starvation (input that never terminates a sentence, e.g., streamed
  JSON) → the cap forces a chunk, no one waits forever. Chunk ordering: controller stamps
  sequence numbers; a slow GPU retry re-emits at the same sequence, client dedupes — same
  wire-retry/dedup discipline as [WhatsApp's delivery protocol](../03.whatsapp/04-deep-dives.md).

## 2. WebSocket audio transport and the jitter buffer

Interactive audio rides a long-lived websocket: text deltas in, audio chunks out, for
minutes per session. Connection mechanics mirror WhatsApp's connection layer
(see [deep dives 2](../03.whatsapp/04-deep-dives.md)): gateways hold the sockets,
a lightweight registry routes a user to their gateway, reconnect storms are handled with
client backoff and TLS resumption. What's *different* here is the media:

- **Timestamps on every chunk**: audio is playback-constrained, not arrival-constrained.
  Each chunk carries media time (seconds of audio it represents) and wall-clock send time.
  Reordering on the wire doesn't matter — late chunks matter.

**Where to buffer: server, client, or nothing?**

- **Option A — no buffer, play on arrival**: absolute minimum latency; any packet hiccup
  or inference stall → audible gap/silence, then audio resumes mid-word. Interactive
  systems cannot tolerate this at all.
- **Option B — client jitter buffer, fixed ~1–2s**: absorbs bursts and 10–20% RTF dips
  completely; playback is smooth. Loses: adds that much latency (fine *after* the first
  byte, but the first byte pays it too).
- **Option C (chosen) — adaptive client jitter buffer**: start near-empty (fast first
  audio), grow only if arrival jitter is observed, capped ~500ms; server side keeps
  *no* playback buffer, it just streams as fast as inference allows.

- **Trade-off stated**: we hold ~0–500ms of playback delay in the client to guarantee
  uninterrupted audio, instead of 0 delay with audible gaps. The first byte is exempt
  (buffer starts small) because first-byte latency is its own SLO. A fixed 1–2s buffer
  would buy nothing — once audio is flowing faster than real time, 500ms absorbs
  everything, and extra buffering hurts agent turn-taking.
- **Backpressure matters more than buffering on the send side**: if the GPU falls behind
  1× real time, the correct behavior is *pause text intake / drain the stream faster, not
  buffer unbounded audio* — audio buffered for 10s is indistinguishable from a dropped
  call, so bound it: a stream whose output buffer exceeds a few seconds of media is
  errored and re-established. Better to fail one stream loudly than to have every stream
  quietly drift.
- **Interruption (barge-in)**: a cancel must stop GPU work mid-chunk and flush
  controller/client buffers for that stream immediately, or the user's next utterance
  overlaps the ghost of the last one.
- **Failure modes**: mobile NAT/idle timeouts on long agent sessions → app-level
  keep-alives shorter than carrier NAT timeouts (same physics, and same trade — battery
  vs reconnect rate — as [WhatsApp](../03.whatsapp/04-deep-dives.md)); mid-stream gateway
  deploy → reconnect + resume-from-latest-sequence protocol, exactly like WhatsApp's
  reconnect catch-up; duplicated chunks after a retry → client dedupes on sequence number,
  a no-op since audio chunks carry IDs.

## 3. Voice model storage, loading, and tenant isolation (per-customer models)

A cloned voice is a *model artifact*, and there are orders of magnitude more of them than
base models: ~500K private voices, two tiers (estimates from [02](02-scale-estimates.md)):

- **Instant clone**: a speaker *embedding* + conditioning audio (~KBs + ~5MB), applied to
  the shared base model at inference time (prompt conditioning). Cheap to store, cheap to
  "load" (it's a lookup).
- **Professional clone**: fine-tuned checkpoint (~500MB per voice), applied as weights.

**Option A — every voice is a fine-tuned checkpoint**:
- wins: best per-voice quality; uniform mechanism.
- loses: 500K × 500MB ≈ 250TB of weights; every GPU can locally hold only hundreds —
  cache hit rates collapse; first-use for a random private voice pays a multi-second
  weights transfer *on the interactive path*; per-request fine-tune variants multiply
  storage.

**Option B — everything is prompt-conditioned embeddings** (never train per voice):
- wins: storage collapses to ~2.5TB of recordings + embeddings; any voice on any GPU is a
  O(seconds) embedding fetch; fleet-wide voice cache is a non-problem.
- loses: quality ceiling for demanding voices (prosody/expressiveness under prompt
  conditioning is below fine-tuning); heavy voices re-purchase their conditioning cost
  per request.

**Option C (chosen) — tiered: instant clone = embedding by default; professional clone
= dedicated checkpoint, opt-in by the customer** (and priced accordingly). Plus a
mechanism that matters regardless of tier: a **two-level voice cache** — a large registry
DB (authoritative, ~28TB of assets) and a per-GPU LRU cache of *hot* voices
(embeddings + the ~most-used few hundred checkpoints). Interactive dispatch is
**voice-affinity-biased**: prefer a GPU that already has the voice, degrade to a load-on
miss with an admission check (a stream must not wait >1s for a cold voice without the
controller shedding or queueing it as batch-like work).

- **Trade-off stated**: we accept two quality tiers and a bias/pinning scheduler's
  complexity so that the fleet's working set of *model artifacts* stays small enough to
  cache. The alternative — uniform fine-tuned everything — spends GPU capacity and
  storage to buy quality that most voices don't demand.
- **Isolation is a data-plane guarantee, not a policy**: voice assets are keyed
  (tenant_id, voice_id) with authorization checked at the registry *before* any asset
  load; inference workers have no path to fetch or serve a voice whose owning tenant
  didn't authorize the session. A leaked checkpoint is a leaked *capability* (anyone with
  the file can synthesize that voice forever), so asset-store objects are encrypted with
  per-tenant keys, and checkpoints are not downloadable by customers — API-synthesize
  only. This is the same "server may route but must not inspect/own" discipline as
  [WhatsApp's E2E design](../03.whatsapp/04-deep-dives.md) in reverse: there the server
  held no capability at all; here the server holds everything, so the file system must
  enforce tenant boundaries.
- **Failure modes**: cache stampede for a suddenly-viral public voice → coalesced
  single-flight loads per voice (N concurrent misses → 1 load); a corrupt/missing
  checkpoint → registry marks voice degraded, streams on it fail fast (not hang) with a
  retry-after-rebuild; a deleted customer account with voices still referenced by live
  streams → tombstone at the registry, streams terminate, asset deletion is async but
  audited.

## 4. The GPU budget: interactive vs batch on shared hardware

From [02](02-scale-estimates.md): interactive needs ~200 GPUs at peak with a tail-latency
SLO; batch's steady-state need is ~10 GPUs, bursting to ~50 for turnaround. They want the
same machines.

**Option A — one pool, batch fills the gaps**:
- wins: best utilization; batch pays near-zero marginal cost; no capacity is idle.
- loses: batch jobs *are* long-running GPU occupations (a segment render is minutes);
  when interactive demand spikes, preempting mid-render stranding partial segments is
  either wasteful (redo) or delay-y (drain); interactive p95 gets correlated to batch
  queue depth — exactly the coupling a latency SLO can't contain.

**Option B — separate pools, hard partition**:
- wins: interactive SLO is isolated by construction; capacity math is honest and simple.
- loses: idle interactive capacity overnight is dead weight (~30–50% of fleet-hours);
  batch queues at interactive peaks even though real spare compute exists.

**Option C (chosen) — separate *schedulers*, shared hardware, preemption asymmetry**:
interactive always wins; batch is preemptible by policy at segment boundaries only.
- interactive pool: sized for peak + headroom (1.75×), voice-affinity scheduling from
  deep-dive 3.
- batch pool: starts small (~50 GPUs of burst capacity), grows *only* into machines the
  interactive scheduler has explicitly released. A batch segment checkpoint is persisted
  at each boundary (manifest from [03](03-architecture.md)), so preemption costs at most
  one segment of re-render, and never a whole job.
- batch jobs that miss their turnaround window fall back to the interactive pool's
  *admission-controlled spare* (bounded %), never to the point of interactive p95 risk.

- **Trade-off stated**: we give up some utilization (Option C's interactive headroom sits
  idle much of the day) in exchange for a hard, provable rule — batch can never make an
  interactive stream audible-degradeable. Pure utilization-max (Option A) would buy a
  better $/GPU-hour and sell our p95 to the batch queue.
- **Why the asymmetry of preemption direction matters**: preemption is safe where
  *recovery is cheap and the victim tolerates it*. Batch tolerates it (segments are
  checkpointed, SLO is minutes); interactive streams don't (their SLO is hundreds of ms
  and a voice agent can't pause mid-conversation). Same lesson as Twitter's celebrity
  fanout asymmetry — the expensive path is the one with the strictest guarantee
  (see [Twitter deep dives](../02.twitter/04-deep-dives.md)); design so the strict one is
  never the one being preempted.
- **Failure modes**: runaway batch load — queue depth and interactive RTF are watched
  jointly, with automatic batch-pool shrink when interactive p95 drifts; a stuck segment
  re-renders forever → per-segment lease with retry cap and poison-segment quarantine
  (job completes minus the segment, flagged); fleet-wide GPU degradation (e.g., slow
  clocks) — RTF telemetry per GPU, and controllers re-route around underperformers the
  way a connection layer avoids a bad gateway.

## 5. Consent and abuse controls for cloned voices

Cloning is the feature whose abuse case is: upload 30s of a public figure's voice from
the internet, then have them *say anything*. Controls must be architectural, because once
audio is generated, the harm is already out.

- **Consent at registration, as data**: a pro/instant clone requires the owner to record
  a fixed, unguessable verification phrase (generated per attempt, not a static script —
  so replaying a found clip doesn't pass) plus owned-source attestation. Stored as an
  **append-only attestation record** bound to the voice's registry entry — synthesized
  audio carries provenance only if the attestation exists and hasn't been revoked.
- **Synthesis-time refusal, not just registration-time**: a request naming a protected
  voice (public figures, voices flagged by a platform, or a voice registered with consent
  *later revoked*) is refused in the controller *before* GPU dispatch. A model-level
  similarity screen also flags requests whose *conditioning audio*, not the named voice,
  resembles a protected speaker — because abuse won't politely use the registration flow.
  (Estimated design point: an embedding-similarity check against a protected-voice index
  runs in tens of ms — negligible next to inference.)
- **Watermarking every output**: inaudible watermarks in all generated audio (including
  batch), so synthesized audio is detectable as ours even after download and re-encoding.
  Trade-off: we spend a little output quality margin on robustness, and accept that no
  watermark is unremovable — it raises cost, not certainty.
- **Metering as abuse telemetry**: characters-per-tenant anomaly detection (abuse is
  typically bursty and high-volume) — the metering stream from F6 is the substrate, the
  same shape of signal as the follower-graph abuse telemetry in
  [Twitter deep dives](../02.twitter/04-deep-dives.md) — rate limits apply *before* the
  expensive/irreversible step (GPU dispatch), not after the audio is out.

**Options and the trade-off, stated**:
- **Option A — verify consent only at registration** (lighter product friction): wins
  onboarding UX badly (30-second path); loses the revocation case (consent exists, then
  the owner says stop — or never gave it and the recording was stolen) — synthesis keeps
  working on a stale yes forever.
- **Option B — verify at every synthesis call** (re-check all attestations inline): wins
  currency of consent; loses hot-path latency and DB load for what is a once-per-voice,
  append-mostly fact.
- **Option C (chosen) — cache with revocation push**: attestations cached at the
  controller with a short TTL (seconds), registry pushes invalidations on revoke/flag.
  Wins: hot path is unaffected and revocation takes effect within seconds; loses: push
  invalidation is another distributed subsystem with its own failure modes, and a few
  seconds of post-revocation staleness is accepted.

- **Failure modes**: forged consent recordings (the verification phrase catches scripted
  replays, not deepfake phrase recordings) → escalate to a live, liveness-checked challenge
  for high-risk voices, accepting slower onboarding for that tier; a leaked per-tenant
  checkpoint key → key rotation + per-tenant encryption (deep-dive 3) bounds the blast
  radius and re-keys without re-registering all voices; refusal-index misses a new public
  figure → detection/watermark + report-driven flagging are the fallback layer — design
  assumes the first line of defense will leak, and builds the second.
