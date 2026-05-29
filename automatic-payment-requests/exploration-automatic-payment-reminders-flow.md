# Exploration: Automatic Payment Reminders — Current Flow & New Feature Analysis

> How does the existing automatic payment reminder system work end-to-end, and how would we add a "Payment Requested" history entry for the Deposits & Milestones epic?

**Related:**
- [Invoice History Auto-Pay Statuses Exploration](../invoice-history/exploration-invoice-history-auto-pay-statuses.md)
- Figma: https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5285-33486

---

## Repos Involved

| Repo | Role |
|------|------|
| `api` (legacy is-api) | Sends invoice emails, inserts PostgreSQL msg rows, handles open tracking |
| `is-messages` | EventBridge-triggered reminder Lambda, email delivery via Mailgun, push notifications, Parse Msg status updates |
| `is-parse-server` | Parse MongoDB — Msg class (invoice history source of truth for clients) |
| `is-web-app` | Web frontend — renders invoice history via `history-base-row.tsx` |
| `is-mobile` | Mobile frontend — renders invoice history via `invoice-history.tsx` |
| `is-services` | Mounts is-messaging domain model at `/messaging/statuses` (read-only) |

---

## Full End-to-End Flow (Current)

### Step 1: Global Setting — "Automatic Payment Reminders" Toggle

**Mobile UI:**
- `is-mobile/src/features/payments/payments-option/components/payments-option-section.tsx` (line 262-274)
- Field: `SettingKeys.PaymentReminders`
- Saves to Realm → syncs to Parse `Setting` collection

**Parse Storage:**
- Collection: `Setting`
- Document: `{ remoteId: "PaymentReminders", valBool: true/false, account: <pointer> }`

**Enforcement (per-message):**
- When is-api inserts a msg row, it sets `suppress_reminder` based on email type and invoice terms
- The reminder handler queries `WHERE suppress_reminder IS NOT true`
- The account-level setting is checked separately by `is-abandoned-cart` service

**Files:**
- `is-mobile/src/services/realm/entities/settings/repository.ts` (line 151)
- `is-parse-server/cloud/collections/setting/settingHooks.ts` (line 63)
- `is-services/packages/domain-model/is-account/src/settings/index.ts` (lines 54-62)

---

### Step 2: User Sends Invoice (Manual Action)

**Web v2 flow:**
```
User clicks "Send Invoice"
  → POST /api/v2/email-invoice
  → is-api: v2EmailInvoiceHandler() in src/controllers/invoice-sending.ts:350
```

**What is-api does:**

1. **Creates Parse Msg** (`src/services/email.ts:347`):
   ```
   createParseMsg({ accountId, from, to, invoiceRemoteId, docVersion, emailType, subject })
   → Parse REST API creates Msg object in MongoDB
   → Returns remoteId (used as client_msg_id in PostgreSQL)
   ```

2. **Inserts PostgreSQL msg row** (`src/services/msg.ts:121`):
   ```sql
   INSERT INTO msg (id, url, parse_account_id, invoice_no, to_address, type, doc_type,
                    client_msg_id, platform, invoice_remote_id, suppress_reminder)
   VALUES (...)
   ```
   - `type = 1` (email)
   - `client_msg_id` = Parse Msg remoteId (links the two records)
   - `suppress_reminder` = true if email type is Reminder or invoice has no terms

3. **Queues email for delivery** (`src/providers/is-msg/email.ts`):
   ```
   sendInvoiceEmailMsg(emailJob)
   → isMsg.email.sendTemplatedEmail()
   → is-messages Lambda receives job via API Gateway
   → SQS → sqsHandler → renders template (SendWithUs API) → sends via Mailgun
   → Updates msg.receipt_id in PostgreSQL
   ```

**Mobile v1 flow (similar but different entry):**
```
User taps "Send"
  → Creates Msg LOCALLY in Realm (status: SENDING)
  → POST /api/v1/email-invoice (multipart form with PDF)
  → is-api: v1EmailInvoiceHandler() — same insertMsg() + email queue
  → Realm syncs Msg to Parse
```

**Key files:**
- `api/src/controllers/invoice-sending.ts` — v1EmailInvoiceHandler, v2EmailInvoiceHandler
- `api/src/services/email.ts:213` — `send()` function (v2 path)
- `api/src/services/msg.ts:121` — `insertMsg()` (PostgreSQL INSERT)
- `api/src/providers/is-msg/email.ts` — queues to is-messages
- `is-messages/packages/code/src/functions/email/sqsHandler.ts` — processes email job
- `is-messages/packages/code/src/common/util/mailers/mailgun.ts` — sends via Mailgun

---

### Step 3: Client Opens Invoice (or Doesn't)

**Open detection trigger:**
Client clicks the invoice link in the email → `GET /v/{hashId}/{invoiceNo}.pdf`

**is-api handling** (`src/controllers/invoice-viewing.ts`):

1. **Sets opened_dt in PostgreSQL** (line 177):
   ```sql
   UPDATE msg SET opened = true, opened_dt = now()
   WHERE short_id = $1 AND opened_dt is null
   ```

2. **Sets pixel_fired** (line 121):
   ```sql
   UPDATE msg SET pixel_fired = now()
   WHERE short_id = $1 AND pixel_fired is null
   ```

3. **Notifies is-messages** (line 387):
   ```
   pushOpened(msg)
   → isMsg.pushNotification.open({ invoiceNo, toAddress, receiptId, msgRemoteId, accountId, invoiceRemoteId })
   → is-messages Lambda receives 'open' push job
   ```

4. **is-messages updates Parse** (`packages/code/src/push/sync-status.ts:23`):
   ```
   convertToSyncPush(job):
     → updateMessageStatus(accountId, msgRemoteId, { eventType: 'opened', eventTime })
       → Queries Parse for Msg by remoteId
       → Sets Msg.status = { eventType: 'opened', eventTime }
     → Creates MsgEvent({ eventType: 'opened', msgRemoteId, invoiceRemoteId })
     → Sends sync push notification to mobile devices
   ```

**If client DOESN'T open:**
- `opened_dt` stays NULL in PostgreSQL
- Parse Msg.status stays at 'sent' or 'delivered'
- This is what the reminder handler checks for

**Tracking mechanism:**
- Mailgun's built-in tracking is DISABLED (`'o:tracking-opens': 'no'`)
- Invoice Simple uses its own tracking: client clicking the view link triggers the open
- The `/v/{hashId}` endpoint in is-api is the tracking point (not a pixel in the email body)

**Key files:**
- `api/src/controllers/invoice-viewing.ts` — viewInvoice(), onMsgViewed(), setMsgAsOpenedByShortId()
- `api/src/services/mailgun.ts:257` — pushOpened()
- `api/src/providers/is-msg/push-notification.ts:28` — queueOpen()
- `is-messages/packages/code/src/push/sync-status.ts` — convertToSyncPush(), updateMessageStatus()
- `is-messages/packages/code/src/push/parse.ts:45` — updateMessageStatus() (Parse write)

---

### Step 4: 2+ Days Pass, Email Not Opened → Reminder Sent

**Trigger:**
- EventBridge rule: cron every 30 minutes (staging + production only)
- CDK: `is-messages/packages/cdk/lib/email-sending.ts` (lines 141-146)

**Handler:** `is-messages/packages/code/src/functions/email-reminder/handler.ts`

**Query** (`packages/code/src/common/db.ts:51`):
```sql
SELECT * FROM msg
WHERE
  suppress_reminder IS NOT true
  AND opened_dt IS NULL
  AND pixel_fired IS NULL
  AND created_dt IS NOT NULL
  AND created_dt < (now() - '2880 minutes'::interval)  -- 2 days
  AND created_dt > (now() - '96 hours'::interval)      -- 4 days max
  AND type = 1                                          -- email type
  AND reminder_sent_dt IS NULL                          -- not already reminded
  AND swu_email IS NOT NULL                             -- has template data
ORDER BY created_dt DESC
LIMIT 1000
```

**Additional checks before sending** (handler.ts:52-56):
```
getInvoiceReminderDetails(invoiceId) → fetches from Parse
  → isInvoiceDueInFuture(dueDate) → SKIP if due date is in the future
  → isInvoiceWithNoTerms(termsDay) → SKIP if terms = NONE
  → Hardcoded skip for account 'BZzYhGpiVc'
```

**Email sending flow:**
```
sendReminder(msg):
  → getEmailReminderExperiment() picks template:
     - SWU_INVOICE_REMINDER_TEMPLATE_ID (standard)
     - SWU_ESTIMATE_REMINDER_TEMPLATE_ID (estimates)
     - SWU_INVOICE_REMINDER_IR_EXPERIMENT_TEMPLATE_ID (US experiment)
  → Builds EmailInvoiceMsgJob with reminder: true
  → sendEmailJob() → SQS queue (EMAIL_SQS_URL)
  → sqsHandler → sendInvoiceUsingMailgun():
     → renderTemplate() via SendWithUs API (gets subject, html, text)
     → mg.messages.create(MAILGUN_DOMAIN, data) — Mailgun API
     → setReminderReceiptId(msgId, messageId) — UPDATE msg
  → setReminderSentDt(msg.id) — UPDATE msg SET reminder_sent_dt = now()
```

**What does NOT happen:**
- No new Msg created in Parse MongoDB
- No MsgEvent created
- No entry appears in invoice history (web or mobile)
- No push notification to mobile

**Key files:**
- `is-messages/packages/cdk/lib/email-sending.ts:141-146` — EventBridge rule
- `is-messages/packages/code/src/functions/email-reminder/handler.ts` — main handler
- `is-messages/packages/code/src/common/db.ts:51` — getReminders() SQL
- `is-messages/packages/code/src/reminder/experiment.ts` — template A/B logic
- `is-messages/packages/code/src/reminder/is-invoice-due-in-future.ts` — due date check
- `is-messages/packages/code/src/reminder/is-invoice-with-no-terms.ts` — terms check
- `is-messages/packages/code/src/email-sending/email-queue-manager.ts` — SQS producer
- `is-messages/packages/code/src/functions/email/sqsHandler.ts` — SQS consumer
- `is-messages/packages/code/src/common/util/mailers/mailgun.ts` — Mailgun delivery

---

## Data Model Summary

### PostgreSQL `msg` table (is-api / is-messages shared)

| Field | Type | Set by | Purpose |
|-------|------|--------|---------|
| id | string | is-api (insertMsg) | Primary key |
| short_id | serial | auto-increment | Used in URL hashId |
| to_address | string | is-api | Recipient email |
| parse_account_id | string | is-api | Account ID |
| invoice_id | string | is-api | Invoice ID |
| invoice_no | string | is-api | Invoice number |
| invoice_remote_id | string | is-api | Links to Parse Invoice |
| client_msg_id | string | is-api | Links to Parse Msg remoteId |
| type | int | is-api | 1=email, 2=sms |
| doc_type | int | is-api | invoice/estimate/statement |
| url | string | is-api | PDF URL on S3 |
| platform | string | is-api | web/ios/android |
| swu_email | jsonb | is-api (UPDATE after insert) | Full email template data |
| receipt_id | string | is-messages | Mailgun message ID |
| suppress_reminder | boolean | is-api | Skip reminders for this msg |
| opened_dt | timestamp | is-api (on view) | When client opened |
| pixel_fired | timestamp | is-api (on view) | When tracking fired |
| pushed_open_dt | timestamp | is-api | When open push was sent |
| reminder_sent_dt | timestamp | is-messages | When reminder was sent |
| reminder_receipt_id | string | is-messages | Reminder Mailgun ID |
| created_dt | timestamp | auto | Row creation time |

### Parse MongoDB `Msg` class (source of truth for invoice history UI)

| Field | Set by | Purpose |
|-------|--------|---------|
| remoteId | is-api / mobile | Unique ID (matches client_msg_id in PG) |
| account | is-api / mobile | Pointer to Account |
| invoiceRemoteId | is-api / mobile | Links to Invoice |
| to | is-api / mobile | Recipient email |
| from | is-api / mobile | Sender email |
| sendTime | is-api / mobile | Timestamp (ms) |
| status | is-messages (on events) | `{ eventType, eventTime, desc? }` |
| docVersion | is-api / mobile | `{ versionId, createdAt, restoredCreatedAt? }` |
| deleted | is-api / mobile | Soft delete flag |

### Parse MongoDB `MsgEvent` class (individual delivery events)

| Field | Set by | Purpose |
|-------|--------|---------|
| msgRemoteId | is-messages | Links to Msg |
| eventType | is-messages | delivered/opened/dropped/sent |
| eventTime | is-messages | Timestamp |
| invoiceRemoteId | is-messages | Links to Invoice |
| account | is-messages | Pointer to Account |

---

## Current Status Types (what shows in history UI)

| eventType | Web Icon | Mobile Icon | Label |
|-----------|----------|-------------|-------|
| sending | IconSendMail | MaterialIndicator (spinner) | "Sending" |
| sent | IconSendMail | email-sent | "Sent" |
| delivered | IconDelivered | email-delivered | "Delivered" |
| opened | IconOpened | email-open | "Opened" |
| signed | IconSigned | signature | "Signed" |
| dropped | IconFailed | email-dropped | "Failed to Send" |
| failed_to_send | IconFailed | email-dropped | "Failed to Send" |
| email_limit_reached | IconFailed | email-dropped | "Failed to Send" |
| version_saved_manual | (hidden) | VersionManualSaveIcon | "Invoice Version Saved" |
| version_restored | IconVersionRestored | VersionRestoredIcon | "Version Restored" |
| queued | (hidden) | (not rendered) | — |

---

## Gap Analysis: What's Missing for "Payment Requested" in History

The existing reminder system sends the email but **never creates a Msg in Parse**. The history entry requires:

1. A new Msg object in Parse MongoDB with `status.eventType = 'payment_request_sent'`
2. New UI rendering in web (`history-base-row.tsx`) and mobile (`invoice-history.tsx`)

---

## Proposed: "Payment Requested" for Deposits & Milestones

### Option 3 Context (from Seth)

For **non-vaulted invoices** (client hasn't set up auto-pay):
- If "Automatic Payment Requests" is enabled in Settings
- On the **due date** of a milestone/upcoming payment
- System sends a reminder email to the client
- "Payment Requested" is logged in invoice history

This is a **new flow** — different trigger conditions from the existing 2-day-after-send reminder.

### What's Needed

| Component | Change | Exists Today? |
|-----------|--------|---------------|
| EventBridge rule | New rule: fires on milestone due date | NO — new trigger |
| Lambda handler | Sends reminder + creates Parse Msg | Partially — email send pattern exists in is-messages, Parse write is new |
| Setting | New or reuse "Automatic Payment Reminders" toggle | Existing toggle could be reused |
| Parse Msg write | Create Msg with eventType: 'payment_request_sent' | NO — is-messages has never written to Parse Msg collection |
| Web UI | New icon + label for 'payment_request_sent' | NO — new case in history-base-row.tsx |
| Mobile UI | New icon + localization for 'payment_request_sent' | NO — new case in invoice-history.tsx |

### Architecture Decision: Where to Create the Parse Msg

**Option A: is-messages Lambda (alongside email send)**
- Pro: Single place — send email + log history in one handler
- Con: is-messages has never written to Parse Msg collection (reads only)
- Con: Adds coupling between email delivery and history logging

**Option B: is-api (same as manual sends)**
- Pro: is-api already has `createParseMsg()` and `insertMsg()` — proven pattern
- Con: Would need is-messages to call back to is-api after sending, or is-api to own the trigger
- Con: is-api is legacy, team is moving away from it

**Option C: New Parse Cloud Function called from is-messages**
- Pro: Follows is-stripe/is-paypal pattern (`Parse.Cloud.run('createInvoiceHistoryEntry')`)
- Pro: Keeps Parse write logic centralized in is-parse-server
- Con: is-messages would need Parse masterKey for Cloud.run (it already has it for reads)

**Recommendation:** Option C — add a Cloud function in is-parse-server, call it from is-messages after the email is sent. Lowest risk, follows existing patterns.

---

## Relationship to Auto-Pay Status Events

If we later add auto-pay events (charged, failed, cancelled, enabled), the Parse Cloud function created for "Payment Requested" would be reusable. The difference is:

| Event | Triggered from | Parse write via |
|-------|---------------|-----------------|
| payment_request_sent | is-messages Lambda (new) | Cloud function |
| auto_pay_enabled | is-payments Lambda (createRecurringPayment topic) | Same Cloud function |
| auto_pay_charged | is-payments Lambda (paymentSuccess topic) | Same Cloud function |
| auto_pay_failed | is-payments Lambda (paymentFailure, max retries) | Same Cloud function |
| auto_pay_cancelled | is-payments Lambda (recurringPaymentCancelled) | Same Cloud function |

The Cloud function becomes the single entry point for creating non-email history entries.

---

## Effort Estimate (Payment Requested only)

| Work Item | Estimate |
|-----------|----------|
| New EventBridge rule + Lambda handler (CDK + handler code) | 1.5-2 days |
| Parse Cloud function `createInvoiceHistoryEntry` | 0.5 days |
| Call Cloud function from is-messages after email send | 0.5 days |
| Web UI — new icon + case in history-base-row.tsx | 0.5 days |
| Mobile UI — new icon + case in invoice-history.tsx + localization | 1 day |
| Unit tests | 1 day |
| Integration testing (trigger → email → history entry) | 0.5-1 day |

**Total: ~5-6 engineering days**

---

## Abandoned Cart vs Automatic Payment Reminders — Two Separate Systems

There are **two** reminder systems that both email clients about unpaid invoices. They are completely independent:

### Combined Comparison (Verified Against Code)

| | Abandoned Cart (`is-abandoned-cart`) | Automatic Payment Reminder (is-messages cron) |
|---|---|---|
| **Trigger** | Client views checkout page but doesn't pay (`checkout-viewed` EventBridge event) | Invoice email sent but not opened after 2 days (`opened_dt IS NULL`) |
| **Timing** | 30 min + 3 days after checkout view | 2-4 days after invoice email sent |
| **Architecture** | Event-driven (EventBridge → 2 scheduled rules → Lambda) | Batch cron every 30 min (`cron({ minute: '*/30' })`) |
| **Location** | `is-services/packages/services/is-abandoned-cart/` | `is-messages/packages/code/src/functions/email-reminder/` |
| **CDK** | `is-services/packages/aws/cdk/src/lib/constructs/abandoned-cart.ts` | `is-messages/packages/cdk/lib/email-sending.ts` (lines 141-146) |
| **Controlled by** | Account-level `PaymentReminders` setting (mobile toggle) | Per-message `suppress_reminder` field in PostgreSQL |
| **User-configurable?** | Yes — merchant can toggle off in Settings | No — no user-facing toggle |
| **Suppression logic** | Lambda checks `getPaymentRemindersSetting(accountId)` before sending | is-api sets `suppress_reminder = true` at insert time if: (1) email is already a reminder (`emailType === 'reminder'`), or (2) invoice terms = NONE |
| **Granularity** | Account-level (all or nothing) | Per-message (automatic, based on email type + invoice terms) |
| **Template** | `ABANDONED_CART_EMAIL_TEMPLATE_ID` (dedicated SWU template) | `SWU_INVOICE_REMINDER_TEMPLATE_ID` (+ A/B experiment variants) |
| **Template IDs** | Staging: `tem_R4Jym6MFBBRwx6xYRYkkfptT`, Prod: `tem_Hcqm4wpPTbpMwgJ4cCtkjBbD` | Env var from GitHub Actions vars (test: `tem_ua5xYTnBKJbHJYUoyfLGjP`) |
| **Sender** | `no-reply@invoicesimple.com` (prod) / `no-reply@dev.invoicesimple.com` (non-prod) | `support@getinvoicesimple.com` (hardcoded in handler.ts:68) |
| **Subject** | Defined in SWU template (not in code) | Defined in SWU template (not in code) |
| **CTA button** | "Complete Payment" → checkout URL + `?reminder-email=30mins` or `?reminder-email=3days` | "View invoice {no}" → invoice view URL (`/v/{hashId}`) |
| **Delivery** | `queueSWUJob()` → is-messages API → SQS → Mailgun | `sendEmailJob()` → SQS → `sqsHandler` → Mailgun direct |
| **Cleanup** | `remove-scheduler` Lambda fires on `payment-attempted` event, removes scheduled rules | N/A — one-shot, `reminder_sent_dt` prevents re-send |
| **Creates Parse Msg?** | No | No |
| **History entry?** | No | No |

### Key Distinction

The **"Automatic Payment Reminders" toggle** in Settings (`SettingKeys.PaymentReminders`) controls **abandoned cart only**. The is-messages cron does NOT check this setting — it relies on the per-msg `suppress_reminder` boolean that is-api sets at email send time.

### suppress_reminder Deep Dive

The is-messages cron has no user-facing toggle. Suppression is determined automatically by is-api when the msg row is inserted:

```typescript
// api/src/services/email.ts:376
const suppressReminder = params.emailType === EmailType.Reminder
  || universalInvoice.setting.termsDay === InvoiceTermTypes.NONE;
```

**Suppressed (no cron reminder will fire) when:**
1. The email being sent is itself a reminder (`emailType === 'reminder'`) — prevents reminder-of-reminder loops
2. The invoice has no payment terms (`termsDay === NONE`) — "Due: None" invoices aren't considered outstanding

**Not suppressed (cron reminder WILL fire after 2 days if unopened) when:**
- A regular invoice email (`emailType === 'invoice'`) is sent for an invoice with terms set (Net 30, On Receipt, etc.)

There is no way for a merchant to opt out of the is-messages cron reminder for a specific invoice (other than setting terms to NONE).

### Why Different Architectures? EventBridge Scheduler vs Cron Sweep

| | EventBridge Scheduler (Abandoned Cart) | Cron Sweep (is-messages) |
|---|---|---|
| **Precision** | Exact — fires exactly 30 min after event | Up to 30 min late (worst case: event ages past threshold right after cron runs) |
| **State storage** | The scheduled rule IS the state (no DB table needed) | Relies on existing DB rows + timestamps |
| **Infra overhead** | Creates + deletes a rule per event | One Lambda, one rule, one query |
| **Scales with** | Number of concurrent scheduled reminders | Query result size (LIMIT 1000) |
| **Cleanup** | Must explicitly delete scheduled rules (`remove-scheduler` Lambda on `payment-attempted`) | Self-cleaning — `reminder_sent_dt` prevents re-send |

**Why abandoned cart uses a scheduler:** It needs to fire at a specific time relative to each individual event (30 min after *this* client viewed *this* checkout). A cron would need a new table to track checkout-viewed events and sweep them.

**Why is-messages uses a cron:** The state already exists in the `msg` table (`created_dt`, `opened_dt`, `reminder_sent_dt`). A simple sweep query finds eligible rows. The 30-min imprecision doesn't matter for a 2-day-old email.

---

## How to Simulate the Existing Reminder on Staging

### Prerequisites
- Access to staging PostgreSQL (the `msg` table)
- An Invoice Simple account with a sent invoice
- Email client that doesn't auto-preview links (or be careful not to click)

### Steps

1. **Send a test invoice** via the app (web or mobile) to a test email address

2. **Find the msg row** in PostgreSQL:
   ```sql
   SELECT id, invoice_no, to_address, created_dt, opened_dt, suppress_reminder, reminder_sent_dt, swu_email IS NOT NULL as has_template
   FROM msg
   WHERE invoice_no = 'YOUR_INVOICE_NO'
   ORDER BY created_dt DESC LIMIT 1;
   ```

3. **Backdate `created_dt`** to 3 days ago (must be >2 days, <4 days):
   ```sql
   UPDATE msg SET created_dt = now() - interval '3 days'
   WHERE invoice_no = 'YOUR_INVOICE_NO' AND reminder_sent_dt IS NULL;
   ```

4. **Ensure the row is eligible** — all must be true:
   - `suppress_reminder` = false (or null)
   - `opened_dt` = NULL
   - `pixel_fired` = NULL
   - `reminder_sent_dt` = NULL
   - `swu_email` IS NOT NULL
   - `type` = 1

5. **Wait up to 30 minutes** for the cron to fire

6. **Check `reminder_sent_dt`** was set:
   ```sql
   SELECT reminder_sent_dt, reminder_receipt_id FROM msg WHERE invoice_no = 'YOUR_INVOICE_NO';
   ```

### Common Pitfalls

- **Email client auto-opens links**: Some email clients (Apple Mail, Outlook) preview URLs in the email body, which triggers the `/v/{hashId}` endpoint and sets `opened_dt`. If this happens, reset it:
  ```sql
  UPDATE msg SET opened_dt = NULL, pixel_fired = NULL, pushed_open_dt = NULL
  WHERE invoice_no = 'YOUR_INVOICE_NO';
  ```

- **Query performance**: Don't filter by `to_address` alone — it's not indexed. Use `invoice_no` instead.

- **Invoice must be past due**: The handler checks `isInvoiceDueInFuture()` and skips if the due date hasn't passed. Use "Due: On Receipt" or backdate the invoice due date.

- **Invoice must have terms**: The handler checks `isInvoiceWithNoTerms()` and skips if terms = NONE.

- **Don't click the invoice link**: Opening the invoice via the universal link (`/v/{hashId}`) sets `opened_dt` and disqualifies it from the reminder.

### What the Reminder Email Looks Like

The reminder uses a different template than the original send:
- Template: `SWU_INVOICE_REMINDER_TEMPLATE_ID` (or experiment variants)
- Subject line is re-rendered from the original `swu_email` data
- Sent via Mailgun (same as original)
- Does NOT create a new Parse Msg — no history entry appears

---

## Open Questions

1. **Trigger condition:** Fire on exact due date? Or N days before? What time of day?
2. **Setting reuse:** Does the existing "Automatic Payment Reminders" toggle control this, or is it a new setting specific to deposits/milestones?
3. **Which invoices:** Only invoices with deposits/milestones (upcoming payments), or all invoices?
4. **Retry if Parse write fails:** Best-effort (miss the history entry) or retry?
5. **Existing reminder interaction:** If the existing 2-day reminder fires AND the new due-date reminder fires, does the client get two emails? Need deduplication logic?
6. **Abandoned cart interaction:** If the client viewed checkout but didn't pay, they may get abandoned cart emails AND the new payment-requested email. Dedup needed?
