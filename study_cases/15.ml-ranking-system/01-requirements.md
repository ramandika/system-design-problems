# 01 — Requirements

## What the system does (functional)

- **Rank a page**: given `user_id` + context (device, time, session history, surface:
  home feed / search results / related-items rail), return an ordered list of ~50 items
  (videos, posts) from a ~500M-item corpus, in one request.
- **Multi-surface ranking**: the same ranker family serves home feed, search result
  re-ranking (query as part of the retrieval context) and related-items ("watch next").
  Same funnel, different retrieval sources and blending rules.
- **Freshness**: an item published < 15 min ago is eligible to rank; if it is trending,
  it can reach the head of a relevant user's feed within ~10 min of going viral.
- **Business/policy filters are first-class**: do-not-recommend lists, seen-and-skipped
  demotion, dedup (never show the same item twice in a session), region/age-appropriateness,
  author diversity — enforced at re-rank, on every request, not best-effort.
- **Closed feedback loop**: impression, click/watch-start, watch-time/watch-fraction, skip,
  like, share events are logged with position + model version, joined into labels, and the
  model is retrained on a fixed cadence.
- **Exploration**: a controlled fraction of slots is used to gather feedback on uncertain
  items (new videos, new creators) — a product requirement, not an ML nicety (deep dive 5).

## Explicitly OUT of scope

- **Video delivery/CDN, transcoding, upload pipeline** — see
  [01.youtube deep dives](../01.youtube/04-deep-dives.md); the ranker ranks URLs.
- **Training-infrastructure internals** (feature pipelines, model registry mechanics,
  distributed training): this case treats them as services with SLAs; the serving-side
  consequences are covered in [deep dive 3](04-deep-dives.md#3-feature-freshness-realtime-counters-vs-precomputed-features);
  the production-ML infra itself is [case 14](../14.ml-production-system/04-deep-dives.md).
- **Spam/authenticity classification and ad ranking**: upstream policy models run before
  us and ads run in parallel slots; we only honor their verdicts as filters.
- **Real-time federated / on-device inference**: server-side ranking only; client work is
  limited to sending context and rendering.
- **Long-term preference extraction beyond embeddings**: we consume a user embedding
  produced elsewhere; how it's trained is out of scope.

## Non-functional requirements (with the numbers 02 scales from)

- **Latency**: end-to-end ranking ≤ 100 ms p99, split ~10 ms retrieval, ~10 ms feature
  fetch, ~25–50 ms ranker+re-rank, rest queue/headroom. p50 ≈ 30–50 ms.
- **Throughput**: ~500M DAU, ~46K ranking QPS average (4B/day ÷ 86,400 s), ~200K QPS peak (~4×; justified in 02).
- **Recall quality (not just latency)**: the top-k from retrieval must contain essentially
  everything the top-50 would have chosen — retrieval recall@2K measured offline ≥ 95%
  (an engineering target, labeled as our target, not a public benchmark).
- **Availability**: graceful degradation always — if the ranker or feature store fails,
  fall back to a cheap fallback ranker (recency × popularity × follow-graph) rather than
  an empty feed; ranking is part of the product's core read path, feed down = product down.
- **Model updates without pause**: new model deploys and daily re-trainings go out with
  shadow evaluation and canary; serving never stops to "rebuild the index" — new shards
  warm up behind the old ones.
- **Determinism for experimentation**: a request's model version and feature values are
  logged so any impression can be replayed/debugged (needed for the interleaving/A/B
  machinery in deep dive 6).

## Scope decisions (what we deliberately choose to be)

- We are a **recommender for a large corpus with rich interaction history**; cold-start is
  a bounded problem (new creators), handled in the retrieval blending, not a separate system.
- We optimize a **composite objective** (watch-completion × diversity × policy), expressed
  as a multi-objective ranker; the product decides the mixture weights, the system makes
  them per-request tunable.
- We accept **eventual consistency in labels** (watch events land minutes later) and design
  around the delay (deep dive 4) rather than promising fresh labels.
