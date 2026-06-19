# APR Scaling — Splitting Parse Writes to a Worker Lambda

## Problem

APR's main Lambda does 3 things per invoice:
1. Send SWU email
2. Write `paymentRequestSentDate` to Invoice (idempotency)
3. Write Msg object (invoice history)

Steps 2 and 3 are Parse SDK writes (~350ms each). Sequential processing caps out at ~2,500 invoices before hitting Lambda's 15-min limit.

---

## Solution: SQS Fan-Out

Split into two Lambdas:

```
┌──────────────────────────────────────┐
│  Lambda 1: Sweep (runs once/day)      │
│                                       │
│  - Query MongoDB: invoices due today  │
│  - Filter: settings enabled          │
│  - Send SWU email per invoice        │
│  - Write paymentRequestSentDate      │
│    (direct Mongo updateMany — fast)  │
│  - Publish invoiceId to SQS          │
└──────────────┬───────────────────────┘
               │ SQS message per invoice
               ▼
┌──────────────────────────────────────┐
│  Lambda 2: Parse Writer (worker)      │
│                                       │
│  - Triggered per SQS message          │
│  - Receives: invoiceId, accountId,   │
│    sentAt timestamp                  │
│  - Creates Parse Msg object          │
│    (invoice history entry)           │
└──────────────────────────────────────┘
```

---

## Why Split Here

| Operation | Lambda 1 (Sweep) | Lambda 2 (Worker) | Why |
|---|---|---|---|
| MongoDB invoice sweep | ✅ | — | Batch read, fast |
| Settings filter | ✅ | — | Batch read, fast |
| SWU email | ✅ | — | Async queue, no wait |
| `paymentRequestSentDate` | ✅ (direct Mongo `updateMany`) | — | No hooks care about this field, instant at any scale |
| Parse Msg object | — | ✅ | Needs hooks, slow (~350ms) — offloaded to worker |

**Key insight:** `paymentRequestSentDate` stays in Lambda 1 as a direct MongoDB write — it's a new field with zero hook logic. Only the Msg history write (which needs Parse hooks) goes to the worker.

---

## Lambda 2 — SQS Trigger Config

```typescript
// CDK construct
const parseWriterLambda = new lambda.Function(this, 'ParseWriterHandler', {
  timeout: Duration.minutes(1),   // each invocation is just 1 Parse write
  memorySize: 256,
  // ...
});

parseWriterLambda.addEventSource(new SqsEventSource(aprQueue, {
  batchSize: 1,          // 1 message per invocation (simple, predictable)
  maxConcurrency: 50,    // up to 50 parallel Lambda invocations
}));
```

With `maxConcurrency: 50` and ~350ms per write:
- 1,000 invoices → ~7 sec wall-clock
- 10,000 invoices → ~70 sec wall-clock
- 1,000,000 invoices → ~2 hours wall-clock (AWS scales workers automatically)

---

## Lambda 2 — Handler

```typescript
// packages/services/is-payment-requests/src/lambda-handlers/parse-writer/handler.ts

import Parse from 'parse/node';
import { SQSHandler } from 'aws-lambda';

type ParseWriterPayload = {
  invoiceId: string;
  invoiceRemoteId: string;
  accountId: string;
  sentAt: string; // ISO timestamp
};

export const handler: SQSHandler = async (event) => {
  Parse.initialize(process.env.PARSE_APP_ID!, undefined, process.env.PARSE_MASTER_KEY!);
  Parse.serverURL = process.env.PARSE_SERVER_URL!;

  for (const record of event.Records) {
    const payload: ParseWriterPayload = JSON.parse(record.body);

    // Create Msg object for invoice history
    const Msg = Parse.Object.extend('Msg');
    const msg = new Msg();
    msg.set('account', { __type: 'Pointer', className: 'Account', objectId: payload.accountId });
    msg.set('invoiceRemoteId', payload.invoiceRemoteId);
    msg.set('eventType', 'payment_request_sent');
    msg.set('caption', 'Payment Requested');
    msg.set('deleted', false);
    msg.set('updated', new Date(payload.sentAt));

    await msg.save(null, { useMasterKey: true });
  }
};
```

---

## Lambda 1 — Changes

Lambda 1 adds two things:
1. Direct Mongo `updateMany` for `paymentRequestSentDate` (replaces Parse SDK write)
2. Publish to SQS after email is sent

```typescript
// After sending emails, batch-update idempotency field directly in Mongo
await invoiceCollection.updateMany(
  { _id: { $in: sentInvoiceObjectIds } },
  { $set: { paymentRequestSentDate: new Date() } }
);

// Publish to SQS for Parse history writes
await sqsClient.send(new SendMessageBatchCommand({
  QueueUrl: process.env.APR_PARSE_WRITER_QUEUE_URL,
  Entries: sentInvoices.map((inv, i) => ({
    Id: String(i),
    MessageBody: JSON.stringify({
      invoiceId: inv._id,
      invoiceRemoteId: inv.remoteId,
      accountId: inv.accountId,
      sentAt: new Date().toISOString(),
    }),
  })),
}));
```

---

## SQS Queue Config

```typescript
// CDK
const aprParseWriterQueue = new sqs.Queue(this, 'AprParseWriterQueue', {
  queueName: 'apr-parse-writer',
  visibilityTimeout: Duration.seconds(30),  // > Lambda 2 timeout
  retentionPeriod: Duration.days(1),        // if Lambda 2 fails, retry within 1 day
  deadLetterQueue: {
    queue: new sqs.Queue(this, 'AprParseWriterDLQ'),
    maxReceiveCount: 3,  // 3 retries before DLQ
  },
});
```

---

## Failure Handling

| Failure | Behavior |
|---|---|
| Lambda 2 fails for an invoice | SQS retries up to 3× (visibility timeout) |
| All retries fail | Message goes to DLQ — alert/monitor there |
| Lambda 1 fails mid-sweep | `paymentRequestSentDate` not set → next day's cron re-processes (idempotent) |
| Email sent but SQS publish fails | History entry missing — email still sent. Acceptable (best-effort history) |

---

## Scale Comparison

> **What scales is matched invoices × 2 writes × ~350ms.** The collection size (1M, 10M docs) is irrelevant — with the compound index, the query is always <100ms. Only the invoices that pass all filters (due today + opted in + `paymentRequestSentDate: null`) matter.

| Matched invoices/day | Sequential (Phase 1) | SQS fan-out 50 workers (Phase 3) |
|---|---|---|
| 1,000 | ~6 min ✅ | ~7 sec ✅ |
| 10,000 | ~58 min ❌ | ~70 sec ✅ |
| 100,000 | ~10 hours ❌ | ~12 min ✅ |
| 1,000,000 | ~97 hours ❌ | ~2 hours ✅ |
| 10,000,000 | ~40 days ❌ | ~20 hours ⚠️ |

**Lambda 15-min cutoff hits at ~2,500 matched invoices sequential** — that's the trigger to move to Phase 3.

---

## Recommended Solution

**SQS fan-out (Phase 3) + direct Mongo for idempotency field** is the most reliable and scalable design:

| Concern | Solution |
|---|---|
| `paymentRequestSentDate` at scale | Direct Mongo `updateMany` — batch all matched invoices in one call, milliseconds regardless of count |
| Parse Msg history at scale | SQS → Worker Lambda (50 concurrent) — AWS auto-scales workers, no Lambda timeout risk |
| Reliability | SQS retries (3×) + DLQ — no lost messages even if Parse is slow or down |
| Failure isolation | Email + idempotency written in Lambda 1. History write in Lambda 2 is best-effort — a failure there doesn't re-send emails |
| Simplicity | Lambda 1: query → email → Mongo write → publish to SQS |

**Key principle:** separate writes by their nature:
- `paymentRequestSentDate` — no hooks needed → direct Mongo, instant at any scale
- Msg object — needs Parse hooks → offload to worker, parallelized via SQS

---

## When to Implement

**Phase 1 (MVP):** Sequential Parse writes, no SQS. Sufficient for current scale (<2,500 matched invoices/day).

**Phase 3 (this design):** When daily matched count approaches ~2,000-2,500 (nearing Lambda 15-min limit). Migration is:
1. Add SQS queue + Lambda 2 in CDK
2. Change Lambda 1's Parse write → `updateMany` + SQS publish
3. Deploy

No changes to email logic, settings fetch, or query logic.
