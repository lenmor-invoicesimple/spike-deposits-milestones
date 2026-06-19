# Debounced SQS Pipeline — Unified D&M Architecture Proposal

> Date: 2026-06-09 (based on discussion 2026-06-06)
> Status: Proposal — not yet approved
> Alternative to:
>   - [APR implementation (daily cron)](./automatic-payment-requests/2026-05-28-automatic-payment-requests-implementation.md)
>   - [Invoice dueDate ↔ Milestone sync](./invoice-duedate-milestone-sync/)
>   - [Auto-pay design](./auto-pay/)
>
> This proposal unifies all of the above into a single architecture.

---

## Origin

From a discussion on 2026-06-06: the suggestion was to move all payment scheduling logic out of Parse Server and into `is-payments/` behind a debounced pipeline. The original idea used a 1-hour EventBridge schedule that gets replaced on every save. This doc refines that into a faster, simpler approach using SQS.

Related context:
- Liz's pivot toward Deposits-first vaulting (June 5)
- Sonya's Temporal recommendation ($2-5/mo)
- Core team may have a similar pattern (unconfirmed — need to find the code)

---

## Problem Statement

D&M requires several backend operations when a merchant edits invoice payment scheduling:

1. **Milestone-due-date sync** — recalculate `Invoice.dueDate` from milestones
2. **Automatic Payment Requests (APR)** — schedule reminder emails on due dates
3. **Auto-pay (Phase 3)** — charge a vaulted card on due date

These currently live in (or are planned for) Parse Server hooks, a daily cron, and an unspecified future system — three separate architectures for one logical workflow. Meanwhile, invoice saves happen frequently (especially on mobile/web during editing), and executing expensive operations on every save is wasteful.

---

## Core Idea

Every invoice save (web or mobile) fires a call to a new `is-payments` endpoint. The endpoint doesn't execute expensive operations immediately — it queues an SQS message with a 60-second delay. If another save arrives within that window, the newer message supersedes the older one (via a PG timestamp comparison). Only when the invoice is "quiet" for 60s does the Lambda process.

This single architecture covers Phase 1 (APR emails), Phase 2 (scheduling UI), and Phase 3 (auto-pay / charging).

---

## Key Rationale

1. **Decoupling from Parse Server** — Parse should only handle core invoicing + mobile offline sync. Payment scheduling logic shouldn't live there.
2. **Flexibility** — We own is-payments entirely. Requirements change? We change our service, no Parse Server PR reviews or risk of breaking sync.
3. **Debounce solves the "too many edits" problem** — Users editing on every keypress don't trigger 50 Lambdas. Only the final state matters.
4. **Single Lambda covers all phases** — APR, due date sync, auto-pay all live in one processor. No scattered cron jobs.
5. **Proven pattern** — is-payments already runs Lambdas that read Parse, write PG, send emails, schedule retries via EventBridge, and charge Stripe/PayPal (recurring payments).

---

## Architecture

```
┌──────────────────┐       ┌─────────────────────────┐       ┌───────────────────────────┐
│  web/mobile      │       │  is-payments             │       │  SQS Queue                │
│  saves invoice   │──────▶│  POST /invoice-schedule  │──────▶│  DelaySeconds = 60        │
│                  │       │                          │       │  (message per save)        │
└──────────────────┘       │  • Upsert PG row:        │       └────────────┬──────────────┘
                           │    last_saved_at = NOW() │                    │
                           │    payload = milestones  │                    │ (60s later)
                           │  • SendMessage to SQS    │                    │
                           └─────────────────────────┘                    ▼
                                                               ┌──────────────────────────┐
                                                               │  Processing Lambda        │
                                                               │                          │
                                                               │  1. Read PG row           │
                                                               │  2. Compare last_saved_at │
                                                               │     vs message timestamp  │
                                                               │                          │
                                                               │  MATCH → Process:         │
                                                               │    • Calc due dates       │
                                                               │    • Write to Parse       │
                                                               │    • Schedule APR email   │
                                                               │    • Schedule auto-charge │
                                                               │                          │
                                                               │  STALE → Discard (noop)  │
                                                               └──────────────────────────┘
```

---

## How the Debounce Works

### On Every Invoice Save (the endpoint)

```
POST /is-payments/invoice-schedule
Body: { invoiceId, milestones, depositConfig, ... }
```

1. Upsert `invoice_payment_schedule` row:
   - `last_saved_at = NOW()`
   - `payload = { milestones, deposit, settings }`
2. Send SQS message with `DelaySeconds = 60`:
   - `{ invoiceId, savedAt: <same timestamp as PG> }`
3. Return `202 Accepted` (or computed due date for optimistic UI — see below)

### When the Lambda Fires (60s after each message)

1. Read PG row for this `invoice_id`
2. Compare: does `last_saved_at` match the message's `savedAt`?
   - **Yes** → Invoice has been quiet for 60s. Process it.
   - **No** → A newer save happened. Discard. Exit.
3. Processing (only on match):
   - Calculate next due date from milestones
   - Update `Invoice.dueDate` on Parse Server
   - Schedule APR reminder email (if enabled)
   - Schedule auto-charge (Phase 3, if vaulted card exists)
   - Update PG row: `last_processed_at = NOW()`, `status = 'done'`

### Under Rapid Edits (30 saves in 2 minutes)

| Event | PG `last_saved_at` | SQS message delivers at | Lambda action |
|-------|--------------------|--------------------------|----|
| Save 1 (10:00:01) | 10:00:01 | 10:01:01 | Reads PG → sees 10:02:30 → **discard** |
| Save 2 (10:00:05) | 10:00:05 | 10:01:05 | Reads PG → sees 10:02:30 → **discard** |
| ... | ... | ... | **discard** |
| Save 30 (10:02:30) | 10:02:30 | 10:03:30 | Reads PG → sees 10:02:30 → **process** ✓ |

**29 Lambda invocations do a single PG read and exit (< 50ms, ~$0.000001 each). Only the 30th does real work.**

---

## Data Model

```sql
CREATE TABLE invoice_payment_schedule (
  invoice_id         TEXT PRIMARY KEY,        -- Parse Invoice remoteId
  account_id         TEXT NOT NULL,           -- for querying by merchant
  last_saved_at      TIMESTAMPTZ NOT NULL,    -- debounce comparator
  payload            JSONB NOT NULL,          -- milestone config snapshot at last save
  computed_due_date  DATE,                    -- result of last processing
  last_processed_at  TIMESTAMPTZ,            -- when Lambda last ran successfully
  status             TEXT DEFAULT 'pending',  -- pending | processing | done | error
  apr_scheduled_for  DATE,                    -- when next APR email is scheduled
  charge_scheduled_for DATE,                  -- when next auto-charge is scheduled (Phase 3)
  created_at         TIMESTAMPTZ DEFAULT NOW(),
  updated_at         TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ips_account ON invoice_payment_schedule(account_id);
CREATE INDEX idx_ips_status ON invoice_payment_schedule(status) WHERE status != 'done';
CREATE INDEX idx_ips_apr ON invoice_payment_schedule(apr_scheduled_for) WHERE apr_scheduled_for IS NOT NULL;
```

---

## What the Processing Lambda Does

All-in-one per invoice, covering Phase 1 through 3:

| Step | Phase | Logic |
|------|-------|-------|
| 1. Calculate due date | 1-2 | Read `payload.milestones`, find earliest unpaid, derive `dueDate` |
| 2. Write to Parse | 1-2 | `Invoice.dueDate = computed`, update Payment objects if needed |
| 3. Schedule APR email | 1-2 | If merchant has `AutomaticPaymentRequests` enabled, queue email via is-msg for `dueDate` |
| 4. Schedule auto-charge | 3 | If vaulted card exists + merchant enabled auto-pay, schedule charge for `dueDate` |
| 5. Update PG row | All | `last_processed_at`, `computed_due_date`, `apr_scheduled_for`, `status = 'done'` |

For APR and auto-charge scheduling: these are future-dated events. The Lambda can either:
- **Option A:** Write the date to PG and have a separate daily cron check for `apr_scheduled_for = today` (simple, reuses existing cron pattern)
- **Option B:** Create an EventBridge schedule for the exact date (precise, no daily sweep, but more schedules to manage)

Recommendation: **Option A for APR** (daily cron is cheap, and APR doesn't need sub-day precision), **Option B for auto-charge** (money movement should fire at exact time).

---

## Optimistic Due Date (Solving the 60s Delay for UI)

The 60s debounce means Parse `Invoice.dueDate` won't update for at least 1 minute. If the user is looking at the invoice preview, they'd see stale data.

**Solution:** The endpoint computes and returns the due date synchronously in the response:

```json
// POST /is-payments/invoice-schedule response
{
  "accepted": true,
  "computedDueDate": "2026-07-15"  // optimistic, for client display
}
```

Client shows this immediately. Parse gets updated 60s later by the Lambda. No user-visible lag.

---

## Why SQS Over EventBridge Scheduler for the Debounce

| | EventBridge Scheduler | SQS DelaySeconds |
|--|--|--|
| Debounce mechanism | Create/update schedule per invoice per save | Let all messages queue; discard stale ones |
| API calls per save | CreateSchedule or UpdateSchedule (heavier, has throttle limits) | SendMessage (lightweight, 3000+ TPS) |
| Rate limits at scale | 720 creates/min or 720 updates/min per region (soft limit) | Effectively unlimited |
| Race condition handling | Schedule fires before update arrives → stale execution | Harmless — Lambda discards stale messages |
| Cleanup | Must delete schedule after processing | Messages auto-delete after processing |
| Cost | $1/million scheduled invocations + schedule storage | $0.40/million messages |

**SQS wins for debounce at sub-minute scale.** EventBridge Scheduler is better for "fire once at a specific future date" (use for APR/charge scheduling, not the debounce itself).

---

## Debounce Window Tuning

| Window | Behavior | Best for |
|--------|----------|----------|
| 30s | Near-instant, more Lambda invocations during edits | Time-sensitive ops (auto-pay close to due date) |
| 60s | Sweet spot — editing finishes, result within a minute | General use |
| 120s | Fewer invocations, slightly delayed | High-traffic invoices |
| 300s | Batch-friendly | Non-time-sensitive (reporting, analytics) |

Could be adaptive: shorter window when due date is today/tomorrow, longer otherwise.

---

## What This Replaces

| Current / Planned Component | What Happens to It |
|---|---|
| `recalculateInvoiceDueDate` (Parse Server hook) | Replaced by Lambda step 2 |
| APR daily cron (planned) | Replaced by Lambda step 3 + simple daily trigger |
| Auto-pay architecture (TBD) | Covered by Lambda step 4 |
| Parse Server milestone hooks | No longer needed for scheduling logic |
| Client-side milestone validation | Stays (UI concern, not backend) |

---

## What Stays the Same

- **Parse Server** remains the source of truth for Invoice/Payment documents
- **Mobile offline sync** still goes through Parse
- **is-payments HTTP server** still manages its PG tables via Drizzle
- **Phase 1/2 UI prototypes** (label field, long press, date picker) — all still valid, no backend dependency
- **Feature flags** — still client-side Optimizely for UI features

---

## Integration Points

### Web (is-web-app)

After any invoice save that modifies payments/milestones:
```typescript
// In invoice save handler (after Parse save succeeds)
await fetch(`${IS_PAYMENTS_URL}/invoice-schedule`, {
  method: 'POST',
  body: JSON.stringify({ invoiceId, milestones, depositConfig }),
  headers: { 'x-api-key': IS_PAYMENTS_MASTER_KEY }
});
```

### Mobile (is-mobile)

After Parse sync completes for an invoice with payment changes:
```typescript
// In sync completion handler or after updateUpcomingOrDepositPayment
await callIsPaymentsSchedule({ invoiceId, milestones, depositConfig });
```

**Open question:** Should mobile call this directly, or should Parse Server trigger it via a hook? (Parse hook = simpler mobile integration, but adds Parse coupling. Direct call = cleaner but needs mobile code change.)

### Existing is-payments Infrastructure

Already has:
- ✅ PostgreSQL via Drizzle ORM (`IS_PAYMENTS_DATABASE_URL`)
- ✅ Parse SDK with master key (`PARSE_SERVER_URL`, `PARSE_MASTER_KEY`)
- ✅ SQS queues (payment events queue, payment execution queue)
- ✅ EventBridge Scheduler (retry scheduling)
- ✅ Email via is-msg (`IS_MESSAGES_API_KEY`)
- ✅ Stripe/PayPal access (`IS_STRIPE_SERVICE_URL`, `IS_PAYPAL_SERVICE_URL`)
- ✅ Lambda functions with same access pattern

**No new infrastructure categories needed.** Just a new endpoint, table, queue, and Lambda.

---

## Tradeoffs

### Advantages

1. **Unified architecture** — One pattern serves Phase 1, 2, and 3. No need to redesign between phases.
2. **Decoupled from Parse** — Aligns with long-term direction (moving away from Parse).
3. **Debounce is elegant** — Solves the real problem of rapid invoice edits triggering expensive operations.
4. **Owned service** — Full control over is-payments without Parse Server constraints (no Optimizely, limited testing infra).
5. **Scalable** — Per-invoice scheduling (vs daily cron) means no "sweep 100k invoices" spike at 2pm.
6. **Auditable** — PG table provides a clear record of what was scheduled and when.
7. **Flood-proof** — Wasted Lambda invocations cost nearly nothing (single PG read + exit).

### Disadvantages / Risks

1. **60s delay** — Not instant like Parse hooks. Mitigated by optimistic UI response.
2. **New endpoint integration** — Web and mobile both need to call is-payments on every invoice save. Mobile offline sync complicates this.
3. **More infra** — New PG table, new Lambda, new SQS queue. More to monitor and maintain.
4. **Lambda reads/writes Parse** — Still coupled to Parse for data — just moved the coupling to a different layer.
5. **Debounce race conditions** — What if the schedule fires while a save is in-flight? Mitigated by timestamp comparison (PG will be newer → Lambda discards).
6. **Current Phase 1/2 Parse Server prototypes** — The backend changes (updatePayment guards, invoiceHooks fallbacks) would no longer be needed under this architecture. UI prototypes (mobile) are still valid.

---

## Comparison to All Alternatives

| Dimension | This Proposal (SQS Debounce) | 1hr EventBridge Debounce | Current Plan (Parse hooks + daily cron) | Temporal |
|-----------|-----|-----|-----|-----|
| Debounce mechanism | SQS DelaySeconds + PG check | EventBridge schedule overwrite | None (fires per save) | Workflow signal debounce |
| Latency | ~60s after last edit | ~1hr after last edit | Instant (hooks) / daily (cron) | Configurable |
| APR scheduling | PG row + daily cron or EventBridge | Same Lambda schedules it | Daily cron sweep | Temporal timer |
| Auto-pay | Same Lambda schedules it | Same Lambda schedules it | Separate system TBD | Temporal activity |
| Parse coupling | Low (Lambda reads/writes) | Low (same) | High (logic IN Parse) | Low |
| New infra | 1 table, 1 queue, 1 Lambda | 1 table, EventBridge rules, 1 Lambda | Minimal | Temporal cluster ($2-5/mo) |
| Flood protection | Excellent (cheap discards) | Good (schedule overwrite) | None | Built-in |
| Operational complexity | Low (SQS is fire-and-forget) | Medium (schedule lifecycle) | Low but scattered | Medium (new system to learn) |
| Covers Phases 1-3 | ✅ | ✅ | ❌ (one per phase) | ✅ |
| Mobile offline | Needs call after sync | Same | Works naturally (Parse hooks) | Same as SQS |

---

## Relationship to Liz's Deposits-First Pivot

Both this proposal and Liz's pivot decouple from Parse. They're compatible:
- Liz simplifies the **product scope** for v1 (Deposits only, no milestone automation)
- This proposal simplifies the **technical architecture** (one pipeline for whatever product decides)
- If v1 is Deposits-only, this pipeline still works — just fewer steps in the Lambda initially
- When milestones/auto-pay come back in v2, the architecture is already in place

---

## Open Questions

- [ ] **Mobile integration path:** Direct call to is-payments, or Parse hook triggers it?
- [ ] **Optimistic UI:** Should endpoint return computed due date, or does client calculate locally?
- [ ] **APR precision:** Daily cron (simple) or per-invoice EventBridge schedule (precise)?
- [ ] **Core team pattern:** Similar pattern may exist — find the code for reference
- [ ] **Liz/Seth alignment:** Does this work with the Deposits-first pivot direction?
- [ ] **SQS queue:** New queue or reuse existing payment events queue with a new message type?
- [ ] **Payload size:** How much data to snapshot in PG `payload` column vs re-read from Parse at processing time?

---

## Next Steps (If Approved)

1. **Spike:** Add the endpoint + PG table + SQS message send (no Lambda yet — just prove the debounce)
2. **Lambda:** Implement processing logic (start with due-date sync only)
3. **Integration:** Wire up web → endpoint call after invoice save
4. **APR:** Add email scheduling to the Lambda
5. **Phase 3:** Add auto-charge scheduling when vaulting ships

---

## References

- [APR implementation plan (daily cron)](./automatic-payment-requests/2026-05-28-automatic-payment-requests-implementation.md)
- [Invoice dueDate ↔ Milestone sync](./invoice-duedate-milestone-sync/)
- [Auto-pay design](./auto-pay/)
- [APR cost analysis](./automatic-payment-requests/apr-cost-analysis.md)
- is-payments recurring payments Lambda (existing reference implementation)
