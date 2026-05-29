# Existing Email Reminder Flows — Technical Reference

> Context: APR (Automatic Payment Requests) will be the third email reminder system. This doc captures the two existing flows in full detail so we can make informed decisions about APR's architecture.

---

## Overview: Three Email Systems

| System | Trigger | What it sends | Where it lives | Setting key |
|--------|---------|---------------|----------------|-------------|
| **is-messages cron** | Every 30 min (EventBridge) | "Unopened invoice" reminder | `is-messages/` | `suppress_reminder` field on `msg` row |
| **Abandoned cart** | Event-driven (checkout-viewed) | "Complete your payment" reminder | `is-services/packages/services/is-abandoned-cart/` | `PaymentReminders` |
| **APR** (new) | Daily cron (EventBridge) | "Payment due today" | `is-services/` (new package) | `AutomaticPaymentRequests` |

---

## Flow 1: is-messages Cron (Unopened Invoice Reminder)

### What It Does

Sends a reminder email to recipients who haven't opened an invoice email after 2 days. Fires every 30 minutes, processes up to 1000 messages per run.

### Trigger

- **Type:** EventBridge scheduled rule
- **Schedule:** `cron(*/30 * * * ? *)` — every 30 minutes
- **Environments:** Staging + Production only
- **Target:** `ReminderHandler` Lambda
- **CDK:** `is-messages/packages/cdk/lib/email-sending.ts` (lines 142-146)

### The Query

Queries the PostgreSQL `msg` table for eligible messages:

```sql
SELECT * FROM msg
WHERE
  suppress_reminder IS NOT true
  AND opened_dt IS NULL
  AND pixel_fired IS NULL
  AND created_dt IS NOT NULL
  AND created_dt < (now() - '2880 minutes'::interval)   -- older than 2 days
  AND created_dt > (now() - '96 hours'::interval)       -- but not older than 4 days
  AND type = 1                                           -- email type only
  AND reminder_sent_dt IS NULL                           -- not already reminded
  AND swu_email IS NOT NULL                              -- has template data
ORDER BY created_dt DESC
LIMIT 1000
```

**Window:** Only emails between 2–4 days old get reminded. Before 2 days = too early. After 4 days = too late.

**Index:** `idx_msg_reminder` on `(opened_dt, pixel_fired, created_dt, type, subtype, reminder_sent_dt)`

### Data Flow

```
EventBridge (every 30 min)
    │
    ▼
ReminderHandler Lambda
    │
    ▼
getReminders(2880, 1000) → PostgreSQL msg table
    │
    ▼ [up to 1000 messages]
For each msg:
    ├─ Fetch invoice metadata from Parse (due date, terms)
    ├─ Skip if: invoice due in future
    ├─ Skip if: invoice has termsDay === NONE (-2)
    ├─ Skip if: hardcoded excluded account
    ├─ Get template ID (A/B experiment logic)
    ├─ Build EmailInvoiceMsgJob { reminder: true, tags: ['is-reminder'] }
    ├─ Send to EMAIL_SQS_URL (FIFO queue)
    └─ Mark: UPDATE msg SET reminder_sent_dt = now() WHERE id = $1
             │
             ▼
SQS Handler Lambda (batch size 10)
    │
    ▼
renderTemplate() via SendWithUs API
    │
    ▼
sendMailgunMessage() → Mailgun sends email
    │
    ▼
Store reminder_receipt_id on msg row
```

### Idempotency / Preventing Re-sends

Multiple layers:

| Layer | Mechanism | When set |
|-------|-----------|----------|
| `reminder_sent_dt IS NULL` | Primary — query excludes sent messages | Set immediately after SQS send |
| `opened_dt IS NULL` | Skip opened emails | Set when recipient opens email |
| `pixel_fired IS NULL` | Skip tracked-open emails | Set when tracking pixel fires |
| `suppress_reminder = true` | Explicitly suppressed | Set at initial email send time |
| 4-day window | Expires naturally | Time-based |

### suppress_reminder — When It's Set

Set in `api/src/services/email.ts` (lines 376-390) at the time the original invoice email is sent:

```typescript
const suppressReminder = params.emailType === EmailType.Reminder
  || universalInvoice.setting.termsDay === InvoiceTermTypes.NONE;
```

Suppressed if:
- The email itself is a reminder (don't remind about reminders)
- The invoice has "No Terms" setting (termsDay === -2)

### Settings Check

**None at the cron level.** This system does NOT check a merchant setting like `PaymentReminders`. It's controlled entirely by:
- The `suppress_reminder` flag (set at send time)
- Whether the email was opened

### Email Template

- **Invoices (control):** `SWU_INVOICE_REMINDER_TEMPLATE_ID`
- **Invoices (treatment/A/B):** `SWU_INVOICE_REMINDER_IR_EXPERIMENT_TEMPLATE_ID`
- **Estimates:** `SWU_ESTIMATE_REMINDER_TEMPLATE_ID`
- **Render data:** Reuses the original email's `msg.swu_email.renderData` (same content, just re-sent)

### Key Files

| File | Purpose |
|------|---------|
| `is-messages/packages/cdk/lib/email-sending.ts` | CDK infra, EventBridge rule (line 142) |
| `is-messages/packages/code/src/functions/email-reminder/handler.ts` | Main handler, orchestration |
| `is-messages/packages/code/src/common/db.ts` | `getReminders()` query, `setReminderSentDt()` |
| `is-messages/packages/code/src/reminder/experiment.ts` | A/B test template selection |
| `is-messages/packages/code/src/reminder/parse.ts` | Fetch invoice metadata from Parse |
| `is-messages/packages/code/src/functions/email/sqsHandler.ts` | SQS → Mailgun sending |
| `api/src/services/email.ts` | Sets `suppress_reminder` at send time |

### Lambda Config

- **Runtime:** Node.js 20.x
- **Memory:** 256 MB
- **Timeout:** 3 minutes
- **Retries:** 0

---

## Flow 2: Abandoned Cart (Outstanding Checkout Reminders)

### What It Does

When a client views a checkout page but doesn't pay, the system schedules two reminder emails:
1. **30 minutes later** — "You haven't completed your payment"
2. **3 days later** — "Don't forget to pay"

### Trigger

- **Type:** Event-driven (NOT a cron)
- **Event:** `checkout-viewed` on `payment-event-bus` (EventBridge)
- **Source:** `checkout` (emitted by is-unifiedxp when client opens checkout page)
- **Cleanup event:** `payment-attempted` — cancels pending schedules

### Architecture: Three Lambdas

| Lambda | Trigger | Purpose |
|--------|---------|---------|
| `schedule-reminder` | `checkout-viewed` event | Creates 2 EventBridge Scheduler schedules (30min + 3days) |
| `send-reminder` | EventBridge Scheduler (at scheduled time) | Validates eligibility + sends email |
| `remove-schedulers` | `payment-attempted` event | Deletes pending schedules |

### Data Flow

```
Client views checkout page
    │
    ▼
is-unifiedxp emits "checkout-viewed" to payment-event-bus
    │ { invoiceId, currencyCode, checkoutUrl, recipientEmailAddress }
    ▼
═══════════════════════════════════════════════════════════════
PHASE 1: SCHEDULE CREATION (schedule-reminder Lambda)
═══════════════════════════════════════════════════════════════
    │
    ├─ Check DynamoDB: schedule already exists for this invoice+recipient?
    │  └─ YES → return (idempotent, no duplicate)
    │  └─ NO → continue
    │
    ├─ Create EventBridge Scheduler schedule: 30 min from now
    ├─ Create EventBridge Scheduler schedule: 3 days from now
    │   (both target: send-reminder Lambda, ActionAfterCompletion: DELETE)
    │
    └─ Store in DynamoDB: { invoiceId, recipientEmail, 30mins.scheduleName, 3days.scheduleName }
       └─ TTL: 150 days (auto-cleanup)

═══════════════════════════════════════════════════════════════
PHASE 2: SEND REMINDER (send-reminder Lambda, at scheduled time)
═══════════════════════════════════════════════════════════════
    │
    ├─ Input: { invoiceId, currencyCode, checkoutUrl, recipientEmailAddress, scheduledIn: '30mins'|'3days' }
    │
    ├─ Parallel fetch:
    │  ├─ Parse.Cloud.run('publicInvoice', { id: invoiceId }) → invoice data
    │  └─ Parse.Cloud.run('invoiceGetPayments', { invoiceId }) → payments
    │
    ├─ CHECK 1: AccountSettings.getSettingPaymentReminders(accountId)
    │  └─ disabled → RETURN (no email)
    │  └─ error → defaults to true (send anyway)
    │
    ├─ CHECK 2: isInvoiceDueInFuture(invoice)
    │  └─ due date in future → RETURN (skip — not overdue yet)
    │
    ├─ CHECK 3: isInvoiceWithNoTerms(invoice)
    │  └─ termsDay === NONE → RETURN
    │
    ├─ CHECK 4: isInvoicePayable({ invoice, payments })
    │  └─ balanceDue <= 0 or deleted → RETURN
    │
    └─ All checks pass → sendReminderEmail()
       │
       ├─ Calculate balance due
       ├─ Fetch merchant user: Users.getUserByAccountId(accountId)
       ├─ Build render data: { client_name, amount, company_*, pay_url, invoice_no, date }
       ├─ pay_url = checkoutUrl + ?reminder-email=30mins|3days
       ├─ queueSWUJob({ recipient, sender: noreply, renderData, templateId })
       │     └─ templateId: ABANDONED_CART_EMAIL_TEMPLATE_ID
       │        (staging: tem_R4Jym6MFBBRwx6xYRYkkfptT)
       │        (prod: tem_Hcqm4wpPTbpMwgJ4cCtkjBbD)
       └─ trackSendEmail() → IsEvents { action: 'reminder-email-sent' }

═══════════════════════════════════════════════════════════════
PHASE 3: CLEANUP (remove-schedulers Lambda, on payment)
═══════════════════════════════════════════════════════════════
    │
    ├─ "payment-attempted" event received with { invoiceId }
    ├─ Query DynamoDB: all schedules for this invoiceId
    ├─ For each: delete both EventBridge Scheduler schedules
    └─ DynamoDB record remains (TTL auto-deletes after 150 days)
```

### Idempotency / Preventing Re-sends

| Mechanism | How |
|-----------|-----|
| DynamoDB check before scheduling | If schedule exists for invoice+recipient, skip |
| EventBridge `ActionAfterCompletion: DELETE` | Schedule auto-deletes after firing |
| Invoice payability check at send time | If already paid, skip |
| Payment-attempted cleanup | Deletes pending schedules immediately |

### Settings Check

- **Setting:** `PaymentReminders` (checked via `AccountSettings.getSettingPaymentReminders(accountId)`)
- **Package:** `@is/account` (wraps `@is/mongo` → MongoDB `Setting` collection)
- **Default:** `true` if lookup fails (defensive — send rather than miss)
- **Checked at:** Send time (not schedule time) — so if merchant disables between checkout and 3-day reminder, the 3-day email won't send

### Email Sending

```
sendReminderEmail()
    → queueSWUJob(emailJobData)           ← @is/messaging package
    → IsMessages SDK HTTP client           ← @invoice-simple/is-msg npm package
    → HTTP POST /email-sending             ← is-messages service API
    → is-messages receives job
    → SQS → sqsHandler
    → renders SWU template
    → Mailgun sends email
```

### State Tracking: DynamoDB

**Table:** `CheckoutReminderSchedule`

| Key | Type | Description |
|-----|------|-------------|
| `invoiceId` (PK) | STRING | Invoice being reminded |
| `recipientEmail` (SK) | STRING | Recipient email address |
| `30mins` | Object | `{ scheduleName: string }` |
| `3days` | Object | `{ scheduleName: string }` |
| `expiredAt` | NUMBER | Unix timestamp (TTL), now + 150 days |

**Operations:**
- `GET` — idempotency check before creating
- `PUT` — store after creating schedules
- `QUERY` — find all schedules for invoiceId (cleanup)
- **Auto-delete** via DynamoDB TTL after 150 days

### CDK Infrastructure

| Resource | Type | Purpose |
|----------|------|---------|
| `payment-event-bus` | EventBridge Bus | Shared event bus |
| `checkout-viewed` rule | EventBridge Rule | Triggers schedule-reminder |
| `payment-attempted` rule | EventBridge Rule | Triggers remove-schedulers |
| `checkout-reminder-schedule-group` | Scheduler Group | Groups all reminder schedules |
| `CheckoutReminderSchedule` | DynamoDB Table | State tracking |
| `schedule-reminder` | Lambda | Creates schedules |
| `send-reminder` | Lambda | Sends emails |
| `remove-schedulers` | Lambda | Cleans up on payment |
| DLQs (x2) | SQS Queues | Dead-letter for failed events |

### Key Files

| File | Purpose |
|------|---------|
| `is-services/packages/services/is-abandoned-cart/src/lambda-handlers/schedule-reminder/event-bridge-handler.ts` | Schedule creation logic |
| `is-services/packages/services/is-abandoned-cart/src/lambda-handlers/send-reminder/scheduled-handler.ts` | Eligibility checks + send |
| `is-services/packages/services/is-abandoned-cart/src/lambda-handlers/remove-schedulers/event-bridge-handler.ts` | Cleanup on payment |
| `is-services/packages/services/is-abandoned-cart/src/utils/send-reminder-email.ts` | Email building + queueSWUJob |
| `is-services/packages/services/is-abandoned-cart/src/utils/dynamo-db.ts` | DynamoDB operations |
| `is-services/packages/services/is-abandoned-cart/src/utils/event-bridge-schedule.ts` | Scheduler create/delete |
| `is-services/packages/services/is-abandoned-cart/src/utils/is-invoice-payable.ts` | Balance check |
| `is-services/packages/services/is-abandoned-cart/src/utils/provider/account/get-payment-reminders-enabled.ts` | Setting lookup |
| `is-services/packages/services/is-abandoned-cart/src/utils/provider/parse/universal-invoice.ts` | Parse read |
| `is-services/packages/aws/cdk/src/lib/constructs/abandoned-cart/` | All CDK constructs |

### Environment Variables

```
# Shared (abandonedCartStackEnv)
APP_ENV, PARSE_SERVER_URL, PARSE_APP_ID, PARSE_MASTER_KEY, IS_MESSAGES_API_KEY, MONGODB_URI

# schedule-reminder specific
SCHEDULER_GROUP_NAME, SEND_REMINDER_LAMBDA_ARN, SEND_REMINDER_LAMBDA_ROLE_ARN, CHECKOUT_REMINDER_SCHEDULE_TABLE

# remove-schedulers specific
SCHEDULER_GROUP_NAME, CHECKOUT_REMINDER_SCHEDULE_TABLE
```

---

## Comparison: Key Differences

| Dimension | is-messages cron | Abandoned cart | APR (planned) |
|-----------|-----------------|----------------|---------------|
| **Trigger** | Fixed schedule (every 30 min) | Event-driven (checkout-viewed) | Fixed schedule (daily cron) |
| **Data source** | PostgreSQL `msg` table | Parse Cloud Functions + DynamoDB | `@is/mongo` Invoice collection + `@is/account` |
| **What's swept** | `msg` rows (emails already sent) | Invoices (via Parse) at send time | Invoice documents in MongoDB |
| **Timing logic** | 2–4 day window after email sent | 30min + 3days after checkout viewed | On `dueDate = today` |
| **Setting check** | None (uses `suppress_reminder` flag) | `PaymentReminders` via `@is/account` | `AutomaticPaymentRequests` via `@is/account` |
| **Idempotency** | `reminder_sent_dt` field on `msg` row | DynamoDB record + schedule auto-delete | `paymentRequestSentDate` on Invoice (method TBD) |
| **Email sending** | Direct SQS → Mailgun (internal) | `queueSWUJob()` → is-messages → SQS → Mailgun | `queueSWUJob()` → is-messages → SQS → Mailgun |
| **State write-back** | `UPDATE msg SET reminder_sent_dt` (PostgreSQL) | None (DynamoDB only) | TBD (Parse CF / DynamoDB / direct Mongo) |
| **Cancellation** | N/A (time window expires) | `payment-attempted` event → delete schedules | N/A (query condition handles it) |
| **Lambdas** | 1 | 3 | 1 (planned) |
| **Infra** | EventBridge rule + Lambda | EventBridge rules + Scheduler + DynamoDB + 3 Lambdas | EventBridge rule + Lambda (minimal) |

---

## Implications for APR Design

1. **APR is closer to is-messages cron in shape** (single cron, sweep, filter, send) but lives in is-services (modern codebase, MongoDB access)
2. **Email sending:** Use abandoned-cart's `queueSWUJob()` pattern (not is-messages' internal SQS)
3. **Setting check:** Use abandoned-cart's `@is/account` pattern
4. **Reads:** Use `@is/mongo` (the underlying layer abandoned-cart also uses indirectly)
5. **Idempotency write:** Open question — no existing pattern perfectly matches. is-messages writes to its own DB. Abandoned-cart uses DynamoDB. APR needs to write to Invoice (Parse/Mongo).
