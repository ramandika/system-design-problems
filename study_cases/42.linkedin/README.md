# Case 42 — LinkedIn

**The system**: a professional network — member profiles, a *symmetric* connection graph,
degree-bounded traversal ("you know who through whom"), a professional news feed,
recommendations (People You May Know, jobs), and Who's Viewed Your Profile. Sounds like
"Twitter plus profiles"; almost every hard problem here is different from
[Case 02 — Twitter](../02.twitter/README.md).

**Why it's hard**:

- The graph is **undirected and stateful**: edges require the *other person's consent* —
  a pending/accept state machine Twitter never needs (a follow is one-sided and instant).
- Traversal is **degree-bounded, not broadcast**: the product constantly asks "who is a
  2nd-degree contact, through whom?" — a hop computation on a ~120B-edge graph, which is
  a completely different workload from fanning a tweet to followers.
- **Profiles are read-dominated and per-viewer permissioned**: profile reads dwarf profile
  writes by orders of magnitude, and what a viewer may see depends on degrees and privacy
  mode — the ACL must be checked without making the cache per-viewer.
- The feed is not reverse-chron: it is a multi-stage ranking funnel over signals (job
  changes, endorsements, comments) that are themselves event pipelines.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: symmetric-edge storage and dist-2 precompute vs on-demand hops,
read-path scaling for per-viewer ACLs, state-machine writes with idempotency, multi-stage
retrieval-then-rank, real-time event pipelines (Kafka lineage — see [13.kafka](../13.kafka/README.md)).

**Try it first**: before reading 03, answer: (1) how do you compute "2nd degree" for a
member with 500 connections *and* 30,000 connections — same algorithm? (2) two members tap
"Connect" at each other within a second — what happens? (3) a Private-mode member views
someone's profile — what do you log, store, and show?
