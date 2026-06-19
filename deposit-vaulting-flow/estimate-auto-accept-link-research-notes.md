# Estimate Auto-Accept Link — Research Notes

**Date:** 2026-06-11 (updated 2026-06-17)
**Status:** Technical deep-dives and verification findings

This document contains all the technical deep-dives, code verification, and decision rationale that informed the [main spec](./estimate-auto-accept-link-spec.md).

---

## Flow Comparison

### Original Plan (from Dip's proposal)

```
1. Merchant sends estimate (with "Request Mandatory Card on File" enabled)
2. System generates one-time token → email with accept link
3. Client clicks link → hits endpoint (GET /accept?token=abc123)
4. Server validates + consumes token (atomic, single step)
5. Server converts estimate → invoice (server-side)
6. Server redirects client → card vaulting → deposit checkout
7. Server sends silent sync push to merchant
8. Merchant's app syncs
```

Client clicks one link → conversion + payment happen in one shot. Client never "sees" the estimate — just lands on checkout.

### Seth's Prototype Flow (6 screens)

```
1. Merchant sends estimate (with deposit configured)
2. System generates link → email shows deposit amount + "Review Estimate" CTA
3. Client clicks "Review Estimate" → lands on public estimate view page (NEW)
4. Client reads the full estimate (line items, totals, deposit amount)
5. Client clicks "Approve Estimate" button
6. Server validates + consumes token (atomic)
7. Server converts estimate → invoice (server-side)
8. Client lands on checkout — deposit amount + online fee shown, picks payment method
9. Client checks "Save Payment Method" → sees authorization for remaining balance
10. Client pays deposit
11. Success page confirms deposit paid + card saved for remaining balance
12. Server sends silent sync push to merchant
```

Client reviews first, explicitly approves, then pays. Three distinct steps visible to the client.

### Key Differences

| | Original plan | Seth's prototype |
|---|---|---|
| **Client sees estimate?** | No — skips straight to checkout | Yes — full view page first |
| **Approval is explicit?** | No — clicking link = acceptance | Yes — separate "Approve" button |
| **Conversion trigger** | Link click | Approve button click |
| **Steps for client** | 1 (click → pay) | 3 (view → approve → pay) |
| **New pages needed** | Accept endpoint only | Estimate viewer + approve endpoint + checkout |
| **Vaulting consent** | Implicit | Explicit checkbox + authorization text with remaining balance |
| **Deposit shown in email** | Not specified | Yes — "DEPOSIT USD $100.00" in email |

**Key insight:** This isn't just "convert and view" — it's **vault card + convert + charge deposit**. The conversion is sandwiched between two payments operations. Seth's prototype makes this explicit to the client with clear authorization language.

### Seth's Prototype Screens (summary)

1. **Email** — Shows estimate number, date, total ($500), deposit amount ($100), "Review Estimate" CTA
2. **Estimate view** — Full public estimate page on invoicesimple.com with "Approve Estimate" button (+ PDF/Print options)
3. **Payment form** — Deposit $100 + online payment fee $3.49 = $103.49, card fields
4. **Payment methods** — US Bank, Pay Later, PayPal, Apple Pay, Link, Google Pay + "Save Payment Method" checkbox
5. **Save card auth** — "Acme Landscaping will be authorized to charge your selected payment method for this invoice's remaining balance. Balance: $413.96"
6. **Success** — Deposit paid confirmed + "Payment Method Saved" with authorization details (amount, method, contact to modify)

---

## Current State

### Estimate-to-Invoice Conversion (Today)

The conversion is **entirely client-side** — no backend endpoint exists.

| Platform | Entry Point | Core Logic |
|----------|-------------|------------|
| Web | `ConvertToInvoiceButton.tsx` | `InvoiceModel.convertEstimate()` → `_convertEstimateData()` |
| Mobile | `convert-estimate-to-invoice-modal.tsx` | `makeInvoiceFromEstimate()` in realm use-cases |

**Field mapping summary:**
- `docType` flips from ESTIMATE (1) → INVOICE (0)
- New `remoteId` assigned (new document identity)
- `estimateId` set on invoice settings (back-reference)
- Estimate-specific fields stripped (`estimateSignatureRequired`, etc.)
- Items, client, company, photos, signature preserved
- Totals recalculated

**Payments are runtime-determined** — any invoice with `balanceDue > 0` and payment not suppressed is automatically payable. No explicit "enable payments" step needed after conversion.

### Link/Share Infrastructure (Today)

| Pattern | Purpose | Auth | Tracking |
|---------|---------|------|----------|
| `/vm/{msgId}/{invoiceNo}` | Mobile share (tracked) | None | `msg` table, open events |
| `/vm/{msgId}/{invoiceNo}?p=1` | Share + payment redirect | None | Same + payment_viewed |
| `/v/{documentId}` | Web public link | None | hashids → msg_archive |

Links are permanent (no expiry), identified by UUID obscurity, and tracked via the `msg` PostgreSQL table.

---

## Public Estimate View Page (Screen 2 of Seth's Prototype)

### What it shows (from Seth's prototype):
- Estimate number (EST0001)
- Date
- Merchant logo + company name + business number + address + contact
- Client name + phone + email ("TO" section)
- Line items (description, rate, qty, amount)
- Total
- Deposit amount
- "Approve Estimate" button (primary CTA)
- PDF + Print buttons (secondary)
- "Powered by Invoice Simple" footer

### Key Finding: Public estimate view ALREADY EXISTS (updated 2026-06-17)

**A public estimate view page already exists** at the same route as invoices:

| Component | Location |
|-----------|----------|
| Next.js page | `is-web-app/nextjs/app/(public)/v/[documentId]/page.tsx` |
| Feature component | `is-web-app/nextjs/app/(public)/v/[documentId]/public-document.feature.tsx` |
| Document type detection | Checks `docType === DocTypes.DOCTYPE_ESTIMATE` at runtime |
| Estimate signature modal | `estimate-signature-modal.tsx` — already renders for estimates |
| Express route | `is-web-app/server/app.ts` — `/v/:id` |
| URL pattern | `/v/{documentId}` — same for both invoices and estimates |

Both invoices and estimates use the **same `/v/{documentId}` route**. The component detects the document type at render time and renders accordingly (e.g., shows signature modal for estimates when `estimateSignatureRequired` is true).

### Revised Approach: Reuse is-web-app's public view page

Instead of building a new page in a separate service, **reuse the existing public estimate page in is-web-app** and add the "Approve Estimate" button as a new affordance:

```
Email "Review Estimate" CTA
  → is-web-app: /v/{documentId}/approve (NEW route)
    → validate approvedAt (NOT consumed yet — just checking)
    → render existing public estimate page + "Approve Estimate" button

Client sees existing public estimate page + "Approve Estimate" button
  → Client clicks "Approve Estimate"
  → is-web-app calls Parse Cloud Function: approveEstimate({ documentId })
    → consume approvedAt guard, convert, redirect to checkout
```

**Benefits:**
- No new rendering — reuse existing Next.js page with full branding, responsive design, PDF/Print
- Consistent styling with the rest of the public document experience
- Parse Cloud Function stays focused: conversion + state management only, no rendering responsibility
- Existing page already handles data fetching from Parse, merchant branding, mobile responsiveness

**What's new on the existing page:**
- "Approve Estimate" button (conditionally shown when route is `/approve`)
- API call to Parse Cloud Function `approveEstimate` on button click
- "Already approved" state handling (if approvedAt already set)
- "Link expired" state handling (optional)

**Data fetching:** Not needed in a separate service — is-web-app already fetches the full document from Parse/Mongo for the public view. Parse Cloud Function only needs to fetch the document at approval time (for conversion).

---

## is-checkout Compatibility Check (verified 2026-06-17)

Juan flagged: is-checkout has eligibility checks + email logic — does it work with estimates?

### 1. Email sending — No issue

The recipient email comes from the document's `client.email` field. No docType filtering anywhere in the send flow — estimates can already be emailed via web and mobile. Our "Send Estimate" flow already works.

- **Web**: `is-web-app/nextjs/app/(authenticated)/(core)/(documents)/components/email-document/modal-email-document-form.tsx` — pulls from `props.document.client?.email`
- **Mobile**: `is-mobile/src/services/send-invoice/api.ts` — gets from client object on the document
- **Backend** (`api/src/controllers/invoice-sending.ts`): receives `toAddress` from client request, no docType validation

### 2. Public page `/v/{documentId}` — No issue

The `publicInvoice` Parse cloud function (`is-parse-server/cloud/collections/invoice/functions/publicInvoice/`) fetches by `invoiceId` with **no docType check**. Estimates are already viewable at `/v/{documentId}`. Our approve page route sits on top of this.

### 3. Payment eligibility — Rejects estimates, but NOT a problem for our flow

**The check** (`is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts`, lines 58-60):
```
documentType !== 0 → GeneralNotPayableReason.docTypeNotInvoice
```

Only `docType === 0` (invoice) shows payment buttons. Estimates (`docType === 1`) are rejected.

**Why this doesn't affect us:** By the time the buyer reaches checkout, the estimate has already been converted to an invoice by the cloud function. Checkout receives an `invoiceId` pointing to a real invoice with `docType === 0`. Eligibility passes.

**What to ensure:** The approve page must redirect to `/checkout/{newInvoiceId}` (from cloud function response), NOT `/checkout/{estimateId}`. The estimate itself will never hit the checkout flow.

### Full eligibility check chain (for reference)

`findNotPayableReason()` checks in order:
1. No checkout data → not payable
2. `documentType !== 0` → **rejects non-invoices** (this is the only docType check in the system)
3. Document deleted → not payable
4. Total ≤ $0 → not payable
5. Balance due ≤ $0 → not payable
6. Balance due ≥ $1,000,000 → not payable
7. Pending orders exist → not payable

**Endpoint**: `GET /checkout/eligibility/{accountId}/{documentRemoteId}?hasEverSubscribed={boolean}`

### Summary

| Area | Works with estimates? | Action needed? |
|------|----------------------|----------------|
| Email sending | Yes — no docType check | None |
| Public page `/v/{documentId}` | Yes — no docType check | None |
| Payment eligibility | No — rejects non-invoices | None (conversion happens before checkout) |

**No code changes needed in is-checkout.** The conversion-first architecture means eligibility always sees an invoice.

---

## Parse Cloud Function Feasibility (verified 2026-06-17)

Can a Parse Cloud Function really handle everything `approveEstimate` needs to do? Yes — existing cloud functions in this codebase already do each of these individually.

### Can it do atomic double-approve protection?

The ideal approach is a raw Mongo `findOneAndUpdate` with `{ approvedAt: { $exists: false } }` — only one caller wins, true atomicity.

Parse Cloud Functions have access to the underlying Mongo adapter:
```js
const db = Parse.Server.database.adapter;
// raw Mongo queries available
```

Simpler v1 approach: `estimate.fetch()` → check `approvedAt` → set + `.save()`. Race condition window is tiny (two people clicking approve at the exact same millisecond). Fine for v1; can add Mongo-level atomicity later if needed.

### Can it convert estimate → invoice?

Yes — just `new Parse.Object('Invoice')`, set fields, `.save()`. Cloud functions routinely create and save Parse objects. The conversion logic (field mapping) lives directly in the function body.

### Can it set fields on the estimate?

Yes — `estimate.set('approvedAt', new Date())` + `.save()`. Standard Parse operations.

### Can it send push notifications?

Yes — existing cloud functions already call `@invoice-simple/is-msg` for both silent sync and ALERT pushes. The bookkeeping, recurring payments, and Stripe integration cloud functions all use `isMsg.pushNotification.sync()` and alert pushes.

### Can it return data to the caller?

Yes — cloud functions just `return { invoiceId }` and the is-web-app caller receives it:
```js
const result = await Parse.Cloud.run('approveEstimate', { documentId, email })
// result.invoiceId → redirect to /checkout/{invoiceId}
```

### Summary

| Capability | Feasible? | Pattern |
|-----------|-----------|---------|
| Atomic guard (approvedAt check) | Yes (v1: fetch+check; v2: raw Mongo) | Standard |
| Create invoice Parse object | Yes | Standard |
| Set fields on estimate | Yes | Standard |
| Send push notifications | Yes | Existing pattern (bookkeeping, RP, Stripe) |
| Return invoiceId to caller | Yes | Standard cloud function return |

---

## `sentStatus` Field — Reliability for Expiry (verified 2026-06-17)

If we use `sentStatus.updatedAt` for link expiry, is it reliable?

**How `sentStatus` is set:**
1. **On document creation** — Parse `beforeSave` hook (`invoiceHooks.ts:106`) sets `sentStatus: { name: 'draft', updatedAt: now }` if not already present
2. **On send** — Client app (mobile/web) sets `sentStatus: { name: 'sent', updatedAt: now }` and syncs to Parse
3. **On email events** — Webhooks update to 'delivered', 'opened', 'viewed', etc.
4. **Server-initiated sends** (e.g., recurring invoices) — Backend explicitly calls `updateInvoiceSentStatus(invoiceId, 'sent')`

**Reliability concern:** A comment in `api/src/util/document-version/utils.ts` calls this field **"deprecated"**:
> "AFAIK this field is deprecated. Parse server sets it in the beforeSave hook but it gets stripped somewhere down the line."

**Safer alternatives for expiry:**
- `sentStatus.updatedAt` with `name: 'sent'` — should work but may not be reliable
- `document.updated` — always accurate, but changes on any edit (not just send)
- Query the `msg` PostgreSQL table — has the actual send timestamp per message, most accurate
- `document.createdAt` — simplest, doesn't reset on re-send

**For v1:** Expiry is likely not needed at all (product question open). If we add it, `msg` table timestamp is the most reliable source.

---

## Technical Deep-Dive: Moving Conversion Server-Side

### Issue 1: Sync Ownership — NOT A PROBLEM

Mobile uses `remoteId` as primary key with `UpdateMode.All` (upsert). When the server creates an invoice via Parse, the next sync pull detects "this remoteId doesn't exist locally" and cleanly inserts it. No duplicates, no conflicts. The sync logic doesn't distinguish between "I created this" and "server created this."

**Silent sync push already exists:** `@invoice-simple/is-msg` SDK has `pushNotification.sync()` — used by bookkeeping, stripe, paypal, recurring invoices, document restore, expense OCR. On mobile, `PushNotificationType.SYNC = 4` triggers `syncStore.sync()`. With `full: false`, it's completely silent — no banner, no notification tray, no sound. Data just appears in the list.

**Gap:** Merchant won't *know* the client accepted unless they're watching their list. Probably need an ALERT notification ("Your estimate was accepted!") in addition to the silent sync.

### Issue 2: Platform Behavior Differences — DESIGN DECISION

A server endpoint needs to pick one behavior:

| | Mobile | Web | Recommendation |
|---|---|---|---|
| **Hidden items** | Includes ALL | Only `visibleItems` | Web (hidden = intentional) |
| **Custom notes** | Appends default invoice note | Preserves as-is | Web (safer, don't add text user didn't ask for) |
| **Deposit fields** | Strips `depositType`, `depositRate`, `depositAmount` | Keeps them | Mobile (strip, since D&M would configure deposits separately) |
| **UUID format** | v4 (random) | v1 (time-based) | Either (both globally unique) |

None are blockers — just needs a decision.

### Issue 3: On-Device Side Effects — ONE REAL CONCERN

**a) Estimate marked `fullyPaid: true`** — moves from "Open" to "Closed" tab.
- Server can do this on Parse. Client picks it up on sync.
- Small window where estimate stays in "Open" until sync arrives. Low risk — merchant knows they sent a link.

**b) `InvoiceLastNo` bump** — ~~duplicate number risk~~ **Not actually a problem.**
- `invoiceNo` is just a display label, not a unique key. Identity is `remoteId`.
- Users can already manually set duplicate numbers today.
- Worst case: counter is stale for next local create. Minor UX annoyance at most.

**c) Instant availability** — currently invoice appears immediately (Realm write).
- With server-side creation, merchant waits for sync push (~seconds).
- Silent sync push (`full: false`) handles this — already a pattern.

---

## Security Considerations

### Link Security Model (updated 2026-06-17 — no token, uses documentId)

| Concern | Assessment |
|---------|------------|
| **Is URL obscurity enough?** | Yes — same 24-char MongoDB ObjectId as existing share links. Not guessable, same security model we already use for invoice viewing. |
| **Can the link authorize a charge?** | **No.** The link allows approval (estimate → invoice conversion) only. Payment still goes through normal Stripe/PayPal checkout which has its own authorization (card entry, PayPal login). No stored payment method is charged without explicit client action. |
| **Replay protection** | `approvedAt` field on estimate — once set, approval endpoint rejects. Subsequent visits show "already approved" page. |
| **Expiry** | Check estimate `sentAt` — if older than threshold (30 days?), show "link expired." |
| **Enumeration** | MongoDB ObjectId (24-char hex) — same security as existing `/v/{documentId}` share links. |

### Comparison to Current Share Links

| Property | Current Share Links | Proposed Approve Link |
|----------|--------------------|--------------------|
| Auth | None (ObjectId obscurity) | None (same ObjectId) |
| Expiry | Never | 30 days from send (soft check) |
| Single-use | No (permanent) | Yes (approve action is one-time via `approvedAt`) |
| Action | View document | View + approve + redirect to pay |
| Payment auth | Separate (checkout flow) | Separate (checkout flow) |

**Conclusion:** The approve link has the **same security model** as existing share links (ObjectId obscurity) plus a single-use semantic on the approve action. No token table needed — the estimate's own state is the guard.

### Token Proves Intent, Not Identity (raised by Dip — 2026-06-17)

**The problem:** The token is a magic link — unauthenticated, no login required. Anyone who has the URL can view the estimate and click "Approve." If the client forwards the email (intentionally or accidentally), whoever receives it can approve on the client's behalf.

**What the token guarantees:**
- Estimate is approved **at most once** (single-use, atomic consume)
- Approval happened within the expiry window
- The approver had access to the link (i.e., they received or were forwarded the email)

**What the token does NOT guarantee:**
- The person who clicked "Approve" is the intended recipient
- The merchant has any record of *who* approved

**Why "confirm your email" doesn't fix it:**
- The email address is visible in the forwarded email — anyone who received the forward already knows it
- Not a real identity barrier, just friction

**Realistic options:**
1. **Accept it** — same tradeoff as any magic link system (Stripe invoices, DocuSign lite, Calendly). Assumption: whoever received the email is the intended party. Document as known limitation.
2. **Full auth** — client must log in (or create an account) to approve. Verifies identity but kills frictionless UX — a significant product tradeoff.

**Decision needed (Liz/Seth):** Is frictionless approval (no login) worth accepting that the token proves "someone clicked" but not "the right person clicked"?

For v1, the recommendation is to accept it and document the limitation — the financial exposure is limited because the token does not authorize any charge (payment still requires explicit card entry or PayPal auth in checkout).

**Why this is largely a non-issue in practice:**
- At deposit checkout: card entry (or PayPal/Apple Pay auth) is required — a third party can't pay with the real client's card without having it
- At buyer-initiated remaining balance payment: a PIN is sent to the client's email/phone before the saved card can be used — only the real client can receive and enter it
- The PIN effectively acts as the identity gate at payment time, which is where it actually matters

An unauthorized approver can create an invoice but cannot pay it without the real client's credentials. The system self-corrects at the payment step.

**Recording buyer info (from Liz — 2026-06-17):**

Liz suggested recording buyer info with the consent and surfacing it to the merchant in-app.

| Record | Field | Captured when | Source | New? |
|--------|-------|---------------|--------|------|
| **Estimate** | `approvedBy: { email?, timestamp }` | "Approve Estimate" click | Email input on approve page + request timestamp | Yes — new |
| **Invoice** | Payer info (name, email, last4, brand) | Checkout payment | Stripe/PayPal payment response | No — already exists |

Only the estimate `approvedBy` field is new. Invoice payer info is already captured by Stripe/PayPal at checkout time and visible to merchants today.

**Open question (Liz/Seth):** Should the approval page include an email input field? This would give the merchant a record of who approved — and if it doesn't match the client email on file, that's a signal someone else approved. Not a hard gate (can be skipped or faked), but useful metadata.

---

## Code Paths to Reuse

| What | Where | Reuse Strategy |
|------|-------|----------------|
| Field mapping logic (web) | `is-web-app/client/src/models/InvoiceModel.ts:1296` (`_convertEstimateData`) | Extract to Parse Cloud Function as pure logic |
| Field mapping logic (mobile) | `is-mobile/src/services/realm/entities/invoice/use-cases.ts:22` (`makeInvoiceFromEstimate`) | Extract to Parse Cloud Function as pure logic |
| Link tracking infra | `api/src/services/msg.ts` | Pattern reference (existing infra) |
| Payment eligibility | `is-packages/is-stripe-sdk/invoice-payable.ts` | Already runtime — no changes needed |
| Checkout redirect | `api/src/controllers/invoice-viewing.ts:280-296` | Pattern reference for redirect logic |
| Silent sync push | `is-services/.../is-bookkeeping/src/utils/push-notification.ts` | `isMsg.pushNotification.sync({accountId, ...})` — one-liner |
| ALERT push notification | Existing "Payment received" push pattern | Same infra, new event type + copy |
| Mobile sync handler | `is-mobile/src/services/notifications/handlers.ts:139` | `PushNotificationType.SYNC = 4` → `syncStore.sync()` — already wired |
| Mobile ALERT handler | `is-mobile/src/services/notifications/handlers.ts` | Existing ALERT handling — just add new notification type routing |
| In-app messaging | `is-messages` / `@invoice-simple/is-msg` | Existing notification/message infrastructure — add new event type |
