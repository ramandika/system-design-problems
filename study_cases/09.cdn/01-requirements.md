# 01 — Requirements

## Scope decisions (estimated design targets)

A multi-customer CDN serving a broad mix of sites and APIs (media, e-commerce, software
distribution, SaaS front pages). Not a single-tenant video CDN — that is YouTube's shape
([01.youtube](../01.youtube/04-deep-dives.md)); multi-tenancy is a requirement here, which
changes cache design more than anything else (see deep dive 2).

## Functional requirements

**Data path (per request, must all happen in-line):**

- F1. **Anycast routing to the nearest healthy POP**, with automatic re-routing around
  POP/regional failure (see deep dive 1).
- F2. **TLS termination at the edge**: customer-managed and managed certificates, SNI-based
  routing, HTTP/2 and HTTP/3 (QUIC) negotiation, session resumption.
- F3. **Caching for standard HTTP semantics**: respect/re-interpret `Cache-Control`,
  `ETag`/`Last-Modified` conditional requests, `Vary` handling, stale-while-revalidate,
  per-customer overrides of all of it.
- F4. **Purge API**: individual URL purge, tag/surrogate-key purge (one API call invalidates
  everything tagged), "purge everything" per customer (see deep dive 4).
- F5. **Proxying non-cacheable traffic (dynamic acceleration)**: POST/PUT/preflight-free
  pass-through, connection reuse to origin, routing over the CDN's own backbone between
  POP and origin region.
- F6. **Cache-control customization**: customer-defined cache keys (strip/keep query params,
  cookie/header segmentation), signed URLs (auth-token in URL expiring without origin hit),
  per-response headers.
- F7. **DDoS/WAF protection in-line**: L3/4 volumetric absorption, L7 rate limiting per
  (IP, customer, path), bot verification challenges (see deep dive 6).
- F8. **Request/edge logs and analytics**: per-customer log delivery near-real-time; billing
  derived from it (billing is why logs are on the critical path).

**Control plane (not per-request):**

- F9. Customer config (domains, rules, cache settings) pushed to all POPs in seconds.
- F10. Certificate issuance and rotation (ACME-style) and deployment of custom certs.

## Non-functional requirements

| Dimension | Target | Why |
|---|---|---|
| Edge overhead on cache HIT | p99 < ~10 ms added latency | the product *is* lower latency |
| Edge overhead on MISS (proxying) | p99 < ~50 ms added beyond origin RTT | indirection must stay cheap |
| Cache hit rate (static, well-configured) | ≥ 95% of requests, ≥ 85% of bytes | economics; see deep dive 2 |
| Invalidation propagation | globally < ~30 s (target: single-digit seconds end-to-end for a hot-content purge, push-fanout bound ~5 s, est.) | correctness promise; deep dive 4 |
| Availability | ≥ 99.99% edge availability; POP failure must not break the service | anycast + health-based de-routing |
| Peak absorbable request volume | tens of millions of req/s fleet-wide; ~200M+ req/s L7 floods without origin impact (publicly reported ~200M req/s HTTP-flood mitigations, vendor-reported) | deep dive 6 |
| Certificate rotation | zero-downtime, automated | TLS is the entry gate |

## Explicitly OUT of scope

- **Edge compute / functions at the POP** (Cloudflare Workers-style): a real product for
  real vendors, but a different system — scheduling tenant code safely at the edge is its
  own case. We only forward non-cached traffic.
- **Video-transcoding / packaging**: the CDN consumes ABR ladders produced upstream (see
  [YouTube](../01.youtube/04-deep-dives.md)); we do not transcode.
- **Private backbone construction and PoP-to-PoP circuit engineering**: assumed given; we
  design routing and cache behavior over it, not its physics.
- **DNS authoritative service** as a product (DNS-based steering is used internally where
  noted, but we don't design a DNS platform).
- **Multi-cloud origin storage** (the blob-store side of origins): treated as a black box
  reachable via HTTP.
