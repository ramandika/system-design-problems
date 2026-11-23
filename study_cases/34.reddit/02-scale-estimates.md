# 02 — Scale estimates

Assumptions (adjust and redo the arithmetic if they change). Public anchors are old enough
to label; treat growth since then as free upside, and note where the conclusion is robust
to it.

- ~430M MAU (publicly reported by Reddit in its 2019 year-in-review; corroborated by
  third-party summaries of the same report), ~50–60M DAU (publicly reported ~57M around
  2023; treat as an order-of-magnitude estimate — Reddit's later SEC filings show higher).
- ~100K+ active communities (Reddit has publicly said "over 100K active communities"; total
  created communities is far larger).
- Votes: 23.1B votes in 2019 (Reddit's 2019 year-in-review, treated as an annual total)
  → 23.1B / 365 ≈ **63M votes/day**.
  Scale to 2024-era usage ×3 (estimate) → 63.3M × 3 ≈ **190M votes/day** as our working
  number.
- Comments: 1.7B comments in 2019 (same review, treated as an annual total) → 1.7B / 365
  ≈ **4.7M/day**; ×3 (estimate) → 4.7M × 3 ≈ **14M comments/day** working number.
- Posts: ~13.2M posts in October 2019 (same review — a single-month figure; the vote and
  comment totals above are treated as annual) → 13.2M / 30 ≈ 440K/day; ×3 (estimate) →
  **~1.3M/day**. (External cross-check: Reddit's SEC filings publicly report a cumulative
  corpus of roughly 2B posts and 22B comments through 2025 — consistent with a single-month
  post rate around 440K/day in 2019 and with the ×3 post estimate, and with comment volumes
  closer to the ×1–2 range than a full ×3.)
- Reads: reads dominate. Estimate DAU × ~20 page-ish reads/day (feed pages, thread pages,
  infinite-scroll chunks) → 57M × 20 ≈ 1.1B read requests/day. Each feed request hydrates
  ~25–50 posts; each thread request hydrates ~200–500 comment nodes.

## Write path

**Posts**: 1.3M/day ≈ **15 QPS average**, peak ~5× → ~75 QPS. Trivially small — the write
problem for posts is never throughput, it's the read demand a single post generates.

**Comments**: 14M/day ≈ **160 QPS average**, peak ~5× → **~800 QPS site-wide**. A single viral
thread that absorbs 5–10K comments/hour is only ~1–3 QPS at the comment level (5,000–10,000
÷ 3,600), but a live event concentrates many simultaneous hot threads on one subreddit
shard, pushing that shard into the low hundreds of QPS. Note: a viral article's comments
arrive as a fast sequential stream at tree depth 1–3, all targeting one post — a partition
hotspot by construction, not by skew across keys.

**Votes**: 190M/day ≈ **2.2K QPS average**, peak 10–20× during a live sports/politics event
or a front-page event: **~20K–45K QPS global vote ingest for hours**. Votes are tiny (~50–100 bytes)
and extremely amenable to batching — this is a message-throughput problem, not a transaction
problem. See [13.kafka](../13.kafka/04-deep-dives.md) for the event-log pattern.

## Read path

**Feed reads**: 1.1B requests/day ≈ 12K QPS average, peak ~5× → ~60K QPS. Each request
requires ranking candidates: per-community reads re-rank that community's own recent posts
(~100s of candidates); r/all must merge across communities (deep dive 6).

**Comment tree reads**: a hot thread with 50K comments, viewed by (say) 5M users over a day
→ effectively the tree is re-read tens of millions of times while changing every second.
Conclusion reads out of the arithmetic: **per-request tree rendering from durable storage
cannot stand; we need (a) prefix-range-friendly storage and (b) aggressive caching keyed by
(post, sort) with invalidate-on-mutation rather than count-based recomputation.**

Hydration arithmetic per thread read: ~500 comment nodes × (render only ~100–200 per page +
a "load more" stub for the rest). Per-subreddit feed read: ~25 posts × rank+hydrate.

## Storage

**Posts**: 1.3M/day × ~2 KB (title + body/text + metadata) ≈ 2.6 GB/day ≈ **~1 TB/year
raw**. Small.

**Comments**: 14M/day × ~500 B (body + parent + author + metadata) ≈ 7 GB/day ≈ **~2.6
TB/year raw, ~1 TB/year compressed**. Small — which is the point: comment *storage* is a
non-problem; comment *tree read patterns* are the problem.

**Votes**: two artifacts per vote — an ingest log (batched, time-partitioned, dropped after
reconciliation) ≈ 190M × ~60 B ≈ 11 GB/day (transient), and per-content counters +
per-user vote ledger for undo (≈ same order). Keep the per-user ledger long; the raw event
log can be aged out after reconciliation windows (days).

**Comment tree index overhead** (materialized path, deep dive 1): path strings add ~16 B/level
× avg depth ~2 ⇒ negligible.

## Consistency check

- 190M votes/day ÷ 57M DAU ≈ 3.3 votes/user/day — plausible for drive-by voting.
- 14M comments/day ÷ 57M DAU ≈ 0.25 comments/user/day — plausible; commenters are a minority.
- Read:write asymmetry: 1.1B reads vs ~15M content writes/day (1.3M posts + 14M comments)
  ≈ 70:1 by request; but
  per-request fan-in (a hot thread read 5M times against 50K writes) is the real ratio,
  ~100:1 against a single key. Every design choice downstream bends around this.

## Calibration against public numbers

- Reddit's 2019 year-in-review (their own published totals — 23.1B votes, 1.7B comments,
  13.2M posts/late Oct) lands at **~63M votes, ~4.7M comments, ~440K posts per day**. Our
  ×3 scaling for later years is an estimate, labeled as such; the write path is comfortably
  small either way (even ×10 is under 8K QPS of comments).
- Reddit engineering has publicly shared (2015–2020: their blog posts and talks, e.g. on
  moving comment storage to Cassandra; the traffic/edge infrastructure talk at SRE-era
  meetups) that comment reads and the frontend/edge layer, not comment writes, were the
  scaling pressure point. Our arithmetic agrees; the architecture in 03 allocates effort
  accordingly.
- Vote-score display is fuzzed by Reddit publicly (they've said as much for anti-brigading);
  so is vote *counting* precision. Our design tolerates approximate live scores for the same
  reason — the ranking function only needs the score to be right within a few percent.
