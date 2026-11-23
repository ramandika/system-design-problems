# 03 — Architecture

## High-level diagram

```
                                   SERVING PATH (p99 ~100 ms budget)
 client ──(user_id, context, surface)──▶ API/edge
                                            │
              ┌─────────────────────────────▼──────────────────────────────┐
              │  RETRIEVAL (candidate generation)  ~10 ms                  │
              │  ┌──────────────┐ ┌──────────────┐ ┌─────────────────────┐ │
              │  │ two-tower ANN│ │ graph/co-watch│ │ trending & fresh    │ │
              │  │ shard fleet  │ │ candidate DB │ │ source (<15min items│ │
              │  │ (HNSW or IVF)│ │ (item→items) │ │  not in ANN yet)    │ │
              │  └──────┬───────┘ └──────┬───────┘ └─────────┬───────────┘ │
              │         └────────────────┴──── merge/dedupe ─┘ → ~2,000 cands
              └─────────────────────────────┬───────────────────────────────┘
                                            │
              ┌─────────────────────────────▼──────────────────────────────┐
              │  FEATURE ASSEMBLY  ~10–15 ms                               │
              │  user features (profile store, per-request)                │
              │  item features: precomputed stats + realtime counters      │
              │  (mixed freshness tier, cached embeddings)                 │
              └─────────────────────────────┬───────────────────────────────┘
                                            │  batched feature vectors
              ┌─────────────────────────────▼──────────────────────────────┐
              │  HEAVY RANKER  ~20–40 ms                                   │
              │  DNN (multi-objective: watch, skip, like) over 2K cands    │
              └───────────────┬────────────────────────────────────────────┘
                              │  predicted scores
              ┌───────────────▼────────────────────────────────────────────┐
              │  RE-RANK + FILTERS  ~5–10 ms                               │
              │  policy filters (seen/dedupe/policy verdicts/do-not-rank)  │
              │  diversification (author/topic), freshness inject,         │
              │  exploration slot allocation (bandit), blend ads/relateds  │
              └───────────────┬────────────────────────────────────────────┘
                              ▼  ~50 ranked items (ordered, position-tagged)
                          back to client
```

```
                                   TRAINING LOOP (async, hours cadence)
 events (impressions, watch, skip — ~6M/s avg, ~23M peak)  ──▶ event log (~45 TB/day, by user_id)
                                               │  label joiner (waits out label delay)
                                               ▼
        training examples ──▶ trainer (daily/hourly retrain) ──▶ offline eval
                                               │                    │ compare vs serving
                                               ▼                    ▼
                                         model registry ◀── shadow/canary evaluation
                                               │
                       rollout: ANN index rebuild + embedding tables + ranker weights
                       (blue/green: new shards warm up before serving switches)
```

## Components

- **Retrieval fleet**: sharded ANN over item embeddings (two-tower retrieval) + non-ML
  candidate sources (follow graph / co-visit item→item lists, trending, fresh items from a
  stream-backed index). Each source returns a few hundred candidates in ~10 ms; the merger
  unions, dedupes and caps at ~2,000. Retrieval is judged on **recall** (does it contain
  what the ranker would pick), never on score quality (deep dives 1–2).
- **Feature assembly**: two-tier. *User tier* — one precomputed profile row per user
  (embedding, aggregate history), fetched once per request. *Item tier* — precomputed daily
  statistics plus a realtime counter layer (last-5-min/1-hour views and watch rate) plus
  cached item embeddings; gathered as batched multi-gets (~128–256 keys per batch). Served
  by a shared feature store so training and serving read the same definitions (offline/
  online skew is a failure mode, deep dive 3).
- **Heavy ranker**: the DNN, scoring the full ~2K candidate set batched. Multi-objective:
  predicts P(watch), expected watch-fraction, P(skip), P(like/share); a blend function turns
  these into one score. Runs on scoring fleets; per-request cost is the main
  cost/latency knob. A small **pre-ranker** (tiny model over cached features) may cut 2K →
  500 if capacity requires (deep dive 1).
- **Re-rank + filters**: deterministic and product-owned. Policy verdicts, dedup vs items
  the user already saw, author/topic diversification (e.g., no more than N from one author
  per page), exploration slot allocation, injection of fresh/trending items, final ordering
  pass. Cheap, fast, and the only place where "business rules" may override the model.
- **Event log & label joiner**: impression events carry position, model version, feature
  snapshot reference; watch/skip events join back to impressions by (user, item, session).
  The delay between impression and label is handled explicitly (deep dive 4).
- **Model registry & rollout**: retrained weights + embedding tables + ANN index ship
  together as one version; shadow-serve against traffic before switch; blue/green shard
  warm-up so index rebuilds never pause serving.
- **Experimentation harness**: normal A/B plus **interleaving** trials on ranker candidates
  (deep dive 6).

## Data flows

1. **Request → page**: edge → retrieval (parallel sources) → merge (~2K) → features (user
   row + batched item vectors) → ranker scores → filters/diversify → ordered 50, every
   impression logged with position + model version.
2. **Impression → label → training**: impression event (logged at serve time) and later
   watch/skip events (logged at play time) are joined by the label joiner; joined examples
   become next training window. Minutes of lag in labels, not seconds — architecture
   assumes it (deep dive 4).
3. **Events → realtime features**: the same event stream feeds the counter store so "views
   in last hour" moves within seconds of reality (deep dive 3).
4. **Training → serving**: registry ships ranker weights and embeddings; index rebuilds
   stream behind serving shards; rollouts are shadowed, canaried, and reversible.

## Failure posture

- Retrieval down → skip ML retrieval, serve recency × popularity × follow-graph page
  (fall back to something like [Twitter's](../02.twitter/03-architecture.md) pre-ranking
  timeline shape). Ranker down → same fallback. Feature store partial outage → rank with
  default/damped features rather than fail the page. Anything ML may degrade; the page
  itself must not.
