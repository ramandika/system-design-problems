# 04 — Deep dives

## 1. Fanout on write vs fanout on read (THE dive)

**Option A — fanout on write** (materialize on post; reads = merge a list):
- wins: reads are O(1), always fast, no read-time query cost; reads still outnumber tweet
  writes ~2:1 (~1B reads/day vs ~600M tweets/day), and every read is latency-critical, so
  the cost sits where it matters.
- loses: write fanout is O(followers) — 100M inserts for a celebrity post; storage cost is
  O(sum of all followers) = ~180B timeline entries/day; deleted tweets need "delete everywhere
  that I inserted" (tombstones everywhere, eventually).

**Option B — fanout on read** (no materialization; reads = union of followed users' tweets):
- wins: write is O(1); celebrity tweets cost nothing extra.
- loses: read is O(following_count) per timeline read — a user following 1000 accounts
  triggers a massive read-time query, 60K QPS of timeline reads becomes an unacceptable
  fan-out to the tweet store; read cost scales with #reads, not #writes (worse when loads
  spike — exactly when you can't afford it).

**Chosen: hybrid.** Fanout-on-write for everyone below a follower-count threshold
(~99.999% of posts); celebrity authors skip the fanout and their recent tweets are merged
into readers' timelines on read. The merge window is bounded: on-read service looks up a
small fixed list of "high-follower authors you follow" and pulls their recent tweets.

- Trade-off made explicit: we accept slightly more complex read path (a merge + dedup) to
  bound the worst-case write fanout, and we accept a bounded memory cost on read for
  celebrity-followers only.

**As actually built (2012, publicly documented)**: fanout-on-write via a Fanout component
writing each tweet into a replicated Redis timeline cache keyed off recipient, pipelined
4,000 recipients per batch; celebrity tweets skipped fanout and were served on read by
query expansion against the **Earlybird** search index — the home timeline became a search
query like `home_timeline:raffi OR user_timeline:taylorswift13`, running >30K QPS alongside
300K+ QPS of poll-based timelines. Same hybrid split we chose; the real system used its
existing search index as the celebrity "recent tweets" index rather than a bespoke one.
Sources: [QCon SF 2012 slides](https://qconsf.com/sf2018/sf2012/dl/qcon-sanfran-2012/slides/RaffiKrikorian_TimelinesAtScale.pdf),
[Earlybird ICDE 2012 paper](https://cs.uwaterloo.ca/~jimmylin/publications/Busch_etal_ICDE2012.pdf).

## 2. Timeline cache design

One Redis cluster, consistent-hash sharded by user_id. Value = list of tweet IDs (compact
integer IDs, not objects) + a small per-user "last updated" pointer.

- **Multi-get timeline**: a read fetches O(1) Redis keys — a timeline entry is a pre-computed
  list, not per-follower queries.
- **Trim**: only the last ~800 entries per user are cached; older reads go to an
  on-demand/read path from tweet store filtered by followed users (slower, rate-limited).
- **Cache miss recovery**: on miss (new/cold user), a rebuild job fans out from the social
  graph service — expensive (O(followers)) but rate-limited so it doesn't stampede; the
  timeline read for that user waits (degraded read latency) or is served empty-and-fast
  while the rebuild runs.
- **Consistency**: not transactionally consistent — a follower might briefly see an empty
  window if their timeline was mid-rebuild; acceptable.

## 3. Tweet ID: Snowflake-style time-sortable IDs

Naive `AUTO_INCREMENT` in a single DB cannot shard. Twitter uses **Snowflake**:
64-bit = timestamp(ms) | machine-id | sequence-per-ms. IDs roughly sort by creation time.

- Why it matters: reverse-chron timelines and pagination are expressed as "entry IDs >
  last seen ID" — sortability by ID is everything for the merge/dedup/page logic.
- **Trade-off**: IDs are not random (security: timing/tweet-count inference possible) vs
  the ordering guarantee. We pick ordering; random IDs would need a secondary sort field.
- Machine-id uniqueness is via ZooKeeper / manual assignment; clock skew between generators
  shows up as slightly out-of-order IDs — acceptable at ~ms level.

## 4. Deleting tweets across millions of timelines

Once fanned out, a tweet ID lives in ~followers' Redis timelines. Options:

- **(a) Tombstone in hydrate**: keep the tweet in timelines but have hydration return
  "deleted" and the read service filter it out. Wins: no write storm at delete time.
  Loses: zombie pointers until timelines trim naturally.
- **(b) Reverse-insert deletes**: find every timeline that has this tweet ID and remove it.
  Needs O(followers) work per delete — same as the original fanout, which is only cheap for
  low-follower authors; for celebrities this is as bad as the original problem.

**Chosen: (a) for celebrities and (b)+(a) combined for normal users** (real deletion from
recently-touched timelines; tombstones for cold ones). Trade-off accepted: some deleted
tweets linger as tombstones until timelines rotate, and that's fine — nobody can tell.

## 5. Read-your-own-writes vs eventual follower consistency

You must instantly see your own post. Followers may see it up to seconds later. Design:

- Post handler writes to tweet store synchronously (authoritative), returns to author.
- Author's own timeline render, for a short TTL window after a post, layers "your own
  recent tweets directly from store" on top of the cache-derived timeline.
- Followers' timelines get the entry asynchronously (fanout workers).

**Trade-off**: complexity of a "merge with fresh store data" path on the hot read route,
in exchange for a user-perceptible guarantee. Worth it: posting into a void is a killer bug.

## 6. Rate limiting & abuse

Bots, scraping, spam floods. Limits per-account (post/hour), per-IP, and per-follower-graph
(an account that suddenly follows 10K people looks like spam). Rate limits apply BEFORE
fanout: a spam tweet that gets 10K followers is bad, but a spam tweet from a bot that gets
10K followers via follows is exponentially worse — the follower graph itself is rate-limited
and audited asynchronously.
