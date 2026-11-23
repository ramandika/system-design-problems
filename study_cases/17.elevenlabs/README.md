# Case 17 — ElevenLabs (Voice AI)

**The system**: text-to-speech and voice cloning. Two coexisting workloads: **real-time
streaming TTS** (a user or an autonomous voice agent speaks text and hears audio back with
sub-second first-byte latency) and **batch synthesis** (long documents, audiobooks, dubbed
videos — minutes of audio per job, measured in throughput not latency). On top sit a
**voice model registry** (public library + per-customer private voices) and **per-customer
voice training** (cloning from a consented recording).

**Why it's hard**:

- TTS is the first workload in this corpus where the *output* is a strict real-time
  constraint: interactive synthesis must produce audio faster than real time, forever, or
  the user hears a gap. Chat/Twitter tolerate 300ms p99; a voice agent tolerates it once
  in the first response and never again mid-sentence.
- The GPU fleet serves two customers with opposite SLOs (interactive wants tail latency,
  batch wants cost per output hour) sharing one hardware type — the scheduling split is a
  first-class design decision, not an ops detail.
- Voice cloning makes *the model weights themselves* a per-customer asset: storage,
  loading, caching, and tenant isolation apply to artifacts that are megabytes, not rows.
- A consumer can clone any voice from 30 seconds of YouTube audio. Consent verification
  and abuse controls are architectural components (refusals, watermarks, detection), not a
  terms-of-service page.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: real-time production constraints vs batch throughput on shared
hardware (a new axis versus the request/response workloads in
[Twitter](../02.twitter/README.md) and [WhatsApp](../03.whatsapp/README.md)); long-lived
websocket transport (compare [WhatsApp's connection layer](../03.whatsapp/04-deep-dives.md));
per-tenant model weights as storage problem; rate limiting and abuse where the abuse
creates durable artifacts ([Twitter's rate limits](../02.twitter/04-deep-dives.md)).

**Try it first**: before reading 03, answer: (1) why not render the whole utterance and
ship it as one file when the user asks for streaming, (2) what exactly happens on the GPU
when a customer's private cloned voice is requested for the first time, (3) if batch jobs
and interactive streams run on the same GPUs, who wins and what does the loser experience.

**Primary sources**: this case is written as a design exercise over known product
behavior, not from published internals. Scale numbers in [02](02-scale-estimates.md) are
our own rough estimates from public reporting (ElevenLabs' ~$100M ARR milestone and
~1M+ registered users were publicly reported around 2025); no internal fleet or latency
figures are cited because none have been published.
