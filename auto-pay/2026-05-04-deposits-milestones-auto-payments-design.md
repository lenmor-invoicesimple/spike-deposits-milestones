# Deposits & Milestones — Automatic Payments: Technical Design

> See [decisions.md](./2026-05-04-deposits-milestones-auto-payments-decisions.md) for all decision points and trade-off analysis.

## Key Invariants

**Vault once per invoice.** The client vaults exactly once — when they pay the first payment at checkout (deposit or first milestone) and opt into auto-pay. That single vault record (`upcoming_payment`, keyed by `invoice_id UNIQUE`) covers every current and future milestone on the invoice. The client never sees a vault/checkout prompt again unless the vault is invalidated (max retries, merchant cancels, invoice-level edit). Re-vaulting replaces the existing record.

**Deposit is not required.** The vaulting payment can be a deposit OR the first milestone — both are valid entry points. The checkout scenarios are: (1) Deposit + milestones, (2) Milestones only (no deposit), (3) Deposit + single milestone. In all cases, auto-pay is offered for remaining future milestones after the first payment.

**Execution time = vault time in UTC.** Milestone due dates are date-only (no time component). The EventBridge schedule fires at: `dueDate @ time-of-day(consent_granted_at)` in UTC. `consent_granted_at` is stored on the vault record. Timezone defaults to UTC — same as RP today. Neither checkout nor the vault captures client/merchant timezone. Future improvement: use merchant account timezone. To confirm before V1 ship.

## Critical Semantic Change: `dueDate`

**Today:** `dueDate` on an upcoming payment is purely informational — "I'd like to be paid by this date." Nothing happens when the date passes. No enforcement, no reminders.

**With auto-pay:** `dueDate` becomes the trigger date — "The client's card WILL be charged on this date." Real money moves.

### Implications

1. **Merchant awareness when setting due dates.** When auto-pay is active (client has vaulted) and the merchant views due dates, the UI must reinforce that "Client will be charged $X on this date." Casual date-setting now has real financial consequences. Note: per Decision 14, edits are blocked while vault is active — merchant must disable auto-pay first.

2. **Due date required for auto-pay scheduling.** A milestone without a due date cannot be scheduled. The UI should nudge: "Add a due date to enable automatic payment for this milestone." Deposits are exempt (paid at checkout, no schedule needed).

3. **Past due dates at vault time.** Per Decision 8: if milestones are already overdue when the client vaults, they are charged immediately. The merchant should understand this when enabling auto-pay on an invoice with overdue milestones.

4. **Deposits have no due date — by design.** The deposit is the checkout payment (the vaulting moment). It never needs a schedule. Only future milestones with due dates get automated.

---

## Architecture: Hybrid Approach

Separate tables and handlers for upcoming payments (isolation from RP). RP serves as an architectural blueprint — most code will be new, referencing RP patterns rather than directly reusing them.

### Shared Utilities (extract from existing RP code)

> **Note:** Most RP execution code cannot be reused directly because it's tightly coupled to `recurringInvoiceSeriesId` — the RP series ID appears in function signatures, schedule names, idempotency keys, and payload shapes throughout. The shared layer is the low-level primitives (Stripe SDK wrapper, AWS SDK calls, email utilities), not the RP-specific orchestration handlers.

| Component | Current Location | Notes |
|---|---|---|
| Stripe charge execution (PaymentIntent off_session) | `stripe-payment-intent.ts` (`createStripeSDKPaymentIntent`) | Thin Stripe SDK wrapper: takes `Stripe.PaymentIntentCreateParams` + merchant ID. **Not** `recurring-payment-execute.ts` — that's coupled to RP's series/fee model. We write a new `upcoming-payment-execute.ts` that calls this same underlying primitive. |
| Retry scheduler (create/delete EventBridge `at()` schedules) | `is-payments/src/lambda-handlers/util/scheduler.ts` | `recurringInvoiceSeriesId` is baked into naming and payload — not a rename. Write parallel `createUpcomingPaymentSchedule` / `deleteUpcomingPaymentSchedule` functions in the same file (or new `upcoming-payment-scheduler.ts`). Same AWS SDK calls, same schedule group. |
| Success/failure handling | `payment-execution/handle-success.ts` → `handlePaymentSuccess` / `payment-execution/handle-failure.ts` → `handlePaymentFailure` | Both tightly coupled to `recurringInvoiceSeriesId`. Not reusable. Write new `upcoming-handle-success.ts` / `upcoming-handle-failure.ts`. |
| Email send utilities | `email/send-automatic-payment-successful-email.ts` → `sendAutomaticPaymentSuccessfulEmail` / `payment-execution/send-failure-emails.ts` → `sendPaymentFailureEmail`, `sendPaymentCancelledEmail` | `recurringInvoiceSeriesId` is a required param but is **only used for a dead `fetchRecurringInvoice` lookup** (result unused in template) and logging. Making it optional is a small, safe change. These functions are otherwise generic and **can be reused for upcoming payment emails** after making the param optional. |
| EventBridge schedule group | `cdk/src/lib/constructs/is-payments/is-payments-retry-schedule-group-construct.ts` (`IsPaymentsRetryScheduleGroupConstruct`) | Reuse directly — upcoming payment schedule names use distinct prefix (`upcoming-{invoiceId}-{paymentId}`) so no collision with RP names (`payment-retry-{seriesId}-{count}`). |
| Events queue (FIFO) + events Lambda | `is-payments-events-sqs-construct.ts` (`IsPaymentsEventsSqsConstruct`) + `is-payments-events-lambda-construct.ts` (`IsPaymentsEventsLambdaConstruct`) + handler: `lambda-handlers/payment-events.ts` | Four new topic cases added to the switch in `payment-events.ts` (see table below). No CDK changes to queue or Lambda constructs. FIFO `MessageGroupId` set to `invoiceId` for upcoming payment messages (vs `recurringInvoiceSeriesId` for RP) — requires new publish functions in `is-payments/src/server/services/payment-events-queue/payment-events-queue.ts` and `is-stripe/src/services/payment-events-queue.ts`. |

#### New topics for `payment-events.ts` switch

| New topic | Equivalent RP topic | Purpose |
|-----------|---------------------|---------|
| `createUpcomingPayment` | `createRecurringPayment` | Published by is-stripe after vault → handler creates 1 `upcoming_payments` row (ACTIVE) for the next unpaid milestone + its EventBridge schedule pair. No other rows created (Decision 21). |
| `upcomingPaymentSuccess` | `paymentSuccess` | Published by execution Lambda on success → handler creates NEXT schedule, sends emails, logging |
| `upcomingPaymentFailure` | `paymentFailure` | Published by execution Lambda on failure → handler manages retry scheduling, emails |
| `upcomingPaymentReminder` | (no RP equivalent) | Published by Automatic Payment Request (reminder) schedule firing → handler sends "Upcoming Scheduled Payment" email with Pay Now CTA |

Note: No `cancelUpcomingPaymentSchedules` topic needed. Cancel-all is handled directly by the `POST /upcoming-payment/disable` REST endpoint (synchronous, not via SQS). This is simpler than RP's `cancelRetrySchedules` topic because cancel is always triggered by a REST call (mobile, checkout, or admin), never by an async event.

### New Upcoming Payment Components

| Component | Purpose |
|---|---|
| `upcoming_payment` table (is-stripe) | Vault per invoice |
| `upcoming_payments` table (is-payments) | Per-milestone execution state |
| Upcoming payment execution handler | Consumes from SQS queue, validates status, calls is-stripe to charge |
| Automatic Payment Request (reminder) handler | Fires N days before due date, sends "Upcoming Scheduled Payment" email |
| Upcoming payment SQS topics on events queue | `createUpcomingPayment`, `upcomingPaymentSuccess`, `upcomingPaymentFailure`, `upcomingPaymentReminder` |
| CDK construct for upcoming payment execution Lambda | Same permission pattern as RP retry Lambda |
| EventBridge schedule PAIRS (one at a time) | Charge schedule + Automatic Payment Request (reminder) schedule for current payment only (created when chain reaches it — Decision 21) |

### Fallback

If vaulting extraction or shared queue message format creates too much noise, fall back to full isolation (Approach 2): duplicate vault storage, separate execution queue, fully separate handlers. Shared utilities (Stripe SDK calls, retry math, email templates) can still live in a `packages/payments/shared/` module.

---

## Data Model

### is-stripe: `upcoming_payment` — invoice vault

One record per invoice (not per milestone). Stores the vaulted payment method.

```sql
CREATE TABLE upcoming_payment (
  id              SERIAL PRIMARY KEY,
  invoice_id      TEXT UNIQUE NOT NULL,                        -- NEW: per-invoice scope (replaces recurringInvoiceSeriesId)
  account_id      TEXT NOT NULL,
  customer_id     TEXT NOT NULL,
  vaulted_token   TEXT NOT NULL,
  payment_method_type TEXT NOT NULL,
  payment_method_last4 TEXT,                                   -- derived from Stripe PaymentMethod at vault time (for display)
  currency_code   TEXT NOT NULL,
  consent_granted_at TIMESTAMPTZ NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);
-- Omitted vs RP: amount_cents (no fixed amount on vault — each milestone has its own amount)
```

Vaulting happens ONCE per invoice. The single vault record covers all current and future milestones. Client never sees a vaulting prompt again unless the vault is invalidated.

**Timezone is not stored on the vault table.** Each scheduling request (initial `createUpcomingPayment` SQS message, and subsequent `POST /upcoming-payment/add` from mobile) includes `timezone` in its payload. Mobile knows the device timezone and passes it at call time. This avoids storing it centrally and removes the need for is-payments to call back to is-stripe just to get a timezone when adding new milestones later.

### is-payments: `upcoming_payments` — individual milestone execution records

One record per milestone, created only when the chain reaches it (Decision 21). At any point, at most one row per invoice has `status = ACTIVE`.

```sql
CREATE TABLE upcoming_payments (
  id                    SERIAL PRIMARY KEY,
  invoice_id            TEXT NOT NULL,                   -- NEW: replaces recurringInvoiceSeriesId
  milestone_id          TEXT NOT NULL,                   -- NEW: Parse Payment remoteId — no RP equivalent
  account_id            TEXT NOT NULL,
  provider              TEXT NOT NULL DEFAULT 'stripe',
  amount_cents          INTEGER NOT NULL,                 -- snapshot at row creation time
  currency_code         TEXT NOT NULL,
  status                TEXT NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE | INACTIVE (no QUEUED — Decision 21)
  retry_count           INTEGER NOT NULL DEFAULT 0,
  disabled_reason       TEXT,                            -- NEW values vs RP: MILESTONE_COMPLETED, MANUALLY_PAID
  failure_code          TEXT,
  failure_description   TEXT,
  is_async_payment_method BOOLEAN DEFAULT FALSE,
  scheduled_date        DATE NOT NULL,                   -- NEW: the trigger date for this milestone — no RP equivalent
  last_executed_at      TIMESTAMPTZ,
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(invoice_id, milestone_id)                       -- NEW: no RP equivalent
);
-- Omitted vs RP: count (RP series invoice counter, no equivalent), consentGrantedAt (on vault table),
--               failedInvoiceId (not applicable), paymentMethodLast4 (not stored in V1 — not shown in UI)
```

**`milestone_id`** — kept in the table for three reasons: (1) **EventBridge schedule naming** — `upcoming-{invoiceId}-{paymentId}` used when creating and deleting schedules; (2) **row lookup** — execution Lambda receives `{ invoiceId, milestoneId }` from EventBridge and fetches this row to validate `status = ACTIVE` and get `amountCents`; (3) **cancel-one / skip targeting** — `POST /upcoming-payment/cancel-one` and `POST /upcoming-payment/skip` target one row by `milestoneId`. **Not** needed for Parse write-back — that happens automatically via Stripe webhook. See note below.

**How Parse gets updated on successful milestone charge (same pattern as RP):**
- **Checkout** → client's browser hits return URL → `is-stripe /payment-intent/confirm/:documentId` (`payment-intent-confirm.ts`, synchronous) → `addPaymentToDocument()` → `Parse.Cloud.run('invoiceAddPayment')`
- **RP execution Lambda** → creates off-session PaymentIntent → Stripe webhook fires → `payment-intent-updated.ts` → `addPaymentToDocument()` → `Parse.Cloud.run('invoiceAddPayment')`
- **Upcoming payment execution Lambda** → same as RP: creates off-session PaymentIntent → Stripe webhook fires → `payment-intent-updated.ts` → `addPaymentToDocument()` → `Parse.Cloud.run('invoiceAddPayment')`

Both RP and upcoming payment execution rely on the Stripe webhook to record the payment to Parse. No new Parse integration needed — the webhook handler already fires for any PaymentIntent success regardless of what created it.

**`scheduled_date`** — date-only (`DATE`), not a timestamp. Mobile stores `dueDate` as a full ISO timestamp (midnight UTC, e.g. `2026-06-01T00:00:00.000Z`) but we store only the date portion. Combined with `timezone` passed in the scheduling request, is-payments reconstructs the correct local fire time for EventBridge.

### Example

Merchant creates a $5,000 invoice with 3 milestones:

**Parse Payment objects (existing, unchanged):**
```
remoteId: "m1"  label: "Deposit"    paymentValue: 500   paymentMode: FLAT  paymentType: DEPOSIT  dueDate: null
remoteId: "m2"  label: "Midpoint"   paymentValue: 1500  paymentMode: FLAT  paymentType: NONE     dueDate: 2026-06-01
remoteId: "m3"  label: "Final"      paymentValue: 3000  paymentMode: FLAT  paymentType: NONE     dueDate: 2026-07-01
```

Client pays $500 deposit and vaults on May 5:

**`upcoming_payment` (is-stripe) — 1 row:**
```
invoice_id: "inv_abc123", customer_id: "cus_xyz", vaulted_token: "pm_xyz", payment_method_type: "card", consent_granted_at: "2026-05-05T19:30:00Z"
```

**`upcoming_payments` (is-payments) — 1 row only (Decision 21):**
```
invoice_id    milestone_id  amount_cents  status   scheduled_date  retry_count
"inv_abc123"  "m2"          150000        ACTIVE   2026-06-01      0    ← has EventBridge schedule pair
```

Note: `m1` (deposit) has no row — it was paid at checkout. `m3` has no row yet — it lives only in Parse until the chain reaches it.
**Single row creation (Decision 21):** Only the next payment (`m2`) has a row + EventBridge schedule pair. When `m2` succeeds, handle-success reads `m3` from Parse (current amount/date), creates a new row (ACTIVE) + schedule pair. This means merchant can freely edit `m3` in Parse — changes are always picked up.

---

## Execution Flow

### Phase 1: Setup (at checkout)

```
Client pays first payment at checkout (is-unifiedxp)
  - This can be a deposit OR the first milestone — deposit is not required
  → Client opts in to automatic payments
  → PaymentIntent confirmed with setup_future_usage: 'off_session' (vaults card — NOT a SetupIntent)
  → is-stripe: create `upcoming_payment` row (invoiceId, vaultedToken, customerId, consent_granted_at)

  NOTE: Vaulting happens ONCE per invoice. The single vault record covers all
  current and future milestones on this invoice. The client never sees a
  vaulting prompt again unless the vault is invalidated (invoice-level edit
  or explicit cancellation).

  → is-stripe publishes SQS to is-payments:
      { topic: 'createUpcomingPayment', invoiceId, accountId, milestones[], consentGrantedAt }
      - milestones[]: all milestone data included — no Parse fetch needed
      - consentGrantedAt: the UTC instant of vault (time-of-day used for EventBridge schedule fire time)
  → is-payments receives message:
      1. From milestones[] in the payload, pick the FIRST (soonest) unpaid milestone.
      2. Create 1 `upcoming_payments` row (status: ACTIVE, amountCents/scheduledDate from milestone).
         No other rows created — future milestones stay in Parse only (Decision 21).
      3. Create EventBridge schedule PAIR for this milestone:
         - CHARGE schedule: at(scheduledDate, time-of-day(consentGrantedAt)) in UTC
         - AUTOMATIC PAYMENT REQUEST (reminder) schedule: `upcoming-reminder-{invoiceId}-{paymentId}` → fires N days before due date
         - Triggers "Upcoming Scheduled Payment" email (with "Pay Now" CTA)
         - If due date is < N days from vault → skip reminder (or send immediately — TBD)
         - ⚠️ OPEN: What is N? (3 days? 7 days?) — confirm with design
         - NOTE: timezone defaults to UTC (same as RP). Future improvement: use merchant account timezone.
      4. Future milestones: no rows, no schedules. Created one-at-a-time by handle-success
         when the chain advances (reads current values from Parse at that time).
         Merchant can freely edit future payment amounts/dates in Parse — always picked up.
      5. If any upcoming payments are OVERDUE (dueDate < today):
         - ⚠️ Decision 8 REVISITING (2026-05-11) — two options:
             a) Schedule for immediate execution via normal pipeline (at(now + 1min))
                → Each gets own PaymentIntent, own webhook, own Parse write-back
                → Reuses same code path. But client gets rapid-fire charges + emails.
             b) Bundle overdue into the checkout charge (single PaymentIntent)
                → Cleaner UX (one charge). But requires new multi-payment Parse write-back.
                → Only FUTURE payments get scheduled.
           To confirm with team. See Decision 8 in decisions doc for full comparison.
```

### Phase 2: Execution (on due date)

```
EventBridge fires at scheduled time (two types of schedules fire for each milestone):

  AUTOMATIC PAYMENT REQUEST (reminder) schedule (fires N days before due date):
  → Sends "Upcoming Scheduled Payment" email to client
    - Contains: amount, date, method ending in ####, remaining payment schedule table, "Pay Now" CTA
    - "Pay Now" links to existing checkout payment flow for that milestone
    - If client pays via "Pay Now", the charge schedule is cancelled (same as "Client pays milestone via checkout link" in Phase 3)

  CHARGE schedule (fires on due date):
  → Puts message directly on upcoming payment execution SQS queue: { invoiceId, milestoneId }
  (no scheduler Lambda — matches RP pattern, execution Lambda validates status = ACTIVE)

Upcoming payment execution Lambda receives message:
  1. Fetch upcoming_payments row by (invoiceId, milestoneId)
  2. Validate status = ACTIVE
  3. POST is-stripe /backend/upcoming-payment/execute { invoiceId, milestoneId, amountCents }
     → is-stripe fetches upcoming_payment vault internally by invoiceId
     → is-stripe creates PaymentIntent off_session:
        - amount: amountCents (from request)
        - payment_method: upcoming_payment.vaultedToken (fetched internally)
        - customer: upcoming_payment.customerId (fetched internally)
        - idempotency_key: upcoming-{invoiceId}-{paymentId}-{YYYY-MM-DD}
     → returns { paymentIntentId, status }
     (matches RP pattern — Lambda calls is-stripe to execute, is-stripe owns vault lookup internally)
  5a. SUCCESS (ordering matters — see Decision 21 + chain-break recovery):
      - Step 1: Read next unpaid milestone from Parse for this invoice (by dueDate ASC, exclude already-paid)
        - If found: create 1 new `upcoming_payments` row (ACTIVE) + EventBridge schedule PAIR (charge + reminder)
        - If none found: all payments complete, no further action
      - Step 2: Update current upcoming_payments row: status → INACTIVE, disabledReason → MILESTONE_COMPLETED
      - Step 3: Send payment confirmation email to client + payment received email to merchant
      - Step 4: Return success to SQS
      (Parse Payment marked as paid automatically via Stripe webhook — no direct Parse call needed)
      
      NOTE: Step 1 BEFORE Step 2 is intentional. If Lambda fails between Step 1 and Step 2,
      SQS retries the message. Step 1 is idempotent (same schedule name → EventBridge upserts,
      row insert uses UNIQUE constraint on invoice_id+milestone_id → upsert or no-op).
      Charge already succeeded (Stripe idempotency). No double-charge, no lost schedule.
  5b. FAILURE:
      - Increment retryCount
      - POST is-payments /upcoming-payment/record-failure → returns { shouldScheduleRetry, nextRetryDate }
          is-payments decides: shouldScheduleRetry = !isAsyncPaymentMethod && retryCount <= MAX_RETRIES
          (same pattern as RP — Lambda acts on the boolean, doesn't check paymentMethodType directly)
      - If shouldScheduleRetry:
          - Create EventBridge retry schedule (1/3/6 day intervals)
          - Send "retrying" notification to merchant
      - Else (max retries reached):
          - Set THIS milestone: status → INACTIVE, disabledReason → MAX_RETRIES
          - Chain stops naturally — no future row or schedule ever gets created
          - Vault record in is-stripe is kept (NOT deleted) — gets replaced if client re-vaults
          - Merchant tile reverts to non-vaulted state ("client can authorize at checkout")
          - Send failure emails to merchant + client
          - Client must check out again to re-vault and restart auto-pay
          NOTE: With single-row creation (Decision 21), max retries cleanup is trivial:
            - Only 1 row exists (the failed one) — set it INACTIVE
            - Only the failed payment's retry schedule (if any) needs cleanup
            - No QUEUED rows to clean up (they were never created)
```

### Phase 3: Lifecycle Events (post-vault)

> **Decision 14 (revised): Only the NEXT scheduled payment is locked.** With single-row creation
> (Decision 21), only one payment has an is-payments row + EventBridge schedule at any time. That payment is
> immutable. Future payments remain in Parse only (editable: amount, date, add, delete) — no row or schedule
> exists for them, so changes are picked up when the chain reaches them.

Communication pattern: Mobile calls is-payments directly (fire-and-forget REST). See Decision Q1.

```
Merchant edits/deletes a future payment (no is-payments row exists):
  → No confirmation needed — no row or schedule exists for this payment
  → Mobile edits Parse directly (normal flow)
  → When the chain reaches this payment (prior one succeeds), handle-success reads
    current values from Parse (amount, date) to create the row + schedule
  → If payment was deleted from Parse, handle-success skips it and moves to the next unpaid

Merchant attempts to edit/delete the ACTIVE (next scheduled) payment:
  → Confirmation modal: "This will cancel the next scheduled payment. Auto-pay will
    continue with the following payment. Continue?"
  → If merchant confirms:
      → Mobile: POST is-payments /backend/upcoming-payment/skip { invoiceId, milestoneId }
        → is-payments: delete EventBridge schedule PAIR for that payment (charge + reminder)
        → is-payments: set that row to INACTIVE, disabledReason → SKIPPED
        → is-payments: read next unpaid milestone from Parse, create new row (ACTIVE) + schedule pair
      → Mobile: proceed with Parse edit/delete as normal
  → If merchant cancels modal: no action

Merchant edits the invoice (total, client, etc.):
  → Confirmation modal: "This will cancel all automatic payments. Client will need to
    authorize again at checkout. Continue?"
  → If confirms: POST /backend/upcoming-payment/disable → full reset
  NOTE: Invoice-level edits (total, client) still trigger full cancel. Only per-payment
  edits on QUEUED rows are free.

Client pays an upcoming payment via checkout link / "Pay Now" in Automatic Payment Request (reminder) email:
  → Client pays through is-unifiedxp checkout
  → Parse Payment marked as paid automatically via Stripe webhook
  → Checkout (is-unifiedxp): POST is-payments /backend/upcoming-payment/cancel-one { invoiceId, milestoneId }
    → is-payments: delete EventBridge schedule PAIR for that payment (charge + reminder)
    → is-payments: set that upcoming_payments row to INACTIVE, disabledReason → MANUALLY_PAID
    → is-payments: read next unpaid milestone from Parse, create new row (ACTIVE) + schedule pair (chain advances)
  NOTE: Auto-pay continues — chain just advances to the next payment.

Merchant or Admin cancels ALL automatic payments:
  (triggered from: Cancel AP button on invoice in mobile app, or Admin tool via support)
  → POST is-payments /backend/upcoming-payment/disable
    → Delete EventBridge schedule PAIR for ACTIVE payment (only one exists)
    → Set the ACTIVE upcoming_payments row to INACTIVE, disabledReason → USER_DISABLED
    → (No QUEUED rows to clean up — they were never created. Decision 21.)
  NOTE: Vault record is NOT deleted. Client checks out again to re-vault if needed.
  NOTE: No client self-service cancel. Client must contact merchant to cancel.
```

**API surface:**

| Endpoint | Purpose | Who calls |
|----------|---------|-----------|
| `POST /backend/upcoming-payment/disable` | Cancel ALL (full reset). Vault record kept (not deleted) — gets replaced on re-vault. (Note: vault deletion may be added later for consistency with incoming RP work; if so, delete here too.) | Mobile, Admin, Checkout (invoice-level) |
| `POST /backend/upcoming-payment/cancel-one` | Cancel ONE payment (paid early), advance chain | Checkout |
| `POST /backend/upcoming-payment/skip` | Skip/cancel next ACTIVE payment, advance chain | Mobile (merchant action) |

Future payments (no is-payments row) are edited directly in Parse — no is-payments API needed. System reads current Parse values when the chain advances and creates the next row + schedule.

### Client Pays via Checkout: Manual Payment vs Re-vault

When the client opens a checkout link for an upcoming payment, they see the auto-pay checkbox
(same as initial vault). This handles two scenarios: recovering from failure, and re-authorizing
with a new card. Matches RP behavior (`payment-intent-confirm.ts`).

**In both cases, retry schedules for the current payment are always cancelled** (same as RP).

```
Scenario: Client opens checkout link to pay an upcoming payment
  (typical after max retries reached — all schedules already cancelled,
  or merchant manually sent checkout link for an outstanding payment)

  → Client pays the current payment at checkout

  CASE A — Client does NOT check "Enable Automatic Payments" (manual payment):
    → Payment succeeds, Parse Payment marked paid via Stripe webhook
    → Cancel any active retry schedules for this payment (if any exist)
    → Cancel ALL remaining upcoming payment schedules (if any still active)
    → Set all remaining upcoming_payments rows to INACTIVE, disabledReason → USER_DISABLED
    → Vault record is kept (not deleted, same as RP) but auto-pay is effectively OFF
    → Remaining payments revert to manual — merchant must send links or mark paid
    → Rationale: client explicitly chose NOT to consent. Keeping active schedules
      charging their card contradicts their choice.

  CASE B — Client checks "Enable Automatic Payments" (re-vault):
    → Payment succeeds, Parse Payment marked paid via Stripe webhook
    → Cancel ALL existing schedules (retries + remaining charge/reminder pairs)
    → Set any existing ACTIVE row to INACTIVE
    → Vault record REPLACED with new token + new consentGrantedAt
    → Create 1 fresh `upcoming_payments` row for the NEXT unpaid milestone:
      - status: ACTIVE, retryCount: 0
      - amountCents/scheduledDate from Parse (current values)
    → Create EventBridge schedule PAIR for that payment (charge + Automatic Payment Request)
    → Auto-pay restarted with new card and clean slate (Decision 21 — one row at a time)
    → Client sees remaining schedule at checkout and re-consents to those specific payments
    → Rationale: fresh consent = fresh start. New card may succeed where old one failed.
```

**Example flow (max retries → re-vault):**
```
May 1:   Client vaults, pays Payment 1. Payments 2-5 scheduled.
Jun 1:   Payment 2 charged successfully.
Jul 1:   Payment 3 fires — FAILS (card expired).
Jul 2:   Retry 1 — fails.
Jul 5:   Retry 2 — fails.
Jul 11:  Retry 3 — fails. MAX RETRIES → Payments 4, 5 cancelled. All schedules deleted.
Jul 12:  Merchant sends checkout link to client for Payment 3.
Jul 13:  Client opens link, sees:
           - Payment 3: $1,500 (pay now)
           - ☐ Enable Automatic Payments
             Payment Schedule:
               Payment 4  Aug 15  $1,000
               Payment 5  Sep 15  $1,000

         Client enters new card, checks auto-pay box → pays Payment 3 ($1,500),
         vaults new card, Payment 4 gets ACTIVE schedule, Payment 5 QUEUED.
```

**Example flow (manual payment — no re-vault):**
```
Same scenario as above, but at step Jul 13:
  Client opens link, pays Payment 3 ($1,500) but does NOT check auto-pay box.
  → Payment 3 paid. No vault. Payments 4 & 5 remain manual.
  → Merchant must send links for Payments 4 and 5 when they're due.
```

**Backend implementation:**
- Checkout (`payment-intent-confirm.ts` equivalent for upcoming payments):
  1. Always: cancel retry schedules for this payment
  2. If consent NOT granted: `POST /upcoming-payment/disable` (cancel all remaining)
  3. If consent granted: `POST /upcoming-payment/disable` + `POST /upcoming-payment/vault` (cancel all → re-create with new vault)
- This matches RP's pattern: `publishCancelRetrySchedulesEvent()` always fires,
  `handleConsentGranted()` / `setupAndPublishRecurringPayment()` only fires with consent

**API surface addition:**

| Endpoint | Purpose | Who calls |
|----------|---------|-----------|
| `POST /backend/upcoming-payment/re-vault` | Cancel all + create fresh schedules with new vault | Checkout (on re-consent) |

---

## Checkout / Client UX (is-unifiedxp)

### What the client sees

The opt-in is an **inline checkbox on the checkout page**, above the payment methods list. The milestone schedule (dates + amounts) is shown inline before the client checks the box — they see exactly what they're consenting to.

```
┌─────────────────────────────────────────────────────┐
│  DEPOSIT DUE           USD $400.00                  │
│  DEPOSIT               20%                          │
│  ONLINE PAYMENT FEE    USD $13.96                   │
│  PAYMENT AMOUNT        USD $413.96                  │
│                                                     │
│  ☐ Enable Automatic Payments                        │
│    By enabling, you authorize charging your         │
│    selected payment method for this invoice.        │
│                                                     │
│    Payment Schedule:                                │
│    Milestone Date      Jan 15, 2026   $400.00       │
│    Milestone Date      Feb 15, 2026   $400.00       │
│    Milestone Date      Mar 28, 2026   $400.00       │
│    Milestone Date      Mar 28, 2026   $400.00       │
│                                                     │
│  Email Address                                      │
│  [hello@johnsmith.com             ]                 │
│  ✓ Your information will be securely stored         │
│                                                     │
│  PAYMENT METHODS                                    │
│  [Apple Pay]  [G Pay]  [Pay with Link]              │
│  ○ Card  ○ US Bank Account  ○ Pay Later  ○ PayPal   │
│                                                     │
│  [Pay Now]                                          │
└─────────────────────────────────────────────────────┘
```

### Key UX Rules

1. **Opt-in is optional.** Client can pay deposit without auto-pay. No vault created, future milestones stay manual.
2. **Schedule visible before consent.** The full milestone schedule (dates + amounts) is shown inline above the payment methods, before the client checks the box. PERCENT milestones show resolved dollar amount.
3. **Consent copy.** "By enabling, you authorize charging your selected payment method for this invoice." Specific to this invoice, not open-ended like RP's "future invoices."
4. **Email required.** Email address is mandatory before auto-pay can be set up (validated before vault creation).
5. **PayPal not supported for auto-pay.** When PayPal is selected, the "Enable Automatic Payments" checkbox becomes disabled ("This feature is available only for select payment methods"). Stripe card/ACH and Stripe Link only.
6. **No milestones at checkout = no auto-pay opt-in.** If merchant only created a deposit with no future milestones, don't show the opt-in.
7. **No client self-service cancel.** Client must contact merchant to cancel. Emails say: "To make changes to or cancel automatic payments, please contact {Merchant Business Name} at {Merchant Business Email}." Only merchant (mobile Cancel AP button) or admin (support tool) can disable.
8. **"Pay Later" not eligible for auto-pay.** Only card and US bank account payment methods support vaulting for off-session charges.

### Reuse from RP Checkout

| Component | Reuse | Changes |
|---|---|---|
| Stripe Elements (card input) | As-is | None |
| PaymentIntent with `setup_future_usage` | Pattern | Same vaulting mechanism as RP (NOT a SetupIntent — single call charges + vaults) |
| Opt-in checkbox | Structure | Different copy, conditional on milestones existing |
| `handleRecurringPaymentConfirm` | Logic | New handler writing to `upcoming_payment` table |
| Success redirect | Structure | Show schedule confirmation |

### New Components

- Milestone schedule display (list of upcoming payments with amounts + dates)
- Email templates:
  - "Automatic Payments Enabled" — sent at vault time (summary + full schedule table)
  - "Upcoming Scheduled Payment" — Automatic Payment Request (reminder), sent N days before due date (amount, date, method, schedule table, "Pay Now" CTA)
  - "Scheduled Payment Successful" — sent after charge (amount charged, remaining schedule table, "View Invoice" CTA)
  - Payment failure / retrying — sent to merchant on failure
  - All cancelled — sent to merchant + client on max retries

---

## Terminology

"Recurring payment" strongly implies regularity (same amount, fixed interval). Milestones are irregular and open-ended.

**PRD terminology mapping:**
- **Automatic Payment Request** (PRD) = automated email about an upcoming payment. Works for all users including non-Stripe. This doc calls it "Automatic Payment Request (reminder)" for clarity.
  - **Non-vaulted context (Era 2):** Sends a checkout link email **on the due date** (PRD v1) or 3 days before (PRD v2). No card on file — this IS the payment request. Equivalent to "Email Invoice" but triggered automatically.
  - **Vaulted context (Era 3, this design):** Sends a "heads up, we're charging your card" email **N days before** the due date. Card is on file — the charge fires on due date regardless. "Pay Now" CTA lets client pay early and cancel the auto-charge.
- **Vaulted Payment Automation** (PRD) = card vaulting + auto-charge on due date. Stripe only. This is the core of what this design doc covers.
- **Payment Request Automation + Vaulted Payment Automation** (PRD) = the full combined experience (reminders + auto-charge). Tier 1 GTM.

**Era → PRD mapping:**

| Era | Scope | PRD equivalent | GTM |
|-----|-------|----------------|-----|
| Era 1 | Unified Scheduling System — consolidate Payments Schedule + Invoice Terms + Abandoned Cart into one system; enforce dates (not-in-past, required for auto-pay) | "Clean up: Unified Scheduling System for Payments Schedule + Invoice Terms + Abandoned Cart Automation" | Prerequisite |
| Era 2 | Automatic Payment Request (reminder emails, non-vaulted) | "Payment Request Email Automation" | Tier 2 |
| Era 3 | Vaulted Payment Automation (card vaulting + auto-charge) | "Payment Request Automation + Vaulted Payment Automation" (combined) | Tier 1 |

**User-facing term:** "Automatic payments" — covers both RP and milestones. RP already uses this in its UI copy.

**Internal codebase terms:**
- `recurring_payment` / `recurring_payments` — existing RP tables (series-based)
- `upcoming_payment` / `upcoming_payments` — new tables (invoice-based, Decision 13)
- Both are types of "automatic payment" from the user's perspective.

---

## Infrastructure: AWS (EventBridge, SQS, Lambda)

### Current RP Architecture

```
                                 ┌─────────────────────────┐
is-stripe / is-recurring-invoices│                         │
         │                       │  EventBridge Scheduler  │
         ▼                       │  (retry schedule group) │
┌─────────────────────┐          │                         │
│ payment-events-queue│          └───────────┬─────────────┘
│      (.fifo)        │                      │ at(scheduled time)
└─────────┬───────────┘                      ▼
          ▼                       ┌──────────────────────┐
┌─────────────────────┐           │ retry-scheduler      │
│ events-lambda       │           │ Lambda               │
│ (topic dispatcher)  │           │ (posts to exec queue)│
└─────────────────────┘           └──────────┬───────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │ payment-execution    │
                                  │ queue (standard)     │
                                  └──────────┬───────────┘
                                             │
                                             ▼
                                  ┌──────────────────────┐
                                  │ payment-execution    │
                                  │ Lambda               │
                                  │ (charges Stripe)     │
                                  └──────────────────────┘
```

### What's Reused Directly (no changes)

| Resource | Name | Why reusable |
|---|---|---|
| Schedule group | `is-payments-retry-schedule-group` | Container for schedules. Upcoming payment schedules coexist — names don't collide (`payment-retry-{seriesId}-{count}` vs `upcoming-{invoiceId}-{paymentId}`) |
| Events queue (FIFO) | `is-payments-events-queue.fifo` | Add new upcoming payment topics. Low-volume, FIFO with `messageGroupId: invoiceId` ensures ordering per invoice |
| Events Lambda | `is-payments-events-lambda` | Already a topic dispatcher (switch statement). Add upcoming payment topic cases |
| DLQ alarm SNS topic | Existing alarm construct | Same monitoring for upcoming payment failures |
| IAM permission patterns | `scheduler:CreateSchedule`, `scheduler:DeleteSchedule`, `iam:PassRole` | Same permissions needed |

### What's New

#### 1. Upcoming Payment Execution Lambda

EventBridge fires directly to SQS execution queue on each milestone's due date (no intermediate scheduler Lambda — Decision 11). The execution Lambda validates status and charges.

Processes upcoming payment execution (initial + retries).

```
CDK: IsPaymentsUpcomingPaymentExecutionLambdaConstruct
  Function name: is-payments-upcoming-payment-execution
  Handler:       is-payments/src/lambda-handlers/upcoming-payment-execution.ts
  Runtime:       Node.js 22.x
  Memory:        256MB
  Timeout:       5 min
  Trigger:       SQS event (batchSize: 1)
  Queue:         is-payments-upcoming-payment-execution-queue
  Env vars:
    - IS_STRIPE_SERVICE_URL
    - IS_STRIPE_API_KEY
    - IS_PAYMENTS_SERVICE_URL (self, for record-success/failure)
    - IS_PAYMENTS_API_KEY
    - RETRY_SCHEDULE_GROUP_NAME (reuse existing group)
    - UPCOMING_PAYMENT_EXECUTION_QUEUE_URL (for retry schedules — posts back to own queue)
  Permissions:
    - scheduler:CreateSchedule + scheduler:DeleteSchedule (for retry schedules)
    - sqs:SendMessage (for retry — EventBridge target is this queue)
    - iam:PassRole (for scheduler role)
```

Handler logic:
```typescript
// Receives from SQS (EventBridge fires directly to queue)
{ invoiceId: string, milestoneId: string }

// Steps:
// 1. Fetch upcoming_payments row by (invoiceId, milestoneId) → validate status = ACTIVE
// 2. POST is-stripe /backend/upcoming-payment/execute { invoiceId, milestoneId, amountCents }
//    (is-stripe fetches vault internally by invoiceId)
// 3a. Success → read next unpaid milestone from Parse
//              → If found: create new row (ACTIVE) + EventBridge schedule pair
//              → Update current row: INACTIVE, MILESTONE_COMPLETED
//              → Parse Payment marked paid via Stripe webhook (no direct Parse call)
//              → Publish upcomingPaymentSuccess topic (for emails)
// 3b. Failure → increment retryCount
//              → If shouldRetry: create EventBridge retry schedule → fires back to this queue
//              → If max retries: set row INACTIVE (no other rows to clean up — Decision 21)
//              → Publish upcomingPaymentFailure topic (for emails)
```

#### 2. Upcoming Payment Execution SQS Queue

```
CDK: IsPaymentsUpcomingPaymentExecutionSqsConstruct
  Queue:              is-payments-upcoming-payment-execution-queue (standard)
  DLQ:               is-payments-upcoming-payment-execution-dlq (standard)
  Visibility timeout: 360s (6 min, 1.2x Lambda timeout)
  Max receive count:  3
  Alarm:             DLQ message count > 0 → SNS (reuse existing alarm topic)
```

#### 3. New Topics on Existing Events Queue

New topics added to `is-payments-events-queue.fifo`, processed by existing `events-lambda`:

```typescript
"createUpcomingPayment"      // is-stripe → is-payments after vault
"upcomingPaymentSuccess"     // for logging, emails, analytics
"upcomingPaymentFailure"     // for logging, emails, analytics
"upcomingPaymentReminder"    // reminder schedule fires → send email with "Pay Now" CTA
```

Note: No `cancelUpcomingPaymentSchedules` topic. Cancel-all is handled synchronously via `POST /upcoming-payment/disable` REST endpoint (mobile, checkout, or admin).

FIFO ordering with `messageGroupId: invoiceId` ensures upcoming payment events for the same invoice are processed sequentially.

### Why Separate Execution Queue (not shared with RP)

| Aspect | Shared queue | Separate queue (chosen) |
|---|---|---|
| Isolation | RP + upcoming payment in same Lambda | Independent deploys, no RP regression risk |
| Scaling | Scale together | Scale independently |
| Monitoring | Shared metrics | Upcoming payment-specific alarms and dashboards |
| Complexity | Discriminator on messages, dispatch logic | Clean separation, slightly more CDK |

Chosen: **separate queue + Lambda** for full isolation during active development.

### Upcoming Payment Architecture Diagram

```
Client vaults at checkout
        │
        ▼
is-stripe: create vault record
        │
        ▼ SQS (FIFO)
┌────────────────────────────┐
│ payment-events-queue.fifo  │
│ topic: createUpcoming-     │
│        Payment             │
└────────────┬───────────────┘
             ▼
┌────────────────────────────┐
│ events-lambda              │
│ → creates 1 upcoming_      │
│   payments row (ACTIVE)    │
│   for next milestone       │
│ → creates EventBridge      │
│   schedule PAIR for it     │
│   (charge + reminder)      │
└────────────────────────────┘

On due date (no scheduler Lambda — EventBridge fires directly to SQS):
┌─────────────────────────┐
│ EventBridge Scheduler   │
│ at(2026-06-01T{vaultTime}) │
└───────────┬─────────────┘
            │ (direct to SQS)
            ▼
┌──────────────────────────┐
│ upcoming-payment-        │
│ execution queue          │
│ (standard)               │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│ upcoming-payment-        │
│ execution Lambda         │
│ → validate ACTIVE        │
│ → charge via is-stripe   │
│ → record success/failure │
│ → schedule retry if fail │
└──────────────────────────┘

On retry (same path):
EventBridge at(retry date) → SQS execution queue → execution Lambda
```

### EventBridge Schedule Naming

```
Charge:    upcoming-{invoiceId}-{paymentId}
Reminder:  upcoming-reminder-{invoiceId}-{paymentId}
Retry:     upcoming-retry-{invoiceId}-{paymentId}-{retryCount}
```

All in the shared `is-payments-retry-schedule-group`. No collision with RP schedules (`payment-retry-{seriesId}-{retryCount}`).

Schedules are created/cancelled as **pairs** (charge + reminder). All schedules use `ActionAfterCompletion: 'DELETE'` — auto-cleanup after firing.

### Summary: Resource Count

```
REUSED (unchanged):                          5 resources
  ✅ is-payments-retry-schedule-group
  ✅ is-payments-events-queue.fifo
  ✅ is-payments-events-lambda
  ✅ DLQ alarm SNS topic
  ✅ IAM permission patterns

NEW (created):                               4 resources
  🆕 is-payments-upcoming-payment-execution-queue
  🆕 is-payments-upcoming-payment-execution-dlq
  🆕 is-payments-upcoming-payment-execution-lambda
  🆕 CDK construct files

MODIFIED:                                    2 changes
  ✏️ events-lambda handler (add upcoming payment topic cases)
  ✏️ new upcoming-payment-scheduler.ts utility (parallel to RP scheduler)
```

No separate scheduler Lambda — EventBridge fires directly to SQS (Decision 11).

### RP Code as Reference (not direct reuse)

Most RP code is tightly coupled to `recurringInvoiceSeriesId`. We write new implementations referencing these patterns:

| What to build | RP reference | Notes |
|---|---|---|
| `upcoming-payment-execute.ts` | `recurring-payment-execute.ts` | New file, same Stripe SDK primitives |
| `upcoming-payment-scheduler.ts` | `scheduler.ts` | New schedule create/delete functions, same AWS SDK calls |
| `upcoming-handle-success/failure.ts` | `handle-success.ts` / `handle-failure.ts` | New handlers, similar flow |
| Vault token extraction | `stripe-handle-recurring-payment-confirm.ts` | Same Stripe SDK operation |
| Email sends | `send-automatic-payment-successful-email.ts` | May be reusable if `recurringInvoiceSeriesId` made optional |

### Potential Issues

1. **Vaulting extraction** — `stripe-handle-recurring-payment-confirm.ts` hardcodes `recurringInvoiceSeriesId`. Safest path: thin wrapper that both RP and upcoming payment handlers call, or duplicate for upcoming payments (small amount of code).
2. **Schedule naming collisions** — Mitigated by distinct prefixes (`payment-retry-` vs `upcoming-` vs `upcoming-retry-`).
3. **Amount for PERCENT mode** — Resolved at schedule creation (snapshot). Invoice-level edits trigger full reset, so stale amounts aren't charged.
4. **EventBridge Scheduler limits** — AWS limit is 1,000,000 schedules per account. Even at scale, upcoming payment schedules are bounded (few per invoice). Not a concern.

---

## Merchant-Side Mobile UI

### Component Integration

Current mobile payment scheduling structure:
```
Invoice Screen → PaymentSchedulingScreen
  ├── InvoiceTotalSection
  ├── UpcomingPaymentsSection
  ├── PaymentHistorySection
  └── CheckDepositsSection
```

With auto-pay:
```
Invoice Screen → PaymentSchedulingScreen
  ├── InvoiceTotalSection
  ├── AutomaticPaymentsSection  ← NEW
  ├── UpcomingPaymentsSection   ← MODIFIED (schedule indicators)
  ├── PaymentHistorySection
  └── CheckDepositsSection
```

### AutomaticPaymentsSection States

Auto-pay is implicit — no merchant toggle. The section appears when eligible and reflects vault state:

**Not vaulted (client hasn't opted in yet):**
```
┌─────────────────────────────────────────────┐
│  Automatic Payments                         │
│  Your client will be prompted to set up     │
│  auto-pay when they pay via checkout.       │
└─────────────────────────────────────────────┘
```

**Vault active, upcoming payments scheduled:**
```
┌─────────────────────────────────────────────┐
│  Automatic Payments                    ● On │
│  Next charge: $2,000 on Jun 1, 2026         │
│                                             │
│  [Disable Automatic Payments]               │
└─────────────────────────────────────────────┘
```
Note: V1 shows generic status (no card details). Card last4/type can be passed from is-stripe
at creation time (optional fields on create endpoint) but not stored on `upcoming_payments` table
in V1 — same as RP. Future: may add columns and show
"Visa ****4242" if design requests it.

**Vault disabled (failed / cancelled):**
```
┌─────────────────────────────────────────────┐
│  Automatic Payments                   ● Off │
│  Disabled: payment failed after 3 attempts  │
│                                             │
│  Client can re-enable at next checkout.     │
└─────────────────────────────────────────────┘
```

**Offline (network unavailable, vault active):**
```
┌─────────────────────────────────────────────┐
│  OfflineSectionCover                        │
│  Editing disabled while offline.            │
│  Automatic payments are active.             │
└─────────────────────────────────────────────┘
```
Per Decision 17: when vault is active and device is offline, the entire Payment Scheduling
section is covered with `OfflineSectionCover` (same pattern as RP). Edits require network
to sync cancellations with is-payments.

### UpcomingPaymentsSection Modifications

Rows show different states based on scheduling status (Decision 14 revised):
```
┌─────────────────────────────────────────────┐
│  Upcoming Payments                          │
│                                             │
│  Midpoint  $2,000  Due Jun 1  🔒 Scheduled │  ← ACTIVE (locked, has is-payments row)
│  Final     $2,000  Due Jul 1  ⚡ Auto      │  ← future (editable, Parse only — no row yet)
│                                             │
│  [Add Upcoming Payment]                     │  ← enabled (adds to Parse only)
└─────────────────────────────────────────────┘
```

### Locked State (Decision 14 revised — only next payment locked)

With single-row creation (Decision 21), only the ACTIVE payment (which has an is-payments row + schedule) is locked:

1. **ACTIVE payment (has is-payments row + schedule):** Edit/delete disabled. Shows "Scheduled for [date]" badge. Actions available:
   - "Skip this payment" → confirmation → `POST /skip` → chain advances (creates next row + schedule)
   - "Disable All Automatic Payments" → confirmation → `POST /disable` → full reset
2. **Future payments (Parse only, no is-payments row):** Fully editable. Edit amount, date, add new, delete — all normal.
   No confirmation needed. System reads current Parse values when the chain reaches them.
3. **"Add Upcoming Payment" button:** Enabled (new payment added to Parse only).

"Mark as Paid" on the ACTIVE payment → "Skip this payment" confirmation → chain advances.
"Mark as Paid" on a future payment → no confirmation, just mark in Parse (chain skips it naturally).

### Merchant Flows

**Auto-pay eligibility is implicit** — no merchant toggle. If the account is Stripe-eligible and the feature flag is enabled, checkout automatically shows the auto-pay consent option to the client. The merchant doesn't "turn on" auto-pay; the client opts in by vaulting.

**Skip next payment:** Merchant taps "Skip" on the ACTIVE payment → confirmation: "This will cancel the next scheduled payment. Auto-pay will continue with the following payment." → `POST /skip` → chain advances to next QUEUED payment.

**Disable all:** Merchant taps "Disable Automatic Payments" → confirmation: "This will cancel all scheduled payments. Your client will need to set up auto-pay again at their next checkout." → full reset: cancel ACTIVE payment's EventBridge schedule pair, set the ACTIVE row INACTIVE with reason `USER_DISABLED`. No QUEUED rows to clean up (Decision 21). Vault record kept (not deleted) — gets replaced on re-vault. (Note: vault deletion may be added later for consistency with incoming RP work.)

### Data Requirements

```typescript
type AutoPayStatus = {
  vaulted: boolean
  activePayment?: {                    // from is-payments (the single ACTIVE row)
    milestoneId: string
    amountCents: number
    scheduledDate: string
    status: 'ACTIVE'
    retryCount: number
    disabledReason?: string
  }
  completedPayments: Array<{           // from is-payments (INACTIVE rows — history)
    milestoneId: string
    amountCents: number
    disabledReason: string             // MILESTONE_COMPLETED, SKIPPED, etc.
  }>
}
// Future milestones (not yet scheduled) come from Parse — already loaded by mobile/web
```

Fetched from: `GET /backend/upcoming-payment/status?invoiceId={id}` (is-payments endpoint)

**UI uses is-payments + Parse data together to determine editability:**
- **ACTIVE row (from is-payments)** → locked row (🔒 badge, edit/delete disabled, "Skip" action available)
- **Future milestones (from Parse, no is-payments row yet)** → editable row (⚡ badge, normal edit/delete controls)
- **INACTIVE rows (from is-payments)** → completed/cancelled (greyed out or hidden)

### High-Level Web Plan

Same concepts apply to web (is-web-app):
- New `AutomaticPaymentsSection` in `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/`
- Schedule badges: 🔒 Scheduled (ACTIVE, has is-payments row) vs ⚡ Auto (future, Parse only)
- Only ACTIVE (next) payment locked — future milestones fully editable in Parse
- "Skip" action on ACTIVE row, "Disable All" as nuclear option
- "Add Upcoming Payment" enabled (adds to Parse only)
- Offline: not applicable (web always online)
- Same is-payments API endpoints for status/skip/disable

---

## Emails & Notifications

### Email Triggers

| Event | Recipient | Content |
|---|---|---|
| Client vaults successfully | Client | "Automatic payments set up. Your [card ending 4242] will be charged on the dates below: [schedule list]" |
| Client vaults successfully | Merchant | "Your client has set up automatic payments for invoice #123" |
| Upcoming payment reminder (before charge) | Client | "Reminder: $2,000 will be charged to your card ending 4242 on [date] for [milestone label]" |
| Upcoming payment charged successfully | Client | "Payment of $2,000 was charged to your card ending 4242 for [milestone label]" |
| Upcoming payment charged successfully | Merchant | "Payment received: $2,000 for [milestone label] on invoice #123" |
| Upcoming payment charge failed (retrying) | Merchant | "Payment of $2,000 failed for [milestone label]. We'll retry automatically." |
| Upcoming payment charge failed (final — max retries) | Merchant | "Payment of $2,000 failed after 3 attempts for [milestone label]. Action needed." |
| Upcoming payment charge failed (final — max retries) | Client | "Your payment of $2,000 could not be processed. Contact [merchant] to arrange payment." |
| Auto-pay cancelled (merchant disabled) | Client | "Automatic payments for invoice #123 have been cancelled." |

### Reuse from RP Emails

RP has templates for payment success, payment failure (retrying + final), and cancellation. The structure (sender, amount, date, payment method) is the same. Milestone emails need new templates with the same layout/branding but milestone-specific copy (milestone label, specific dates).

### Push Notifications (Mobile)

Key events for merchant via Firebase Cloud Messaging:
- "Payment received: $2,000 for Midpoint on invoice #123"
- "Payment failed for Midpoint. We'll retry in 24 hours."
- "Payment failed after 3 attempts. Tap to manage."

### What's NOT Notified

- Individual retry attempts (only first failure + final failure)
- Vault idle with no active schedules
- Schedule changes (Decision 14 revised: QUEUED edits are free, only ACTIVE payment is locked)

---

## Edge Cases & Safety

### Merchant sends invoice while vault is active

When the merchant taps "Send" on an invoice that has an active vault (client has already vaulted), a **confirmation modal** must appear before the invoice is sent:

```
┌─────────────────────────────────────────────────────┐
│  Automatic Payments are Enabled for this Invoice    │
│                                                     │
│  By sending this invoice to {Client Name}, you are  │
│  making the following charge:                       │
│                                                     │
│  Method    {Method} - {account/card} ending ######  │
│  Amount    $400.00                                  │
│  Online Payment Fee  $13.96                         │
│  Total     $413.96                                  │
│                                                     │
│  [Agree & Continue]                                 │
└─────────────────────────────────────────────────────┘
```

This is a new merchant confirmation step — sending a subsequent milestone invoice while auto-pay is active is effectively triggering an upcoming charge. The merchant must explicitly agree before the invoice is dispatched to the client.

**Scope:** Mobile send flow (Ticket 5b). Web send flow to follow.

---

### Milestone paid via checkout link while schedule is active

A client can pay an upcoming milestone online via a "View online payment" or "Send deposit email" link — not just at the initial deposit checkout. If auto-pay is active for that invoice and the client pays the milestone manually through checkout, the schedule for that milestone must be cancelled to avoid double-charging.

**Behavior:** When checkout records a successful payment for a milestone (sets `amount`, `date`, `method` on the Parse Payment), it must also call `POST /backend/upcoming-payment/cancel-one` with `invoiceId` and `milestoneId`. Cancel that EventBridge schedule pair, set `upcoming_payments` row to INACTIVE with disabledReason: `MANUALLY_PAID`.

**Note:** This is an additional trigger point in is-unifiedxp's payment confirmation flow, separate from the mobile merchant mark-paid flow below.

### Milestone manually marked as paid while schedule is active

Merchant marks a milestone as paid (cash, check, etc.) through the app.

**Behavior (depends on status):**
- **ACTIVE (next scheduled):** Confirmation modal: "This will skip this scheduled payment and advance auto-pay to the next one." → `POST /backend/upcoming-payment/skip`. Schedule cancelled, chain advances.
- **QUEUED (future):** No confirmation needed. Mark as paid in Parse. When handle-success reaches it, it sees the payment is already completed and skips to the next QUEUED row.

### Client pays deposit WITHOUT opting into auto-pay

**Behavior:** Normal payment. No vault created, no schedules. Milestones remain manual. Merchant can re-send invoice later for client to set up auto-pay.

### Invoice deleted or voided

**Behavior:** Full cleanup — cancel all EventBridge schedules, set all rows INACTIVE, delete vault record.

### Stripe payment method expires before milestone date

**Behavior:** Charge fails at execution time (Stripe returns `card_expired`). Normal failure/retry flow. If all retries fail, merchant notified. No proactive "card expiring" check (same as RP).

### Merchant changes client email on invoice

Invoice-level edit → full reset (Decision 14). Confirmation modal → cancel all schedules, vault invalidated. New client would need to vault fresh.

### Multiple overdue milestones at vault time, partial failure

Client vaults and 3 milestones are overdue. First two charge successfully, third fails.

**Behavior:** Each overdue milestone is an independent execution. First two: MILESTONE_COMPLETED. Third: enters retry flow (1/3/6 days). No rollback of successful charges.

### Milestone with no due date

Deposits have no due date (by design — paid at checkout). A non-deposit milestone should always have a due date (form requires it).

**Behavior:** If somehow a milestone has no due date, no schedule is created. It stays as a manual upcoming payment. UI nudges merchant: "Add a due date to enable automatic payment for this milestone."

### Merchant edits a future payment while vault is active

Per Decision 14 (revised) + Decision 21, future payments (no is-payments row) are **freely editable**. No confirmation, no API call to is-payments. Merchant edits directly in Parse. When the chain reaches that payment, handle-success reads the current values (amount, date) from Parse to create the row + schedule.

### Merchant edits the ACTIVE (next scheduled) payment while vault is active

The ACTIVE payment is immutable (has is-payments row + EventBridge schedule). Merchant must confirm: "This will skip this payment. Auto-pay continues with the next one." → `POST /backend/upcoming-payment/skip` → chain advances (creates new row + schedule for next unpaid milestone from Parse).

### Vault-time validation: overpayment gate & remaining balance (Decision 2026-05-29)

> Decided in Slack thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779994686.931629
> Participants: Lenmor, Liz, Seth

**Overpayment gate — block vaulting if milestones sum > invoice total:**

At vault time, checkout validates: `sum(milestone.amountCents for all unpaid milestones) <= invoice.total`. If the sum exceeds the invoice total, vaulting is blocked:
- Client does NOT see the auto-pay opt-in checkbox (or it's disabled with explanation)
- Merchant sees an inline warning on the Payment Scheduling screen: "Automatic payments unavailable — scheduled payments exceed the invoice total. Adjust payment amounts to enable."
- Rationale: merchants wouldn't intentionally schedule more than the total (typo prevention), and this mitigates chargebacks (buyer never auto-charged beyond invoice balance)

**Remaining balance — milestones don't cover full invoice total:**

When `sum(milestones) < invoice.total`, vaulting is allowed. Auto-pay only charges the explicitly scheduled milestones. The remaining balance is handled manually:

| Scenario | Behavior |
|----------|----------|
| `sum(milestones) > invoice.total` | Block vaulting, show warning to merchant |
| `sum(milestones) < invoice.total` | Allow vaulting, auto-pay scheduled milestones only, remaining = manual |
| `sum(milestones) = invoice.total` | Happy path, full automation |

**What happens after all scheduled milestones complete:**
1. Last milestone fires and succeeds
2. `handle-success` reads next unpaid milestone from Parse → finds none → no new row/schedule created
3. Auto-pay is effectively complete (no more ACTIVE rows, no more schedules)
4. `invoice.balanceDue` reflects only the unscheduled remainder (already updated by successful milestone payments via Stripe webhooks → Parse `invoiceAddPayment`)
5. Client can pay remaining balance manually via checkout link (normal flow, no re-vault needed)

**No special remaining-balance logic needed:**
- The schedule system is purely milestone-driven — it doesn't track or reason about `invoice.total`
- `invoice.balanceDue` is updated naturally by each successful payment (existing Parse webhook flow)
- When client opens checkout for the remaining, they see the current `balanceDue` and pay normally
- No "auto-generate remaining balance milestone" logic
- No vault-aware balance tracking in the schedule system

**Can the merchant add new milestones after all scheduled ones complete?**
- Yes — the vault has transitioned to an effectively dormant state (no ACTIVE row, no schedules)
- To auto-pay new milestones, client would need to re-vault (new consent for new amounts/dates)
- More likely: merchant sends a checkout link for the remaining and client pays manually

**Implementation:**
- Overpayment check lives in checkout (is-unifiedxp) at the point where the auto-pay opt-in is rendered
- Simple comparison: `sumMilestoneAmounts > invoiceTotal` → hide/disable opt-in
- No is-payments or is-stripe involvement — this is a pure UI/checkout gate

### Negative balance / overpayment (pre-existing, not auto-pay specific)

The system intentionally allows negative balances (Parse backend comments: "can be negative if the customer overpaid"). Form validation is inconsistent: upcoming payment forms cap at invoice total, but "Mark as Paid" and "Record Payment" forms have no max. This is a pre-existing gap unrelated to auto-pay.

**For auto-pay:** Not a risk. Auto-pay charges the snapshotted `amountCents` from the milestone — it won't exceed what was agreed. The race condition (auto-pay fires + merchant marks paid) is guarded by `status = ACTIVE` check. Separate concern from auto-pay, but worth noting for the broader payment scheduling feature.

### Concurrent charge + merchant skip/disable race condition

Merchant skips or disables auto-pay at the exact moment EventBridge fires for the ACTIVE payment.

**Behavior:** The execution Lambda checks `status = ACTIVE` before charging. If the skip/disable sets INACTIVE before the Lambda runs, the charge is skipped. If the Lambda runs first and succeeds, the payment is already MILESTONE_COMPLETED when the skip/disable tries to cancel — no-op (already completed). Use optimistic locking (`updatedAt` check) on the `upcoming_payments` row to prevent double-processing.

---

## API Contracts

All endpoints follow existing patterns: Bearer token auth, Zod validation, `{ success: boolean, data?, message?, error? }` response shape.

### is-payments Endpoints

#### Create upcoming payment schedules (on vault)

```
POST /backend/upcoming-payment
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Request:
{
  invoiceId: string
  accountId: string
  provider: "stripe"
  milestones: [                    // all unpaid milestones included — only the FIRST (soonest) gets a row + schedule (Decision 21)
    {
      milestoneId: string        // Parse Payment remoteId
      amountCents: number        // resolved at call time
      currencyCode: string       // "USD"
      scheduledDate: string      // ISO date "2026-06-01"
    }
  ]
  consentGrantedAt: string       // ISO datetime
  timezone: string               // IANA timezone, e.g. "America/Toronto" (from client at vault time)
  vaultTime: string              // ISO time "14:30:00" — schedules fire at this time on each due date
  isAsyncPaymentMethod?: boolean
  paymentMethod?: { type: "card" | "us_bank_account", last4: string }  // V2 — not stored in V1 (no column in upcoming_payments table yet)
}

Response (201):
{ success: true, data: { invoiceId: string, scheduledMilestoneId: string } }
```

#### Get upcoming payment status

Used by checkout (is-unifiedxp) and mobile/web UI to display auto-pay state.

```
GET /backend/upcoming-payment/status?invoiceId={id}
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Response (200):
{
  success: true,
  data: {
    invoiceId: string
    vaulted: boolean
    // V1: paymentMethodType and paymentMethodLast4 stored but NOT returned.
    // Mobile shows "Auto-pay enabled" without card details (same as RP).
    activePayment?: {              // the single ACTIVE row (null if none scheduled)
      milestoneId: string
      amountCents: number
      scheduledDate: string
      retryCount: number
      lastExecutedAt?: string
    }
    completedPayments: [           // INACTIVE rows (history — MILESTONE_COMPLETED, SKIPPED, etc.)
      {
        milestoneId: string
        amountCents: number
        disabledReason: string
      }
    ]
    // Future milestones (not yet scheduled) are NOT in this response — client reads from Parse
  }
}

Response (404):
{ success: false, message: "No upcoming payment found for invoice" }
```

#### Record success (called by execution Lambda)

```
POST /backend/upcoming-payment/record-success
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Request:
{
  invoiceId: string
  milestoneId: string
  isRetry?: boolean
  transactionId: string          // Stripe PaymentIntent ID
}

Response (200):
{ success: true, data: { id: string, status: "INACTIVE", disabledReason: "MILESTONE_COMPLETED" } }
```

#### Record failure (called by execution Lambda)

```
POST /backend/upcoming-payment/record-failure
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Request:
{
  invoiceId: string
  milestoneId: string
  isRetry?: boolean
  errorDetails?: { failureCode: string, failureDescription: string }
}

Response (200):
{
  success: true,
  data: {
    id: string
    retryCount: number
    shouldScheduleRetry: boolean
    nextRetryDate?: string
    status: "ACTIVE" | "INACTIVE"
    disabledReason?: "MAX_RETRIES"
    amountCents: number
    currencyCode: string
    paymentMethod?: { type: string, last4: string }  // V2 — not stored yet
  }
}
```

#### Skip next scheduled payment (merchant action)

```
POST /backend/upcoming-payment/skip
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Request:
{
  invoiceId: string
  milestoneId: string
}

Response (200):
{
  success: true,
  data: {
    skippedId: string,
    disabledReason: "SKIPPED",
    nextActive?: { milestoneId: string, scheduledDate: string }  // null if no more QUEUED
  }
}
```

Cancels the ACTIVE payment's EventBridge schedule pair, sets it INACTIVE with reason `SKIPPED`, then reads the next unpaid milestone from Parse, creates a new row (ACTIVE) + schedule pair (chain advances). If no unpaid milestones remain, auto-pay is effectively complete.

#### Disable all upcoming payments (full reset)

```
POST /backend/upcoming-payment/disable
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Request:
{
  invoiceId: string
  reason: "USER_DISABLED" | "ADMIN_DISABLED" | "MAX_RETRIES"
}

Response (200):
{ success: true, data: { disabledCount: number } }
```

Disables the ACTIVE upcoming payment (sets INACTIVE), cancels its EventBridge schedule pair. No QUEUED rows to clean up (Decision 21 — they were never created). Vault record is kept (not deleted) — gets replaced on re-vault. (Note: vault deletion may be added later for consistency with incoming RP work; if so, delete here too.) Nuclear option — merchant must use this for invoice-level edits or full cancellation.

#### Cancel single upcoming payment (client pays early via checkout)

```
POST /backend/upcoming-payment/cancel-one
Authorization: Bearer <IS_PAYMENTS_API_KEY>

Request:
{
  invoiceId: string
  milestoneId: string
  reason: "MANUALLY_PAID"
}

Response (200):
{
  success: true,
  data: {
    id: string,
    milestoneId: string,
    status: "INACTIVE",
    disabledReason: "MANUALLY_PAID",
    nextActive?: { milestoneId: string, scheduledDate: string }
  }
}
```

Cancels the payment's schedule (if ACTIVE), sets INACTIVE, and advances the chain — reads next unpaid milestone from Parse, creates a new row (ACTIVE) + schedule pair. Auto-pay continues with remaining payments.

---

### is-stripe Endpoints

#### Execute upcoming payment charge

```
POST /backend/upcoming-payment/execute
Authorization: Bearer <IS_STRIPE_API_KEY>
Idempotency-Key: upcoming-{invoiceId}-{paymentId}-{YYYY-MM-DD}

Request:
{
  invoiceId: string
  milestoneId: string
  amountCents: number
  currencyCode: string
}

Response (200):
{
  success: true,
  data: {
    paymentIntentId: string
    status: "succeeded" | "processing"
    amountCents: number
  }
}

Response (400/402):
{
  success: false,
  error: "payment_failed",
  errorDetails: { failureCode: string, failureDescription: string }
}
```

#### Get vault details

```
GET /backend/upcoming-payment?invoiceId={id}
Authorization: Bearer <IS_STRIPE_API_KEY>

Response (200):
{
  success: true,
  data: {
    invoiceId: string
    customerId: string
    vaultedToken: string          // masked: "pm_****xyz"
    paymentMethodType: "card" | "us_bank_account"
    paymentMethodLast4: string
    consentGrantedAt: string
  }
}

Response (404):
{ success: false, error: "No vault found for invoice" }
```

#### Create vault (after SetupIntent confirmation)

```
POST /backend/upcoming-payment/vault
Authorization: Bearer <IS_STRIPE_API_KEY>

Request:
{
  invoiceId: string
  accountId: string
  customerId: string
  vaultedToken: string            // pm_xxx
  paymentMethodType: "card" | "us_bank_account"
  currencyCode: string
  consentGrantedAt: string
  timezone: string                // IANA timezone from client, e.g. "America/Toronto"
  milestones: [                   // included so SQS message to is-payments is self-contained (Decision Q2)
    { milestoneId: string, amountCents: number, currencyCode: string, scheduledDate: string }
  ]
}

Response (201):
{ success: true, data: { id: string, invoiceId: string } }
```

#### Delete vault

Not implemented. Vault is never deleted — it gets replaced when the client re-vaults (same as RP pattern). The `POST /backend/upcoming-payment/vault` endpoint upserts by `invoice_id`.

---

---

## Resolved Questions (from self-review)

### 1. Who proxies merchant actions to is-payments?

**Decision: Mobile calls is-payments directly (fire-and-forget), matching the bookkeeping sync pattern.**

**Context:** Parse currently has NO outbound communication pattern (no SQS publishing, no REST calls to downstream services). Adding one would be a new pattern. However, mobile already calls is-bookkeeping directly after Parse saves using a fire-and-forget REST call (`syncPaymentToBookkeeping`). This is a proven production pattern.

**Flow:**
1. Mobile saves milestone to Parse (existing `invoiceAddPayment` / `invoiceUpdatePayment`)
2. Mobile then fires a REST call to is-payments (fire-and-forget) to create/update/cancel the schedule
3. Errors logged to Sentry but don't block the merchant's save action
4. Web follows the same pattern when built (call from web service layer)

**Trade-offs accepted:**
- Duplication across clients (mobile + web) — bounded to 2 clients, manageable
- If Parse save succeeds but is-payments call fails, milestone exists without schedule — mitigated by Sentry alerting and potential reconciliation endpoint
- No Parse changes needed (preferred by team)

### 2. How does is-payments get milestone data?

**Decision: Client sends all data in the request payload.**

Mobile includes resolved milestone data (amountCents, dueDate, invoiceId, milestoneId, accountId) in the REST payload to is-payments. is-payments trusts the data and creates the schedule. No is-payments→Parse fetch needed.

**Rationale:** Matches bookkeeping sync pattern (payload is self-contained). Avoids adding is-payments→Parse dependency. Amount is already resolved at schedule creation time (Decision 2a: snapshot).

### 3. How does the status endpoint return vault info?

**Decision: Denormalize onto is-payments (same as RP) but don't expose to client in V1.**

- Store `paymentMethodType` and `paymentMethodLast4` on `upcoming_payments` table at creation time
- Mobile UI shows "Auto-pay enabled" (generic status) — same as current RP mobile UI
- No card details displayed in V1 (RP doesn't show them either)
- Data is available for future use if design wants to show "Visa ****4242" later

### 4. What timezone/time for schedule execution?

**Decision: Use vault time as execution time, in client's timezone. Same approach as RP.**

**How RP does it:** Uses merchant's timezone (`ScheduleExpressionTimezone` on EventBridge). Time of day is captured from the setup moment.

**How milestones will do it:** Since milestone due dates are date-only (no time component), use the vault creation time as the execution time. If client vaults at 2:30 PM ET, all milestone schedules fire at 2:30 PM ET on their respective due dates.

- Timezone: derived from client/merchant timezone at vault time (RP infra already supports this via `date-fns-tz`)
- DST: handled automatically (same as RP)
- Future: if design adds a time picker UI field, use that instead of vault time

**Note:** Design team will evaluate whether a time picker field should be added to the milestone form.

---

## Out of Scope (V2 / Future)

- **SMS channel for Automatic Payment Requests** — PRD v2 mentions sending reminders via SMS in addition to email. V1 is email only.
- **Merchant edits payment request before sending** — PRD "Payment Request Actions (Stripe Payment Not Vaulted)" section. Non-vaulted Era 2 flow only; in vaulted flow, charges fire automatically with no review step.
- **Global settings for Automatic Payment Requests** — PRD says "store settings at Global Settings level, not at invoice level." Era 2 scope: a merchant toggle like "auto-send payment requests for all my invoices." V1 (vaulted) doesn't need this because auto-pay is consent-driven per invoice at checkout.
- **Cross-invoice vaulting** — PRD v2: "buyer associated with multiple invoices can choose to use saved card across invoices." V1 vault is per-invoice (`invoice_id UNIQUE`).
- **PayPal vaulting** — PRD explicitly defers. Separate scope.
- **Merchant-initiated manual charge on vaulted card** — PRD v2 user story. Not designed.
