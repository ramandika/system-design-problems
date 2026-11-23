# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- ~10M new links/day (shortens) — roughly the order large public shorteners report for
  healthy years; treat as an estimate.
- ~1B redirects/day, i.e. ~23K redirects/s if the day's traffic all landed in the busy half
  of the day (1B ÷ 43,200 s); users click old
  links long after they were made, so the redirect corpus is cumulative, not daily.
- Read:write ratio ~100:1 — a link gets clicked ~100× over its life on average, with a
  heavy tail (a handful of links take most of the traffic).
- Record size (estimate): long URL ~200 bytes + ~50 bytes metadata → round to
  ~300 bytes/record.

## The asymmetry, in numbers

**Writes (shorten)**: 10M/day ÷ 86,400 s ≈ **~116 QPS average**, peak ~5× (estimate) →
**~600 QPS**. This is one modest database. Nothing about the write path is hard.

**Reads (redirect)**: 1B/day ≈ **~12K QPS average**, peak (email campaigns, viral shares —
bursts are correlated, not Poisson) ~5× (estimate) → **~60K QPS**. This is the design
target.

Read:write ≈ 12K ÷ 116 ≈ **~100:1**. Consequences:

- No read replica / shard tuning can rescue an un-cached design: even an in-memory KV at
  60K QPS needs its own fleet and headroom. The only sane target is *most redirects never
  reach the origin* (deep dive 2).
- A single-digit percentage of extra latency on shorten is invisible; 10 ms on redirect is
  the entire latency budget. Optimize only the read path.

## Key-space math: how long is the code?

Alphabet: `[a-zA-Z0-9]` = 62 symbols. Capacity of a base62 code of length L = 62^L:

- 62^5 ≈ 0.9B — exhausted in ~90 days at 10M/day. Too short.
- **62^6 = 56,800,235,584 ≈ 56.8B** — 10M/day × 365 × 10 years = 36.5B links in a decade → 6 chars covers
  ~10 years at ~64% fill. Workable but tight for a *random-allocation* strategy (see below).
- 62^7 ≈ 3.5T — effectively unlimited (~1000 years at this write rate). Cheap insurance:
  one extra character costs ~1 byte per link and ~1 character of visual noise.

So: **6 chars is the floor, 7 is the sane default**; custom aliases may be longer but are
bounded (< ~40 chars) to stop them doubling as a free-text field.

Allocation strategies constrain the usable space differently:

- **Sequential counter + base62**: uses 100% of the space (56.8B/56.8B). 6 chars is honest
  headroom math; 7 chars gives a huge buffer.
- **Random codes**: the birthday bound gives n ≈ sqrt(2N·ln(1/(1−p))) picks before the
  first collision with cumulative probability p. With N = 56.8B and p = 50%,
  n = sqrt(2 × 56.8B × ln 2) ≈ 280M picks — the first collision arrives at only ~0.5% fill
  (280M/56.8B). What actually hurts the check-and-retry loop is the *per-insert* collision
  rate, which tracks fill: at fill f, a fraction ~f of random picks hit an existing code,
  so retries are routine by ~30% fill and punishing at ~64% (roughly two picks in three
  collide).
  Random over a 6-char space with 36.5B stored links sits at ~64% fill — painful, not
  merely wasteful; over 7 chars it's a non-issue.
- This is the same constraint family as Twitter's ID generation
  ([02.twitter, dive 3: Snowflake-style IDs](../02.twitter/04-deep-dives.md#3-tweet-id-snowflake-style-time-sortable-ids)):
  64 bits there, 36 base62 bits here.

## Storage

- Records: 36.5B × ~300 bytes ≈ ~11 TB over 10 years — smaller than one modern NVMe node's
  worth. Storage is NOT the problem; key-value style with a compressed KV store handles it.
- Cache: only the hot tail matters. Heavy-tail shares mean a few TB of RAM
  (consistent-hash sharded, see [10.distributed-cache](../10.distributed-cache/README.md))
  holds the links that get clicked this week; cold links fall through to the KV store.
- **Egress dominates**: a redirect is a lookup plus a ~300-byte Location header (~1 KB
  response).
  1B/day × ~1 KB ≈ ~1 TB/day of egress — cheaper to absorb at CDN/edge than from origin
  (see deep dive 2).

## Consistency check

- 1B/day ÷ 10M/day = 100 clicks per link over its lifetime — matches the read:write ratio.
- 60K QPS peak × ~1 KB ≈ ~0.5 Gbps — trivial bandwidth; latency, not throughput of bytes,
  is the constraint.
- Code space: 36.5B (10-year cumulative) < 56.8B (6-char space) → 6 chars barely fits; 7
  chars removes the pressure. Arithmetic shows the conclusion, not vibes.
