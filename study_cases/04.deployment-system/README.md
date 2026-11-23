# Case 04 — Deployment System (CI/CD)

**The system**: a multi-tenant continuous-delivery platform (GitHub Actions / Spinnaker /
Buildkite-shaped). Commits arrive → the platform compiles a pipeline into a DAG of jobs →
builds and tests on ephemeral runners → publishes content-addressed artifacts → then the
fun part: it *changes production* with progressive rollouts (canary, blue/green), gates each
step on health metrics, and rolls back automatically when the gate fires.

**Why it's hard**:

- It is a scheduler first: **~100k pipelines run concurrently** (working-hours design point,
  see [02](02-scale-estimates.md)), each a fan-in/fan-out DAG, on a heterogeneous runner
  fleet, across thousands of tenants who all think they are first in line.
- It holds **the keys to production** for everyone who uses it: a runner executes untrusted
  PR code next to a broker that mint production credentials. Isolation and secrets are the
  load-bearing design work, not an add-on.
- The deploy loop is a **distributed control system**: an evaluator watching slowly-updating
  metrics races a promote controller that is eager to move on. Get the races wrong and you
  flap between versions or roll back good deploys.
- Everything is retry-unsafe by default: artifacts, cache restores, job claims, promotions
  all run on fallible hardware with at-least-once delivery underneath (same physics as
  [WhatsApp's delivery layer](../03.whatsapp/04-deep-dives.md)).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: DAG scheduling and fairness at scale (contrast the request/response
worlds of [Twitter](../02.twitter/README.md) and [YouTube](../01.youtube/README.md) — here the
unit of work is a *long-running job tree*, not a request), content-addressable artifact
storage, secret brokering over ephemeral compute, a promote/evaluate state machine with
single-writer fencing, and wave-based fleet rollouts — the platform deploys itself with the
mechanism it sells.

**Try it first**: before reading 04, answer: (1) how do you keep one noisy tenant from
starving 9,999 others when the queue is 500k jobs deep, (2) a metric gate says "healthy" but
the promote controller moved the traffic before the metric was written — how do you make
that race benign, (3) why is a shared build cache between PRs from untrusted forks a
security problem and what do you do about it?

**Primary sources**: GitHub's published limits and usage disclosures (e.g. their docs' per-
-org concurrency caps and public statements about millions of daily Actions executions —
order-of-magnitude anchors only), Google's SRE book on progressive delivery and error
budgets, Spinnaker documentation on deployment strategies, the AWS OIDC/federation model for
short-lived build credentials. This case is a design exercise: all numbers below are rough
estimates, not measurements.
