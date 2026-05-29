# Exploration: Invoice History (Msg) — Payment Request & Auto-Pay Statuses

> How does the invoice history (Msg) system work, and how do we add Payment Request and Automatic Payment status entries to it?

**Figma references:**
- Payment Requests: https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5285-33486
- Automatic Payment statuses: https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5358-33446

---

## Current Architecture: How Msg (Invoice History) Works

### The Pattern: Direct Parse Save

Msg entries are saved directly to the Parse `Msg` class. There is no cloud function or is-services API for creation. The `beforeSaveMsg` hook only validates (checks `invoiceRemoteId` exists) — it doesn't create anything.

### Web Flow

```
User clicks "Send Invoice"
→ sendInvoiceEmailAction() (nextjs/actions/send-invoice-email.action.tsx)
→ POST /api/v2/email-invoice (to Parse Server REST API)
→ Parse creates Msg with status: SENDING
→ Email service sends the email
→ Delivery webhooks update Msg.status (DELIVERED, OPENED, etc.)
```

### Mobile Flow

```
User taps "Send"
→ emailInvoice() (features/documents/use-cases/share.ts)
→ Creates Msg LOCALLY in Realm first (status: SENDING)
→ POST /api/v3/app/email (to is-services)
→ Sync uploads Msg to Parse → MongoDB
→ Delivery webhooks update status later
```

### Status Updates (Delivery Tracking)

Separate from creation — `is-messaging` service (`GET /messaging/statuses`, `POST /messaging/swu`) receives SendWithUs/email provider webhooks and updates the Msg's status field.

---

## Key Insight

The current system only tracks **email/messaging events**. The Msg class is essentially an email delivery audit trail. Adding payment-related history entries expands its purpose to a general "invoice activity log."

---

## Proposal: New Msg Event Types for Auto-Pay

To show "Payment Request Sent", "Automatic Payment Enabled", "Payment Charged $500", etc. in the invoice history, we'd:

1. Create new Msg entries with new `eventType` values
2. Same mechanism — direct Parse save to Msg collection with `invoiceRemoteId` as the foreign key
3. Both clients pick them up automatically via existing sync (mobile) and fetch (web) patterns
4. UI changes — add icons + localized text for new event types in both `history-base-row.tsx` (web) and `invoice-history.tsx` (mobile)

### New Event Types

| Event Type | Description |
|---|---|
| `PAYMENT_REQUEST_SENT` | Merchant sent a payment request/reminder to client |
| `AUTO_PAY_ENABLED` | Client vaulted their payment method at checkout |
| `AUTO_PAY_CHARGED` | Scheduled payment was successfully charged |
| `AUTO_PAY_FAILED` | Payment failed after max retries exhausted |
| `AUTO_PAY_CANCELLED` | Merchant or client disabled automatic payments |

### Where to Trigger Creation

| Event | Where it happens | Where to create Msg |
|---|---|---|
| Payment request sent | is-services (email send) | Same place email Msg is created |
| Auto-pay enabled (vaulted) | is-stripe / is-payments | After vault confirmation |
| Auto-pay charged | is-payments Lambda (handle-success) | After successful charge |
| Auto-pay failed | is-payments Lambda (handle-failure) | After max retries |
| Auto-pay cancelled | is-payments API (disable) | After disable confirmed |

---

## Technical Considerations

### Parse Write from is-payments (Backend → Parse)

Today, Msg entries are created from:
- Web: via Parse REST API from Next.js
- Mobile: locally in Realm, then synced to Parse

For auto-pay events (enabled, charged, failed, cancelled), the source is **is-payments** (Lambda or API). is-payments would need to write to Parse — options:

1. **Parse Cloud Function call** (`Parse.Cloud.run('...')`) from is-payments Lambda/handler — preferred; handles permissions and formatting correctly. Same pattern as is-stripe/is-paypal use for `invoiceAddPayment`.
2. **Direct Parse REST API call** from is-payments — simpler but bypasses cloud function validation/formatting; not recommended per Juan Angel (backend systems should read via Mongo directly, write via cloud functions).
3. **SQS message to a handler that writes Parse** — decoupled, resilient to Parse downtime
4. **is-services endpoint that writes Parse** — adds a hop but keeps Parse writes in one place

**Recommendation:** Option 1 (Parse Cloud Function). A new cloud function (e.g. `msgAddAutoPayEvent`) would be created in is-parse-server to handle Msg creation with proper permissions and formatting. is-payments already initializes Parse with masterKey, so `Parse.Cloud.run()` works today. Precedent: is-stripe and is-paypal both write to Parse this way.

### Msg Schema Additions

Current Msg fields (relevant):
- `invoiceRemoteId` — links to invoice
- `status` — delivery status (SENDING, DELIVERED, OPENED, etc.)
- `type` — message type (email, sms, etc.)

New fields needed:
- `eventType` — discriminator for the new event types (or overload `type`)
- `metadata` — flexible object for event-specific data (e.g., `{ amountCents: 50000, milestoneId: "abc123" }`)

### UI Changes

**Web** (`is-web-app/nextjs/.../history-base-row.tsx`):
- New icon mapping for each eventType
- New copy/localization strings
- Amount display for charge events

**Mobile** (`is-mobile/src/.../invoice-history.tsx`):
- Same icon + copy additions
- Realm schema update to handle new eventType values

### Existing Clients Pick Up Automatically

Both web and mobile already fetch/sync ALL Msg entries for an invoice. New event types will appear in the list automatically — the only work is rendering them with appropriate icons and text (currently unrecognized types would be ignored or show a default).

---

## Open Questions

1. **Should "Payment Request Sent" be a new Msg eventType, or is it just a regular email Msg?** — Today, sending a payment reminder email already creates a Msg. Does the design want a *separate* "Payment Request" entry distinct from the email delivery entry?

2. **Msg metadata structure** — What data should be stored on each event? At minimum: `amountCents`, `milestoneId`, `paymentMethodLast4`. But how much is needed for the UI?

3. **Timing of "Auto-pay enabled" entry** — Created when vault is confirmed (is-stripe webhook), or when the first schedule is created (is-payments)?

4. **Retry events** — Should individual retry attempts show in history, or only the final outcome (success/failure after all retries)?

5. **Parse write permissions** — Does is-payments have Parse credentials today? (RP's handle-success already writes to Parse via `addPaymentToDocument`, so likely yes.)

---

## Relationship to Other Decisions

- **Decision 15 (successive scheduling):** "Auto-pay charged" Msg is created in `handle-success` — same place that chains to the next schedule
- **Decision 14 (only next payment locked):** "Skip this payment" action should also create a Msg entry
- **Decision 8 (overdue payments):** If overdue milestones are charged immediately at vault time, multiple "Auto-pay charged" entries appear in quick succession — acceptable?
- **Decision 12 (reminder emails):** The reminder email itself already creates a Msg via the email path. The "Payment Request" eventType may be separate from or replace the email-delivery Msg.
