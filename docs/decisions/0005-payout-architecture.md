# ADR-0005 — Hybrid payout architecture, ledger-first finance

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-05-19 |
| **Authors** | Founder + finance design session |

## Context

Engineering work for the pilot is feature-complete (see [`CLAUDE.md`](https://github.com/ChopNow-app/chopnow-api/blob/develop/CLAUDE.md)). The remaining critical path before any vendor sees a payout is:

1. RCCM + NIU + Campay Go-Live (tracked: chopnow-api [#181](https://github.com/ChopNow-app/chopnow-api/issues/181))
2. A working money-distribution path

The first is regulatory and out of code's hands. The second is the subject of this ADR.

### What's broken or missing today

| Piece | Current state | Problem |
|---|---|---|
| Commission rate | Single `Vendor.commissionRate Decimal @default(0.10)` field | INFORMAL vendors are silently set to 10% at submission. Business model says 6%. We are over-charging the segment we most need to retain. |
| Per-order snapshot | None — `Order` carries `subtotalXAF`, `deliveryFeeXAF`, `totalXAF` only | When (not if) commission rates change in M5, M8, etc., we can't compute historical vendor balances. Reconciliation drifts. |
| Payout job | None — Epic 7 story 7.1 ("Paiement Vendeur Quotidien") is unstarted | Vendors will not be paid by the platform when the pilot launches. Manual MoMo transfers are unsustainable beyond ~10 cmd/day. |
| Ledger | None — there's a `VendorPenalty` table (shipped 2026-05-19 with pre-orders) that's a sketch of accounting | We have no append-only, double-entry source of truth. Every reconciliation is bespoke SQL. This is how marketplaces lose track of money around 100+ orders/day. |
| Refund flow | Manual; `paymentStatus=REFUND_PENDING` exists but no Campay API integration | Story 7.10 unstarted. Refund is a worklist today. |

### What forced this decision now

The previous default plan was a single daily payout cron (Story 7.1). After advisor review (transcript captured in the founder's notes 2026-05-19), three things became clear:

1. **A single weekly cadence for all vendors mis-prices the moat.** INFORMAL vendors buy ingredients daily. A 7-day cashflow gap pushes them off the platform. They are also the segment our pricing strategy depends on (6% commission, see [business-model.md §6](https://github.com/ChopNow-app/chopnow-docs/blob/main/_bmad-output/planning-artifacts/business-model.md)).
2. **A money-moving cron at Saturday 22:00 (peak dinner)** is a self-inflicted incident waiting to happen. The originally-proposed slot overlaps the daily order spike.
3. **The ledger has to exist before the payout job.** Without it, every refund / dispute / negative-balance scenario corrupts the books. Shipping a payout cron over the current schema would be paving over the problem.

## Decision

We will adopt a **hybrid payout policy, backed by a double-entry append-only ledger, with money-moving crons running at Sunday 02:00 Africa/Douala**.

### Vendor payout policy

| Vendor type | Cadence | Trigger | Reserve window |
|---|---|---|---|
| **INFORMAL** | On-demand | Vendor taps "Demander virement" in PWA → **admin approves** in dashboard → Campay transfer fires | Trusted threshold below; otherwise 24h hold |
| **SEMI_FORMAL** | Weekly | Sunday 02:00 cron, idempotent on `(vendorId, periodStart)` | Refuse payout when `pendingRefundsXAF > 0` or `openDisputes > 0` |
| **RESTAURANT** | Weekly | Same Sunday 02:00 cron | Same |
| **B2B (Phase 2)** | Net-30 invoiced | Out of scope this ADR | — |

**"Trusted vendor"** definition for the informal on-demand path:

```
trusted = (completed_orders ≥ 10)
       AND (open_disputes = 0)
       AND (days_since_first_order ≥ 7)
```

Trusted vendors get same-day on-demand transfers (still admin-approved in v1). Untrusted vendors get a 24h hold from the moment they request. Revisit thresholds at M4 based on fraud data.

### Rider payout policy

| Cadence | Trigger | Reason for rejecting alternatives |
|---|---|---|
| **Daily batch** | 06:00 Africa/Douala cron | Riders wake to yesterday's earnings — captures 90% of the psychological win of instant payout at 10% of the operational risk (per-delivery Campay fees, fraud surface, webhook noise) |

### Cron schedule (Africa/Douala)

| When | Job | Scope |
|---|---|---|
| **Sunday 02:00** | Weekly vendor payout sweep | SEMI_FORMAL + RESTAURANT vendors with `netXAF > 0` |
| **Daily 06:00** | Rider payout batch | All riders with `pendingXAF ≥ MIN_RIDER_PAYOUT_XAF` (initial: 1 000 FCFA) |
| **Daily 03:00** | Reconciliation report | Cross-check Campay statement vs `LedgerEntry` for the previous 24h |
| (on-demand) | Informal vendor cashout | Vendor request → admin approve → Campay transfer; not scheduled |

### Data model shape

**`LedgerEntry` (new — double-entry, append-only):**

```prisma
enum LedgerAccount {
  CUSTOMER_ESCROW          // liability — money customer paid us, not yet earned
  VENDOR_PAYABLE           // liability — owed to vendor
  RIDER_PAYABLE            // liability — owed to rider
  PLATFORM_REVENUE         // income — commission + delivery margin
  PLATFORM_RESERVE         // asset — kept on platform for refunds/disputes
  CAMPAY_FLOAT             // asset — what Campay holds on our behalf
  REFUND_PAYABLE           // liability — owed back to a customer
}

enum LedgerEventType {
  PAYMENT_RECEIVED
  ORDER_DELIVERED
  REFUND_ISSUED
  VENDOR_PAYOUT
  RIDER_PAYOUT
  PENALTY_APPLIED
  ADJUSTMENT                // admin manual entry, must reference a reason
}

model LedgerEntry {
  id           String           @id @default(uuid())
  // Same eventId on every leg of a double-entry transaction — entries with
  // the same eventId must net to zero across debit/credit.
  eventId      String
  eventType    LedgerEventType
  account      LedgerAccount
  // Positive = debit increases, credit decreases (asset/expense convention).
  // For liability accounts the sign is reversed in reporting, not at write.
  amountXAF    Int              // never zero; sign indicates debit (+) / credit (−)
  // Optional foreign keys — at least one is required at the service layer.
  orderId      String?
  vendorId     String?
  riderId      String?
  payoutId     String?
  refundId     String?
  description  String?          // human-readable, surfaced in admin dashboard
  createdAt    DateTime         @default(now())

  @@index([eventId])
  @@index([account, createdAt])
  @@index([vendorId, createdAt])
  @@index([riderId, createdAt])
  @@index([orderId])
  @@map("ledger_entries")
}
```

Append-only by convention enforced in the service layer; no DELETE / UPDATE on `ledger_entries` outside migrations. Net balances are derived via `SUM(amountXAF)` queries with the account-sign reversal applied in the reporting layer.

**`VendorPayout` + `RiderPayout` (new):**

```prisma
enum PayoutStatus {
  PENDING                    // computed, not yet sent
  IN_FLIGHT                  // Campay transfer initiated, awaiting webhook
  PAID                       // confirmed
  FAILED                     // Campay rejected; ops escalation
  CANCELLED                  // admin reversed before in-flight
}

model VendorPayout {
  id           String       @id @default(uuid())
  vendorId     String
  vendor       Vendor       @relation(fields: [vendorId], references: [id])
  // Period covered. For weekly payouts: Sun-prev to Sat-this. For on-demand:
  // periodStart = first un-paid-out order's paidAt, periodEnd = request time.
  periodStart  DateTime
  periodEnd    DateTime
  grossXAF     Int          // sum of order subtotals in period
  commissionXAF Int         // platform commission
  penaltyXAF   Int          @default(0)  // sum of VendorPenalty rows applied
  adjustmentsXAF Int        @default(0)  // admin manual adjustments
  netXAF       Int          // gross − commission − penalty + adjustments
  momoPhone    String
  campayRef    String?      @unique
  status       PayoutStatus @default(PENDING)
  failureReason String?
  scheduledFor DateTime     // when the cron picked it up OR request time
  sentAt       DateTime?
  paidAt       DateTime?
  createdAt    DateTime     @default(now())

  // Idempotency: one payout per (vendor, period) — re-running the cron
  // doesn't double-pay.
  @@unique([vendorId, periodStart])
  @@index([status, scheduledFor])
  @@map("vendor_payouts")
}
```

`RiderPayout` is structurally identical with `riderId`, no `commissionXAF` (riders earn the delivery share directly), and a different period (daily). Schema mirrors above.

**`Order` snapshot fields (new):**

```prisma
// Captured at order creation from Vendor.commissionRate at that moment.
// Never recomputed. Drives all downstream payout / ledger arithmetic.
commissionRate      Decimal  @db.Decimal(5, 4)   // e.g. 0.0600
commissionXAF       Int?     // populated on payment confirmation
riderShareXAF       Int?     // populated on payment confirmation
platformFeeXAF      Int?     // commission + delivery margin
payoutId            String?  // FK to VendorPayout once settled
```

### What we explicitly defer (out of scope for S1–S3)

These remain in the backlog but are downstream of getting the foundation right:

- **Self-service on-demand cashout** (no admin gate). All informal cashouts go through admin approval in v1; flip after the first 100 transfers prove the failure modes.
- **Automated commission tier recalculation** (Story 7.x, chopnow-api [#76](https://github.com/ChopNow-app/chopnow-api/issues/76)). The schema accommodates tier flips but the cron stays manual / admin-button until M6.
- **TchopNow Star auto-detection** (informal 6% → 4% on ≥4.5★ + ≥100 cmd/month). Same reasoning.
- **Negative balance reserve waterfall.** v1 rule is: refuse payout when `pendingRefundsXAF > 0` OR `openDisputes > 0`. No reserve accumulation; the platform absorbs short-tail losses out of `PLATFORM_REVENUE` until we have enough volume to model a sensible reserve %.
- **Rider per-delivery instant payout.** Daily batch only.

## Alternatives considered

| Option | Dealbreaker |
|---|---|
| **Daily payouts for all vendors (original Story 7.1)** | Burns ~7× the Campay transfer fees vs weekly for formal vendors; doesn't address the per-order snapshot or ledger gaps, so we'd ship the same bugs faster. |
| **Weekly payouts for all vendors, including INFORMAL** | INFORMAL vendors buy ingredients daily and measure trust in cashflow speed. A 7-day delay risks losing the segment our pricing depends on. Advisor's strongest point. |
| **Instant per-delivery rider payouts** | Per-transfer Campay fees on 400 FCFA payouts destroy margin; creates a fraud-amplification surface (rider games dispatch to farm fees); doubles webhook reconciliation noise. Daily 06:00 captures 90% of the psychological win at 10% of the operational risk. |
| **Saturday 22:00 payout cron** (the original founder proposal) | Overlaps the dinner peak. If the job spikes DB load, slows on Campay, or jams a queue, it degrades the live ordering surface. Live ordering > finance jobs, always. Sunday 02:00 has none of those properties. |
| **No ledger, single `VendorBalance` table** | Works at <50 orders/day. Falls apart on the first refund-after-payout incident, when we need to know how the balance got there. The cost of the ledger now is one Prisma model + a discipline rule; the cost of retro-fitting it later is a multi-week reconciliation project. |
| **Self-service on-demand cashout in v1** | We don't yet know the failure modes. First 100 transfers go through an admin so we can build the runbook from real cases before automating. |
| **External payout provider (e.g. Smobilpay solely for payouts)** | Adds a second aggregator surface, second webhook, second reconciliation — for marginal benefit when Campay already handles outgoing transfers. Re-evaluate at Campay outage > 4h. |

## Consequences

**Positive**

- **Cashflow alignment with vendor reality.** INFORMAL vendors keep their daily-cashflow expectation; formal vendors get a cleaner reconciliation cadence. Both align with how those segments actually run their businesses.
- **Audit-grade financial history.** Every money movement leaves a double-entry trace. Any vendor / rider / customer balance can be derived from `LedgerEntry` at any past point in time with one query. This is the difference between "we think we owe this vendor X" and "we owe this vendor X — here's the trail."
- **Per-order commission immutability.** Tier changes, behavioral penalties, and TchopNow Star step-downs can land in M4–M8 without rewriting history. The 6% an informal vendor earned on order #1042 stays 6% even after they get promoted to Star tier.
- **Off-peak cron eliminates a class of incidents.** Sunday 02:00 vendor sweep + 06:00 rider sweep both run when DB load is ~5% of peak. A flaky Campay or a slow query never hits a live customer.
- **Idempotency baked in.** `VendorPayout @@unique(vendorId, periodStart)` means re-running the cron is safe — no double-pay, no manual "did it actually go through?" panic.

**Negative**

- **Operational load on admin.** Every informal cashout passes through an admin click in v1. Founder time. Mitigation: this surface is explicitly temporary (flip to self-service after ~100 transfers prove the patterns). Worth the cost in exchange for the runbook we'll build.
- **Two payout surfaces (cron + on-demand)** sharing the same `VendorPayout` model. Tested via the same idempotency key; risk is a code path duplication if not careful. Mitigation: shared `PayoutService.compute(vendorId, periodStart, periodEnd) → payout` function called by both paths.
- **Ledger discipline required.** Every service that moves money must call `LedgerService.recordTransaction(eventId, entries[])`. One missed call = silent corruption. Mitigation: code review checklist, eventual `assertLedgerBalanced` integration test after every order/refund flow.
- **The schema grows by 3 tables + 4 Order columns** before the pilot launches. Migration risk on staging exists; mitigated by additive-only changes (no destructive ALTERs).

**Reversal cost** — medium for the policy, large for the ledger.

- **Reversing the policy** (e.g. switching all vendors to daily payouts after pilot data) is a config + cron change. Small.
- **Reversing the ledger** (going back to a single-table balance approach) means re-deriving balances from history. We won't, but if we did, it's a backfill not a rewrite.

## Follow-ups

### ✅ S1 Finance Foundation — shipped 2026-05-19

- **`[7.0a]`** Ledger schema — `LedgerEntry`, `VendorPayout`, `RiderPayout`, append-only convention enforced by `LedgerService.recordTransaction`. ([chopnow-api#195](https://github.com/ChopNow-app/chopnow-api/pull/195))
- **`[7.0b]`** Per-order commission snapshot + type-aware default. Fixed the latent INFORMAL over-charge bug; existing rows backfilled to 0.06. ([chopnow-api#196](https://github.com/ChopNow-app/chopnow-api/pull/196))
- **`[7.0c]`** VendorPenalty → LedgerEntry bridge. Pre-order vendor-cancel-after-accept now writes paired `VENDOR_PAYABLE +` / `PLATFORM_REVENUE −` entries inside the same Prisma transaction as `VendorPenalty.create`. ([chopnow-api#197](https://github.com/ChopNow-app/chopnow-api/pull/197))

### ✅ S2 Finance Operations — shipped 2026-05-20

Backend (chopnow-api):

- **`[7.1a]`** Money-flow ledger writes — `onPaymentSucceeded` (CAMPAY_FLOAT / CUSTOMER_ESCROW) and `markDelivered` (CUSTOMER_ESCROW / VENDOR_PAYABLE / RIDER_PAYABLE / PLATFORM_REVENUE). Every order now leaves 6 ledger rows summing to zero. ([chopnow-api#207](https://github.com/ChopNow-app/chopnow-api/pull/207))
- **`[7.1c]`** `GET /admin/vendors/:id/balance` — signed balance + component breakdown + isTrusted flag. ([chopnow-api#208](https://github.com/ChopNow-app/chopnow-api/pull/208))
- **`[7.1d]`** `GET /admin/riders/:id/balance` — same shape for riders, simpler (no commission/penalty). ([chopnow-api#209](https://github.com/ChopNow-app/chopnow-api/pull/209))
- **`[7.1e]`** Admin financial dashboard backend lists — `vendor-balances`, `rider-balances`, `refund-queue`, paginated. ([chopnow-api#210](https://github.com/ChopNow-app/chopnow-api/pull/210))
- **`[7.2a]`** Sunday 02:00 weekly vendor payout cron for SEMI_FORMAL + RESTAURANT. Idempotent on `(vendorId, periodStart)`. Folds in the **`[7.3a]`** negative-balance refusal rule (skip when `pendingRefunds > 0`). ([chopnow-api#211](https://github.com/ChopNow-app/chopnow-api/pull/211))
- **`[7.2c]`** Daily 06:00 rider payout batch. Same shape, no commission/penalty/dispute leg. ([chopnow-api#212](https://github.com/ChopNow-app/chopnow-api/pull/212))
- **`[7.2d]`** KYC defense-in-depth on rider payouts — refuse if `idCardPhotoUrl OR selfiePhotoUrl IS NULL`, regardless of `RiderStatus`. ([chopnow-api#214](https://github.com/ChopNow-app/chopnow-api/pull/214))
- **`[7.2b]`** INFORMAL on-demand cashout — `POST /vendors/me/cashout-request` (vendor) + `POST /admin/finance/cashout-requests/:id/approve|reject` (admin). New `VendorCashoutRequest` table; v1 admin-gated, self-service flip deferred until first ~100 transfers prove the patterns. ([chopnow-api#215](https://github.com/ChopNow-app/chopnow-api/pull/215))

Frontend (chopnow-app):

- **`[7.3b]`** Admin financial dashboard at `/admin/finance` — four tabs (cashout requests, vendor balances, rider balances, refund queue). Inline approve/reject on the cashout queue with trust badges. ([chopnow-app#138](https://github.com/ChopNow-app/chopnow-app/pull/138))

### ✅ S3 Campay Outbound + Reconciliation — shipped 2026-05-21

After S3, the platform handles every Campay state autonomously and surfaces every failure to admin for one-tap remediation. The only pre-launch blocker that remains is RCCM + NIU + Campay Go-Live ([chopnow-api#181](https://github.com/ChopNow-app/chopnow-api/issues/181)) — engineering is complete.

- **`[7.4-worker]`** Campay outbound transfer worker — `@Cron(EVERY_5_MINUTES)` drains `VendorPayout` / `RiderPayout` PENDING rows. Status-guarded `PENDING → IN_FLIGHT`, calls Campay `/withdraw/`, persists `campayRef`. Webhook handler at `POST /webhooks/campay/transfer` flips `IN_FLIGHT → PAID`. Manual-mode env flag (`CAMPAY_TRANSFERS_ENABLED=false` default) keeps the worker dormant until Campay Go-Live + RCCM lands. ([chopnow-api#217](https://github.com/ChopNow-app/chopnow-api/pull/217))
- **`[7.8]`** Webhook idempotency hardening — new `CampayWebhookEvent` dedup table with `unique(eventType, reference)`. INSERT-first pattern; Prisma `P2002` becomes `isFirst: false`. Wired into both `COLLECT` and `TRANSFER` webhooks. Survives process restarts + multi-instance replicas. ([chopnow-api#218](https://github.com/ChopNow-app/chopnow-api/pull/218))
- **`[7.10]`** Campay refund flow — `RefundProcessor` cron drains Orders in `REFUND_PENDING`, calls `CampayService.initiateRefund` (outbound to consumer's MoMo), writes paired `CUSTOMER_ESCROW + / REFUND_PAYABLE −` ledger entries on initiation. Webhook at `POST /webhooks/campay/refund` flips `REFUND_PENDING → REFUNDED` and writes the settle pair `REFUND_PAYABLE + / CAMPAY_FLOAT −`. ([chopnow-api#219](https://github.com/ChopNow-app/chopnow-api/pull/219))
- **`[7.3c]`** Stuck-PICKED_UP detection + admin rider-fraud resolution — `@Cron(EVERY_30_MINUTES)` flags Orders stuck in PICKED_UP > 2h. `POST /admin/orders/:id/resolve-rider-fraud` atomically queues a consumer refund (REFUND_PENDING — drained by 7.10), writes a vendor-compensation `ADJUSTMENT` ledger entry (`PLATFORM_REVENUE + / VENDOR_PAYABLE −`), and either SUSPENDS the rider (JWT revoke) or WARNS them (reliabilityScore decrement). ([chopnow-api#220](https://github.com/ChopNow-app/chopnow-api/pull/220))
- **`[7.5]`** Failed payout escalation + admin remediation — `PayoutEscalation` cron picks FAILED + stale IN_FLIGHT payouts and stuck refunds. `GET /admin/finance/escalations` lists them oldest-first. Admin endpoints: `POST /admin/finance/vendor-payouts/:id/retry` (flip FAILED → PENDING) and `POST /admin/finance/vendor-payouts/:id/manual-mark-paid` (capture admin-fired Campay reference). Rider equivalents shipped in the same PR. ([chopnow-api#221](https://github.com/ChopNow-app/chopnow-api/pull/221))
- **`[7.9]`** Campay balance pre-check — `CampayService.getBalance()` with 60s cache. `PayoutTransferWorker` fetches once per tick, tracks remaining float as it initiates transfers, refuses any payout that would push remaining negative. Prevents bounced transactions. ([chopnow-api#222](https://github.com/ChopNow-app/chopnow-api/pull/222))
- **`[7.12]`** Campay circuit breaker (mode dégradé) — 5 consecutive failures across any Campay method opens the circuit. While OPEN, every wrapped call (`initiateCollect`, `initiateTransfer`, `initiateRefund`, `getBalance`) throws `campay_circuit_open` immediately for 60s. Next call auto-promotes to HALF_OPEN; probe outcome decides recovery. `GET /admin/finance/campay-circuit` exposes state. ([chopnow-api#223](https://github.com/ChopNow-app/chopnow-api/pull/223))

### Documentation

- ✅ ADR-0005 (this document) — captures the policy + ledger model.
- ✅ `docs/vendor/pre-orders.md` — updated to describe the ledger pairing (S1).
- 📋 `docs/finance/` section (`ledger.md`, `payouts.md`, `commission.md`) — defer until post-launch when there's real production traffic to write about.

### What's actually left before launch

Engineering: nothing finance-related. Pre-launch blockers stay regulatory:

- **RCCM + NIU registration** (TchopNow legal entity) — chopnow-api [#181](https://github.com/ChopNow-app/chopnow-api/issues/181).
- **Campay Go-Live application** — requires the RCCM + NIU above + production credentials. Until that lands, the manual-mode env flags (`CAMPAY_TRANSFERS_ENABLED`, `CAMPAY_REFUNDS_ENABLED`) keep the auto-fire workers dormant. The founder fires transfers + refunds manually in the Campay UI and uses the admin manual-mark-paid endpoint to capture each Campay reference into the bookkeeping.
