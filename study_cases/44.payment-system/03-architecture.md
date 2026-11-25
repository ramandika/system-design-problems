# 03 — High-level architecture

## Diagram

*Q: How does one charge intent get authorized, posted, and reconciled — and what sits at the seam to the banks?*

```mermaid
flowchart LR
    m((🧑 merchant)) ==>|"1· POST /v1/charges<br/>HTTPS · JSON"| gw[api-gateway<br/>TLS termination · idempotency-key interception]
    gw -->|"2· GET idem:key · INSERT idem:key"| idem[(idempotency store<br/>key → result/in-flight · TTL ~24 h)]
    gw -->|"3· gRPC CreateIntent()<br/>gRPC · proto"| pay[payments service<br/>create/confirm → authorize → capture<br/>per-intent CAS transitions · gateway adapters]
    pay -->|"4· gRPC PreCheck()<br/>gRPC · proto"| rk[risk service<br/>velocity · rules+model<br/>≤ ~150 ms inline]
    pay -->|"5· gRPC Authorize()<br/>gRPC · proto"| ga[gateway adapters<br/>per-rail protocol + credentials<br/>timeouts · uniqueness ref to bank]
    ga ==>|"6· Auth request + uniqueness ref<br/>per-rail HTTPS/ISO 8583"| nw[/acquirers + card networks<br/>per-rail · per-acquirer/]
    ga -.->|"status inquiry<br/>per-rail HTTPS/ISO 8583"| nw
    sw[expiry sweeper<br/>re-auth → release hold] -.->|"re-auth / release hold<br/>gRPC · proto"| pay
    pay -->|"7· INSERT postings"| led[(ledger<br/>immutable double-entry postings<br/>account-sharded · balance rollups)]
    pay -.->|"produce: charge.succeeded"| k{{📨 Kafka<br/>payment events}}
    k -.->|"consume: charge.succeeded"| wk[webhooks service<br/>signed events · retry w/ backoff]
    wk -->|"POST /webhooks/charge<br/>signed · HTTPS · JSON"| m
    nw -.->|"PUT /settlement/{acquirer}/{date}<br/>SFTP · CSV"| rec[reconciliation<br/>match settlement files → breaks queue]
    rec -->|"SQL match by scheme-ref + amount<br/>INSERT correction postings"| led
    led -->|"SQL payout-eligibility view"| po[payouts<br/>bank payouts · statement match]
    cu((🧑 cardholder)) -.->|"card capture (client-side)<br/>HTTPS · JSON"| vl[(🔑 vault<br/>token ↔ PAN · isolated credentials<br/>no bulk read path)]
```

## Components

| Component | Job | Key property |
|---|---|---|
| API gateway | auth, rate limit, **intercepts idempotency keys** | the first dedup gate; replays served or joined in-flight (deep dives §1) |
| Payments service | payment state machine, CAS transitions, coordinates gateways | the only writer of payment state |
| Idempotency store | key → first-request result / in-flight marker | unique index on key; TTL ~24 h (deep dives §1) |
| Risk service | inline velocity/rules/model ≤ ~150 ms; async review queue | failing open = fraud; failing closed = lost sales — budget-bound (deep dives §5) |
| Gateway adapters | per-acquirer/per-rail protocol + credentials + timeouts | the *distributed-system seam*: timeouts, ambiguity, idempotency-to-the-bank |
| Ledger | immutable double-entry postings, account-sharded; balance rollups derived | the source of truth; corrections are postings, never edits (deep dives §2) |
| Tokenization vault | token ↔ PAN, isolated credentials, no bulk read path | PCI scope isolation (deep dives §6) |
| Webhooks service | signed events, exponential retry, endpoint health | at-least-once; merchant-acknowledged |
| Reconciliation | ingests settlement files, matches, raises breaks | correctness backstop (deep dives §4) |
| Payouts | funded sweep accounts → bank payouts, with statement match | ledger-funded, externally reconciled |

## Key data flows

**Charge (the flagship flow)**: gateway validates + intercepts idempotency key →
payments service: intent created (or replay/join) → risk check inline (~150 ms budget) →
gateway adapter calls acquirer (auth-only) → on response: ledger postings (authorized hold on
the merchant's pending account; fees anticipated) + state CAS `processing → authorized` →
webhook `charge.succeeded` queued. Timeout/ambiguity → the adapter's *uniqueness key to the
bank* lets a retry be idempotent against the issuer too (deep dives §3); the intent stays
`processing` with a pending-resolution flag until the ambiguity is cleared by inquiry or
reconciliation — never silently re-attempted.

**Capture/void**: merchant captures within the auth hold window (~5–7 days, ballpark) —
capture postings move pending→settled-pending; void before capture cancels the hold (no
network fee), refund after capture creates refund postings + triggers rail refund.

**Reconciliation (daily)**: acquirer settlement file → match against ledger by scheme
reference + amount + date-window → breaks classified (timing / FX rounding / missing) →
breaks queue with owners and SLAs → correction postings *as new rows* when resolved.

## Where the money is

- Not bandwidth, not fleet size: **the coupling seam to banks** (deep dives §3) and the
  idempotency/ledger pair (deep dives §1–2) are where correctness is won or lost.
- Latency budget math: of ~500 ms internal envelope, risk gets ≤ ~150 ms, gateway
  orchestration ~50 ms — the rest is yours to spend on the *ambiguity-handling* job that
  real payment systems live in.
