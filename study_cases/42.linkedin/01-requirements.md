# 01 — Requirements

## Functional

Core v1:

- **Profile**: work history, education, skills, endorsements, headline, photo. Publicly
  viewable parts + member-controlled visibility per section. Read-dominated (see estimates).
- **Connections**: an edge is *symmetric and consented*. Lifecycle: A sends a connection
  request → B sees it as pending → accept / decline / ignore / withdraw; acceptance creates
  a two-way edge instantly. Product limit (publicly documented): **30,000 1st-degree
  connections max**.
- **Degree-bounded graph queries**: "2nd degree" (friends-of-friends, with the connecting
  path), "3rd degree", "People You May Know" candidates, "who in my network works at X".
  Only 1st–3rd degree are ever surfaced — no open-ended BFS.
- **Professional feed**: not reverse-chron. Mix of posts from your network, shared jobs,
  suggested content, comments with reply threading. Ranked (see
  [15.ml-ranking-system](../15.ml-ranking-system/README.md) for the generic funnel; this
  case focuses on the *signal surface*).
- **Who's Viewed Your Profile (WVYP)**: list of who viewed you (unless they browse in
  Private mode), weekly aggregates, "your profile appeared in N searches".
- **Member search / directory**: find people by company/title/school; results annotated
  with degree (1st / 2nd / 3rd+).

Out of scope v1: jobs marketplace internals, messaging (→ [30.facebook-messenger](../30.facebook-messenger/README.md)),
groups, ads bidding (ads as *candidates* in the feed only), Recruiter tooling.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Profile read latency | p99 < ~200 ms incl. ACL | drives read cache + degree-ACL-at-edge design |
| Connection accept → edge live | < 1 s, atomic | both members must see the new 1st-degree edge immediately |
| Feed request latency | p99 ≤ ~500 ms incl. ranking | budget split: retrieval ~ms, ranking is the risk (deep dives) |
| Graph query latency | 2nd-degree answer < ~100 ms | dist-2 cannot be computed hot for heavy users → precompute |
| Availability | profile/search: very high; invite: ordinary; WVYP: can lag/degrade | asymmetric SLOs by user-perceived pain |
| Consistency | connections read-your-writes + atomic; aggregates (degrees, PYMK, WVYP) eventually consistent (minutes-hours) | classic split: state machine vs derived data |
| Durability | profiles/edges: no loss (replicated authoritative store); derived stores (dist-2, PYMK, WVYP) rebuildable from events | authoritative vs derived split (scope decision 2) makes durability cheap where it matters |
| Privacy | degree-gated visibility is a *correctness* requirement, not a cache nicety | leaking contact info to a 3rd-degree viewer is a legal/user-trust incident |
| Scale | ~1B registered members (publicly reported ~1B since 2023) | drives the estimate sheet |

## Scope decisions that shape everything

1. **Degrees are bounded by design**: no arbitrary-depth traversal ever runs in the online
   path. The product only needs 1/2/3, and 2nd-degree is *precomputed* (aggregates + sample
   paths), not searched at read time. This is the core delta vs Twitter's broadcast fanout.
2. **Edge state machine is authoritative and synchronous**; everything the graph touches
   downstream (dist-2 caches, PYMK, feed ranker features) is *derived* and updated
   asynchronously via events ([13.kafka](../13.kafka/README.md)).
3. **ACLs are evaluated per request against cached canonical profile data** — never cached
   per-viewer, otherwise the privacy model forks the cache fleet (deep dives §3).
