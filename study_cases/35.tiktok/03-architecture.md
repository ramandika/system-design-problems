# 03 — Architecture

## High-level shape

Five planes: **serve** (the per-request funnel), **ingest** (upload→transcode→publish),
**observe** (the telemetry loop), **store/edge** (blob + CDN, treated as given), and a
shared **metadata/feature substrate**. The defining property: the serve plane computes
the feed per request from live state — nothing stores "your feed".

Q: How does one swipe run the serve funnel, and how does the event loop close back into features before the next fetch?

```mermaid
flowchart LR
    u(("🧑 user<br/>swipe UI + player<br/>batched telemetry sender"))
    gw["feed gateway<br/>recall → filter → coarse rank → fine rank → slate<br/>budget ~500 ms p99 · degrade to step-3"]
    emb[("🧲 ANN index<br/>video · user embeddings")]
    vd[("video meta<br/>title · ladder · perms")]
    cn[("engagement counters<br/>approximate · sharded")]
    ft[("feature store<br/>online · fresh ~5–30 s")]
    col[edge collectors<br/>durable accept · ack fast]
    k{{"📨 event log bus<br/>partition by user_id"}}
    ups[online feature updater<br/>events → fresh features]
    agg[streaming aggregator<br/>user.events → counter deltas]
    lake[("offline lake<br/>training · analytics")]
    cdn[/"🌍 CDN edge<br/>≥90% segment hits"/]
    ob[("🪣 origin blob store<br/>originals · renditions")]
    upl["upload svc<br/>chunked · resumable<br/>→ moderation gate → enqueue"]
    mod["async moderation review<br/>flags video → takedown (N7)"]
    xc["transcode farm<br/>per-title ladder"]
    pb["publish hook<br/>meta register · ANN upsert<br/>feature bootstrap"]

    u ==>|"1· GET /feed?page<br/>HTTPS · JSON"| gw
    gw -->|"ANN topK user emb"| emb
    gw -->|"GET title · ladder · perms"| vd
    gw -->|"GET fresh features"| ft
    gw -->|"GET counters"| cn
    u -->|"2· GET segments<br/>HTTPS · bytes"| cdn
    cdn -->|"GET segment:{id}"| ob
    u -->|"3· POST /v1/events<br/>HTTPS · JSON"| col
    col -.->|"produce: user.events"| k
    k -.->|"consume: user.events"| ups
    ups -->|"UPSERT features"| ft
    k -.->|"consume: user.events"| agg
    agg -.->|"apply counter deltas"| cn
    k -.->|"consume: user.events"| lake
    u -->|"4· POST /upload/sessions<br/>HTTPS · chunk bytes"| upl
    upl -->|"PUT original"| ob
    upl -.->|"produce: transcode.job"| tq{{"📨 transcode queue<br/>priority by predicted audience"}}
    tq -.->|"consume: transcode.job"| xc
    xc -->|"PUT ladder renditions"| ob
    upl -.->|"produce: video.publish"| k
    k -.->|"consume: video.publish"| pb
    xc -.->|"produce: rendition.ready"| k
    k -.->|"consume: rendition.ready"| pb
    pb -->|"register meta · ladder · perms"| vd
    pb -->|"UPSERT video embedding"| emb
    pb -->|"bootstrap item features"| ft
    mod -.->|"remove video meta"| vd
    mod -.->|"evict segment cache"| cdn
```

## Components

**Serve plane.**

- *Feed gateway*: owns the per-request deadline. Recall retrievers run in parallel;
  the funnel shortens under load (drops stages from the end, never the middle) —
  degrade from "step-5 slate" to "step-3 ordering" rather than to anything that
  ignores relevance.
- *Recall*: per-user candidate retrieval — an interest index over user embeddings,
  ANN over video embeddings, follow graph, trending, sound/hashtag indexes, rewatch
  candidates. Recall is cheap retrieval; it must not score, only retrieve.
- *Coarse rank*: a tiny model over cheap features shared across all candidates; sized
  from 02's arithmetic (~2B scores/s mean fleet-wide, estimates).
- *Fine rank*: the expensive model over the ~600 survivors, with item features cached
  in-process and user features computed once per request. Ranker internals are
  [Case 15](../15.ml-ranking-system/03-architecture.md)'s subject; this plane only
  owns the funnel boundaries and budgets.
- *Slate layer*: turns a ranked list into a feed — interleave by content type and
  creator, apply frequency caps ("user saw this author 50 times today"), inject the
  exploration pool (dive 5), enforce seen-item suppression against a per-user
  impression bloom/log.

**Ingest plane.**

- *Upload service*: chunked, resumable, idempotent upload sessions (mobile networks
  drop); stores the original in the blob store and enqueues transcoding.
- *Pre-publish moderation gate*: a cheap automated check that *blocks serving* — full
  moderation continues async with takedown recall (N7).
- *Transcode farm*: queue-driven workers producing the per-title ladder (dive 2);
  publish does not wait for it (dive 3). Priority is by predicted audience so the
  exploration burst always has renditions ready.
- *Publish hook*: registers video metadata, upserts the video embedding into the ANN
  index, bootstraps item features — the reason a 30-second-old video can be recalled.

**Observe plane.**

- *Collectors → event log bus* ([Case 13](../13.kafka/README.md)-class), partitioned
  by user id; three consumer families: online feature updater (the ≤5–30 s freshness
  budget), streaming aggregator for approximate counters, and the offline lake for
  training. Details in dive 4.

**Metadata substrate** (shared): video metadata service, sharded approximate counters,
item-feature cache, embedding indexes. Read-heavy and cache-served; counters are
eventually consistent by design.

## Data flows

1. **Swipe**: client → gateway → funnel → 10 IDs + rendition URLs + metadata → client
   player prefetches renditions from CDN edges ([Case 25](../25.mobile-newsfeed/README.md)
   owns that half). Every event generated flows to the observe plane within seconds and
   back into the feature store before the next fetch, most of the time (N2).
2. **Publish**: upload completes → moderation gate → *publish* (1st rendition + meta +
   index upsert) → exploration burst begins → transcode completes the ladder in the
   background → enrich renditions as they land.
3. **Takedown**: async review flags a video → removal propagates to meta service →
   feed caches and CDN eviction → next fetch never recalls it.

## What is *not* here, deliberately

No timeline store: [Case 02](../02.twitter/04-deep-dives.md) precomputes timelines
because followers define the audience; here candidate generation is the audience, and
attempting to materialize "a feed" per user would freeze a model that must react within
seconds (dive 1). The feed exists only for the duration of one request's funnel —
plus a short client-side session cache of next-up candidates, which is where any
precomputation actually lives.
