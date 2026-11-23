# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- ~200M DAU, ~500M MAU
- ~500M tweets/day (publicly reported order), of which ~10–50M are "new" original tweets
  with the rest replies/retweets — simplify: 600M tweets/day
- Average user follows ~300 accounts, checks timeline ~5×/day → ~1B timeline reads/day
- Timeline shows the latest ~200 entries by default, 280 chars ≈ 280 bytes for text (UTF-8),
  plus metadata (ID, author, timestamp, counts) ≈ ~100 bytes

## Write path

**Tweet writes**: 600M/day ≈ **7K QPS average**, peak ~3–5× → ~35K QPS at peak. Not huge,
but every write fans out.

**Fanout work per tweet**: average follower count ~300, but heavy skew — median follower
count is tiny (~100 followers) while top accounts have 50–100M+ followers. If timelines are
materialized per user:

- Normal-user tweet: insert into ~300 follower timelines → cheap.
- Celebrity tweet (worst case): a single write needs inserts into **~100M timelines**.
  Even at 1ms per insert that is ~28 hours serialized — pure fanout-on-write dies here.

So the write path is a *distribution and scheduling* problem, not a "write to DB" problem.

## Read path

**Timeline reads**: 1B/day ≈ 12K QPS average, peak ~5× → ~60K QPS on the timeline service.
Each read renders ~200 tweets → the service must return ~200 pre-materialized tweet IDs
(often fewer bytes than it sounds: ~200 × ~50 byte pointers + hydrated tweet bodies fetched
from tweet store, cached).

**Tweet reads (hydration)**: timeline cache gives IDs; tweet bodies are fetched from the
tweet store via cache. Assume heavy caching (a handful of viral + popular tweets dominate):
cache hit ratio ~95%+ at a few-TB Redis fleet; DB reads ~5% of 60K QPS × 200 tweets —
that is 60K × 200 ≈ 12M hydration keys/s, ~5% miss → ~600K tweet-body fetches/s (estimate)
— manageable only if we batch (multi-get ~200 keys at once, not 200 individual round trips).

## Storage

**Tweet store**: 600M tweets/day × ~400 bytes (text + metadata + denormalized fields)

= 240 GB/day ≈ ~96 GB/day after compression (estimate, ~2.5×) ≈ **~35 TB/year**
(96 GB/day × 365 ≈ 35 TB; 1 GB = 10^9 B throughout. If compression only reaches 2×:
120 GB/day → ~44 TB/year).

Sharding: by tweet ID (time-ordered, distributes) → but per-user "my tweets" profile views
need user-keyed secondary access (author index) — a second index keyed by user_id.

**Timeline cache**: ~200M DAU × ~800-timeline-entry-cap; most users' timelines are only
a few hundred entries ≈ ~10 KB/user hot in Redis. 200M × 10 KB = **~2 TB RAM**, ×2 with the
replicated copies ≈ **~4 TB** across the fleet (fits across a few hundred Redis nodes;
consistent-hash by user_id).

## Media bandwidth (estimate)

Media bytes bypass app servers via CDN (see [YouTube](../01.youtube/03-architecture.md)),
but they still need sizing. Assume ~50% of timeline reads render some media, ~1.5 MB of
media per such read: 1B reads/day × 1.5 MB ≈ **~1.5 PB/day ≈ ~17 GB/s ≈ ~140 Gbps average**
(1.5×10^15 B ÷ 86,400 s ≈ 17 GB/s; peak ~5× → ~700 Gbps). At a ~95%+ CDN hit ratio,
origin fetches are ~5% of that (~7 GB/s) — this is why media never touches the app tier.

## Consistency check

- 1B timeline reads/day ÷ 200M DAU = 5 reads/user/day — matches assumption.
- 600M tweets/day ÷ 200M DAU = 3 tweets/user/day — plausible.
- Fanout inserts/day ≈ 600M × avg~300 followers ≈ ~180B timeline-inserts/day (~2M/s) —
  THIS, not the 7K tweet QPS, is the real write workload. Any design that materializes
  timelines must hold this rate; it's why hybrid fanout exists (see deep dives).

## Calibration against real Twitter (2011–2012, publicly reported)

Primary source: Raffi Krikorian, "Timelines at Scale", QCon SF 2012
([slides](https://qconsf.com/sf2018/sf2012/dl/qcon-sanfran-2012/slides/RaffiKrikorian_TimelinesAtScale.pdf),
[InfoQ talk](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)) — all
figures 2012-era, cite them with the date.

- **Timeline reads: >300K QPS @ 1ms p50 / 4ms p99** (poll-based). 300K/s × 86,400s ≈ 26B
  reads/day — ~26× our 1B/day estimate. Our "5 reads/user/day" assumption is far too low;
  real clients re-issue reads on every scroll/app open, not 5×/day. If we redo the read
  path at 26B/day → 300K QPS avg, the O(1)-read requirement gets *stronger*, not weaker —
  conclusion holds.
- **30B timeline deliveries/day** (~350K/s avg). Against our 180B/day fanout-insert
  estimate: 30B ÷ 600M tweets ≈ 50 deliveries per tweet, not ~300 — real Twitter's hybrid
  excluded celebrity followers from fanout entirely, capping per-tweet fanout. This is an
  order-of-magnitude confirmation that the hybrid choice (deep dive 1) is what bounds
  write cost in practice.
- **Tweet ingestion: 5–10K/s normal, ~34K/s peak** (widely re-reported 2013 New Year's
  Eve spike of ~33K/s; attributed to a Krikorian interview but not re-verified against a
  primary source — carry as "roughly") → peak ≈ 3.5–7× average. Our 7K QPS avg / ~35K
  peak (3–5×) lands inside it.
- **Search (2011): ~2,200 tweets/s indexed, 18K QPS queries ≈ 1.6B queries/day**
  (18,000 × 86,400 ≈ 1.55B; Twitter eng blog + Earlybird ICDE 2012 paper).
- **Fanout mechanics**: pipelined 4,000 recipients per batch into a replicated Redis
  timeline cache; p50 delivery to 1M followers in 3.5 s (p99 ~5 min).
- Later-era anchor (2023, Twitter/X eng blog): users-DB reads scaled toward millions of
  QPS not by adding MySQL replicas but with hundreds of stateless Vtgate proxies on Mesos —
  worth remembering when our "read scaling" plans assume replica fan-out.
