# 01 — Requirements

## Functional

Core v1:

- **Post tweet**: text ≤ 280 chars, optionally media (image/video). Tweets are immutable
  once posted; deleting is possible (fanout of deletion, see later).
- **Follow / unfollow**: follows change future timeline contents only (a core simplification:
  posting does not retroactively affect anyone's timeline).
- **Home timeline**: the N most recent tweets from everyone you follow, mostly-reverse-chron.
  ~"last 800 tweets" materialized; older goes to a paged/read path.
- **User profile**: my tweets, followers, following counts.
- **Search**: hashtag/keyword/URL search; trending topics.

Out of scope v1 (but note): DMs (→ see [WhatsApp](../03.whatsapp/README.md) case), quote
tweets' threading nuances, notifications' full design, ads/recommendation ranking beyond
"reverse-chron".

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Timeline read latency | p99 < ~100 ms | this dominates; drives the timeline-cache design |
| Post → visible in follower timelines | < 1–2 s typical | fanout is async but must feel sync |
| Availability | post: high; timeline: higher; search: can degrade | posting failure is user-visible; search outage is not |
| Consistency | read-your-own-writes (you must see your own tweet); follower timelines eventually consistent (~seconds) | subtle: the author must see their tweet immediately — affects where we write first |
| Duplication / abuse | bots must not flood timelines | rate limits per account; fanout throttling |
| Retention | tweets effectively forever | storage tiering; counts as part of the "public archive" |
| Scale | ~200M DAU (publicly reported, order-of-magnitude) | drives the estimate sheet |

## Scope decisions that shape everything

1. **Timelines are materialized**, not queried at read time. Read-perf is the product. The
   design tension (memory cost of pre-building timelines vs read-time cost of building them)
   is the whole case — see deep dives.
2. **Fanout is asynchronous and best-effort but retried to completion** — visibility to
   followers may lag by seconds; a missed fanout would silently hide tweets forever.
3. **Tweet IDs are time-sortable** — reverse-chron ordering and pagination depend on it
   (see deep dives for Snowflake ID design).
