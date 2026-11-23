# Case 34 — Reddit

**The system**: a community/discussion platform. Users join communities (subreddits), submit
posts (text/link/media), and reply in **threaded comment trees** that can go many levels deep.
Every post and comment carries a **vote score** (net up/down votes) that drives ranking into
**per-community and global hot feeds**. Communities are moderated mostly by **volunteer
moderators**, not by a central trust-and-safety team.

**Why it's hard**:

- The core data is not a timeline but a **tree**. Twitter's unit is a flat list; here every
  comment has a parent, and rendering a thread means fetching and ordering a subtree,
  sometimes hundreds of thousands of nodes deep on viral posts.
- **Votes are a firehose with undo semantics**: billions of votes, users can change or delete
  votes, and the same vote must never be counted twice — under a store that also has to
  re-rank content in real time.
- **Sharding by community** feels natural but is wrong at the edges: the biggest subreddits
  are far larger than the median, and r/all, the front page, and per-user feeds must merge
  across all shards.
- **Moderation is federated**: thousands of volunteer moderators with per-community rules,
  with only a thin central layer (site-wide bans, quarantines) above them. Designing the
  permission and action model per community — not globally — is unusual.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: tree storage (adjacency vs materialized path vs closure table),
idempotent event ingestion with undo (voting), ranking functions (hot score, Wilson lower
bound), shard-by-community with escape hatches, fanout-on-read feed assembly (contrast with
[case 02](../02.twitter/04-deep-dives.md)), community-scoped moderation state machines
(contrast with [case 06](../06.apple-app-store/04-deep-dives.md)).

**Try it first**: before reading 03, answer: (1) how would you fetch and render a 50K-comment
thread with a "load more comments" affordance — what does the storage layer return for one
click? (2) a user changes an upvote to a downvote on a viral post — trace where the delta
lands and why it must be idempotent, (3) how does the r/all feed merge candidates from
100K communities without being O(communities)?
