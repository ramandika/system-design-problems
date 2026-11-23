# Case 02 — Twitter

**The system**: users post short messages (tweets); followers see them in a reverse-chron
timeline. Sounds simple; one celebrity's tweet is read by 100M people within an hour.

**Why it's hard**:

- The product looks the same for every user but the load profile is wildly skewed: normal
  users post to ~100 followers, celebrities post to ~100M. One "write" is not one write.
- The timeline must be instant: reads only modestly outnumber tweet writes (~1B reads/day
  vs ~600M tweets/day ≈ ~2:1; fanout inserts outnumber reads the other way, ~180:1), and
  a read must never wait on a fanout.
- Everything is eventually-consistent (counts, ranking) EXCEPT the thing users feel:
  your tweet should appear, mostly-in-order, on every follower's timeline.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: fanout-on-write vs fanout-on-read (the central one), timeline
caching in Redis, Snowflake-style ID generation, heavy-tail load skew, cache-aside patterns.

**Try it first**: before reading 03, answer: (1) where are timelines stored and in what
data structure, (2) what happens when @celebrity posts — trace it, (3) what happens on a
timeline read for a user who follows @celebrity and 500 others?
