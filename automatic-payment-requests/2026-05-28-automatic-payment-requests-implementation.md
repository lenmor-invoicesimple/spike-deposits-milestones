# Automatic Payment Requests — Implementation Plan (2026-05-28)

> Context: Deposits & Milestones epic. When a merchant enables "Automatic Payment Requests," the system sends a reminder email to the client on the invoice's due date. This is distinct from the existing "Outstanding Checkout Reminders" (formerly "Automatic Payment Reminders") which fires after checkout abandonment.
>
> Related:
> - [Invoice dueDate ↔ Milestone Sync (exploration)](../invoice-duedate-milestone-sync/2026-05-21-invoice-duedate-milestone-sync-exploration.md)
> - [Invoice dueDate ↔ Milestone Sync (implementation)](../invoice-duedate-milestone-sync/2026-05-26-invoice-duedate-milestone-sync-implementation.md)
> - [Existing reminder flow exploration](./exploration-automatic-payment-reminders-flow.md)
> - [Cost & Scale Analysis](./apr-cost-analysis.md) — Design A vs B vs C, breakpoints up to 1M invoices, monthly cost estimates
>
> Diagrams:
> - [Before (current systems)](./diagrams/apr-before.png) | [source](./diagrams/apr-before.mmd)
> - [After (with APR)](./diagrams/apr-after.png) | [source](./diagrams/apr-after.mmd)

---

## Summary

A daily cron job queries for invoices where `dueDate = today`, checks the merchant's `AutomaticPaymentRequests` setting, sends a "payment due" email to the client, and logs a "Payment Requested" entry in invoice history.

---

## Two Settings — Clarification

The existing "Automatic Payment Reminders" toggle is being split into two distinct settings:

| Toggle (UI label) | Setting key (Parse `Setting.remoteId`) | What it controls | Architecture |
|---|---|---|---|
| **Automatic Payment Requests** (NEW) | `AutomaticPaymentRequests` | New due-date cron — sends email on `Invoice.dueDate` | Daily cron sweep |
| **Outstanding Checkout Reminders** (RENAMED) | `PaymentReminders` (unchanged) | Existing abandoned cart — 30min + 3 days after checkout view | Event-driven EventBridge scheduler |

- The existing `PaymentReminders` key stays unchanged in backend code — only the UI label changes
- No backend changes needed for abandoned cart / Outstanding Checkout Reminders
- The new `AutomaticPaymentRequests` setting follows the same storage pattern

---

## Why a Cron (Not EventBridge Scheduler)

**Decision:** Daily cron sweep, similar to the existing is-messages reminder.

**Eng. team reasoning:** Using EventBridge would require creating/updating a schedule every time milestones or deposits change on the frontend — overkill when the state already exists on the Invoice object.

**Why it works now:** `Invoice.dueDate` is already persisted and kept in sync with the soonest unpaid milestone via `recalculateInvoiceDueDate` (implemented in is-parse-server). The cron just needs to sweep for `dueDate = today`.

| Concern | EventBridge Scheduler | Daily Cron |
|---|---|---|
| **Complexity** | Create/update/delete a rule per milestone change | One Lambda, one rule, one query |
| **State management** | Schedules must stay in sync with Invoice.dueDate | No sync — reads Invoice.dueDate directly |
| **Precision needed?** | No — "day of" is sufficient | Yes — daily sweep handles this |
| **Infra cleanup** | Must delete rules on payment/edit/delete | Self-cleaning (query condition handles it) |
| **Scale** | N active rules (one per invoice with milestones) | One query, bounded by LIMIT |

---

## High-Level Flow (SQS Fan-Out — Default Architecture)

```
┌─────────────────────────────────────────────────────────────────────┐
│  EventBridge Rule: cron(0 14 * * ? *)   ← daily at 14:00 UTC       │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Lambda 1: Sweep + Send (runs once/day)                             │
│                                                                     │
│  1. Query invoices: dueDate = today AND balanceDue > 0              │
│  2. Check setting: AutomaticPaymentRequests enabled for account     │
│  3. Filter: not already reminded (paymentRequestSentDate: null)     │
│  4. Batch fetch merchant info (User collection)                     │
│  5. For each eligible invoice:                                      │
│     a. Insert PG msg record (for pixel tracking)                    │
│     b. Send email with pixel_url (queueSWUJob)                      │
│  6. Batch write paymentRequestSentDate (direct Mongo updateMany)    │
│  7. Publish invoiceIds to SQS                                       │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ SQS message per invoice
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Lambda 2: Parse Writer (worker, up to 50 concurrent)               │
│                                                                     │
│  - Triggered per SQS message                                        │
│  - Creates Parse Msg object ("Payment Requested" in history)        │
│  - Best-effort: retries 3×, then DLQ                                │
└─────────────────────────────────────────────────────────────────────┘
```

> **Why SQS fan-out as default (not phased):** PG writes for pixel tracking + Parse writes for history + email sends make the per-invoice cost ~380ms sequential. At just 2,500 invoices this hits Lambda's 15-min limit. SQS fan-out removes the scaling ceiling from day one and isolates failures (email + idempotency in Lambda 1; history in Lambda 2).

<details>
<summary><strong>Simplified alternative: Single Lambda (if <500 invoices/day guaranteed)</strong></summary>

If scale is guaranteed small, a single Lambda doing everything sequentially is simpler:
```
Lambda: sweep → PG insert → send email → write paymentRequestSentDate → create Parse Msg
```
Trade-off: hits 15-min timeout at ~2,500 invoices/day, history write failure blocks the loop, no retry isolation. Only appropriate if product confirms <500 matched invoices/day for the foreseeable future.

</details>

---

## Setting Lookup Strategy

### Guidance from Eng. Team (Parse Interaction Pattern)

> "We have a bad practice in general with Parse... we should not be using the Parse SDK to read from backend systems. We should be using Mongo directly. For writing I think it makes sense to use SDK with Cloud Functions — writing is different because of permissions and formatting."

| Operation | Method | Package | Why |
|---|---|---|---|
| **Read** invoices due today | MongoDB via `@is/mongo` | `packages/data-sources/is-mongo` | Typed collection accessors, handles Parse pointer transforms. Same pattern as is-checkout, is-document, etc. |
| **Read** account settings | MongoDB via `@is/mongo` (or `@is/account`) | `packages/data-sources/is-mongo` | Batch `$in` query on Setting collection for all accountIds. `@is/account` is a domain wrapper over `@is/mongo` but designed for single-account lookups; for batch, go direct to `@is/mongo`. |
| **Write** invoice history entry | Parse Cloud Function | Parse SDK (`masterKey`) | Permissions, hooks, formatting — Cloud Functions are the correct write path |
| **Write** idempotency field | TBD — see options below | — | Needs decision |

### MongoDB Access Patterns in is-services (for reference)

| Pattern | Package | Used by |
|---|---|---|
| `@is/mongo` — typed collection accessors | `packages/data-sources/is-mongo` | Most services (checkout, edge, document, financing) |
| `@is/account` — domain methods wrapping `@is/mongo` | `packages/domain-model/is-account` | abandoned-cart, fundbox, acorn-webhook |
| Raw `MongoClient` | Direct `mongodb` driver | is-bookkeeping only (special case) |

### Decision: Reads via `@is/mongo` (same as abandoned-cart)

- **Settings:** `@is/mongo` Setting collection with batch `$in` on accountIds (or `@is/account` for single lookups — it's the same MongoDB underneath, just a domain wrapper)
- **Invoices:** `@is/mongo` exposes an `Invoice` collection with typed accessors and `getCollectionItems()`. Batch query with filter `{ dueDate: today, balanceDue: { $gt: 0 } }`
- **No Parse SDK for reads** — follows eng. team's guidance AND matches the existing abandoned-cart pattern

### Parse Write Patterns — Why Parse Cloud Functions for Writes

Parse Server has **lifecycle hooks** (`beforeSave`, `afterSave`) on every collection. For Invoice specifically (`is-parse-server/cloud/collections/invoice/invoiceHooks.ts`), `beforeSaveInvoice` runs on every save and handles:
- Payment sanitization and syncing to Payment collection
- Company field normalization (iOS vs Android vs web)
- Deposit creation/update/deletion
- Platform field assignment
- RemoteId uniqueness validation (non-prod)

Additionally, Parse enforces **Class Level Permissions (CLP)** on reads/writes.

| Write method | Hooks fire? | CLPs checked? | When to use |
|---|---|---|---|
| **Parse SDK with `masterKey`** | Yes (skips `!isMasterKey` block) | No (masterKey bypasses) | Backend service writes — hooks fire for data integrity, CLPs skipped because we're trusted |
| **Direct Mongo write** | No | No | Avoid — bypasses all Parse safety. Only is-bookkeeping does this (special case) |
| **Parse SDK without masterKey** | Yes (all code runs) | Yes | Client app writes (iOS/Android/web) |

**For APR's `paymentRequestSentDate` write:** The Invoice hooks don't care about this field (they handle payments, company, deposits). But using Parse Cloud Function with masterKey is still preferred because:
1. Follows the established "Parse for writes" guidance
2. Doesn't set a precedent of direct Mongo writes from is-services
3. If hooks are added to Invoice later that care about this field, they'll fire automatically
4. N Cloud Function calls (50-500/day) is acceptable performance cost

### Idempotency Write — Decision

The `paymentRequestSentDate` field needs to be written after email is sent.

**Decision: Parse SDK with `masterKey`** (same as is-recurring-invoices)

```typescript
// After sending email for each invoice:
const invoice = await new Parse.Query('Invoice').get(invoiceId, { useMasterKey: true });
invoice.set('paymentRequestSentDate', new Date());
await invoice.save(null, { useMasterKey: true });
```

**Why `masterKey`:**
- `masterKey` is NOT "skip hooks" — hooks still fire, but the heavy client-specific logic (payment sync, deposits) is skipped via `if (!isMasterKey)` block
- It IS "I'm a trusted backend service" — bypasses CLPs, runs essential validation only
- Appropriate for system-initiated writes (not user-initiated)

**Precedent — other system flows using `masterKey` writes:**

| Service | What it writes | Method |
|---|---|---|
| **is-recurring-invoices** | Invoice + RecurringInvoiceSeries | `object.save(null, { useMasterKey: true })` — **direct precedent** |
| **is-bookkeeping** | Invoice payments | `Parse.Cloud.run('invoiceAddPayment', { useMasterKey: true })` |
| **abandoned-cart** | Nothing — no write-back after sending email | N/A |

**Why not the alternatives:**

| Option | Why rejected |
|---|---|
| **DynamoDB tracking table** | Overkill for a simple date field. Abandoned-cart uses DynamoDB because it tracks schedule state across 3 Lambdas — APR just needs one field on the swept record. |
| **Direct Mongo write** | Bypasses ALL hooks and CLPs. Sets bad precedent. Only is-bookkeeping does this (special case). |
| **Dedicated Parse Cloud Function** | Extra indirection — is-recurring-invoices proves `object.save({ useMasterKey: true })` is the accepted pattern for this exact use case. |

### Considered Alternatives (Read Strategy)

**Option B (Per-Invoice via Parse SDK):** Simple code (3 lines per invoice), but N queries hits Parse rate limits at scale. 500 invoices = 500 Parse queries = ~25s + rate limit risk. Rejected because of eng. team's guidance to avoid Parse SDK for backend reads.

**Hybrid (Chunked batches):** Unnecessary with `@is/mongo` — Mongo handles large `$in` arrays natively without the pagination constraints that Parse SDK imposes.

### Final Flow

```
═══ Lambda 1: Sweep + Send (triggered by EventBridge cron) ═══

  → Step 1: @is/mongo Invoice collection:
      find({ dueDate: today, balanceDue: { $gt: 0 }, paymentRequestSentDate: null })
      → returns N invoices

  → Step 2: Extract unique accountIds from results:
      const accountIds = [...new Set(invoices.map(inv => inv.accountId))]

  → Step 3: @is/mongo Setting collection (batch lookup):
      find({ accountId: { $in: accountIds }, remoteId: 'AutomaticPaymentRequests', valBool: true })
      → returns which accounts have setting enabled

  → Step 4: In-memory filter:
      filter invoices to only those whose accountId is in the enabled set

  → Step 5: Batch fetch merchant user info:
      @is/mongo User collection: find({ accountId: { $in: eligibleAccountIds } })
      → for email template data (company name, sender name)

  → Step 6: For each eligible invoice:
      → 6a. Insert PG msg record (insertMsg → gets short_id for pixel tracking)
      → 6b. Generate pixel_url from hashids.encode(short_id)
      → 6c. Send email with pixel_url (queueSWUJob from @is/messaging)

  → Step 7: Batch write idempotency (direct Mongo, not Parse):
      invoiceCollection.updateMany(
        { _id: { $in: sentInvoiceIds } },
        { $set: { paymentRequestSentDate: new Date() } }
      )

  → Step 8: Publish to SQS for history writes:
      SendMessageBatch → 1 message per invoice (invoiceId, accountId, sentAt)

═══ Lambda 2: Parse Writer (triggered per SQS message) ═══

  → Create Parse Msg object:
      eventType: 'payment_request_sent', caption: 'Payment Requested'
      → appears in Invoice History UI
      → retries 3× via SQS, then DLQ
```

### Lambda 1: Sweep + Send (Reference Implementation)

```typescript
import { v4 as uuid } from 'uuid';
import { Collections } from '@is/mongo';
import { queueSWUJob } from '@is/messaging';
import { insertMsg } from '@is/pg-msg';
import { SQSClient, SendMessageBatchCommand } from '@aws-sdk/client-sqs';
import Hashids from 'hashids';
import * as Sentry from '@sentry/node';

const sqs = new SQSClient({});
const hashids = new Hashids(process.env.HASHID_SALT!);

export const handler = async () => {
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  const tomorrow = new Date(today);
  tomorrow.setDate(tomorrow.getDate() + 1);

  // Step 1: Sweep invoices due today
  const invoiceCollection = await Collections.getInvoiceCollection();
  const invoices = await Collections.getCollectionItems<Collections.Invoice>({
    collection: invoiceCollection,
    filter: {
      dueDate: { $gte: today, $lt: tomorrow },
      balanceDue: { $gt: 0 },
      paymentRequestSentDate: null,
    },
  });

  if (!invoices.length) return;

  // Step 2: Extract unique accountIds
  const accountIds = [...new Set(invoices.map(inv => inv.accountId))];

  // Step 3: Batch settings lookup
  const settingCollection = await Collections.getSettingsCollection();
  const settings = await Collections.getCollectionItems<Collections.Setting>({
    collection: settingCollection,
    filter: {
      ...Collections.getAccountIdFilter(accountIds),
      remoteId: 'AutomaticPaymentRequests',
      valBool: true,
    },
  });
  const enabledAccountIds = new Set(settings.map(s => s.accountId));

  // Step 4: Filter to eligible invoices
  const eligible = invoices.filter(inv => enabledAccountIds.has(inv.accountId));
  if (!eligible.length) return;

  // Step 5: Batch fetch merchant user info (for email template)
  const eligibleAccountIdList = [...new Set(eligible.map(inv => inv.accountId))];
  const userCollection = await Collections.getUserCollection();
  const users = await Collections.getCollectionItems<Collections.User>({
    collection: userCollection,
    filter: { ...Collections.getAccountIdFilter(eligibleAccountIdList) },
  });
  const usersByAccountId = new Map(users.map(u => [u.accountId, u]));

  // Step 6: For each eligible invoice — PG insert + send email
  const sentInvoices: typeof eligible = [];

  for (const invoice of eligible) {
    const merchant = usersByAccountId.get(invoice.accountId);

    try {
      // 6a. Insert PG msg record (for pixel tracking / "Opened" event)
      const pgMsg = await insertMsg({
        parse_account_id: invoice.accountId,
        invoice_no: invoice.invoiceNo,
        invoice_remote_id: invoice.remoteId,
        to_address: invoice.client.email,
        url: `${process.env.CHECKOUT_BASE_URL}/v/${invoice.hashId}`,
      });

      // 6b. Generate pixel URL
      const hashId = hashids.encode(pgMsg.short_id);
      const pixelUrl = `${process.env.BASE_URL}/px/${hashId}`;

      // 6c. Send email with pixel embedded
      await queueSWUJob({
        recipient: { address: invoice.client.email, name: invoice.client.name },
        sender: { address: 'noreply@invoicesimple.com', name: merchant?.name ?? '' },
        templateId: process.env.APR_EMAIL_TEMPLATE_ID!,
        renderData: {
          client_name: invoice.client.name,
          amount: invoice.balanceDue,
          company_name: merchant?.companyName ?? '',
          pay_url: `${process.env.CHECKOUT_BASE_URL}/v/${invoice.hashId}`,
          pixel_url: pixelUrl,
          invoice_no: invoice.invoiceNo,
          due_date: invoice.dueDate,
        },
      });

      sentInvoices.push(invoice);
    } catch (error) {
      Sentry.captureException(error);
      console.error(`Failed to send for invoice ${invoice.remoteId}`, error);
    }
  }

  if (!sentInvoices.length) return;

  // Step 7: Batch write idempotency field (direct Mongo — no hooks needed)
  const sentIds = sentInvoices.map(inv => inv._id);
  await invoiceCollection.updateMany(
    { _id: { $in: sentIds } },
    { $set: { paymentRequestSentDate: new Date() } }
  );

  // Step 8: Publish to SQS for Parse history writes (Lambda 2)
  const batchSize = 10; // SQS max per SendMessageBatch
  for (let i = 0; i < sentInvoices.length; i += batchSize) {
    const batch = sentInvoices.slice(i, i + batchSize);
    await sqs.send(new SendMessageBatchCommand({
      QueueUrl: process.env.APR_PARSE_WRITER_QUEUE_URL!,
      Entries: batch.map((inv, idx) => ({
        Id: String(idx),
        MessageBody: JSON.stringify({
          invoiceId: inv._id,
          invoiceRemoteId: inv.remoteId,
          accountId: inv.accountId,
          sentAt: new Date().toISOString(),
        }),
      })),
    }));
  }
};
```

### Lambda 2: Parse Writer (Reference Implementation)

```typescript
import Parse from 'parse/node';
import { SQSHandler } from 'aws-lambda';

Parse.initialize(process.env.PARSE_APP_ID!, undefined, process.env.PARSE_MASTER_KEY!);
Parse.serverURL = process.env.PARSE_SERVER_URL!;

type Payload = {
  invoiceId: string;
  invoiceRemoteId: string;
  accountId: string;
  sentAt: string;
};

export const handler: SQSHandler = async (event) => {
  for (const record of event.Records) {
    const payload: Payload = JSON.parse(record.body);

    const msg = new Parse.Object('Msg');
    msg.set('remoteId', crypto.randomUUID());
    msg.set('invoiceRemoteId', payload.invoiceRemoteId);
    msg.set('account', Parse.Object.createWithoutData('Account', payload.accountId));
    msg.set('sendTime', Date.now());
    msg.set('eventType', 'payment_request_sent');
    msg.set('caption', 'Payment Requested');
    msg.set('emailType', 'payment_request');
    msg.set('from', 'system');
    msg.set('deleted', false);
    msg.set('timestamp', Math.trunc(Date.now() / 1000));
    await msg.save(null, { useMasterKey: true });
  }
};
```

### Failure Handling — Order of Operations

**Lambda 1 order (per invoice):**
```
6a. Insert PG msg       ← required for pixel tracking
6b. Send email          ← side effect (can't undo)
   (collect into sentInvoices[])
7.  Batch updateMany    ← idempotency (prevents re-send)
8.  Publish to SQS      ← triggers Lambda 2 for history
```

**Lambda 2 (per SQS message):**
```
Create Parse Msg        ← "Payment Requested" in history
   (retries 3× via SQS visibility timeout, then DLQ)
```

| Failure | Consequence | Acceptable? |
|---|---|---|
| Lambda 1: PG insert fails | No pixel URL → email not sent for that invoice. Skips to next. | Yes — safe, no side effects |
| Lambda 1: Email fails | Invoice not added to sentInvoices → idempotency not written → retries next day | Yes — safe |
| Lambda 1: updateMany fails | Emails sent but idempotency not written → next cron may resend | Rare — updateMany is atomic. Small double-email risk. |
| Lambda 1: SQS publish fails | Emails sent ✓, idempotency written ✓, no history entry | Yes — history is best-effort |
| Lambda 2: Parse write fails | SQS retries 3× → DLQ. History missing but email + idempotency intact | Yes — alert on DLQ |

### History Entry — Parse Msg Object

The manual "Send Email" flow in is-api creates two separate records:

| Record | Database | Created by | Purpose |
|---|---|---|---|
| Parse `Msg` object | MongoDB (Msg collection) | `createParseMsg()` in is-api | Invoice activity timeline (merchant-visible: "Invoice sent", "Payment requested") |
| PostgreSQL `msg` row | PostgreSQL | `insertMsg()` in is-api | Email lifecycle tracker (is-messages cron uses this for reminder eligibility) |

APR needs **both** records:
- **Parse Msg object** → "Payment Requested" entry in Invoice History UI
- **PostgreSQL `msg` row** → provides `short_id` for pixel tracking URL → enables "Opened" event automatically

**Why PG msg is needed:** The "Opened" event in Invoice History (see Seth's Figma) is triggered by an email tracking pixel. The pixel URL contains a `hashId` encoded from the PG msg's `short_id`. Without the PG record, there's no hashId, no pixel URL, no "Opened" tracking.

**Pattern borrowed from:** Manual "Send Invoice" flow in `is-api` (`api/src/services/email.ts`). Note: abandoned-cart (flow 2) does NOT create PG msg records — its emails have no "Opened" tracking.

```typescript
// 1. Create PG msg record (for pixel tracking)
const pgMsg = await insertMsg({
  parse_account_id: accountId,
  invoice_no: invoice.invoiceNo,
  invoice_remote_id: invoice.remoteId,
  to_address: clientEmail,
  url: checkoutUrl,
});

// 2. Generate pixel URL from short_id
const hashId = hashids.encode(pgMsg.short_id);
const pixelUrl = `${baseUrl}/px/${hashId}`;

// 3. Send email with pixel embedded
await queueSWUJob({
  recipient: { address: clientEmail },
  templateId: APR_TEMPLATE_ID,
  renderData: { pixel_url: pixelUrl, ... },
});

// 4. Create Parse Msg (Invoice History UI entry)
const msg = new Parse.Object('Msg');
msg.set('invoiceRemoteId', invoice.remoteId);
msg.set('account', { __type: 'Pointer', className: 'Account', objectId: accountId });
msg.set('eventType', 'payment_request_sent');
msg.set('caption', 'Payment Requested');
msg.set('deleted', false);
msg.save(null, { useMasterKey: true });
```

**Result in Invoice History (automatic):**
- "Payment Requested" — from step 4 (Parse Msg)
- "Opened" — from existing pixel infra, triggered when client opens email (no APR code needed)

### Performance Estimate (at 1000 invoices/day)

**Lambda 1 (Sweep + Send):**

| Step | What | Operations | Latency per op | Total time |
|---|---|---|---|---|
| 1 | Sweep invoices | 1 Mongo query (indexed, returns 1000 docs) | 20-100ms | ~50ms |
| 2 | Extract accountIds | In-memory Set | instant | ~0ms |
| 3 | Settings lookup | 1 Mongo query (`$in` 1000 IDs, indexed) | 20-100ms | ~50ms |
| 4 | Filter eligible | In-memory Set.has() × 1000 | instant | ~0ms |
| 5 | Fetch users | 1 Mongo query (`$in` ~500 IDs, indexed) | 20-100ms | ~50ms |
| 6a | PG msg insert | 1000 × `insertMsg()` (PG write for pixel tracking) | ~5-10ms each | ~10s |
| 6b | Send emails | 1000 × `queueSWUJob()` with pixel_url (HTTP POST) | ~100ms each | ~100s (1.7 min) |
| 7 | Idempotency write | 1 × `updateMany` (direct Mongo, batch all 1000) | N/A | ~50ms |
| 8 | SQS publish | 100 × `SendMessageBatch` (10 per batch) | ~20ms each | ~2s |
| | | | **Lambda 1 Total:** | **~1.9 min** |

**Lambda 2 (Parse Writer) — runs in parallel, triggered by SQS:**

| What | Operations | Latency per invocation | Wall-clock (50 concurrent workers) |
|---|---|---|---|
| Create Parse Msg | 1 × `msg.save()` per message | ~120ms | 1000 invoices → ~2.4s |

**Combined wall-clock time: ~2 min** (Lambda 1 dominates; Lambda 2 runs in parallel after)

**Benchmarked latency (staging):**
- Invoice save (simple field update): ~157ms
- Payment/Msg save (new object): ~120ms
- Invoice save (complex, updatePaymentDetails): ~283ms
- Direct Mongo `updateMany`: <100ms regardless of count

**Key improvement vs single-Lambda:** Parse writes (previously 68% of time) are now offloaded to Lambda 2. Lambda 1 bottleneck is just PG inserts + email sends (~110ms/invoice). No scaling ceiling.

**No LIMIT on sweep query:** Flow 1 uses `LIMIT 1000` because it runs every 30 min (catches overflow next run). APR runs once/day — if we LIMIT and there are >1000 due today, the overflow is missed forever. Process all in one run.

**Lambda 1 config:** 10 min timeout, 512 MB memory.
**Lambda 2 config:** 1 min timeout, 256 MB memory, maxConcurrency: 50.

| Matched invoices/day | Lambda 1 wall-clock | Lambda 2 wall-clock | Total |
|---|---|---|---|
| 100 | ~12s | <1s | ~12s ✅ |
| 1,000 | ~1.9 min | ~2.4s | ~2 min ✅ |
| 5,000 | ~9.2 min | ~12s | ~9.2 min ✅ |
| 8,000 | ~14.7 min | ~19s | ~14.7 min ⚠️ (ceiling) |
| 10,000 | ~18.3 min ❌ | ~24s | exceeds 15-min limit |

**Real production data (Periscope, 2026-06-02):** 12,600 invoices due today across all accounts. This is the total pool before APR filtering (opt-in setting, valid email, not already sent). At realistic early adoption (5-25%), matched count is 630–3,150 — well within Lambda 1's capacity.

**Lambda 1 bottleneck at scale:** Email sending (`queueSWUJob`) is the dominant cost — ~100ms per HTTP POST to is-messages, and is 90% of Lambda 1 execution time. PG insert (~10ms each) is only 10% of per-invoice cost. At 8,000 matched invoices, Lambda 1 approaches the 15-min limit (sequential).

**If matched invoices exceed ~8k/day:** Add `p-limit(10)` for concurrent email sends — extends ceiling to ~80k matched invoices. Batch PG INSERT (`INSERT ... VALUES ... RETURNING short_id`) is an optional further optimization but not needed until 50k+ (PG is never the bottleneck before email).

**Note:** The 100ms per `queueSWUJob()` is an estimate for internal HTTP round-trip. Needs staging benchmark to confirm actual latency.

### Required Indexes

APR's sweep query on the Invoice collection needs a compound index — `dueDate` is not currently indexed (confirmed: full collection scan in Compass), and `paymentRequestSentDate` is a brand new field.

| Collection | Index | Query it supports | Status |
|---|---|---|---|
| **Invoice** | `{ dueDate: 1, balanceDue: 1, paymentRequestSentDate: 1 }` | Sweep: `{ dueDate: today, balanceDue > 0, paymentRequestSentDate: null }` | **Must create** — doesn't exist |
| **Setting** | `{ _p_account: 1, remoteId: 1 }` | Batch settings: `{ _p_account: { $in: [...] }, remoteId: "AutomaticPaymentRequests" }` | Likely exists (Parse auto-indexes pointers) — **confirm in Compass** |
| **User** | `{ _p_account: 1 }` | Batch user: `{ _p_account: { $in: [...] } }` | Likely exists (Parse auto-indexes pointers) — **confirm in Compass** |

**Action items:**
1. Create compound index on Invoice collection (can do via Compass or migration script)
2. Verify Setting and User `_p_account` indexes exist in Compass (Indexes tab)
3. Add index creation to the CDK/migration step in implementation

### Why Two Queries + In-Memory (Not `$lookup`)

MongoDB's `$lookup` (aggregation JOIN) could combine Steps 1+3 into one query, but:

| Approach | Performance | Simplicity | Available via `@is/mongo`? |
|---|---|---|---|
| **Two `find()` + in-memory filter** | Both queries use their own optimal indexes. Two round-trips in same VPC = sub-ms overhead. In-memory Set join on hundreds of items is instant. | Simple, readable code | Yes — `getCollectionItems()` supports `find()` |
| **`$lookup` aggregation** | Mongo does a nested loop join internally. If `accountId` isn't indexed in Setting, it's a full collection scan per invoice. Even with index, query planner is less efficient inside pipelines. | Complex aggregation pipeline | No — `@is/mongo` doesn't expose aggregation pipelines (only is-bookkeeping uses raw MongoClient) |

**Bottom line:** Two fast indexed queries is often *faster* than one `$lookup`, and much simpler. `$lookup` is better for complex reports with `$group`/`$unwind` — not for "get A, filter by B."

### Why This Works

| Concern | Answer |
|---|---|
| Parse rate limits on reads? | Gone — `@is/mongo` / `@is/account` bypass Parse SDK entirely |
| Large `$in` clause? | Mongo handles natively, no 1000-result cap |
| Parse SDK overhead? | Only on writes (Cloud Functions) — accepted pattern |
| Code complexity? | `@is/mongo` for invoices (1 query) + `@is/mongo` for settings (1 query with `$in`) + in-memory filter |
| Performance? | Two indexed queries + in-memory join beats `$lookup` for this pattern (see above) |
| Failure mode? | Read failure = no emails sent (safe). Write failure = email sent but no history entry (needs retry/log) |
| Precedent? | Same packages and patterns as abandoned-cart |

---

## Email Sending Pattern (Same as Abandoned Cart)

APR will use the same cross-service email sending pattern as abandoned-cart:

```
APR Lambda
  → queueSWUJob(emailJobData)                       ← shared package: @is/messaging
  → IsMessages SDK (npm: @invoice-simple/is-msg)    ← HTTP client wrapper
  → HTTP POST /email-sending                        ← calls is-messages service API
  → is-messages receives job
  → SQS → sqsHandler → renders SWU template → Mailgun sends email
```

**Key files (abandoned-cart reference):**
- `is-services/packages/services/is-abandoned-cart/src/utils/send-reminder-email.ts` — calls `queueSWUJob()`
- `is-services/packages/domain-model/is-messaging/src/swuMessage.ts` — `queueSWUJob()` definition
- `is-services/packages/domain-model/is-messaging/src/email-sending.ts` — wraps `IsMessages` SDK HTTP client

**What this means for APR:**
- No new email infra needed — import `queueSWUJob` from `@is/messaging`, pass template data
- Needs `IS_MESSAGES_API_KEY` env var (same as abandoned-cart)
- Needs a new SWU template ID (created in SendWithUs dashboard)
- The is-messages service handles rendering, queueing, and Mailgun delivery

---

## Resolved Questions

All originally open questions have been resolved through design walkthrough (2026-06-01):

| # | Item | Resolution |
|---|------|------------|
| 1 | Deduplication with existing reminders | Accept overlap for MVP. Revisit if users complain. |
| 2 | Deduplication with abandoned cart | Accept overlap. Different emails, different triggers, low probability. |
| 3 | What counts as "non-vaulted"? | Not needed Phase 1 — milestone vaulting hasn't been built yet. |
| 4 | Time of day | Fixed UTC (propose 14:00 UTC). Pending product confirmation. |
| 5 | Idempotency tracking | `paymentRequestSentDate` field on Invoice. Written via `invoice.save({ useMasterKey: true })`. |
| 6 | Multiple milestones due same day | One email per invoice. |
| 7 | Invoice fully paid before dueDate | Handled by `balanceDue > 0` in sweep query. |
| 8 | Where to host the cron | is-services — new package `is-payment-requests`. |
| 9 | Retry if Parse write fails | Best-effort history. Log to Sentry, move on. |
| 10 | Default value for new setting | **Disabled by default (opt-in).** No surprise emails on deploy. |
| 11 | Email template / CTA | New SWU template. CTA link pending product (checkout URL vs invoice view). |
| 12 | Only milestone invoices or all? | **Pending product.** Query doesn't care — `dueDate = today` regardless of source. |

## Known Edge Cases

### Milestone date edited after APR already fired

**Scenario:**
1. APR fires on June 2 for `Invoice.dueDate = June 2` → sets `paymentRequestSentDate = June 2`
2. Merchant edits the milestone, pushes due date to June 10
3. `recalculateInvoiceDueDate` runs → `Invoice.dueDate = June 10`
4. APR cron runs on June 10: query filters `paymentRequestSentDate: null` → **invoice excluded — APR silently skipped**

**Root cause:** `paymentRequestSentDate` is never cleared when `Invoice.dueDate` changes.

**Fix:** In `updatePayment.ts` (or `recalculateInvoiceDueDate`), clear `paymentRequestSentDate` whenever `Invoice.dueDate` shifts to a new future date. This lets APR re-fire on the new date.

**Scope:** Applies only when a merchant edits a milestone date after APR has already sent for that invoice. No impact if the milestone date hasn't been sent for yet.

---

## Architecture Decision: Where to Host

**Decision: is-services — new package `is-payment-requests`**

| | is-messages | is-services |
|---|---|---|
| MongoDB access (`@is/mongo`) | ❌ No | ✅ Yes |
| `@is/account` settings lookup | ❌ No | ✅ Yes |
| Parse masterKey (for writes) | ❌ Not configured | ✅ Already wired |
| Email sending | ✅ Direct (internal SQS→Mailgun) | ✅ Via `queueSWUJob()` cross-service |
| Codebase status | Legacy, team moving away | Modern, active development |
| Pattern precedent | — | abandoned-cart is the template |

**Hard blocker:** is-messages has no `@is/mongo` access. APR needs MongoDB reads for Invoice sweep and Settings lookup. This alone rules out is-messages.

**Secondary reasons:** Parse masterKey not configured in is-messages, `@is/account` doesn't exist there, and eng. team guidance requires `@is/mongo` for reads (not Parse SDK).

**Package structure:**
```
is-services/packages/services/is-payment-requests/
  src/
    lambda-handlers/
      sweep-and-send/handler.ts     ← Lambda 1
      parse-writer/handler.ts       ← Lambda 2
    utils/...
  package.json
  tsconfig.json
  jest.config.js

is-services/packages/aws/cdk/src/lib/constructs/payment-requests/
  sweep-lambda-construct.ts
  parse-writer-lambda-construct.ts
  sqs-construct.ts
  eventbridge-construct.ts
```

**CDK resources needed:**
- 1 EventBridge rule (`cron(0 14 * * ? *)` — daily at 14:00 UTC)
- 1 Lambda function: Sweep + Send (10 min timeout, 512 MB)
- 1 Lambda function: Parse Writer (1 min timeout, 256 MB, triggered by SQS)
- 1 SQS queue: `apr-parse-writer` (visibility 30s, retention 1 day)
- 1 SQS DLQ: `apr-parse-writer-dlq` (maxReceiveCount: 3)
- IAM roles (invoke Lambda, SQS send/receive)
- Environment variables:
  - Lambda 1: `MONGODB_URI`, `PG_CONNECTION_STRING`, `IS_MESSAGES_API_KEY`, `APR_EMAIL_TEMPLATE_ID`, `CHECKOUT_BASE_URL`, `BASE_URL`, `HASHID_SALT`, `APR_PARSE_WRITER_QUEUE_URL`
  - Lambda 2: `PARSE_APP_ID`, `PARSE_MASTER_KEY`, `PARSE_SERVER_URL`

**How it works:**
```
EventBridge Rule            Lambda 1 (Sweep + Send)         SQS Queue          Lambda 2 (Parse Writer)
────────────────            ───────────────────────         ─────────          ───────────────────────
cron(0 14 * * ? *)  ─────►  query + email + idempotency  ─────►  messages  ─────►  Parse Msg (history)
(daily at 14:00 UTC)         (PG + Mongo + HTTP)                               (50 concurrent workers)
```

---

## Flow Diagrams

### is-messages cron

```
EventBridge (every 30 min)
    │
    ▼
┌─────────────────────────┐
│  Lambda: reminderHandler │
└─────────────────────────┘
    │
    ▼
PostgreSQL: SELECT FROM msg
  (2-4 day old, unsent, LIMIT 1000)
    │
    ▼
For each msg:
    ├── Parse.Cloud.run('publicInvoice') → validate due date & terms
    │
    ▼
    └── SQS → Mailgun (send email)
             │
             ▼
        PostgreSQL: UPDATE msg SET reminder_sent_dt = now()
```

### abandoned-cart

```
Customer views checkout
    │
    ▼
EventBridge event: "checkout-viewed"
    │
    ▼
┌────────────────────────────────┐
│  Lambda 1: Schedule Reminder    │
└────────────────────────────────┘
    │
    ├── DynamoDB: check if schedule exists
    ├── Create EventBridge Scheduler (30 min)
    └── Create EventBridge Scheduler (3 days)
         │
         ▼ (fires at scheduled time)
┌────────────────────────────────┐
│  Lambda 2: Send Reminder        │
└────────────────────────────────┘
    │
    ├── Parse.Cloud.run('publicInvoice')      → Invoice
    ├── Parse.Cloud.run('invoiceGetPayments') → Payments
    ├── @is/account → MongoDB Setting (opt-out check)
    ├── Filter: due date, terms, payable?
    │
    ▼
    └── SWU email via @is/messaging


Customer pays
    │
    ▼
EventBridge event: "payment-attempted"
    │
    ▼
┌────────────────────────────────┐
│  Lambda 3: Remove Schedulers    │
└────────────────────────────────┘
    │
    └── DynamoDB: find & delete both scheduler rules
```

### APR (new — SQS fan-out)

```
EventBridge (once/day, 14:00 UTC)
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  Lambda 1: Sweep + Send                               │
└──────────────────────────────────────────────────────┘
    │
    ├── @is/mongo Invoice: find({ dueDate: today, balanceDue > 0, paymentRequestSentDate: null })
    ├── @is/mongo Setting: find({ accountId: {$in}, key: 'AutomaticPaymentRequests' })
    ├── @is/mongo User: find({ accountId: {$in} })  → merchant email/name
    │
    ▼
For each eligible invoice:
    ├── PG insertMsg()               → gets short_id for pixel
    ├── hashids.encode(short_id)     → pixel_url
    └── queueSWUJob({ pixel_url })   → email with tracking pixel
    │
    ▼
Batch operations:
    ├── Mongo updateMany({ paymentRequestSentDate })  → idempotency (instant)
    └── SQS SendMessageBatch                          → triggers Lambda 2
         │
         │ 1 message per invoice
         ▼
┌──────────────────────────────────────────────────────┐
│  Lambda 2: Parse Writer (50 concurrent workers)       │
└──────────────────────────────────────────────────────┘
    │
    └── Parse SDK: create Msg object  → "Payment Requested" in history
        (retries 3× → DLQ on failure)
```

---

## Comparison: is-messages cron vs abandoned-cart vs APR

| Aspect | is-messages cron | abandoned-cart | APR (new) |
|--------|-----------------|----------------|-----------|
| **Trigger** | EventBridge cron every 30 min | Event-driven (`checkout-viewed`) | EventBridge cron once/day |
| **Sweep query** | PostgreSQL `msg` table (2-4 day old unsent msgs, LIMIT 1000) | None — event-driven per invoice | MongoDB Invoice collection (due today, balanceDue > 0) |
| **Invoice fetch** | `Parse.Cloud.run('publicInvoice')` per msg | `Parse.Cloud.run('publicInvoice')` per invoice | MongoDB direct via `@is/mongo` (batch) |
| **Settings fetch** | None (no opt-out check) | `@is/account` → MongoDB Setting | `@is/mongo` Setting collection (batch `$in`) — or `@is/account` for single lookups |
| **Payments fetch** | None | `Parse.Cloud.run('invoiceGetPayments')` | None (balanceDue already on Invoice doc) |
| **Write** | PostgreSQL update (`reminder_sent_dt`) | None | PG `insertMsg()` (pixel) + Mongo `updateMany` (idempotency) + Parse Msg via SQS (history) |
| **Email** | SQS → Mailgun | SWU via `@is/messaging` | SWU via `@is/messaging` |
| **Idempotency** | `reminder_sent_dt IS NULL` filter | DynamoDB schedule-exists check | `paymentRequestSentDate` field on Invoice |
| **Lambdas** | 1 | 3 | 2 (sweep+send + Parse writer via SQS) |
| **Processing** | Sequential per msg | 1 invoice per invocation | Lambda 1: sequential sweep+send; Lambda 2: 50 concurrent Parse writes |

---

## Remaining Pending Decisions (for Product/Seth)

| # | Item | Options | Impact on code |
|---|------|---------|----------------|
| 1 | Scope: milestone-only or any invoice with dueDate? | Any invoice (broader) vs milestone-only (tighter) | Adds/removes a filter condition in sweep query |
| 2 | Time of day (exact UTC hour) | 14:00 UTC proposed (~9 AM Eastern) | CDK cron expression |
| 3 | Email CTA: checkout URL vs invoice view URL? | Checkout (direct pay) vs view (see invoice first) | String in `renderData.pay_url` — no logic change |
| 4 | Email template content/design | New SWU template needed | Created in SendWithUs dashboard |

---

## Decisions Log

### 1. Trigger type — daily cron sweep

**Decision:** Daily EventBridge cron → single Lambda. Sweep for `dueDate = today`.
- **Why not EventBridge Scheduler (like abandoned-cart):** Would require creating/updating/deleting a schedule every time milestones change. The state (`Invoice.dueDate`) already exists — just sweep for it.
- **Trade-off accepted:** Less precise (fixed UTC hour vs event-relative timing). Acceptable because "day of" granularity is enough.

### 2. Query scope — `dueDate = today` (not `<= today`)

**Decision:** Only process invoices due exactly today.
- **Why not `<= today`:** On first cron run, `<= today` would email every historically overdue unpaid invoice — spam bomb. (Decision from Seth.)
- **Trade-off accepted:** If cron is down on a specific day, those invoices are missed permanently. Acceptable risk for Phase 1.

### 3. Where to host — is-services, new package `is-payment-requests`

**Decision:** New package in is-services monorepo, separate from abandoned-cart.
- **Hard blocker on is-messages:** No `@is/mongo` access (can't read Invoice/Setting collections), no `@is/account`, no Parse masterKey configured.
- **Why not add to abandoned-cart package:** Different feature, abandoned-cart name doesn't fit, APR is much simpler (1 Lambda vs 3). Clean separation preferred.

### 4. Read strategy — two batch Mongo queries + in-memory filter

**Decision:** Step 1 sweeps Invoice collection, Step 3 batch-queries Setting collection with `$in`, Step 4 filters in-memory.
- **Why not `$lookup`:** `@is/mongo` doesn't expose aggregation pipelines. Two indexed `find()` queries are faster AND simpler than a `$lookup` join.
- **Why not per-invoice settings check (like abandoned-cart):** APR is batch. 1000 individual settings calls would be chatty. `$in` is one call.

### 5. Settings check — at send time, disabled by default

**Decision:** Check `AutomaticPaymentRequests` setting via batch `$in` lookup. Default: **disabled** (if setting not found, don't send).
- **Contrast with abandoned-cart:** `PaymentReminders` defaults to `true` (send rather than miss). APR defaults to `false` because it's a new feature — no surprise emails on deploy.
- Merchants must explicitly enable in settings UI.

### 6. Email sending — `queueSWUJob()` from `@is/messaging`

**Decision:** Same cross-service pattern as abandoned-cart. No new email infra.
- **Why not build own SQS→Mailgun:** `queueSWUJob()` already exists, proven in production. Avoid duplicating email infrastructure.
- **Trade-off:** Extra network hop (HTTP to is-messages). Acceptable — same latency abandoned-cart has daily.

### 7. Template data — live from sweep results + batch user fetch

**Decision:** Invoice data from Step 1 sweep (already in hand), merchant info from batch `@is/mongo` User query.
- **Contrast with abandoned-cart:** Abandoned-cart fetches from Parse (`Parse.Cloud.run('publicInvoice')`) because it only has an invoiceId from the event. APR already has full docs from the sweep.
- **APR has zero Parse reads.** More consistent with eng. team guidance than abandoned-cart.

### 8. Idempotency write — Parse SDK `invoice.save({ useMasterKey: true })`

**Decision:** Write `paymentRequestSentDate` on Invoice via Parse SDK with masterKey.
- **Precedent:** is-recurring-invoices uses same pattern (`object.save(null, { useMasterKey: true })`) for system-initiated Invoice writes.
- **Why masterKey:** Hooks still fire (runs essential validation), but skips `!isMasterKey` block (client-specific payment sync, deposits). Appropriate for backend service writes.
- **Why not DynamoDB:** Overkill for one date field. Abandoned-cart uses DynamoDB because it tracks schedule state across 3 Lambdas.
- **Why not direct Mongo write:** Bypasses all hooks/CLPs, sets bad precedent.

### 9. History logging + "Opened" pixel tracking

**Decision:** APR creates both a **PG `msg` row** (pixel tracking) and a **Parse Msg object** (history entry). Best-effort for Parse Msg — if it fails, log to Sentry and move on.

- **Pattern:** Same as manual "Send Invoice" in is-api (`api/src/services/email.ts`). NOT the same as abandoned-cart (which has no pixel tracking).
- **PG `msg` row (required):** Provides `short_id` → `hashId` → pixel URL embedded in email. When client opens email, existing `/pixel/:hashId` endpoint fires → creates "Opened" MsgEvent automatically.
- **Parse Msg (best-effort):** "Payment Requested" entry in Invoice History UI. If it fails, email was already sent and pixel tracking still works.
- **Implication:** APR Lambda needs a PG connection string. This also makes bidirectional dedup (Decision #10) nearly free since the PG connection already exists.

### 10. Deduplication with existing emails — accept overlap for MVP

**Decision:** Accept that clients may get APR + is-messages cron email on the same day if dueDate falls within the 2-4 day reminder window.
- Different emails, different purposes. Overlap window is small.
- Same decision for abandoned-cart overlap: accept, revisit if users complain.

**Full bidirectional dedup (future, if needed):**

For zero overlap, APR needs both **read** and **write** access to the PostgreSQL `msg` table:

| Direction | APR action | Effect |
|---|---|---|
| Prevent is-messages after APR | **Write** to PG msg table (`reminder_sent_dt`) | is-messages sees field set → skips |
| Prevent APR after is-messages | **Read** from PG msg table | APR sees recent reminder exists → skips |

**Trade-off:** ~~Adds a PG dependency to APR (currently Mongo-only).~~ APR already needs PG access for pixel tracking (Decision #9), so the dedup read is nearly free — no new infra, just an additional query.

**Product question for PCore:** Is zero overlap required (adds one PG read per invoice), or is "different email, different purpose" acceptable?

### 11. Time of day — fixed UTC, pending product

**Decision:** Fixed UTC time for Phase 1 (propose 14:00 UTC = ~9 AM Eastern). Exact time pending product confirmation.
- APR is the first system at IS that picks a specific daily hour — no precedent from Flow 1 or Flow 2.
- Timezone-aware sending is a future improvement (no timezone data on accounts today).

### 12. Scale — SQS fan-out from day one

**Decision:** No LIMIT on sweep query. SQS fan-out architecture (2 Lambdas) from the start. No phased rollout.
- **Why no LIMIT:** Runs once/day. If >1000 invoices are due and we cap at 1000, overflow is missed forever.
- **Why SQS from day one:** PG writes for pixel tracking + email sends make Lambda 1 ~110ms/invoice sequential. Parse writes offloaded to Lambda 2 (50 concurrent workers). This removes the scaling ceiling without adding meaningful complexity.
- **Lambda 1 bottleneck:** PG insert + email send (~110ms/invoice). Hits 15-min limit at ~8,000 invoices. For 10k+: batch PG inserts and parallelize email sends.
- **Lambda 2:** No bottleneck — 50 concurrent workers, each doing one ~120ms Parse write. Effectively unlimited.
- **Timeout is set in CDK construct** (infrastructure config, not application code):
  ```typescript
  // Lambda 1: Sweep + Send
  new lambda.Function(this, 'SweepAndSendHandler', {
    timeout: Duration.minutes(10),
    memorySize: 512,
  });

  // Lambda 2: Parse Writer
  new lambda.Function(this, 'ParseWriterHandler', {
    timeout: Duration.minutes(1),
    memorySize: 256,
  });
  ```

| Matched invoices/day | Lambda 1 | Lambda 2 (50 workers) | Total wall-clock |
|---|---|---|---|
| 100 | ~12s | <1s | ~12s ✅ |
| 1,000 | ~1.9 min | ~2.4s | ~2 min ✅ |
| 5,000 | ~9.2 min | ~12s | ~9 min ✅ |
| 10,000 | ~18 min ⚠️ | ~24s | needs Lambda 1 parallelization |

See [`apr-scaling-parse-writes.md`](apr-scaling-parse-writes.md) for detailed SQS queue config, DLQ setup, and Lambda 2 handler.

### 13. CDK structure — new package `is-payment-requests`

**Decision:** Own package and CDK construct, separate from abandoned-cart.
- 1 EventBridge rule + 2 Lambdas + 1 SQS queue + 1 DLQ. Simpler than abandoned-cart's 3 Lambdas + DynamoDB + Scheduler.
- Clean ownership, independent deployment and scaling.
- Lambda 2 (Parse Writer) is reusable if other features need async Parse writes in the future.

### 14. Multiple milestones due same day

**Decision:** One email per invoice. APR targets the Invoice (not individual milestones). `Invoice.dueDate = today` → one email regardless.

### 15. Vaulted check (non-vaulted = eligible for APR)

**Decision:** Not needed Phase 1. Milestone auto-pay hasn't been built yet — all milestone invoices are effectively "non-vaulted."
- Add in Phase 3 when milestone vaulting ships: query is-payments `recurring_payments` table for ACTIVE record.

### 16. Scope — which invoices get APR?

**Decision:** Pending product. Query uses `dueDate = today` regardless — doesn't inherently care how the date got there (milestones vs manual terms).

### 17. Email template / CTA

**Decision:** New SWU template. CTA link pending product (checkout URL for direct pay vs invoice view URL).

---

## Next Steps

1. Confirm remaining product decisions with Seth/design:
   - Scope: milestone-only or any invoice with dueDate?
   - Exact UTC time for cron
   - Email CTA: checkout URL vs invoice view URL?
   - Email template content/design (SWU template creation)
2. Implementation:
   - Create `is-payment-requests` package in is-services
   - Write Lambda 1 handler (sweep + PG insert + email + Mongo updateMany + SQS publish)
   - Write Lambda 2 handler (Parse Msg writer, triggered by SQS)
   - Create CDK construct (EventBridge rule + 2 Lambdas + SQS queue + DLQ)
   - Add `paymentRequestSentDate` field to Invoice schema
   - Create compound index on Invoice collection
   - Set up PG connection for Lambda 1 (same DB as is-messages `msg` table)
   - Create SWU email template in SendWithUs dashboard (must include `pixel_url` variable)
3. Wire up mobile + web UI:
   - New `AutomaticPaymentRequests` toggle in settings
   - Rename existing "Automatic Payment Reminders" → "Outstanding Checkout Reminders" (UI only)

---

## TL;DR

**Current:** No email is sent on the day an invoice is due. The existing "Automatic Payment Reminders" only fires after checkout abandonment (30min + 3 days).

**Proposed:** A daily cron sends a "payment due today" email to clients whose invoice `dueDate = today`, controlled by a new per-merchant setting (`AutomaticPaymentRequests`).

**Details:**
- Daily EventBridge cron → Lambda in is-services, new package `is-payment-requests`
- Reads invoices via `@is/mongo` — sweep query: `{ dueDate: today, balanceDue > 0, paymentRequestSentDate: null }`
- Reads settings via `@is/mongo` Setting collection — batch `$in` on accountIds for `AutomaticPaymentRequests`
- Reads merchant info via `@is/mongo` User collection — batch `$in` for email template data
- No Parse SDK for reads (per eng. team's guidance) — all reads are direct MongoDB
- Writes idempotency field via Parse SDK with masterKey (`invoice.save()` — same as is-recurring-invoices)
- Writes history entry via Parse Msg object — best-effort (log error, move on)
- Sends email via `queueSWUJob()` from `@is/messaging` → is-messages → SQS → Mailgun
- Uses `= today` (not `<= today`) to avoid spamming historically overdue invoices (Seth decision)
- Setting defaults to **disabled** (opt-in) — no surprise emails on deploy
- One email per invoice regardless of how many milestones are due
- Accept dedup overlap with existing reminders for MVP
- Sequential processing, no LIMIT — ~6 min at 1000 invoices/day, under Lambda 15-min max
- Existing "Automatic Payment Reminders" renamed to "Outstanding Checkout Reminders" (UI only, backend key `PaymentReminders` unchanged)

**Remaining product decisions:**
- Scope: only milestone invoices or any invoice with `dueDate`?
- Exact UTC time for the daily cron (propose 14:00 UTC)
- Email CTA: checkout URL vs invoice view URL?
- Email template content/design

**All engineering decisions resolved.** See Decisions Log above for full rationale on each.
