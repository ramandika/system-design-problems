# Case 35 — TikTok (interest-graph short-video feed)

**The system**: users upload short videos (15 s–3 min); every user watches a personalized
"For You" feed. The critical fact about the product: **the feed is not a follow graph**.
There is no timeline you subscribed to. Every swipe asks the server "what should this
person see next?" and the honest answer is computed from scratch, per request, from an
interest model. Videos circulate to whoever the algorithm thinks will watch them — a
first-post creator can reach millions, and last hour's engagement decides it.

**Why it's hard**:

- The feed read is **compute, not storage**. Twitter can precompute timelines because the
  follow graph is the ground truth ([Case 02](../02.twitter/README.md)). Here there is
  nothing to fan out to: candidate generation → ranking → quota/diversity runs per
  request, at ~3.5×10⁵ QPS, against a corpus of hundreds of millions of videos.
- It is video at extreme egress scale. Every impression transfers megabytes, and mobile
  networks are the constraint — the transcode ladder is a *product* decision, not a
  media-pipeline detail.
- Engagement signals are the fuel and arrive at extreme write scale (~2.4×10⁷ events/second
  mean, estimate) with a freshness requirement: what you just watched must skew what you
  are shown next.
- **Cold start is the business model.** A new video from a new creator has no history.
  The system must spend real impression budget exploring it, or the catalog ossifies and
  creators leave.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Relationship to other cases**:

- [Case 15](../15.ml-ranking-system/README.md) covers ranking primitives (funnels,
  feature stores, offline/online skew). Here we own the *retrieval + slate* layer that
  sits above the ranker.
- [Case 25](../25.mobile-newsfeed/README.md) owns client-side feed mechanics (prefetch,
  decode budgets, scroll jank). This case owns the server side and the video ladder that
  prefetch consumes.
- [Case 18](../18.sora/README.md) is video *generation*: there compute creates the
  artifact. Here the artifact already exists and the hard problem is *selection and
  distribution* — a useful contrast in what "video-scale compute" means.
- [Case 07](../07.blob-store/README.md) and [Case 09](../09.cdn/README.md) are the storage
  and edge tiers this case sits on; they are treated as given here.

**Patterns taught here**: recall→coarse-rank→fine-rank→slate funnels, per-title ABR
ladders, publish-before-transcode pipelines, write-scale telemetry with freshness
budgets, exploration budgets for cold-start content.

**Try it first**: before reading 03, answer: (1) why can't you precompute each user's
For-You feed the way Twitter precomputes timelines — and what does that force the read
path to cost? (2) a creator hits "post" — trace the video to the first viewer, including
which steps block. (3) you have 300B impressions/day (estimate) — where does every
engagement event go within seconds, and what breaks if it's 30 minutes late?
