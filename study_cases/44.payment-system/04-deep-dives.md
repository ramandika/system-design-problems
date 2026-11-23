# 04 — Deep dives

## 1. Exactly-once money effects: idempotency keys + serialization (THE dive)

Networks retry; our systems retry (and *must*, after an ambiguous timeout); merchants' code
retries on their own schedule. Every path that can reach the bank twice must be closed at the
write path, not by post-hoc detection.

**Option A — dedupe by request fingerprint (hash of params + customer)**:
- wins: no client cooperation; works for accidental double-submits.
- loses: a legitimate retry after *any* change (recomputed amount, reordered fields) is a
  *different* fingerprint → duplicate charge; a genuinely different charge that coincidentally
  matches (same customer, same amount, same second — a merchant processing two gumballs) is
  wrongly suppressed. Fingerprinting guesses; keys declare intent.

**Option B — client-supplied idempotency key, first-writer-wins, replay semantics**:
- wins: retries *legally* replay the stored result; concurrent duplicates are serialized;
  the key is proof of intent, not a guess.
- loses: key lifecycle must be designed (TTL, collision policy, cross-request meaning) and
  the storage must handle create-vs-replay races under load.

**Chosen: Option B.** Mechanics:

- Gateway intercepts the key → `GET idempotency[key]`: terminal result? replay it verbatim
  (same status, same response body) for the TTL (~24 h). InFlight marker? join and await the
  original's outcome (HTTP 409 or block-then-replay, per API contract). Miss? `INSERT` with a
  unique index on key — lost inserts mean you lost the race, join the winner.
- Same key, *different* request body than the original → **explicit error**, never silent
  reinterpretation (a silent path here is how a changed amount gets charged once or twice at
  the wrong value).
- Server crash mid-auth: the intent stays `processing`; recovery job inquires with the
  acquirer using the *same* bank-side uniqueness reference (adapter-level idempotency —
  deep dives §3) → found charged → promote to authorized; truly rejected → mark failed and
  replay that result.
- Concurrency inside one payment's lifecycle (capture racing a merchant-initiated void) is
  CAS-guarded state transitions: `UPDATE ... WHERE status='authorized'`, affected-rows gates
  the ledger postings — the same single-writer-CAS pattern as
  [42.linkedin/04-deep-dives.md](../42.linkedin/04-deep-dives.md) §2's invite accept.

**Failure modes**: key generated per-request by a broken client SDK → the protection vanishes
(client-side keygen is a documented contract); replay cache cold (TTL expired) during a
merchant's delayed retry → falls through as a new charge — so the TTL must exceed any
sane retry window and the API docs must say so; idempotency store outage → **reject writes
conservatively** (503), never process unkeyed.

## 2. Double-entry ledger: balances are queries, not columns

**Option A — mutable balance per account (`UPDATE accounts SET balance = balance - 100`)**:
- wins: trivially fast reads; tiny schema.
- loses: no history, no reconstruction, no audit answer for "why are we $12.40 off against
  the acquirer in March"; serialization by row-lock on hot accounts (a busy platform account
  becomes a global mutex); a bug (or an admin UPDATE) is *unprovable after the fact*.

**Option B — immutable journal of double-entry postings + derived balances (chosen)**:
- every movement = ≥2 legs (debit/credit across named accounts) in one transaction; balances
  are materialized rollups (per account, updated by the same txn or a stream job); everything
  reconstructs by replay; corrections are *new reversing postings*; per-account sharding with
  global-consistency checkable by construction (sum of debits = sum of credits, always).
- loses: storage grows (fine: ~110 TB/yr raw, est. sheet); "current balance" is eventually
  consistent for aggregates (read-own-writes for the acting caller from the same txn;
  dashboards lag seconds); rollup rebuild jobs to operate.

**Chosen: Option B.** The day you need it (dispute, audit, a reconciliation break, an
engineer's bug at 3 a.m.) is the day Option A's ledger is worthless. Postings reference the
payment intent + the gateway response + the scheme reference — a single row answers "who, what
bank artifact, when, why."

**Failure modes**: rollup cache stale → merchant sees old balance (acceptable, disclosed);
ledger partition down → that account's writes stop (money *pauses*, it does not go missing);
someone "fixes" a posting in place → rejected by design (append-only store + checksummed
chain per account).

## 3. Auth → capture → settlement: a saga across rails you don't control

The state machine: `created → authorized → (captured | voided | expired) → settling →
settled`, with `refunded`, `disputed` branching after capture. Each transition crosses a
boundary we don't transact over.

- **No 2PC across banks — by fiat and by physics**: no issuer will lock an account pending a
  distributed-commit from our side. The only viable protocol is saga + compensations:
  `void` compensates a not-yet-captured auth (no network fee); `refund` compensates a capture
  (fees partially lost — so design for *pre-capture cancellation* whenever possible, i.e.,
  make merchants capture late).
- **Auth holds expire**: card-network auth lifetime ~5–7 days (ballpark) → capture-after-
  expiry is a decline; an expiry sweeper re-auths (with the cardholder's standing permission,
  per network rules) or releases the hold. The sweeper is a correctness component, not a
  cron-job nicety.
- **Ambiguous gateway calls**: timeout on the acquirer does *not* mean failure — it means
  unknown. The adapter sends a uniqueness reference (our payment ID) that the acquirer
  de-dupes on retry, and a resolution path (status inquiry; reconciliation as last resort)
  resolves `processing` intents within a bounded window. A lost response is an incident
  class with a playbook, not a retry-with-a-different-amount.
- **Settlement is batch**: acquirers sweep daily; funds land 1–3 days later (ballpark); payout
  eligibility is a ledger view, and payout rows must point to the captures that funded them
  (traceable money, again a ledger property — deep dive 2).

**Option A — treat the payment as one row whose status we sync optimistically** (the
"state machine as an afterthought" design): wins: tiny code. loses: *every* race above —
capture-after-expiry, double-capture, orphaned auths, unreconcilable money — lands as
production incidents instead of designed transitions. **Chosen Option B** (explicit machine +
sweeper + ambiguity playbook): more systems, fewer lost dollars.

## 4. Reconciliation: the file is the truth at settlement

**Option A — trust the real-time ledger, spot-check acquirer files occasionally**:
- wins: no ingest pipeline.
- loses: unbounded silent drift — scheme fees, cross-currency rounding, acquirer-side
  adjustments, and our own crash-window losses accumulate until a merchant's or finance
  team's number disagrees with the bank's, at which point *nobody can prove which movement
  was wrong*. For money, trust is not transitive across the bank seam.

**Option B — daily settlement files in, matched line-by-line (chosen)**:
- match key: scheme/trace reference + amount + date-window; matched → ledger confirms (or
  notes scheme fees → new postings); unmatched → **breaks queue** with classification:
  *timing* (crossed the file cut-off — expected, auto-resolves next day), *fee/rounding*
  (auto-post with tolerance thresholds), *missing/extra* (needs investigation: our crash
  window, their file error — each with an owner and an SLA, measured in days not months).
- loses: a real pipeline to operate (~daily per acquirer, ~100s of files/day, est. sheet
  scale is trivial; the *semantics* are the work), and a breaks team.

**Chosen: Option B.** Real-time says what we *believe*; the file says what *happened*; both
are stored; disagreement is a first-class product state, not an error log. Same philosophy as
[27.ev-charging-network](../27.ev-charging-network/README.md)'s CDR reconciliation, applied at
the highest stakes in this corpus.

## 5. In-line risk: ~150 ms to say yes/no, paying for false declines

Velocity counters (card, customer, merchant, device), rules (geo/BIN anomalies), a model
score — all inside the auth envelope (~500 ms internal, est. sheet; risk's share ≤ ~150 ms so
the gateway call still fits).

- **The trade that isn't obvious from the fraud side**: the cost-weighted losses are usually
  dominated by *false declines* — a legitimate ~$25 (mean, est. sheet) sale lost forever, plus
  the customer who doesn't come back. A risk system tuned to minimize fraud will
  unselfconsciously destroy revenue; decline rate is a first-class metric next to fraud rate.
- **Option A — score and block inline only**: sub-150 ms hard, and errors are immediate and
  permanent (a decline can't be retroactively un-declined).
- **Option B — inline fast tier + async review tier (chosen)**: clearly-good → authorize;
  clearly-bad → decline with issuer-consistent codes; gray zone → authorize + post-auth
  review queue/refund automation (chargeback liability rules permitting). Inline tier is
  latency-safe (few features, cached velocity counters); the expensive model runs where latency
  permits. Loses: a reviewed payment can charge back anyway — liability accounting has to be
  explicit policy, not an accident.

**Failure modes**: risk service slow/unavailable → fail open (authorize) or fail closed
(decline)? Policy decision with a real cost both ways; the design decision is to make it an
*explicit, kill-switch-visible* policy and default to failing open for low-risk segments,
closed for high-risk ones — never a silent default. Velocity counters stale under Kafka lag →
the model sees yesterday's frequencies: acceptable for minutes, alarmed beyond that (cross-link
[13.kafka](../13.kafka/README.md) consumer-freshness discipline).

## 6. PCI scope: tokenization as an architecture decision

PCI-DSS makes *handling* PANs a regulated, auditable surface. The design goal is that PANs
never transit or rest unencapsulated on platform systems.

- **Option A — encrypt PANs at rest in our normal stores**: encryption is not scope
  reduction — the decryption path, logs, snapshots, backups, and every engineer with query
  access are all *in scope* forever, and audits scale with that surface.
- **Option B — tokenization + isolated vault (chosen)**: card capture happens client-side →
  vault system (separate credentials, separate audit domain) returns a token; every platform
  component downstream — payments state, ledger, risk, webhooks, exports — sees only tokens.
  PCI scope collapses to the vault + capture UI. Loses: the vault is a load-bearing
  dependency (its outage stops new payments), and "token" must be treated as a credential
  itself (leakable per-scope, rotatable).

The same pattern generalizes (→ [31.netflix](../31.netflix/README.md) rights data, PHI/PII
anywhere): *isolate the most regulated datum behind the narrowest interface, make everything
downstream type-blind to it.*
