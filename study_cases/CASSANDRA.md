# Cassandra across the case catalog

Where Cassandra (and the ScyllaDB/Dynamo-class wide-column store family) appears in the
50 study cases, what role it plays, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md), which names the wide-column store as one of the
four "storage pillars" of the corpus (with blob store, relational DB, KV cache tier).

Headline: **no case studies Cassandra itself — every mention routes through one documented
deployment, Discord's.** The recurring shape is tight: *a wide-column store whose partition
key is the domain query (channel + time bucket, post + path), IDs chosen to be sortable so
reads are scans, quorum reads and writes, and the application layer owning routing and
coalescing.* The other recurring role is smaller: the write-heavy TTL'd store that "fails
small". The LSM storage engine underneath this family is not covered here — that belongs to
[12.bigtable](12.bigtable/README.md).

- Cases that mention Cassandra/ScyllaDB/wide-column/Dynamo-class stores: **7 / 50**
- Designed architectural component: **3** (05, 34, 39)
- Option mention (named, not chosen as the highlighted winner): **1** (28)
- Documented analogue or lineage, by reference: **3** (03, 30, 12)
- Studied as the case itself: **0** (nothing models Cassandra end-to-end; 05 is the
  closest, and its store choice is one deep dive among six)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [05.discord](05.discord/README.md) | Trillion-row message store: partition = channel + time bucket, snowflake IDs | Deep (the reference deployment) |
| [34.reddit](34.reddit/README.md) | Comment side-store: one partition per post, materialized path, prefix scans | Core component |
| [39.e-commerce](39.e-commerce/README.md) | Cart store (DynamoDB/Redis-class): TTL'd, write-behind, fails small | Core component (chosen option) |
| [28.uber](28.uber/README.md) | Trip DB option: wide-column or relational, by city + time | Brief |
| [03.whatsapp](03.whatsapp/README.md) | Discord's store cited as the documented analogue for a non-public message DB | By-reference |
| [30.facebook-messenger](30.facebook-messenger/README.md) | One-line cross-ref: Discord's migration = the same bytes-amplification lesson | Brief cross-ref |
| [12.bigtable](12.bigtable/README.md) | Lineage (Bigtable paper → Cassandra/HBase/ScyllaDB) and its counter-position | By-reference |

## Per-case notes

### The reference deployment

**[05.discord](05.discord/README.md)** — the only case that sizes and decides a
Cassandra-class store at trillion-row scale. The message store's partition design
([04-deep-dives.md](05.discord/04-deep-dives.md), dive 5) is the case's own Option B:
partition key = `channel_id` + static time bucket, rows ordered by snowflake-style
timestamp-sorted IDs. Option A (partition = `channel_id` alone) loses for a stated reason:
a ten-year-old public guild puts billions of rows in one partition, which eventually
outgrows any node and degrades the whole cluster's compaction/repair. Option B wins bounded
partitions (old buckets go cold and become the storage-tiering unit), loses a two-level
routing layer and occasional cross-bucket pagination reads — absorbed by a "data-services"
layer (one gRPC endpoint per query type, consistent-hash-routed so a channel's traffic hits
one instance, request coalescing of n same-row readers into one DB query). The chosen shape
is explicitly the one Discord publicly uses (Cassandra/ScyllaDB, quorum reads and writes).
The migration is the anchor statistic, labeled as Discord's own measurements and not an
independent benchmark ([03.whatsapp §3](../03.whatsapp/04-deep-dives.md) repeats it
caveated): May 2022, Cassandra 177 nodes → ScyllaDB 72 nodes (a ~2.5× *shrink*), p99
historical reads 40–125 ms → 15 ms, p99 inserts steady ~5 ms; the README cites a >1
trillion row move. Retention is "forever, tiered": recent buckets on the fast store, cold
buckets on cheaper storage, the bucket key is the tiering unit.

### Core component: wide-column as the derived store

**[34.reddit](34.reddit/README.md)** — Cassandra appears as a *side-store*, not a system
of record. Postgres keeps comment truth; a Cassandra partition per post holds comment rows
keyed by a materialized path (`(post_id, path)` prefix-contiguous), so every render query
the UI performs — "children of this node", first N pages of a branch — is one prefix scan
([04-deep-dives.md](34.reddit/04-deep-dives.md), dive 1). This mirrors what Reddit
publicly did (moving comments into Cassandra keyed by per-comment paths, per their 2015–20
blog posts and talks). Write-ahead discipline: Postgres first, Cassandra asynchronously
reconciled. Named failure modes: a viral post is a hot partition (every comment write of
one thread lands on one partition) — mitigated by service-layer write batching per post,
not by re-sharding comments, because that destroys the prefix scan; and cache stampedes on
sort-version bump hammering Cassandra with miss traffic. Vote counters get an
"eventually-per-second" aggregation note (Redis HINCRBY or Cassandra counters).

**[39.e-commerce](39.e-commerce/README.md)** — the other recurring role: a Dynamo-class
store for state that must fail cheap. Carts land in a "DynamoDB/Redis-class" store, TTL'd
by access, with a write-behind event stream for analytics ([04-deep-dives.md]
(39.e-commerce/04-deep-dives.md), cart deep dive). Chosen over mutating rows in the
relational core because cart churn (~150K/s concentrated at flash bursts) mostly touches
objects that expire worthless — TTL handles abandonment for free, and the store fails
*small*: a lost guest cart costs a re-add. The limit is stated and designed around: weaker
durability is unacceptable for checkout (which never trusts the cart store alone) and for
logged-in users (whose account cart is mirrored to the order-core database — a lost
logged-in cart is a trust failure, not a UX failure).

### Brief mention

**[28.uber](28.uber/README.md)** — the trip database, which must answer "reconstruct trip"
after app-reopen, is named as "quorum-replicated [Case 12](../12.bigtable/README.md)-style
wide-column or plain relational by city+time" ([03-architecture.md]
(28.uber/03-architecture.md)) — a partition-key-is-query design note in one sentence, with
the choice left open.

### Analogue / by-reference

- **[03.whatsapp](03.whatsapp/README.md)** — WhatsApp's message store is not public, so
  the case argues through the best-documented comparable: Discord's wide-column store
  (partition key = channel + static time bucket, sortable snowflake IDs, quorum
  reads/writes, the 177→72-node migration, the request-coalescing data-services layer,
  [04-deep-dives.md](03.whatsapp/04-deep-dives.md) §3). Two transferable claims are pulled
  out: the routing/coalescing shim in front of the store, and that the migration *shrank*
  the fleet while cutting p99 — a stated argument against "just add Cassandra nodes".
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — one-line cross-reference:
  a corpus growing ~70 PB/year of small rows cannot live on a generic KV store with 3×
  replication of hot data, and Messenger's public HBase → MyRocks trajectory is the same
  fight Discord's Cassandra→ScyllaDB migration taught at trillion-message scale
  ([02-scale-estimates.md](30.facebook-messenger/02-scale-estimates.md)).
- **[12.bigtable](12.bigtable/README.md)** — the family's ancestor: the 2006 Bigtable
  paper is "the ancestor of Cassandra, HBase, ScyllaDB and the wide-column family"
  ([README.md](12.bigtable/README.md)), with WhatsApp §3 named as this corpus's operational
  descendant. The LSM engine and its write path are 12's subject. 12 also draws the
  family's counter-position twice ([04-deep-dives.md](12.bigtable/04-deep-dives.md)):
  Dynamo-style stores are the "LSM-with-hash-partitioning family" — engine-enforced
  balancing bought with sorted-range scans — and Bigtable rejects quorum writes across
  regions where Discord's store accepts quorum reads *and* writes at p99 ~5 ms, explicitly
  to show "which side of the trade you pick is a function of the money you'll spend and
  the RPO you can absorb, not a universal best".

## The portable pattern

Every substantive use reduces to one contract, with [05.discord](05.discord/README.md) as
the fullest worked example:

1. **The partition key is a query, not a table.** Partition = channel + time bucket (05),
   post ID + materialized path (34), city + time (28). The wide-column store is queried
   exactly the way its key was designed; anything else goes to a relational system of
   record.
2. **Bound the partition from day one.** Domain keys that look bounded are not (a
   ten-year-old guild, a viral post): add the time bucket to the key (05) or accept a
   hot-partition failure mode and mitigate upstream (34's per-post write batching) — never
   by re-sharding, which destroys the locality the key bought.
3. **Sortable IDs turn pagination into arithmetic.** Snowflake-style timestamp-sorted IDs
   (05, cross-referenced to [02.twitter §3](../02.twitter/04-deep-dives.md)) make
   "everything after X" a scan, not a sort.
4. **Quorum reads and writes, and application-side routing.** The store is AP-flavored but
   per-consistency-level tunable (quorum both ways in 05; counters "eventually per second"
   in 34); the application owns the routing shim (consistent-hash per key instance,
   request coalescing) and the reconciliation from the system of record (34).
5. **Fail-small state goes to the family too.** TTL'd, write-behind, fails small (39) —
   with the durability line drawn by what a loss costs (guest cart = UX, logged-in cart =
   trust failure).
6. **Hot partitions are where the shape ends** — the same boundary Kafka's log hits in
   [13.kafka](13.kafka/README.md) and ticketing in
   [37.ticketmaster](37.ticketmaster/README.md): default key partitioning cannot spread one converging key, so the fix must
   come from batching, bucketing, or the key design itself. And for the engine-level
   write-path question underneath all of this, see [12.bigtable](12.bigtable/README.md).
