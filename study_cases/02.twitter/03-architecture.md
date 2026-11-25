# 03 — High-level architecture

## Diagram

Q: How does one tweet reach every follower's timeline — and what a celebrity post changes on both the write and read paths?

```mermaid
flowchart LR
    u((🧑 author)) ==>|"POST /tweets<br/>HTTPS · JSON"| a[api · fanout<br/>validate → snowflake → write]
    a -->|"MGET followers:{uid}"| sg[(social graph<br/>followers per user)]
    a -->|"INSERT tweet:{id}"| ts[(tweet store<br/>sharded by tweet_id)]
    a -.->|"produce: post.created"| q{{📨 fanout queues<br/>normal authors only · celebrities skip fanout}}
    a -.->|"produce: post.created"| ci[(celebrity index<br/>recent tweet IDs per top account)]
    q -.->|"consume: post.created"| w[fanout workers<br/>RPUSH timelines per follower]
    w -->|"MGET followers:{uid}"| sg
    w -->|"RPUSH tl:{uid}"| tl[(⚡ timeline cache<br/>tweet-id list per user)]
    r((🧑 reader)) -->|"GET /timeline<br/>HTTPS · JSON"| s[timeline read service<br/>merge · page · hydrate]
    s -->|"LRANGE tl:{uid} 0 200"| tl
    s -->|"ZREVRANGE celebrities:{uid} 0 200"| ci
    s -->|"⚡ MGET tweet:{id}"| tc[(⚡ tweet cache<br/>hot tweets)]
    s -->|"MGET tweet:{ids} 0-199"| ts
```

## Components

| Component | Job | Key data |
|---|---|---|
| API / Fanout service | accepts post, writes tweet store, decides fanout strategy | tweet content, author, timestamp |
| Tweet store | durable tweets, sharded by tweet_id | text, author, counts, media refs |
| Tweet cache | hot tweets (recent + viral) | ~10–20% of fleet serving ~95% of hydration |
| Social graph service + store | who follows whom, in fanout-friendly form | follower lists per user (shardable by followed_id) |
| Timeline cache | per-user materialized list of tweet IDs | Redis consistent-hash sharded by user_id |
| High-fanout queue + workers | normal-author fanout, pipelined batches | author → followers, insert timeline entries in batches |
| Celebrity "recent tweets" index | fallback/merge source for on-read | ~top-N most-followed accounts → their recent tweet IDs |
| Timeline read service | merge + page timelines, handle read-your-own-writes | |
| Search / trend pipeline | ingest tweets into search index, trends aggregation | separate DAG, can lag |
| Media CDN | image/video bytes | bytes bypass app servers entirely (see [YouTube](../01.youtube/03-architecture.md)) |

## Key data flows

**Post (normal user)**: validate → write tweet store (authoritative, ID assigned here by
Snowflake) → synchronously return to author (read-your-own-writes: author's own timeline
render merges the real store, not the cache) → async fanout workers insert into ~300
follower timelines.

**Post (celebrity)**: same write, but fanout is skipped entirely — nothing is enqueued to
the fanout queues, so no follower timeline is written at post time. Timeline reads for
anyone who follows that celebrity
MERGE: timeline-cache entries ⊕ recent-tweets-from-celebrity-index, dedup by tweet ID.
(Real Twitter 2012 went further: no celebrity fanout queue at all — celebrity tweets were
served purely on-read from the Earlybird search index; see
[04-deep-dives](04-deep-dives.md) §1.)

**Read timeline**: read timeline cache (one Redis multi-get) → if author follows any
high-fanout accounts, merge their recent tweets on-read → hydrate tweet bodies from tweet
cache/store in batch → render. One user's timeline = a few cache round trips total, not N.

## Where the money is

- Timeline reads are the product: their p99 latency and their hit ratio define the design.
- The 180B/day fanout-insert rate (from estimates) is why Redis timeline caches exist at all
  — they're the only way to make read path O(1) instead of O(following_count).
