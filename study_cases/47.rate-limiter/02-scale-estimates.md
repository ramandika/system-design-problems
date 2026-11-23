# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- ~2B API requests/day on the platform — roughly the order a large payment/API platform
  reports in requests; treat as an estimate, not a citation.
- ~2M active API keys/day out of ~10M registered (most keys idle in a given day).
- Traffic skew: top ~1% of keys drive roughly half of all requests (heavy-tail, same
  skew family as hot links in 40.tinyurl / hot terms in typeahead) — load is
  concentrated, not uniform.
- Peak: bursts are correlated (batch jobs start on the hour, retries sync after blips),
  so peak ≈ 5× average (estimate).

## API QPS vs check QPS — the multiplier trap

**API requests**: 2B/day ÷ 86,400 s ≈ **~23K QPS average**, peak ~5× → **~115K QPS**.

**Checks per request**: a gateway request carries ~2–4 rules: binding tier limit (API
key), a per-IP check, and a key+route check for metered routes. Assume **3 checks per
request on average** →

- average: 23K × 3 ≈ **~70K checks/s**
- peak: 115K × 3 ≈ **~350K checks/s**

That is the *gateway* multiplier. The trap is fan-out beyond the gateway: if every
internal call site re-checks the same key (worker retries, webhook dispatch, partner
integrations calling through the same front door), the multiplier compounds. A deployment
where each of two internal hops re-checks plus the gateway is ~9 checks per request, and
a pipeline that checks per-element rather than per-batch (the crawler politeness mistake
in [41.web-crawler](../41.web-crawler/04-deep-dives.md) dive 1, inverted) can push the
ratio toward **100× the guarded QPS**. The design consequence, in one line: **checks are
cheaper than features — enforce at the edge, at the outermost point where a check can
still be correct, and never re-check internally what the gateway already certified.**

**Rejections are load too.** Under normal operation ~1% of requests are 429'd; each one
still ran its checks (the check is how it *found out* to reject). During abuse events,
single keys/IPs spike to 50%+ rejection — the limiter's load per admitted request
approaches 2×. So the counter store is sized for the attack, not the dashboard:
peak ~350K checks/s assumes peak traffic *includes* rejections.

## Where the check QPS lands

With 3 checks per request split by tier (02's architecture):

- **~60% burst/throughput checks** → enforced locally on the gateway node from its lease
  bucket: zero network chatter per check (deep dive 2). At peak: ~210K checks/s costing
  in-process memory reads.
- **~35% per-IP abuse checks** → mostly local too (the node saw the IP before), with
  ~1% of unknown/suspicious IPs escalating to the shared counter store.
- **~5% hard cumulative quotas** (hourly/monthly) → centralized counters at
  ~17K ops/s peak — every one a network round trip. This is what 1ms-of-budget-per-call
  buys a Redis-class cluster.

Arithmetic on the central store's load: a Redis-class single shard sustains roughly
~100K simple ops/s (round to the safe operational ceiling ~80K). At 350K peak total
checks/s, a naive *every-check-hits-Redis* design needs ≥ ~5 shards just for reads —
which sounds fine until you remember the skew: 1% of keys = ~50% of traffic, so those 5
shards must be sized so no *one* shard melts on one customer. With leases and chunked
counters (deep dive 4), the centralized tier peaks instead at ~20K ops/s — 17K quota
writes + a few K escalations — comfortably inside ~3 shards on load arithmetic alone (~20K vs ~80K safe ceiling
per shard) — the cluster still carries ~10 shards (03-architecture) mainly so a fat
key's K=8 chunked counters (deep dive 4) can each own a distinct shard.

## Storage: memory for counters, not bytes on disk

A counter entry is (limit-rule-id, subject, window) → count/tokens, ~40 bytes of key +
~16 bytes of value + ~70 bytes of hash-table overhead ≈ **~130 bytes/entry** in a
Redis-class store (estimate; the overhead dominates, which is why entry *count* is the
sizing variable, not value width).

Live entry count:

- API keys: 2M active/day × 2 windows (minute + hour bucket) ≈ 4M
- IPs: ~10M distinct IPs/day (NAT inflates; abuse-relevant IPs only) × 1 short window ≈ 10M
- key+route overrides: ~1M on metered routes × 2 windows ≈ 2M

**≈ 16M live entries → 16M × 130 B ≈ ~2 GB.** Trivial; TTL eviction as windows roll keeps
it flat over time. The counter store's cost was never memory.

- **Config store**: tier defaults (dozens of rules) + ~1M per-key overrides × ~200 B ≈
  **~200 MB** per full revision — small enough to hold a full snapshot in every gateway
  node's memory (deep dive 6) and to ship as a static blob.
- **Telemetry**: each check emits one decision record (~100 B: key, rule, decision,
  tokens-remaining). 70K avg checks/s × 100 B ≈ ~7 MB/s ≈ ~0.6 TB/day of decision
  stream — a Kafka-class fire-hose feeding dashboards and billing rollups, sampled or
  dropped under backpressure (same pattern as tinyurl clicks,
  [40.tinyurl](../40.tinyurl/04-deep-dives.md) dive 4).

## Latency: the budget that forbids the obvious design

Platform API p50 ≈ 20 ms end-to-end; reserving ≤ ~1 ms of it to the limiter p50 means:

- same-DC Redis round trip, ~0.3–0.8 ms — *fits*, but only for the ~5% centralized tier;
- edge PoP → central store across a metro/inter-region link, ~10–100 ms — *fails by 10–
  100×*. This one number is why "just rate limit in Redis" is not an architecture at
  CDN-graded deployments, and why deep dive 2 exists.
- Local/lease enforcement: an in-process check costs ~µs — effectively free, at the price of
  drift, quantified next.

## Consistency check

- Bounds on fleet over-admission: with N ≈ 100 gateway nodes and a 500 ms lease refresh
  interval, a saturated key's worst-case over-admission per interval ≈ N nodes each
  spending its leased slice ≈ one refresh interval's worth ≈ R × 0.5 s ≈ **+50% of one
  second's allowance** — matching the "±10% normally, ≤ 1× window under failure" line in
  01-requirements (the full derivation is deep dive 2's job).
- Counter store ops: 350K checks/s × 5% centralized ≈ ~17K ops/s ≤ ~22% of one shard's
  safe ceiling — the whole hard-quota tier fits in a single-digit-shard cluster.
- Decision stream ≈ 0.6 TB/day, comparable to tinyurl's ~1 TB/day click egress — an
  analytics-side cost that must never appear on the enforcement path.
