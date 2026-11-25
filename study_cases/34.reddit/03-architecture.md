# 03 — Architecture

## High-level diagram

This is too much for one diagram, so it is drawn as two: the write path first, then the
read path (each diagram is self-contained and re-declares the shared nodes).

Q: How does a write (post, comment, vote, mod action) reach its system of record, and who consumes the events it fans out?

```mermaid
flowchart LR
    subgraph Client
        w((🧑 users<br/>web · mobile))
    end

    subgraph Edge
        edge[edge<br/>rate limits · geo · bot checks]
    end

    subgraph Services
        gw[gateway]
        post[post service<br/>CRUD · flair · NSFW]
        cmt[comment service<br/>tree ops]
        sub[community service<br/>metadata · membership · ACLs]
        mod[mod service<br/>mod actions · automod]
        vapi[vote api<br/>ack after publish]
    end

    subgraph Ingest
        k{{📨 event log<br/>votes · content · mod events}}
        vcon[vote consumer<br/>dedupe · apply deltas]
        spam[spam classifier<br/>stream consumer]
    end

    subgraph Stores
        pg[(🐘 Postgres<br/>posts · comments · users<br/>community metadata · mod log<br/>system of record)]
        cass[(🗄️ Cassandra<br/>comment tree per post<br/>materialized path)]
        cnt[(vote counters<br/>per content item)]
        led[(vote ledger<br/>per user per item)]
        s3[(🪣 media store<br/>object storage)]
    end

    w ==>|"1· POST /submit<br/>HTTPS · JSON"| edge
    edge -->|"POST /submit<br/>HTTPS · JSON"| gw
    gw -->|"gRPC createPost()"| post
    gw -->|"gRPC treeOps()"| cmt
    gw -->|"gRPC membership()"| sub
    gw -->|"gRPC modActions()"| mod
    sub -->|"SQL community state"| pg
    gw -->|"gRPC castVote()"| vapi
    post -->|"SQL posts"| pg
    post -->|"PUT media"| s3
    cmt -->|"SQL comments"| pg
    cmt -.->|"PUT comment rows<br/>async reconcile"| cass
    cmt -.->|"produce: content.events"| k
    post -.->|"produce: content.events"| k
    mod -->|"SQL mod log"| pg
    mod -.->|"produce: mod.action"| k
    vapi -.->|"produce: vote.cast"| k
    k -.->|"consume: vote.cast"| vcon
    k -.->|"consume:<br/>content.events · vote.cast"| spam
    k -.->|"consume:<br/>content.events · vote.cast"| mod
    vcon -->|"INCR score:{id}"| cnt
    vcon -->|"PUT vote:{uid}:{id}"| led
```

Q: How do a feed read and a thread read get assembled from caches and counters, and how does the event log keep those caches warm?

```mermaid
flowchart LR
    subgraph Client
        w((🧑 users<br/>web · mobile))
    end

    subgraph Edge
        cdn[/🌍 CDN<br/>media · static/]
        edge[edge<br/>rate limits · geo · bot checks]
    end

    subgraph Services
        gw[gateway]
        feed[feed assembler<br/>ranked by sort]
        usr[user service<br/>karma · prefs]
        cmt[comment service<br/>tree ops]
    end

    subgraph Ingest
        k{{📨 event log<br/>votes · content · mod events}}
        rc[rank cache refill]
    end

    subgraph Stores
        pg[(🐘 Postgres<br/>posts · comments · users<br/>system of record)]
        cnt[(vote counters<br/>per content item)]
        rd[(⚡ Redis<br/>tree caches · rank lists)]
        cass[(🗄️ Cassandra<br/>comment tree per post<br/>materialized path)]
        s3[(🪣 media store<br/>object storage)]
    end

    w ==>|"1· GET /v1/feed<br/>HTTPS · JSON"| edge
    edge -->|"GET /v1/feed<br/>HTTPS · JSON"| gw
    gw -->|"gRPC treeOps()"| cmt
    cmt -->|"GET tree page"| rd
    cmt -->|"CQL prefix scan"| cass
    gw -->|"gRPC sortQuery()"| feed
    gw -->|"gRPC prefs()"| usr
    feed -->|"gRPC getSubs()"| usr
    feed -->|"MGET candidate lists"| rd
    feed -->|"MGET scores"| cnt
    feed -->|"SQL posts hydrate"| pg
    s3 -->|"media · HTTPS"| cdn
    k -.->|"consume: content.events · vote.cast · mod.action"| rc
    rc -->|"SET rank:{subreddit}"| rd
```

## Components

**Edge/GW** — terminates anonymous traffic, applies global + per-user + per-community rate
limits, runs lightweight bot heuristics before content reaches write services. The edge,
not the write services, is the first anti-spam layer (deep dive 5).

**Post service** — owns posts (CRUD, flair, NSFW flags), writes to the system-of-record
Postgres sharded by post ID, publishes to the event log. Media goes to object storage +
CDN; posts store only URLs.

**Comment service** — owns tree operations: `render_thread(post, sort, path_prefix)`,
`create_comment`, `edit`, `delete`. System of record is Postgres; a Cassandra side-store
holds one partition per post with comment rows keyed by materialized path for fast subtree
scans (this split and its rationale is deep dive 1). Write-ahead: Postgres first, Cassandra
asynchronously reconciled, cache invalidated on mutation.

**Community & mod services** — community metadata, membership, moderator lists, ban lists,
user/community permissions, mod action log (removal, restore, lock, ban, distinguished
comment), and the automod engine (rule evaluation on every content event). Everything here
is scoped by `subreddit_id` (deep dive 4).

**Event log (Kafka-class)** — one backbone for three consumer families: (a) vote ingest,
(b) ranking/caching refreshers, (c) abuse classifiers. Content writes publish here too so
counters, caches, and classifiers all see the same ordered stream (pattern as in
[13.kafka](../13.kafka/README.md)).

**Vote ingest consumers** — dedupe (per-user-per-item once), then apply deltas to per-content
counters and the per-user vote ledger. Delivered-at-least-once upstream; made idempotent at
the consumer using `(user_id, content_id, revision)` (deep dive 2).

**Feed assembler** — for `r/home` and `r/all`, merges candidate pools per community and
ranks by the requested sort. It is fanout-on-read (Twitter contrast:
[../02.twitter/04-deep-dives.md](../02.twitter/04-deep-dives.md)) — no per-user materialized
timeline is kept; instead a small set of shared, heavily cached per-community and per-cohort
candidate lists serve every user's merge.

**Caches (Redis)** — comment tree pages keyed `(post_id, sort, prefix_page, version)`,
community hot lists keyed `(subreddit_id, sort, window)`, user vote ledger hot entries, and
rank inputs. Invalidation is explicit and versioned, not TTL-only, because thread reads are
extreme cache-critical (see the arithmetic in 02).

## Data flows

1. **Submit a post** — GW → post service → Postgres (durable) + event log. Feed caches pick
   it up asynchronously; ranking recomputes lazily per read (hot is cheap to compute, deep
   dive 3).
2. **Read a thread** — GW → comment service → Redis tree cache for the requested
   `(post, sort, prefix, sort-version)`; miss → Cassandra prefix scan → Postgres fallback
   only for reconciliation; cache the page, bump version on mutation.
3. **Vote** — GW → vote API → event log (fast ack to the user). Vote consumer applies delta
   to counters + ledger with idempotency; rank caches refresh on a short lag;
   thread caches are version-bumped (score displays refresh, tree version unchanged).
4. **Mod action** — mod service writes an immutable action-log row, produces a mod event;
   the automod engine and the feed assembler receive the event and apply the content state
   (removed content drops out of feeds within seconds, not on full re-render).
5. **Render home feed** — feed assembler reads the user's subscribed communities, takes the
   top-N candidate lists (per community, from cache), merges the ~1–2K candidate pool, ranks
   by the requested sort with per-user boosts, hydrates posts.

**Sharding summary**: Postgres rows by content ID; Cassandra comment tree by `post_id`
(one partition per post — bounded, since a post's comments are bounded by human behavior);
vote counters by content ID; communities are the unit for service-side routing and cache
keys, with a routing rule for mega-communities (deep dive 4).
