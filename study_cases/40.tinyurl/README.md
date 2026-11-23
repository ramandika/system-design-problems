# Case 40 — TinyURL (URL shortener)

**The system**: a long URL goes in, a short code comes out; anyone hitting the short URL is
redirected to the long one. It is the smallest system in this collection — one entity, two
operations — which is exactly why it works as a *compact exemplar*: it isolates a few
patterns (read/write asymmetry, ID generation, cache-till-it's-boring, async side effects)
without a product's worth of features hiding them.

**Why it's hard (all that there is)**:

- The load is violently asymmetric: link creation is rare (writes), redirects are constant
  (reads), roughly 100:1. Designing for "a CRUD app" fails instantly — the entire design
  exists to serve the read path at CDN speeds.
- The whole key space lives or dies on one decision: how are 6–7 character codes minted?
  Sequential, hashed, or pre-allocated — each trades uniqueness guarantees against
  unpredictability and availability.
- Every redirect is an outbound link you do not control: users paste malware, phishing,
  expired destinations. You are a link trust layer, not just a lookup table.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: read/write asymmetry (~100:1), code generation
(counter+base62 vs hash vs pre-allocated ranges), layered redirect caching with a Bloom
filter for negative lookups, async analytics pipelines, link-expiry and abuse handling.

**Siblings**: code generation revisits Snowflake-style ID allocation from
[02.twitter](../02.twitter/04-deep-dives.md); the redirect cache is a pure example of
[10.distributed-cache](../10.distributed-cache/README.md) patterns; edge-offloaded serving
borrows from [09.cdn](../09.cdn/README.md).

**Try it first**: before reading 03, answer: (1) how many characters does the short code
need, shown with arithmetic, (2) what happens on a redirect for a code that does not exist,
at 60K QPS, (3) how do you count clicks without adding 1 ms to the redirect?
