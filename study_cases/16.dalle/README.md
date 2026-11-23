# Case 16 — DALL-E (text-to-image generation)

**The system**: users type a prompt ("an astronaut riding a horse in a watercolor style"),
pick size/count, and get images back. Sounds like "call a model"; one generation burns
seconds of scarce GPU time, every prompt is a potential policy violation, and every image is
a multi-megabyte artifact that has to be stored, moderated and served.

**Why it's hard**:

- The unit of load is **GPU-seconds, not requests**. A text service scales by adding cheap
  stateless web pods; here capacity is thousands of expensive GPUs with a hard throughput
  ceiling and minutes of queueing when you guess capacity wrong.
- Load is **bursty and async** (a viral moment triples submissions; a GPU pool can't
  spin up in 10 seconds), so the product must be async end-to-end: submit → queue →
  generate → deliver, with the user watching a status, not a spinner.
- **Safety is content policy under adversarial pressure**: prompts attack the filter;
  images must be classified before delivery, not after — but in-line checking eats
  latency and GPUs.
- Every result is a billed artifact: the economics hinge on **per-image GPU cost**
  (batching, fewer denoising steps, dedup of identical requests) as much as on
  architecture.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: async job pipeline with a finite compute pool, GPU scheduling
(reserved vs spot, batching, priority queues), per-unit inference cost engineering,
moderation in-line vs after-the-fact, cache-by-exact-request dedup, CDN delivery of
generated artifacts.

**Compare with**: [ChatGPT](../08.chatgpt/04-deep-dives.md) (same GPU-pool + streaming
serving problem, different output shape) and [Twitter](../02.twitter/04-deep-dives.md)
(rate limiting and abuse with a skewed load profile — same physics, opposite unit of cost:
fanout inserts there, GPU-seconds here).

**Try it first**: before reading 03, answer: (1) what happens when 3× traffic arrives with
a fixed GPU pool — queue, shed, or spawn GPUs, and for whom? (2) is the NSFW check before
or after generation, and where does that decision put the latency budget? (3) two users
submit the identical prompt — is the second one free?

*All numbers in this case are rough estimates unless stated as publicly reported; no DALL-E
production capacity figures are public, so 02 derives its own and calibrates against
publicly listed API pricing.*
