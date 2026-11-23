# 01 — Requirements

## Functional

- **Communities**: create/join subreddits; each has name, description, settings, rules,
  moderators, a subscriber list, and a feed of its posts.
- **Posts**: text, link, or media (image/video). Belongs to exactly one subreddit. Has a
  vote score, comment count, karma for the author.
- **Threaded comments**: each comment has exactly one parent (a post or another comment),
  arbitrary nesting depth (bounded ~10–12 visible levels; deeper levels open as a new page).
  Comments support editing, deletion, and per-community moderation states (removed, locked,
  approved).
- **Voting**: upvote/downvote on posts and comments; one vote per user per content item;
  users can change or retract their vote; scores change in (near) real time.
- **Ranking**: per-community feeds and per-user home feed in multiple sorts — **new**
  (chronological), **rising** (sudden velocity), **hot** (score vs age), **top** (score over
  a window); comment threads additionally sort by **best** (Wilson vote-confidence, a
  comment-level sort, not a feed sort).
- **Feeds**: r/<subreddit> feed, user home feed (their subscribed communities), r/all
  (every public community), r/popular. Comment threads render as trees inline.
- **Moderation**: volunteer moderators per community can remove/restore content, ban users
  from that community, pin/distinguish, apply automod rules. Site admins sit above with
  site-wide bans and quarantine/NSFW labeling.
- **Karma & identity**: user karma from posts and comments; no real-name identity; pseudonymous
  accounts mean abuse resistance leans on rate limits and ML, not on hard identity.

## Non-functional

- **Read latency**: post list page p50 < ~300 ms; comment tree render p50 < ~500 ms; feed
  page p99 < 1 s. Reads dominate writes by roughly 70:1 in request volume (arithmetic in
  [02](02-scale-estimates.md)), and far more fan-in per content item on hot posts.
- **Vote visibility**: score change visible to the voter and to ranking within seconds;
  exact live-score consistency to every viewer is NOT required (displayed counts are
  approximations anyway — Reddit publicly shows "fuzzed" counts for anti-abuse).
- **Durability**: a submitted post/comment must survive at essentially one write-ahead
  guarantee; votes can be treated as best-effort durable within minutes (recount jobs
  reconcile).
- **Scale**: ~50M DAU class (roughly [see 02](02-scale-estimates.md)); ~100K+ active
  communities; single hot posts reaching tens of thousands of comments.
- **Availability**: writes degrade gracefully — accepting votes is more important than
  immediately reflecting them; a community outage should not take down unrelated communities.
- **Isolation by design**: a community's workload (big thread, mod action, automod rule)
  must not blow up other communities or the global feed.

## Scope decisions

- In scope: posts, comment trees, voting, ranking formulas, community sharding, per-community
  moderation, feed assembly, spam/anti-abuse at the architectural level.
- Out of scope, but noted: private messaging & chat (see [05.discord](../05.discord/README.md)
  for real-time chat), live-streaming, ads serving and auctions, native mobile push
  infrastructure, ML model training pipelines (covered in
  [14.ml-production-system](../14.ml-production-system/README.md)).
- Simplification: we treat ranking as stateless formulas over stored aggregates. Real
  systems layer learned models on top; the structural problem (recompute cheaply, cache by
  feed, invalidate on vote bursts) is the same either way.
