# Existing Email Reminder Flows — Deep Dive

> Context: Written to fully understand the two existing email reminder systems before implementing APR (Automatic Payment Requests). Covers architecture, data flow, files, code, tables, and fields for each system.

---

## TL;DR

| | Flow 1: is-messages cron | Flow 2: Abandoned Cart |
|--|-------------------------|----------------------|
| **What it does** | Re-sends unopened invoice email 2 days after send | Sends "complete your payment" after client abandons checkout |
| **Trigger** | EventBridge every 30 min | Event-driven (`checkout-viewed` event) |
| **Lambdas** | 1 | 3 |
| **Data swept** | PostgreSQL `msg` rows | Per-invoice, scheduled individually |
| **Reads** | PostgreSQL + Parse (invoice metadata) | Parse Cloud Functions |
| **Settings check** | None — `suppress_reminder` baked in at send time | `AccountSettings.getSettingPaymentReminders()` at send time |
| **Email sending** | Internal SQS (is-messages only) | `queueSWUJob()` → HTTP to is-messages → SQS → Mailgun |
| **Idempotency** | `reminder_sent_dt` on `msg` row | DynamoDB + payability check at send time |
| **State write** | `UPDATE msg SET reminder_sent_dt` (PostgreSQL) | None to Mongo/Postgres — DynamoDB only |
| **Infra** | 1 Lambda + EventBridge rule | 3 Lambdas + EventBridge Scheduler + DynamoDB |
| **Codebase** | `is-messages/` (legacy) | `is-services/packages/services/is-abandoned-cart/` |

---

## Background: The msg Table

Before understanding either flow, you need to understand the PostgreSQL `msg` table. It's the shared lifecycle log for every invoice/estimate email ever sent.

### Who creates the row

When a merchant taps "Send Invoice" in the app or web:

1. Request hits `POST /api/v2/email-invoice` on **is-api**
2. **File:** `api/src/controllers/invoice-sending.ts` → calls `api/src/services/email.ts` → `send()`
3. `send()` calls `insertMsg()` → **creates the `msg` row** in PostgreSQL
4. Also sets `suppress_reminder` on the row at this point (see below)
5. Then calls `sendInvoiceEmailMsg()` → `@invoice-simple/is-msg` SDK → HTTP POST to is-messages `/email-sending`
6. is-messages queues to SQS → renders via SendWithUs → sends via Mailgun
7. After Mailgun accepts: is-messages writes `receipt_id` back to the `msg` row

### Schema

**File:** `api/schema.pgsql` (lines 150–183)

```sql
CREATE TABLE msg(
  id text PRIMARY KEY,               -- UUID
  to_address text,                   -- client's email address
  parse_account_id text NOT NULL,    -- merchant's Parse account ID
  invoice_id int,                    -- numeric invoice ID
  invoice_remote_id text NOT NULL,   -- Parse object ID (e.g. 'xKj3nP1')
  invoice_no text NOT NULL,          -- human number (e.g. 'INV-001')
  url text NOT NULL,                 -- invoice view URL
  receipt_id text,                   -- Mailgun ID after original send
  opened boolean default false,
  short_id SERIAL,                   -- auto-increment (used in pixel tracking URL)
  created_dt timestamptz NOT NULL,   -- when email was sent
  opened_dt timestamptz,             -- when client opened email (Mailgun webhook)
  pixel_fired timestamptz,           -- when tracking pixel loaded (GET /p/{shortId}.gif)
  type integer,                      -- 1 = email
  doc_type int NOT NULL,             -- 1 = invoice, 2 = estimate
  swu_email json,                    -- full email job snapshot (renderData, templateId, sender)
  reminder_sent_dt timestamptz,      -- when is-messages cron sent the reminder
  reminder_receipt_id text,          -- Mailgun ID of the reminder email
  suppress_reminder boolean,         -- if true, cron never touches this row
  client_msg_id text,
  subtype text,
  platform varchar(64),
  ...
);

-- Built specifically for the reminder cron query
CREATE INDEX idx_msg_reminder ON msg
  (opened_dt, pixel_fired, created_dt, type, subtype, reminder_sent_dt);
```

### Lifecycle of a row

```
is-api (merchant taps Send)
  → CREATE row: suppress_reminder=false, opened_dt=null, reminder_sent_dt=null

is-messages (after Mailgun accepts)
  → UPDATE: receipt_id = mailgun_id

Mailgun webhook → is-api (client opens email)
  → UPDATE: opened_dt = now()

Client browser loads tracking pixel → is-api GET /p/{shortId}.gif
  → UPDATE: pixel_fired = now()

is-messages cron (2 days later, if not opened)
  → UPDATE: reminder_sent_dt = now(), reminder_receipt_id = mailgun_id
```

### `swu_email` field

Stores the full email job as JSON at send time — a snapshot:

```json
{
  "templateId": "tem_abc123",
  "recipient": { "address": "client@email.com" },
  "sender": { "name": "John's Plumbing", "address": "john@...", "reply_to": "..." },
  "renderData": {
    "company_name": "John's Plumbing",
    "client_name": "Bob Smith",
    "invoice_no": "INV-042",
    "amount": "$500.00",
    "pay_url": "https://checkout.invoicesimple.com/...",
    "due_date": "June 5",
    ...
  },
  "locale": "en-US",
  "tags": ["is-invoice", "is-track", "swu-template:tem_abc123"]
}
```

The is-messages cron reuses `msg.swu_email.renderData` for the reminder email — same dynamic content, different template wrapper.

### `suppress_reminder` — when it's set

**File:** `api/src/services/email.ts` (lines 376–390)

```typescript
const suppressReminder =
  params.emailType === EmailType.Reminder         // merchant manually resent from app
  || universalInvoice.setting.termsDay === InvoiceTermTypes.NONE;  // no terms set

const v: MsgInsert = {
  ...
  suppress_reminder: suppressReminder,
};
```

`suppress_reminder = true` in two cases:
1. **Manual resend from app** (`emailType: 'reminder'`): merchant already followed up, cron should stay out of it
2. **No Terms** (`termsDay === -2`): no due date set, automated reminder doesn't make sense

### `opened_dt` vs `pixel_fired`

Both mean "client read the email" but tracked via different mechanisms:

| Field | Set by | Triggered by |
|-------|--------|-------------|
| `opened_dt` | is-api (Mailgun webhook handler) | Mailgun's "opened" event fires when client's mail client loads images |
| `pixel_fired` | is-api (GET `/p/{shortId}.gif`) | 1×1 transparent pixel embedded in email body — browser fetches it when email renders |

Some email clients block Mailgun tracking but not embedded pixels, or vice versa. Using both gives better coverage.

---

## Flow 1: is-messages Cron (Unopened Invoice Reminder)

### What it does

2 days after an invoice email is sent, if the client hasn't opened it, send them a "reminder" of the same email using a different template.

### Trigger

**File:** `is-messages/packages/cdk/lib/email-sending.ts` (lines 142–146)

```typescript
if (['staging', 'production'].includes(process.env.APP_ENVIRONMENT || '')) {
  const eventRule = new events.Rule(this, 'ReminderScheduleRule', {
    schedule: events.Schedule.cron({ minute: '*/30' }),
  });
  eventRule.addTarget(new targets.LambdaFunction(reminderHandler));
}
```

- EventBridge scheduled rule fires **every 30 minutes**
- Staging + production only (not local dev)
- Invokes `ReminderHandler` Lambda with no payload — Lambda fetches its own data

**Lambda config:**
- Runtime: Node.js 20.x
- Memory: 256 MB
- Timeout: 3 min
- Retries: 0 (failure waits for next 30-min tick)

### The sweep query

**File:** `is-messages/packages/code/src/common/db.ts` (lines 51–70)

```sql
SELECT * FROM msg
WHERE
  suppress_reminder IS NOT true          -- is-api said: never remind this
  AND opened_dt IS NULL                  -- Mailgun: not opened
  AND pixel_fired IS NULL                -- pixel: not opened
  AND created_dt IS NOT NULL
  AND created_dt < (now() - '2880 minutes'::interval)   -- at least 2 days old
  AND created_dt > (now() - '96 hours'::interval)       -- but not older than 4 days
  AND type = 1                           -- emails only
  AND reminder_sent_dt IS NULL           -- not already reminded
  AND swu_email IS NOT NULL              -- has template data
ORDER BY created_dt DESC
LIMIT 1000
```

Window: 2–4 days after send. Earlier = too soon. Later = expired naturally.
Batch size: 1000 per run.

### Handler logic

**File:** `is-messages/packages/code/src/functions/email-reminder/handler.ts`

```
findAndProcessReminders()
  → getReminders(2880, 1000)     ← query above
  → for each msg (in parallel, try/catch per item):
      → sendReminder(msg)
```

**Inside `sendReminder(msg)`:**

```typescript
// 1. Fetch invoice metadata from Parse
const details = await getInvoiceReminderDetails(msg.invoice_remote_id);
// File: is-messages/packages/code/src/reminder/parse.ts (lines 34-58)
// → Parse.Cloud.run() to get invoice.dueDate + invoice.setting.termsDay

// 2. Eligibility checks
if (isInvoiceDueInFuture(details)) return;            // due date > today → skip
if (details.termsDay === InvoiceTermTypes.NONE) return; // no terms → skip
if (msg.parse_account_id === 'BZzYhGpiVc') return;    // hardcoded skip

// 3. Pick template (A/B experiment)
const { templateId, addIRTag } = await getEmailReminderExperiment({
  docType: msg.doc_type, accountId: msg.parse_account_id, payUrl: msg.url
});
// File: is-messages/packages/code/src/reminder/experiment.ts
// → estimates: SWU_ESTIMATE_REMINDER_TEMPLATE_ID
// → invoices control: SWU_INVOICE_REMINDER_TEMPLATE_ID
// → invoices treatment (A/B): SWU_INVOICE_REMINDER_IR_EXPERIMENT_TEMPLATE_ID

// 4. Build job — reuses snapshot from msg row
const body = {
  jobType: 'mailgun-email',
  reminder: true,
  templateId,
  recipient: { address: msg.to_address },
  sender: { address: 'support@getinvoicesimple.com', replyTo: fromAddress, name: appName },
  renderData: emailData,               // ← from msg.swu_email.renderData
  tags: ['is-reminder', `swu-template:${templateId}`],
};

// 5. Queue to SQS
await sendEmailJob(JSON.stringify(body));
// File: is-messages/packages/code/src/email-sending/email-queue-manager.ts
// → pushes to EMAIL_SQS_URL (internal FIFO queue)

// 6. Mark as sent immediately
await setReminderSentDt(msg.id);
// → UPDATE msg SET reminder_sent_dt = now() WHERE id = $1
```

### SQS → Mailgun

**File:** `is-messages/packages/code/src/functions/email/sqsHandler.ts` (lines 104–121)

SQS handler Lambda (separate, always polling) picks up the job:

```
sqsHandler
  → sendInvoiceUsingMailgun(job)
      → renderTemplate(templateId, renderData, locale)  ← SendWithUs API
      → sendMailgunMessage(rendered)                    ← Mailgun API
      → if job.reminder:
          UPDATE msg SET reminder_receipt_id = mailgunId
```

**File:** `is-messages/packages/code/src/common/util/mailers/mailgun.ts`

### Idempotency — all prevention layers

| Layer | Mechanism | Set by |
|-------|-----------|--------|
| `reminder_sent_dt IS NULL` | PRIMARY — query excludes already-reminded rows | is-messages cron (immediately after SQS send) |
| `opened_dt IS NULL` | Exclude if opened | is-api (Mailgun webhook) |
| `pixel_fired IS NULL` | Exclude if pixel loaded | is-api (GET /p/{shortId}.gif) |
| `suppress_reminder IS NOT true` | Exclude manual resends + no-terms | is-api (at original send time) |
| 4-day window | Natural expiry | Time-based |

`reminder_sent_dt` is set **after queueing, not after delivery.** Better to miss one reminder than to risk duplicates.

### Settings check

**None at cron level.** No `AccountSettings.getSettingPaymentReminders()` call. Controlled entirely by `suppress_reminder` baked in at send time. This means: if a merchant disables reminders after sending an invoice, the cron will still send the reminder (the `msg` row's flag was set when the invoice was sent).

### Key files summary

| File | Purpose |
|------|---------|
| `is-messages/packages/cdk/lib/email-sending.ts` | CDK: EventBridge rule (line 142) |
| `is-messages/packages/code/src/functions/email-reminder/handler.ts` | Main handler — sweep, loop, per-msg logic |
| `is-messages/packages/code/src/common/db.ts` | `getReminders()` query, `setReminderSentDt()` |
| `is-messages/packages/code/src/reminder/parse.ts` | Parse Cloud call for invoice metadata |
| `is-messages/packages/code/src/reminder/experiment.ts` | A/B template selection |
| `is-messages/packages/code/src/functions/email/sqsHandler.ts` | SQS → Mailgun |
| `is-messages/packages/code/src/common/util/mailers/mailgun.ts` | Render + send + write receipt |
| `api/src/services/email.ts` | Sets `suppress_reminder` at send time (line 376) |
| `api/src/services/msg.ts` | `insertMsg()` — creates msg row |
| `api/schema.pgsql` | PostgreSQL schema (line 150) |

### Full flow diagram

```
Merchant taps Send → is-api
  → insertMsg(): CREATE msg row (suppress_reminder=false)
  → HTTP → is-messages → SQS → Mailgun → email sent
  → UPDATE msg SET receipt_id = mailgunId

--- Client opens email ---
  → Mailgun webhook → is-api: UPDATE msg SET opened_dt = now()
  OR
  → Browser loads pixel → is-api: UPDATE msg SET pixel_fired = now()

--- Every 30 min: EventBridge → ReminderHandler Lambda ---
  → SELECT * FROM msg WHERE reminder_sent_dt IS NULL AND opened_dt IS NULL ...
  → [client never opened → row returned]
  → Parse.Cloud.run() for invoice metadata
  → Skip if: due in future / no terms / hardcoded account
  → Build EmailInvoiceMsgJob { reminder: true, renderData: msg.swu_email.renderData }
  → sendEmailJob() → EMAIL_SQS_URL
  → UPDATE msg SET reminder_sent_dt = now()

--- SQS handler Lambda ---
  → renderTemplate() via SendWithUs
  → Mailgun sends reminder
  → UPDATE msg SET reminder_receipt_id = mailgunId
```

---

## Flow 2: Abandoned Cart (Outstanding Checkout Reminders)

### What it does

When a client views the checkout page but doesn't complete payment, schedule two follow-up emails:
- **30 minutes later:** "Don't forget to complete your payment"
- **3 days later:** second nudge

Triggered by the client's action (viewing checkout), not by a time sweep.

### Architecture: Three Lambdas

```
checkout-viewed event ──► schedule-reminder Lambda   creates the schedules
                                    │
                         EventBridge Scheduler
                          ┌──────────┴──────────┐
                      at T+30min            at T+3days
                          └──────────┬──────────┘
                                     ▼
                          send-reminder Lambda       validates eligibility + sends email

payment-attempted event ─► remove-schedulers Lambda  cancels pending schedules
```

Each Lambda has one responsibility. They communicate through AWS (EventBridge events, Scheduler, DynamoDB) — not direct calls.

### Trigger: checkout-viewed event

When a client opens the checkout link, **is-unifiedxp** emits to EventBridge:

```json
{
  "EventBusName": "payment-event-bus",
  "DetailType": "checkout-viewed",
  "Source": "checkout",
  "Detail": {
    "invoiceId": "abc123",
    "currencyCode": "USD",
    "checkoutUrl": "https://checkout.invoicesimple.com/pay/abc123",
    "recipientEmailAddress": "client@email.com"
  }
}
```

**CDK rule:** `is-services/packages/aws/cdk/src/lib/constructs/abandoned-cart/event-bridge-construct.ts` (lines 24–38)

Matches `detail-type: checkout-viewed, source: checkout` → invokes `schedule-reminder` Lambda. 2 retries + DLQ (`checkout-viewed-event-dlq`).

### Lambda 1: schedule-reminder

**File:** `is-services/packages/services/is-abandoned-cart/src/lambda-handlers/schedule-reminder/event-bridge-handler.ts`

```typescript
// 1. Idempotency check
const existing = await getCheckoutReminderSchedule({ invoiceId, recipientEmailAddress });
if (existing) return;  // already scheduled (e.g. client refreshed checkout page)

// 2. Create two one-time EventBridge Scheduler schedules
await createEventBridgeSchedule({
  scheduleName: `${invoiceId}-30mins`,
  scheduleTime: thirtyMinutesLaterFormatted(),  // ISO 8601
  targetLambdaArn: SEND_REMINDER_LAMBDA_ARN,
  payload: { invoiceId, currencyCode, checkoutUrl, recipientEmailAddress, scheduledIn: '30mins' },
  actionAfterCompletion: 'DELETE',  // auto-cleans after firing
});

await createEventBridgeSchedule({
  scheduleName: `${invoiceId}-3days`,
  scheduleTime: threeDaysLaterFormatted(),
  targetLambdaArn: SEND_REMINDER_LAMBDA_ARN,
  payload: { invoiceId, currencyCode, checkoutUrl, recipientEmailAddress, scheduledIn: '3days' },
  actionAfterCompletion: 'DELETE',
});

// 3. Store schedule names in DynamoDB (for cleanup reference)
await putCheckoutReminderSchedule({
  invoiceId,
  recipientEmailAddress,
  thirtyMinutesLaterScheduleName: `${invoiceId}-30mins`,
  threeDaysLaterScheduleName: `${invoiceId}-3days`,
});
```

### DynamoDB: CheckoutReminderSchedule

**CDK:** `is-services/packages/aws/cdk/src/lib/constructs/abandoned-cart/dynamodb-construct.ts`

```typescript
new dynamodb.Table(this, 'CheckoutReminderSchedule', {
  partitionKey: { name: 'invoiceId', type: AttributeType.STRING },
  sortKey:      { name: 'recipientEmail', type: AttributeType.STRING },
  timeToLiveAttribute: 'expiredAt',
  billingMode: BillingMode.PAY_PER_REQUEST,
});
```

**Row shape:**
```json
{
  "invoiceId":     "abc123",
  "recipientEmail": "client@email.com",
  "30mins":        { "scheduleName": "abc123-30mins" },
  "3days":         { "scheduleName": "abc123-3days" },
  "expiredAt":     1780000000   ← unix timestamp, 150 days from now (TTL auto-delete)
}
```

**Why DynamoDB and not MongoDB/PostgreSQL:**
- AWS-native — zero setup for Lambdas (no VPC credentials, no connection pool)
- Native TTL — rows auto-delete after 150 days, no cleanup job needed
- Simple key-value access — only GET and QUERY operations, no joins
- This is short-lived operational state, not business data

### Lambda 2: send-reminder

**File:** `is-services/packages/services/is-abandoned-cart/src/lambda-handlers/send-reminder/scheduled-handler.ts`

Invoked by EventBridge Scheduler at T+30min or T+3days. Input = what was stored in the schedule payload:

```json
{
  "invoiceId": "abc123",
  "currencyCode": "USD",
  "checkoutUrl": "https://...",
  "recipientEmailAddress": "client@email.com",
  "scheduledIn": "30mins"
}
```

**Inside the handler:**

```typescript
// 1. Fetch live data from Parse (not a snapshot — fresh at send time)
const [invoice, payments] = await Promise.all([
  getUniversalInvoiceById(invoiceId),
  // → Parse.Cloud.run('publicInvoice', { id: invoiceId })
  getPaymentsFromParse(invoiceId),
  // → Parse.Cloud.run('invoiceGetPayments', { invoiceId })
]);

// 2. Settings check — live, at send time
const enabled = await AccountSettings.getSettingPaymentReminders(invoice.accountId);
// → @is/account package → MongoDB Setting collection
if (!enabled) return;

// 3. Due date check
if (isInvoiceDueInFuture(invoice)) return;   // due date > today → skip

// 4. Terms check
if (isInvoiceWithNoTerms(invoice)) return;   // termsDay === NONE → skip

// 5. Payability check
if (!isInvoicePayable({ invoice, payments })) return;  // balanceDue <= 0 → skip

// 6. Send email
await sendReminderEmail({ invoice, recipientEmailAddress, checkoutUrl, payments, scheduledIn });
```

**Inside `sendReminderEmail()`** (`is-abandoned-cart/src/utils/send-reminder-email.ts`):

```typescript
const { balanceDue } = calculate(invoice, payments);
const user = await Users.getUserByAccountId(invoice.accountId);  // @is/account

const renderData = {
  client_name:   invoice.client?.name || 'Customer',
  amount:        formatCurrency(balanceDue, currencyCode),
  company_name:  invoice.company.name,
  company_email: invoice.company.email || user.email,
  company_logo:  invoice.company.logo?.url,
  invoice_no:    invoice.invoiceNo,
  date:          formattedDate(invoice.localeCode),
  pay_url:       `${checkoutUrl}?reminder-email=${scheduledIn}`,  // tracking param
};

// Cross-service email send
await queueSWUJob({
  recipient: { address: recipientEmailAddress },
  sender: noReplyToSender,
  renderData,
  templateId: ABANDONED_CART_EMAIL_TEMPLATE_ID,
  // staging:    tem_R4Jym6MFBBRwx6xYRYkkfptT
  // production: tem_Hcqm4wpPTbpMwgJ4cCtkjBbD
});
// → @is/messaging package → @invoice-simple/is-msg SDK → HTTP POST to is-messages
// → is-messages → SQS → SendWithUs render → Mailgun send

// Analytics (not state management)
await trackSendEmail({ userId, accountId, invoiceId, recipientEmailAddress, scheduledIn });
// → IsEvents.sendEvent({ action: 'reminder-email-sent' })
```

**Key difference from Flow 1:** Data is fetched **live** at send time, not reused from a snapshot. If merchant updated the invoice between checkout view and the 3-day email, the email reflects the update.

**No write-back:** Nothing is written to MongoDB, PostgreSQL, or Parse after sending. No idempotency field on the Invoice. The only state tracking is DynamoDB (managed by the other two Lambdas).

### Lambda 3: remove-schedulers

**File:** `is-services/packages/services/is-abandoned-cart/src/lambda-handlers/remove-schedulers/event-bridge-handler.ts`

When client pays, is-unifiedxp emits `payment-attempted`. EventBridge rule triggers this Lambda:

```typescript
// 1. Find all schedule records for this invoice
const schedules = await queryCheckoutReminderScheduleByInvoiceId(invoiceId);
// → DynamoDB QUERY: PK = invoiceId

// 2. Delete each schedule from EventBridge Scheduler
for (const item of schedules) {
  await Promise.all([
    deleteEventBridgeSchedule({ scheduleName: item['30mins'].scheduleName }),
    deleteEventBridgeSchedule({ scheduleName: item['3days'].scheduleName }),
  ]);
  // Errors are caught and logged — silent failure on already-deleted schedules
}
// DynamoDB record stays (TTL handles cleanup after 150 days)
```

**Two safety layers against sending after payment:**
1. `remove-schedulers` deletes pending schedules immediately on payment
2. `send-reminder` re-checks `isInvoicePayable()` at send time — so even if deletion races with the schedule firing, the email is skipped

### CDK infrastructure summary

**File:** `is-services/packages/aws/cdk/src/lib/constructs/abandoned-cart/`

| Resource | Type | Purpose |
|----------|------|---------|
| `payment-event-bus` | EventBridge Bus | Shared event bus (cross-service) |
| `checkout-viewed` rule | EventBridge Rule | → schedule-reminder Lambda |
| `payment-attempted` rule | EventBridge Rule | → remove-schedulers Lambda |
| `checkout-reminder-schedule-group` | Scheduler Group | Groups all per-invoice schedules |
| `CheckoutReminderSchedule` | DynamoDB Table | State/idempotency tracking |
| `schedule-reminder` | Lambda | Creates schedules |
| `send-reminder` | Lambda | Validates + sends email |
| `remove-schedulers` | Lambda | Cancels schedules on payment |
| `checkout-viewed-event-dlq` | SQS Queue | DLQ for failed schedule-reminder |
| `payment-attempted-event-dlq` | SQS Queue | DLQ for failed remove-schedulers |

### Environment variables

```
# Shared (abandonedCartStackEnv)
APP_ENV, PARSE_SERVER_URL, PARSE_APP_ID, PARSE_MASTER_KEY
IS_MESSAGES_API_KEY, MONGODB_URI

# schedule-reminder only
SCHEDULER_GROUP_NAME, SEND_REMINDER_LAMBDA_ARN
SEND_REMINDER_LAMBDA_ROLE_ARN, CHECKOUT_REMINDER_SCHEDULE_TABLE

# remove-schedulers only
SCHEDULER_GROUP_NAME, CHECKOUT_REMINDER_SCHEDULE_TABLE
```

### Key files summary

| File | Purpose |
|------|---------|
| `is-abandoned-cart/src/lambda-handlers/schedule-reminder/event-bridge-handler.ts` | Schedule creation |
| `is-abandoned-cart/src/lambda-handlers/send-reminder/scheduled-handler.ts` | Eligibility + send |
| `is-abandoned-cart/src/lambda-handlers/remove-schedulers/event-bridge-handler.ts` | Cleanup on payment |
| `is-abandoned-cart/src/utils/send-reminder-email.ts` | Build renderData + queueSWUJob |
| `is-abandoned-cart/src/utils/dynamo-db.ts` | DynamoDB GET / PUT / QUERY |
| `is-abandoned-cart/src/utils/event-bridge-schedule.ts` | Scheduler create / delete |
| `is-abandoned-cart/src/utils/is-invoice-payable.ts` | balanceDue check |
| `is-abandoned-cart/src/utils/provider/account/get-payment-reminders-enabled.ts` | Settings check |
| `is-abandoned-cart/src/utils/provider/parse/universal-invoice.ts` | Parse.Cloud.run invoice |
| `is-abandoned-cart/src/utils/provider/parse/get-payments-from-parse.ts` | Parse.Cloud.run payments |
| `is-services/packages/aws/cdk/src/lib/constructs/abandoned-cart/` | All CDK constructs |

### Full flow diagram

```
Client opens checkout → is-unifiedxp
  → EventBridge: checkout-viewed { invoiceId, checkoutUrl, recipientEmail }
  → schedule-reminder Lambda:
      DynamoDB GET: already scheduled? → return (idempotent)
      Create Scheduler: T+30min → send-reminder { scheduledIn: '30mins' }
      Create Scheduler: T+3days → send-reminder { scheduledIn: '3days' }
      DynamoDB PUT: { invoiceId, recipientEmail, scheduleName30, scheduleName3d, TTL }

--- At T+30min: EventBridge Scheduler fires send-reminder ---
  → Parse.Cloud.run('publicInvoice') + Parse.Cloud.run('invoiceGetPayments')
  → AccountSettings.getSettingPaymentReminders() → off? return
  → isInvoiceDueInFuture() → future? return
  → isInvoiceWithNoTerms() → no terms? return
  → isInvoicePayable() → paid? return
  → calculate balanceDue, fetch user, build renderData
  → queueSWUJob() → HTTP to is-messages → SQS → SendWithUs → Mailgun → email sent
  → trackSendEmail() → IsEvents

--- Client pays → is-unifiedxp ---
  → EventBridge: payment-attempted { invoiceId }
  → remove-schedulers Lambda:
      DynamoDB QUERY: get schedule names for invoiceId
      EventBridge Scheduler: delete 3-day schedule
      (30-min already auto-deleted after firing)
```

---

## Side-by-Side Comparison

| Dimension | Flow 1: is-messages cron | Flow 2: Abandoned Cart | APR (planned) |
|-----------|--------------------------|------------------------|---------------|
| **What it reminds about** | Unopened invoice email (2–4 days after send) | Abandoned checkout (30min + 3days after view) | Invoice due today |
| **Trigger** | EventBridge every 30 min | `checkout-viewed` EventBridge event | EventBridge daily cron |
| **Trigger timing** | Approximate (next 30-min tick after 2 days) | Precise (exactly 30min / 3days after checkout) | Approximate (one fixed daily time) |
| **Lambdas** | 1 | 3 | 1 |
| **Data source (reads)** | PostgreSQL `msg` + Parse (metadata only) | Parse Cloud Functions | `@is/mongo` + `@is/account` |
| **Settings check** | None — `suppress_reminder` baked in at send time | `AccountSettings.getSettingPaymentReminders()` at send time | `AccountSettings.getSettingRemoteId('AutomaticPaymentRequests')` at cron time |
| **Settings respects mid-flight change?** | No | Yes | Yes |
| **Email data** | Snapshot (`msg.swu_email.renderData`) | Live fetch from Parse | Live fetch from `@is/mongo` |
| **Email sending** | Internal SQS (is-messages only) | `queueSWUJob()` → HTTP → is-messages → SQS → Mailgun | `queueSWUJob()` → HTTP → is-messages → SQS → Mailgun |
| **Idempotency** | `reminder_sent_dt` on `msg` row | DynamoDB record + payability check | `paymentRequestSentDate` on Invoice (write method TBD) |
| **State write-back** | `UPDATE msg SET reminder_sent_dt` (PostgreSQL) | None to Mongo/Postgres | TBD |
| **Cleanup on payment** | N/A (payability check at send time) | `payment-attempted` event → delete schedules | N/A (`balanceDue > 0` filter) |
| **Infra** | 1 Lambda + EventBridge rule | 3 Lambdas + Scheduler group + DynamoDB + 2 EventBridge rules + 2 DLQs | 1 Lambda + EventBridge rule |
| **Codebase** | `is-messages/` (legacy) | `is-services/is-abandoned-cart/` | `is-services/` (new package) |

---

## Implications for APR Design

1. **Shape:** APR is closest to Flow 1 in shape (single daily cron, sweep, filter, send) but lives in is-services (modern codebase, MongoDB access).

2. **Email sending:** Use Flow 2's `queueSWUJob()` pattern (cross-service HTTP to is-messages) — not Flow 1's internal SQS.

3. **Settings check:** Use Flow 2's `@is/account` pattern for a live settings check at cron time. Flow 1's baked-in approach doesn't respect mid-flight changes.

4. **Data reads:** Use `@is/mongo` for invoice queries (typed collection accessors). Use `@is/account` for settings (`AccountSettings.getSettingRemoteId(accountId, 'AutomaticPaymentRequests')`). Neither Flow 1 nor Flow 2 uses this for invoice reads — APR is the first.

5. **Idempotency:** Open question. Flow 1 writes `reminder_sent_dt` to its own PostgreSQL table. Flow 2 uses DynamoDB. APR needs to write to the Invoice document — method TBD (Parse Cloud Function / DynamoDB / direct Mongo).

6. **Infra simplicity:** APR needs far less infra than Flow 2. No per-invoice scheduling, no DynamoDB, no cleanup Lambda. One cron rule + one Lambda is sufficient.
