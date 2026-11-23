# 01 — Requirements

## Functional

Core v1 (what a Stripe/GitHub-class API gateway needs from its limiter):

- **Limit dimensions**: a request is checked against several rules, each on a
  (limit-class, subject) key — per **API key** (the contract the customer signed up for),
  per **user** (many users can share a key; abuse isolation within an org), per **IP**
  (unauthenticated endpoints, login attempts, brute-force defense), and per
  **key+route** (a customer may be cheap on `/balance` and expensive on `/charges`).
  Rules match most-specific-wins: a per-key override beats a per-tier default.
- **Hard vs soft limits**:
  - *Hard limits* are enforced contracts — rejecting is correct and expected (tier
    throughput caps, per-key request quotas, billing-adjacent meters).
  - *Soft limits* are protective thresholds — the platform prefers the request to pass
    (anomaly/IP-abuse thresholds, burst allowances for well-behaved customers). Soft
    limits may be enforced with approximate, drifted counters; hard limits may not drift
    far below what the customer paid for.
- **Windows and burst**: short windows (1 s, 10 s, 1 min) for throughput shaping with an
  explicit *burst allowance* (a customer may spend idle capacity immediately — that is
  what token buckets are for), plus long hard windows (1 h, month) as cumulative quotas.
- **429 behavior**: on rejection, respond fast (`429 Too Many Requests`), never queue or
  throttle-by-delay in v1. Include a machine-readable reason (which rule was hit) and a
  suggested retry interval derived from the rule's window/refill rate.
- **Header exposure**: every admitted response carries `RateLimit-Limit` (the rule),
  `RateLimit-Remaining`, and `RateLimit-Reset` for the binding rule; the 429 carries
  `Retry-After`. Customers build automation on these headers, so they must come from the
  same decision state that admitted/rejected the request — not a separate best-effort
  counter.
- **Rejection vs degradation of protected resources**: the limiter also guards internal
  resources (login attempts per IP, webhook dispatch quotas). Same mechanism, different
  keys and different failure posture (see non-functional).
- **Config-driven rules**: limits are data, not code — tier defaults plus per-key
  overrides, changeable without a deploy, effective fleet-wide in seconds-to-minutes.

Out of scope v1: usage-based dynamic pricing of limits (a billing problem that only needs
the counters we already produce), per-request weighted cost accounting (each request costs
one unit; weighted units are an extension of the same atomic update), global admission
control / load shedding (a different mechanism that happens to read the same counters),
and a customer-facing "usage so far this month" dashboard (built from the telemetry
pipeline, not the enforcement path).

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Limiter latency overhead | ≤ ~1 ms p50, ≤ ~3 ms p99 added to the API call; the limiter must never be the slowest hop | forbids a synchronous network round trip on every check at edge PoPs; drives the lease/local-enforcement design (deep dive 2) |
| Limiter availability | limiter must be at least as available as the API it protects — a limiter outage must not become an API outage | fail-open/fail-closed is per limit class (deep dive 5); the counter store is on no request's critical path |
| Accuracy | hard limits: over-admission bounded at roughly ±10% under normal operation, ±1× window's worth under sync failure; soft limits: unbounded error acceptable, self-correcting | the explicit price of enforcing without a per-request round trip; stated in requirements so it can be defended arithmetic (02, deep dive 2) |
| False-negative tolerance (over-admission) | tolerated within bounds for throughput limits; near-zero for abuse limits protecting security resources | an attacker exploiting ±10% drift on a login-attempt limit is a compromise; drives per-class posture |
| False-positive tolerance (over-rejection) | must be effectively zero for paid limits outside failure mode — locking a paying customer out is the worst limiter failure | drives wedge-avoidance in counters (deep dive 3) and last-known-good config (deep dive 6) |
| Scale | ~2B API requests/day; ~70K limit checks/s average, ~350K peak (02-scale-estimates) | counter store sizing and hot-key math |
| Fleet consistency | all nodes enforce the *same* revision of a rule within seconds-to-minutes; brief skew is acceptable, flapping decisions are not | config versioning with coordinated activation (deep dive 6) |

## Scope decisions that shape everything

1. **The decider is a tiered system, not one algorithm.** Burst shaping (token bucket,
   lease-enforced locally) and cumulative quotas (counters, centralized) are different
   problems bolted into one pipeline; pretending one mechanism serves both is where most
   limiter designs go wrong. See deep dive 1.
2. **Accuracy is traded for latency and availability, in writing.** We accept ±10%
   over-admission on hard limits in exchange for no per-request round trip; we do *not*
   accept it for abuse limits. This is a requirement, not an implementation detail,
   because it is what makes the lease design legal (deep dive 2).
3. **Every rejected request is also load.** 429'd traffic still consumes check QPS before
   rejection — during an abuse event the limiter's load roughly doubles (02-scale-estimates).
   Rejections must be as cheap as admissions.
4. **Headers come from the enforcement state.** Adding a parallel "display counter" would
   double the cost and drift from what actually decided; we accept that the displayed
   `Remaining` is approximate (it is, by scope decision 2, and that is fine — it is not a
   contract).
