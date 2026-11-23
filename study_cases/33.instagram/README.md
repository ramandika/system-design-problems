# Case 33 — Instagram

**The system**: a photo/video social network — users share media to followers, view ranked
feeds, and post ephemeral stories that live 24 hours. Sounds like Twitter with pictures;
three things make it a different problem.

**Why it's hard**:

- The unit of content is a *file*, not a string. Every post is an upload that must be
  normalized into many renditions (resolutions, codecs, thumbnails) before the platform can
  serve it cheaply — so the write path is a pipeline, not a row insert. Twitter's hardest
  write-path problem was fanout; ours starts *before* fanout.
- Stories are a second, parallel content stream with a hard TTL: every item is scheduled to
  die 24 hours after posting, views are private, and the "unread ring" state is per-viewer.
  Ephemeral-at-scale is its own storage and consistency problem, not a feature flag on the feed.
- The feed is *ranked*, not reverse-chron, which reshapes the fanout problem: the timeline
  cache holds candidate pools and precomputed features rather than an ordered log, and the
  hybrid fanout decision interacts with ranking (see ../15.ml-ranking-system).

**Deliberately out of scope / covered elsewhere**: feed fanout fundamentals and the
celebrity-problem taxonomy (02.twitter — this case only covers the *deltas*), blob store and
erasure-coding basics (07.blob-store), CDN caching mechanics (09.cdn), the client-side newsfeed
rendering (20.frontend-newsfeed, 25.mobile-newsfeed), generic two-stage ranking architecture
(15.ml-ranking-system), video *generation* (18.sora — contrast only: Instagram moves pixels,
it doesn't synthesize them).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: sync/async split in an upload pipeline, media renditions as
write-time precomputation, TTL-driven ephemeral storage, ranking-oriented timeline caches
(candidate pools, not logs), two-stage retrieval-then-ranking hooks, heavy-tail media traffic.

**Try it first**: before reading 03, answer: (1) what exactly can the user be told "shared"
when they hit post — the original bytes, or something smaller — and why does that question
dominate the whole pipeline, (2) how does a story disappear at 24h if nothing scans a
billions-of-items table, (3) why can't we reuse Twitter's timeline cache design unchanged
once the feed is ranked?
