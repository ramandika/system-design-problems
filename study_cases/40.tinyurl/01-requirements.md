# 01 — Requirements

## Functional

Core v1:

- **Shorten**: long URL (≤ ~2 KB) in → short code out, e.g. `sho.rt/aB3xK9`. Optionally a
  custom alias (`sho.rt/my-launch`) if available. Same long URL may be shortened many times;
  deduplication is NOT required (each request gets its own code — see scope decisions).
- **Redirect**: GET of a valid code → HTTP redirect to the long URL. This is the whole
  product. Unknown code → 404 or an interstitial, never a hang.
- **Custom aliases**: user-picked codes, first-come-first-served, plus a reserved-word
  policy (see deep dives).
- **Click counts**: per-link click count (eventually consistent is fine) and coarse
  aggregate stats (clicks per day, referrer/geo buckets). No per-click forensic timeline
  for v1.
- **Expiry (optional links)**: a link may carry a TTL; expired links still resolve to an
  expiry page rather than silently 404.

Out of scope v1: link editing (retargeting an existing code), user accounts/teams, QR code
rendering (trivially derived once we have the code), A/B destination splitting.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Redirect latency | p99 < ~10 ms from cache, < ~50 ms on a cache miss; ideally served at the edge | the redirect is the hot path; drives cache-then-KV design |
| Redirect availability | effectively "never down" (~99.99%+); must fail open | if a cache AND KV shard are down, we still redirect on cached/recent data or degrade visibly, not hang |
| Shorten latency | p99 < ~200 ms | non-critical; can involve a DB write and alias check |
| Consistency | read-your-own-writes for the creator (your new code must work immediately); click counts eventually consistent (seconds) | creation writes through to cache/KV before responding |
| Code uniqueness | absolute — a code must never point at two long URLs | drives generation strategy (deep dive 1) |
| Code unpredictability | auto codes should not be enumerable | private links must not be guessable by neighbor-probing |
| Abuse | known-bad destinations must be blocked or interstitial-ed; creation rate-limited | we are a redirector; shorteners are a spam weapon |
| Retention | links effectively forever (shorteners who expire links die as products); expiry only when explicitly set | storage grows unbounded but each record is tiny (see estimates) |
| Scale | ~10M links/day, ~1B redirects/day (roughly the publicly reported order for large shorteners) | drives the estimate sheet |

## Scope decisions that shape everything

1. **The read path is the product.** Writes are two orders of magnitude lighter than reads.
   Every downstream choice (generation, storage, status codes) is judged on what it does to
   redirect latency (see deep dives 2 and 4).
2. **No dedup on shorten.** Same long URL shortens twice → two codes. Wins: creation is
   stateless and cheap, no uniqueness check on the *long URL* side. Loses: code space is
   burned on duplicates and click stats split across codes. We accept this; dedup is a
   lookup (hash of long URL) that adds a cache query to a path that is already fine, and
   it must be opt-in to avoid leaking "was this URL ever shortened here".
3. **Redirect responses are cacheable** at the edge in principle — but see deep dive 2:
   301 vs 302 controls whether the browser may cache it, and that choice trades redirect
   count against analytics and editability.
4. **Analytics never block the redirect.** Click events are emitted fire-and-forget (or not
   at all under load — better to lose clicks than to lose redirects). See deep dive 4.
