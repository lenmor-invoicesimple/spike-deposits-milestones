# Automatic Payment Requests — Implementation Plan (2026-05-28)

> Context: Deposits & Milestones epic. When a merchant enables "Automatic Payment Requests," the system sends a reminder email to the client on the invoice's due date. This is distinct from the existing "Outstanding Checkout Reminders" (formerly "Automatic Payment Reminders") which fires after checkout abandonment.
>
> Related:
> - [Invoice dueDate ↔ Milestone Sync (exploration)](../invoice-duedate-milestone-sync/2026-05-21-invoice-duedate-milestone-sync-exploration.md)
> - [Invoice dueDate ↔ Milestone Sync (implementation)](../invoice-duedate-milestone-sync/2026-05-26-invoice-duedate-milestone-sync-implementation.md)
> - [Existing reminder flow exploration](./exploration-automatic-payment-reminders-flow.md)
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

## High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  EventBridge Rule: cron(0 14 * * ? *)   ← daily at 14:00 UTC       │
│  (or configurable — see "Time of Day" section)                      │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Lambda Handler                                                     │
│                                                                     │
│  1. Query invoices: dueDate <= today AND balanceDue > 0             │
│  2. Filter: not vaulted (no active recurring payment)               │
│  3. Check setting: AutomaticPaymentRequests enabled for account     │
│  4. Filter: not already reminded today (idempotency)                │
│  5. Send email for each qualifying invoice                          │
│  6. Log "Payment Requested" in invoice history (Parse Msg)          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Setting Lookup Strategy

### Guidance from Eng. Team (Parse Interaction Pattern)

> "We have a bad practice in general with Parse... we should not be using the Parse SDK to read from backend systems. We should be using Mongo directly. For writing I think it makes sense to use SDK with Cloud Functions — writing is different because of permissions and formatting."

| Operation | Method | Package | Why |
|---|---|---|---|
| **Read** invoices due today | MongoDB via `@is/mongo` | `packages/data-sources/is-mongo` | Typed collection accessors, handles Parse pointer transforms. Same pattern as is-checkout, is-document, etc. |
| **Read** account settings | MongoDB via `@is/account` | `packages/domain-model/is-account` | Wraps `@is/mongo` with domain methods like `AccountSettings.getSettingRemoteId(accountId, key)`. Same pattern as abandoned-cart. |
| **Write** invoice history entry | Parse Cloud Function | Parse SDK (`masterKey`) | Permissions, hooks, formatting — Cloud Functions are the correct write path |
| **Write** idempotency field | TBD — see options below | — | Needs decision |

### MongoDB Access Patterns in is-services (for reference)

| Pattern | Package | Used by |
|---|---|---|
| `@is/mongo` — typed collection accessors | `packages/data-sources/is-mongo` | Most services (checkout, edge, document, financing) |
| `@is/account` — domain methods wrapping `@is/mongo` | `packages/domain-model/is-account` | abandoned-cart, fundbox, acorn-webhook |
| Raw `MongoClient` | Direct `mongodb` driver | is-bookkeeping only (special case) |

### Decision: Reads via `@is/mongo` + `@is/account` (same as abandoned-cart)

- **Settings:** `AccountSettings.getSettingRemoteId(accountId, 'AutomaticPaymentRequests')` — same pattern abandoned-cart uses for `PaymentReminders`
- **Invoices:** `@is/mongo` exposes an `Invoice` collection with typed accessors and `getCollectionItems()`. Batch query with filter `{ dueDate: today, balanceDue: { $gt: 0 } }`
- **No Parse SDK for reads** — follows eng. team's guidance AND matches the existing abandoned-cart pattern

### Idempotency Write — Options (TBD)

The `paymentRequestSentDate` field needs to be written after email is sent. Three options:

| Option | Pattern precedent | Pro | Con |
|---|---|---|---|
| **Parse Cloud Function** | Eng. team's "Parse for writes" guidance | Consistent with guidance, triggers hooks | Adds N Cloud Function calls (one per invoice sent) |
| **DynamoDB tracking table** | Abandoned-cart uses DynamoDB for state | Proven pattern in this exact service area, no Parse schema change | New table, external state |
| **Direct Mongo write** | Only is-bookkeeping does this | Simplest code, no extra service call | Novel pattern, bypasses Parse hooks |

### Considered Alternatives (Read Strategy)

**Option B (Per-Invoice via Parse SDK):** Simple code (3 lines per invoice), but N queries hits Parse rate limits at scale. 500 invoices = 500 Parse queries = ~25s + rate limit risk. Rejected because of eng. team's guidance to avoid Parse SDK for backend reads.

**Hybrid (Chunked batches):** Unnecessary with `@is/mongo` — Mongo handles large `$in` arrays natively without the pagination constraints that Parse SDK imposes.

### Final Flow

```
Cron fires
  → @is/mongo Invoice collection: find({ dueDate: { $lte: today }, balanceDue: { $gt: 0 }, paymentRequestSentDate: { $ne: today } })
  → @is/account: AccountSettings batch lookup for 'AutomaticPaymentRequests' enabled
      (or @is/mongo Setting collection with $in filter on accountIds)
  → In-memory join: filter invoices to only enabled accounts
  → For each qualifying invoice:
      → Send email (via @is/messaging queue)
      → Write paymentRequestSentDate (method TBD)
      → Parse.Cloud.run('createInvoiceHistoryEntry', {
          accountId, invoiceRemoteId, eventType: 'payment_request_sent'
        })
```

### Why This Works

| Concern | Answer |
|---|---|
| Parse rate limits on reads? | Gone — `@is/mongo` / `@is/account` bypass Parse SDK entirely |
| Large `$in` clause? | Mongo handles natively, no 1000-result cap |
| Parse SDK overhead? | Only on writes (Cloud Functions) — accepted pattern |
| Code complexity? | `@is/account` for settings (1 call) + `@is/mongo` for invoices (1 query) + in-memory filter |
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

## Open Questions / Unknowns

| # | Item | Category | Detail |
|---|------|----------|--------|
| 1 | **Deduplication with existing reminders** | Risk | is-messages cron sends "unopened invoice" email 2-4 days after send. New cron sends "payment due today" on dueDate. If dueDate is 3 days after invoice was sent, client gets **both** same day. Need to suppress old one or accept overlap. |
| 2 | **Deduplication with abandoned cart** | Risk | If client viewed checkout but didn't pay, abandoned cart fires 30min + 3 days. If that coincides with dueDate, client gets 2 emails. Lower risk (requires checkout view). |
| 3 | **What counts as "non-vaulted"?** | Unknown | How to query whether client has a vaulted payment method. Where does this live? (Stripe Customer → PaymentMethods? Flag on Payment/Invoice? RecurringPayment object?) |
| 4 | **Time of day** | Decision | "On due date" — what time? Morning in merchant's timezone? UTC fixed time? What timezone data exists on accounts? |
| 5 | **Idempotency tracking** | Design | Need a field to prevent re-sending. Options: (a) new column on Invoice, (b) field on Payment/milestone, (c) separate tracking table, (d) check if Parse Msg with `eventType='payment_request_sent'` already exists. |
| 6 | **Multiple milestones due same day** | Edge case | If M1 and M2 both due today, send one email or two? Probably one — Invoice.dueDate is already the soonest, and the email is about the invoice, not individual milestones. |
| 7 | **Invoice fully paid before dueDate** | Edge case | Must check `balanceDue > 0`. If auto-pay or manual payment already covered it, skip. |
| 8 | **Where to host the cron** | Architecture | is-messages (existing reminder infra) vs. new Lambda in is-services (co-located with abandoned-cart). See section below. |
| 9 | **Retry if Parse write fails** | Design | If email sends but history logging fails — retry whole thing (double email risk) or track separately? |
| 10 | **Default value for new setting** | Product | Enabled by default (all accounts get emails immediately)? Disabled (opt-in, lower adoption)? Enabled for new accounts only? |
| 11 | **Email template** | Design | New SWU template needed. What content? Reuse checkout URL or invoice view URL? |
| 12 | **Only milestone invoices or all?** | Scope | Does this fire for any invoice with dueDate = today, or only those with milestones/deposits? |

---

## Architecture Decision: Where to Host

### Option: is-messages (existing reminder cron lives here)

**Pro:**
- Email-sending pipeline already built (SQS → Mailgun)
- Existing CDK patterns for EventBridge → Lambda
- Has PostgreSQL access (msg table)

**Con:**
- Legacy codebase — team moving away
- Never written to Parse Msg collection (only reads)
- Would need to add Parse Cloud Function call for history logging

### Option: is-services (new Lambda, alongside abandoned-cart)

**Pro:**
- Modern codebase, active development
- abandoned-cart is a good pattern to follow
- Has Parse masterKey access
- Can call Parse Cloud Functions directly
- Co-located with domain model (`is-account` settings lookup)

**Con:**
- Would need email-sending capability (currently uses `queueSWUJob()` from `@is/messaging`)
- New CDK construct needed

### Recommendation

**is-services** — new package `is-payment-requests` or add to existing `is-abandoned-cart` as a second flow. Follows the same pattern: EventBridge rule → Lambda handler → check setting → send email via `@is/messaging` → log to Parse.

---

## Pending Decisions (for Seth/Product)

1. Default value for `AutomaticPaymentRequests` setting on existing accounts
2. Time of day for the reminder (timezone handling)
3. Email template content/design
4. Whether this applies to all invoices with dueDate or only milestone invoices
5. Deduplication strategy with existing reminders (suppress old one? accept overlap?)

---

## Decisions Log

### 1. Deduplication with is-messages cron (existing "unopened invoice" reminder)

**Decision:** Accept overlap for MVP. If we dedupe later, APR email takes priority.
- Future dedup approach: APR Lambda writes `suppress_reminder = true` on the `msg` row for that invoice, preventing the old cron from picking it up. Requires cross-service PostgreSQL write.
- Other options considered:
  - Suppress APR if old cron already fired (check `reminder_sent_dt IS NOT NULL`) — less useful since APR email is more relevant
  - Consolidate into one email — bigger scope change, not worth it for Phase 1
  - Suppress old cron at invoice-send time (set `suppress_reminder = true` if dueDate within 4 days) — speculative and fragile

### 2. Vaulted check (non-vaulted = eligible for APR)

**Decision:** Not needed for Phase 1. Milestone auto-pay (vaulting for one-off invoices) hasn't been built yet — all milestone invoices are effectively "non-vaulted." Add this check in Phase 3 when milestone vaulting ships.
- When needed: query is-payments `recurring_payments` table for ACTIVE record matching the invoice's series
- Other options considered:
  - Check Payment collection for vaultId/setupIntentId fields — unclear if these exist on milestone payments
  - Check Parse RecurringPayment object linked to invoice

### 3. Scope — which invoices get APR?

**Decision:** To confirm with design/product. APR's source of truth is `Invoice.dueDate` — it doesn't inherently care how the date got there (milestones vs manual terms). The syncing of milestones to Invoice.dueDate is a separate ticket.
- Options to present:
  - Only milestone invoices (has Payment records in Payment collection)
  - Any invoice with `dueDate = today` (regardless of milestones)
  - Milestone-only for Phase 1, expand later

### 4. Default value for new setting

**Decision:** To confirm with design/product.
- Options to present:
  - **Disabled by default (opt-in):** Safer rollout, no surprise emails, lower initial adoption
  - **Enabled by default for all:** Higher adoption, but existing merchants start getting due-date emails immediately on deploy (potentially surprising)
  - **Enabled for new accounts only:** New signups get it on, existing stay off. Requires migration/backfill distinction.

### 5. Time of day

**Decision:** Fixed UTC time for Phase 1 (e.g. 14:00 UTC). Exact time to confirm with product. Timezone-awareness is a future improvement.
- Context: Both existing systems (is-messages cron, abandoned cart) use fixed UTC with no timezone logic. APR follows precedent.
  - is-messages cron: runs every 30 min around the clock, no preferred hour
  - Abandoned cart: fires relative to event time, no preferred hour
  - APR is the first system that picks a specific daily hour — no precedent
- Other options considered:
  - Run multiple times daily (e.g. every 6h) — reduces timezone mismatch but needs idempotency
  - Merchant timezone-aware (e.g. 9am local) — requires timezone data on accounts + bucketed approach. Future improvement.

### 6. Idempotency — prevent double-send

**Decision:** Add `paymentRequestSentDate` field on Invoice document. Cron query includes `paymentRequestSentDate != today` to exclude already-sent invoices. Follows the is-messages pattern (field on the swept record).
- Context: is-messages uses `reminder_sent_dt` on the `msg` row. For APR, the swept record is the Invoice (not a msg row), so the field goes on Invoice.
- Other options considered:
  - **Batch check Msg collection:** After finding invoices due today, query Msg collection for existing `payment_request_sent` entries. Filter in-memory. Pro: no Invoice schema change. Con: extra batch query.
  - **Separate collection (`PaymentRequestLog`):** New collection with (invoiceId, sentDate). Pro: clean separation. Con: new collection to maintain.
  - **Why not on `msg` row:** The `msg` row doesn't exist before APR fires — it's the *output* of the process, not the input being swept.

### 7. Multiple milestones due same day

**Decision:** One email per invoice. APR is about the Invoice being due, not individual milestones. `Invoice.dueDate = today` → one email regardless of how many milestones exist.

### 8. Retry/failure handling (email sends, history write fails)

**Decision:** Best-effort history. Log error to Sentry/CloudWatch, move on. Email is the important part — don't retry (avoids double-email risk). History entry is nice-to-have.
- Other options considered:
  - Separate tracking: mark "email sent" (idempotency field) separately from "history logged." Retry only the history write without re-sending email.
  - Retry entire operation: use SWU dedup key to prevent double-send on retry. More complex.

### 9. Deduplication with abandoned cart

**Decision:** Accept overlap for now. Different emails, different purposes ("complete checkout" vs "payment due today"). Revisit if users complain.
- Other options considered:
  - Suppress abandoned cart if APR fires: cancel pending EventBridge schedules for that invoice. Requires cross-service coordination with `remove-scheduler` Lambda.

### 10. Where to host the cron Lambda

**Decision:** is-services — new package alongside abandoned-cart. Modern codebase, has MongoDB access via `@is/data-sources`, Parse masterKey, `@is/messaging` for email sending.
- To confirm with eng. team
- Other options considered:
  - **is-messages:** Already has email pipeline + EventBridge cron patterns. But legacy codebase, no MongoDB access, team moving away from it.
  - Could also be a new function within existing `is-abandoned-cart` package rather than a brand-new package (same infra, shared CDK construct).

### 11. Email template / CTA

**Decision:** New SWU template with checkout URL CTA ("Complete Payment" → checkout page). Client can pay immediately.
- Other options considered:
  - Invoice view URL CTA ("View Invoice" → `/v/{hashId}`). Client sees invoice first, then decides to pay. Less direct.

---

## Next Steps

1. Confirm open product decisions (items 3, 4, 5) with Seth/design
2. Confirm architecture (item 10) with eng. team
3. Detail the implementation: Lambda handler code, CDK construct, Parse Cloud function, setting creation, email template
4. Define the MongoDB query shape (Invoice collection + Setting collection)
5. Wire up mobile + web UI for new toggle + renamed toggle

---

## TL;DR

**Current:** No email is sent on the day an invoice is due. The existing "Automatic Payment Reminders" only fires after checkout abandonment (30min + 3 days).

**Proposed:** A daily cron sends a "payment due today" email to clients whose invoice `dueDate = today`, controlled by a new per-merchant setting (`AutomaticPaymentRequests`).

**Details:**
- Daily EventBridge cron → Lambda in is-services (same pattern as abandoned-cart)
- Reads invoices via `@is/mongo` (typed collection accessors from `packages/data-sources/is-mongo`)
- Reads settings via `@is/account` (`AccountSettings.getSettingRemoteId()` — same as abandoned-cart)
- No Parse SDK for reads (per eng. team's guidance) — both packages are MongoDB under the hood
- Filters: `dueDate <= today`, `balanceDue > 0`, setting enabled, not already sent (`paymentRequestSentDate` idempotency field)
- Uses `<= today` (not `= today`) so overdue invoices still get caught if cron was down, with idempotency preventing duplicates
- Sends email via `queueSWUJob()` from `@is/messaging` → is-messages → SQS → Mailgun
- Logs history via `Parse.Cloud.run('createInvoiceHistoryEntry')` — best-effort (log error, move on)
- New SWU template with checkout URL CTA
- One email per invoice regardless of how many milestones are due
- Existing "Automatic Payment Reminders" renamed to "Outstanding Checkout Reminders" (UI only, backend key `PaymentReminders` unchanged)

**Confirm:**
- Scope: only milestone invoices or any invoice with `dueDate`? (product/design)
- Default value for new setting on existing accounts: opt-in vs enabled by default? (product)
- Exact UTC time for the daily cron (product)
- Where to host: is-services confirmed? (eng. team)
- Idempotency write method: Parse Cloud Function vs DynamoDB (abandoned-cart precedent) vs direct Mongo write? (engineering)
- Dedup with existing is-messages reminder if dueDate overlaps with reminder window — accept overlap for now, revisit if users complain
- Timezone-aware sending as a future improvement (no account timezone data today)
