# Case 31 — Netflix (video-on-demand)

**The system**: a subscription VOD catalog — a fixed library of licensed and original
titles served to ~300M+ paying households (publicly reported; Netflix stopped quarterly
subscriber reporting in 2025), on every kind of device, in ~190 countries. Nobody uploads
anything. Every title is curated, licensed or produced, localized into dozens of languages,
and delivered from Netflix's *own* CDN.

**Why it's hard** — and how it differs from [Case 01 (YouTube)](../01.youtube/README.md),
which looks similar and is not:

- **Ingest is a pipeline, not a floodgate.** Studios deliver mezzanine masters in batches
  (a season drops, then gets dubbed into 30+ languages within days). Ingest volume is ~3.5
  orders of magnitude below YouTube's (measured in hours, ~3 in bytes), so the interesting problems shift from throughput to
  *quality, packaging correctness, and deadline-driven transcoding per title*.
- **Delivery is planned, not reactive.** A single Netflix CDN appliance is small (a 2U box
  of high-capacity disks, tens of TB raw — public OCA specs list up to ~120 TB), so Netflix *prepositions the library into ISP networks during off-peak
  hours* instead of racing to cache tomorrow's viral hit on demand. Demand forecasting
  replaces cache-miss luck (deep dive 2).
- **The catalog is small; the ranking surface is huge.** With only a few thousand titles per
  region (roughly — regional licensed subsets of a ~20K-title global catalog; 02),
  retrieval is easy — but Netflix instead pushes personalization into the
  UI itself (personalized rows, personalized artwork), and 80%+ of watch-hours are
  publicly reported to come from recommendations.
- **Revenue is the entitlement system.** Every playback checks a subscription, a plan's
  stream/quality limits, and a multi-DRM key server. Get it wrong in either direction: lost
  money, or friction that cancels accounts.
- **Resilience is a product feature.** Netflix treats chaos engineering in production
  (Chaos Monkey and successors, publicly documented for over a decade) as the main reason
  its control plane survives regional failure at all.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Reality-checked (Nov 2026)**: subscriber counts (~300M+ memberships), the Open Connect
CDN (publicly reported ~thousands of sites in ~175+ countries), the ~2 h/day viewing
figure, the 80%-from-recommendations figure (Gomez-Uribe & Hunt, ACM TMIS 2015), and the
chaos-engineering practice are public. Everything derived below is labeled arithmetic from
stated assumptions, not company data.

**Patterns taught here** (see [case index](../README.md)):
ingest/pipeline design, prepositioned vs on-demand CDN, adaptive bitrate, entitlements/DRM
as a distributed-systems surface, personalization over a small catalog, chaos engineering.

**Companion cases**: the client player itself (ABR algorithms, buffering, the browser
fight) is [Case 23](../23.frontend-video-streaming/README.md); generic CDN mechanics are
[Case 09](../09.cdn/README.md); ranking-funnel internals are [Case 15](../15.ml-ranking-system/README.md).

**Try it first**: before reading 03, sketch a design answering: (1) how does a 4-season
show, delivered as one giant master file, become playable in 190 countries with 30+ audio
languages inside a release window, (2) where do the bytes come from at Friday 9 PM, given
your appliances hold only tens of TB each, (3) how do you stop a subscriber's neighbor from
running a 12-screen pirate business on one $10 plan?
