# Estimate Auto-Accept Link — Specification

**Date:** 2026-06-11 (updated 2026-06-17)
**Status:** Architecture finalized after Juan sync
**PRD Reference:** [Deposits Optimizations PRD](https://everpro-tech.atlassian.net/wiki/spaces/PM/pages/1213300774/Deposits+Optimizations+PRD)

---

## Overview

**Goal:** Client receives an estimate email → clicks "Review Estimate" → sees full estimate → clicks "Approve" → estimate converts to invoice (server-side) → redirects to checkout → pays deposit + vaults card → merchant notified.

**Seth's prototype:** 6-screen flow — email → estimate view → approve → deposit checkout → card vault → success.

---

## Architecture & Service Boundaries

### Proposed Code Split (simplified after Juan sync — 2026-06-17)

```
is-web-app/
  nextjs/app/(public)/v/[documentId]/
    approve/page.tsx                    ← NEW route: renders estimate + "Approve Estimate" button
                                           If already approved → shows public invoice (redirect or inline)

is-parse-server/
  cloud/functions/
    approveEstimate.ts                  ← NEW Parse Cloud Function
                                           - validates approvedAt (guard)
                                           - converts estimate → invoice (logic lives here)
                                           - sets approvedAt + approvedBy on estimate
                                           - saves new invoice
                                           - returns { invoiceId } for redirect to checkout

is-services/
  packages/services/is-unifiedxp/       ← ALREADY EXISTS (Next.js customer-facing UI)
    src/app/checkout/[documentId]/
      page.tsx                          ← payment form (screens 3-5 of Seth's prototype)
      success/page.tsx                  ← success page (screen 6)

  packages/services/is-checkout/        ← ALREADY EXISTS (Express backend API)
    src/checkout/                        ← payment intent creation, eligibility, orchestration
```

**Key simplification (Juan):** No new `is-estimates` microservice. No `@is/common` shared package extraction. All conversion logic lives in a Parse Cloud Function — same pattern as other server-side document operations. The is-web-app approve page calls the cloud function directly.

### Service Roles

| Service | Type | Role | Seth's prototype screens |
|---------|------|------|------------------------|
| `is-web-app` (exists, new route) | Next.js | `/v/{documentId}/approve` — renders estimate view + "Approve" button | Screen 2 (estimate view) |
| Parse Cloud Function (NEW) | Cloud Function | `approveEstimate` — validates state, converts, saves invoice, returns invoiceId | — (backend logic) |
| `is-unifiedxp` (exists) | Next.js | Customer-facing checkout UI — payment form, methods, success | Screens 3-6 (payment → success) |
| `is-checkout` (exists) | Express | Backend payment API — intent creation, eligibility checks | Backend for screens 3-6 |
| `is-payments` (exists) | Express | Payment methods, recurring payment orchestration | Card vaulting / `setup_future_usage` |

**Removed:** `is-estimates` microservice and `@is/common` shared package — conversion logic lives directly in the Parse Cloud Function.

---

## Request Flow

```
Email "Review Estimate" CTA
  → is-web-app: /v/{documentId}/approve (NEW route on existing public page infra)
    → fetch estimate from Parse (existing pattern)
    → check approvedAt — if set → show public invoice page (works out of the box)
    → otherwise → render estimate view + email input + "Approve Estimate" button

"Approve Estimate" button click
  → Parse Cloud Function: approveEstimate({ documentId, email? })
    → fetch estimate, check approvedAt (guard against double-approve)
    → set approvedAt + approvedBy { email, timestamp }
    → convert estimate → invoice (logic inside cloud function)
    → save new invoice to Parse
    → send silent sync push + ALERT push to merchant
    → return { invoiceId } → client-side redirect to checkout

Checkout + Payment (deposit paid)
  → is-unifiedxp: /checkout/{invoiceId} (already exists)
    → shows deposit amount + fee
    → payment methods (Card, Bank, PayPal, Apple Pay, Link, Google Pay)
    → "Save Payment Method" checkbox → Stripe setup_future_usage
    → Pay Now → is-checkout backend creates PaymentIntent
    → success page shows deposit paid + card authorized for remaining balance

  → is-checkout: payment confirmation webhook / callback
    → mark estimate fullyPaid: true (→ "Closed")

Already-approved re-visit (Juan's insight)
  → Client clicks same link again after approval
  → is-web-app: /v/{documentId}/approve detects approvedAt is set
  → Shows public invoice page with "Pay Online" button → checkout
  → Works out of the box — invoice is already a public document at /v/{invoiceId}
```

**Why no token table:** The estimate's own state (`approvedAt`) is the single-use guard — once set, approval can't happen again. Expiry can be checked against when the estimate was sent. Invalidation on edit is handled by checking estimate's `updatedAt` vs a threshold. No separate token infrastructure needed.

**Why no new microservice (Juan):** Parse Cloud Functions are the existing pattern for server-side document operations. A whole microservice for one endpoint is over-engineering. The cloud function has direct access to Parse/Mongo — no extra data-fetching layer needed.

---

## Architecture Diagram

![Estimate Accept Flow](./diagrams/estimate-accept-flow.png)

(Source: [estimate-accept-flow.mmd](./diagrams/estimate-accept-flow.mmd))

---

## Bulleted Flow Summary

**Mobile / Web (merchant)**
- Merchant hits "Send Estimate" with deposit configured

**API + is-messages (EXISTING)**
- Send flow composes email with `/v/{documentId}/approve` link embedded
- No token generation step needed — documentId is the identifier

**End Client (estimate recipient)**
- Receives email showing deposit amount + "Review Estimate" CTA

**is-web-app — NEW ROUTE on existing public page infra**
- `/v/{documentId}/approve` — fetches estimate, checks state (`approvedAt`), renders estimate view page with "Approve Estimate" button + optional email input
- If already approved → shows "Already approved" message + link to checkout
- Client reviews full estimate (line items, totals, deposit)
- Client clicks "Approve Estimate"

**Parse Cloud Function `approveEstimate` (NEW — replaces is-estimates service + @is/common)**
- Called from is-web-app approve page: `approveEstimate({ documentId, email? })`
- Checks `approvedAt` (double-approve guard via atomic findOneAndUpdate)
- Sets `approvedAt` + `approvedBy { email?, timestamp }` on estimate
- Converts estimate → invoice (conversion logic lives directly in cloud function)
- Saves new invoice to Parse
- Sends silent sync push + ALERT push to merchant
- Returns `{ invoiceId }` for client-side redirect to checkout

**is-unifiedxp + is-checkout (EXISTING)**
- `/checkout/{invoiceId}` — deposit amount + online fee, all payment methods (Card, Bank, PayPal, Apple Pay, Link, Google Pay)
- "Save Payment Method" checkbox — authorization text for remaining balance, Stripe `setup_future_usage`
- "Pay Now" — is-checkout backend creates PaymentIntent + SetupIntent
- `/checkout/{invoiceId}/success` — deposit paid confirmed + card authorized for remaining balance

**Merchant Device — NEW: sync trigger + notifications for estimate acceptance**
- Silent sync push received (push infra exists, but this trigger point is new)
- New invoice appears in list
- Estimate moves to **"Approved"** state (`approvedAt` set, `fullyPaid` still false — client has approved but not yet paid)
- Estimate only moves to **"Closed"** (`fullyPaid: true`) after deposit payment is confirmed in checkout
- Visible ALERT push notification: "Your estimate was approved!" (confirmed by Seth — needed for when merchant is not in-app)
- In-app messaging linking estimate → converted invoice (confirmed by Liz — merchant needs to see what happened and navigate to the new invoice)

---

## Flow Mapped to Endpoints

| Step | Client action | Endpoint | Service | What happens |
|------|--------------|----------|---------|--------------|
| Send | Merchant hits "Send Estimate" | (existing send flow) | is-messages (exists) | Composes email with `/v/{documentId}/approve` link |
| Email click | "Review Estimate" | `GET /v/{documentId}/approve` | is-web-app (exists, new route) | Fetch estimate, check `approvedAt`. If approved → show public invoice. Otherwise → render view page + "Approve" button |
| Approve | "Approve Estimate" button | Parse Cloud Function `approveEstimate({documentId, email?})` | is-parse-server (exists, new function) | Check `approvedAt`, set it + `approvedBy`, convert → invoice, save, push notify, return `{ invoiceId }` |
| Redirect | Client-side | `/checkout/{invoiceId}` | is-unifiedxp (exists) | Existing checkout page — deposit amount + fee, payment methods |
| Pay | "Pay Now" | (existing checkout POST) | is-checkout (exists) | Stripe PaymentIntent + SetupIntent for future charges |
| Success | — | `/checkout/{invoiceId}/success` | is-unifiedxp (exists) | Show deposit paid + authorization for remaining balance |
| Re-visit | Client clicks link again | `GET /v/{documentId}/approve` | is-web-app | Detects `approvedAt` set → shows public invoice with "Pay Online" button (works out of the box) |

---

## What's New vs What Exists

**Build new:**
1. Parse Cloud Function `approveEstimate` (validates state, converts estimate → invoice, returns invoiceId)
2. `/v/{documentId}/approve` route in is-web-app (new Next.js page — reuses existing public document components, adds "Approve" button + email input)
3. Email template update (add deposit amount + "Review Estimate" CTA with `/v/{documentId}/approve` link)
4. "Approved" state on estimates — new field `setting.approvedAt: Date` + `setting.approvedBy: { email?, timestamp }` (backwards-compatible)
5. Visible ALERT push notification for "estimate approved" (new event type + copy)
6. In-app message linking estimate → converted invoice (new notification type)
7. (Optional) `convertedTo: invoiceId` field on estimate if we want to link them

**Already exists (reuse as-is):**
- `is-unifiedxp` checkout page (screens 3-5 — payment form, all methods)
- `is-unifiedxp` success page (screen 6 — with minor copy additions)
- "Save Payment Method" / Stripe `setup_future_usage: 'off_session'`
- Online payment fee calculation
- Silent sync push (`isMsg.pushNotification.sync()`)
- All payment methods (Card, Bank, PayPal, Apple Pay, Google Pay, Link)
- Push notification infra (ALERT type — same as "Payment received" today)
- In-app notification/messaging infra (is-messages)

**Minor modifications to existing:**
- `is-unifiedxp` checkout may need a `deposit=true` mode to show deposit amount vs full balance
- `is-unifiedxp` success page needs vaulted card authorization blurb ("authorized to charge $413.96")
- Mobile/web need to handle new "Approved" state on estimates (UI for the state, probably a badge or tab)

---

## Estimate Lifecycle (NEW — from Liz's feedback 2026-06-16)

### Current "Make Invoice" Behavior (verified 2026-06-17)

**Finding: The estimate is NOT marked after manual conversion.**

Tested on mobile: after "Make Invoice", the estimate record in Parse only gets an `updated` timestamp bump — no `fullyPaid: true`, no state change. The estimate stays fully open and can be converted again (unlimited invoices from the same estimate).

**Team consensus (Slack thread 2026-06-17):**
- Liz: "In theory, you could simply update a single Estimate and convert it to a new Invoice each time" — valid contractor workflow (reusable estimates)
- Juan/Seth/Liz agree: merchant-initiated conversion stays as-is (unlimited, no guard)
- Buyer-flow enforces 1:1 via `approvedAt` (new field)
- **If product ever wants unique conversion:** just update `docType` + `title` on the estimate in-place (same Mongo collection — invoices and estimates share the Invoice collection, just different `docType`). No migration needed.
- **If we need to link estimate → invoice:** add a `convertedTo: invoiceId` field on the estimate (adding a field to a collection is straightforward)

### Proposed Estimate States

| State | Trigger | What merchant sees | Estimate field |
|-------|---------|-------------------|----------------|
| **Open** | Estimate created / sent | In "Open" tab, waiting for client response | no `approvedAt`, `fullyPaid` unreliable |
| **Approved** (NEW) | Client clicks "Approve Estimate" via link | In "Approved" tab or badge, invoice created but deposit not yet paid | `approvedAt: timestamp` |
| **Closed** | Deposit payment received | In "Closed" tab, fully converted + paid | `fullyPaid: true` (if we fix the persistence bug) |

### Implementation Decision: `approvedAt` field

Current codebase has **no status enum on estimates** — state is supposedly derived from `setting.fullyPaid`, but this field is not reliably set.

**Decision: add `setting.approvedAt: Date | undefined`** — minimal, backwards-compatible change:
- Open = no `approvedAt`
- Approved = `approvedAt: Date` (set by approve link flow)
- Closed = `fullyPaid: true` (set when deposit is actually paid — needs the persistence bug fixed)

`approvedAt` also serves as the **single-conversion guard** for the approve link flow — the first real enforcement that an estimate can only be converted once via this path. (The manual "Make Invoice" path remains unguarded — existing behavior.)

**Files that need changes:**
1. `is-packages/packages/domain-invoicing/src/document/estimate.ts` — add `approvedAt` to EstimateSettings
2. `is-parse-server/cloud/collections/invoice/invoiceValidation.ts` — allow new field
3. `is-services/packages/domain-model/is-document/src/pagination/filter.ts` — add Approved filter
4. Mobile Realm schema — add field + sync
5. Mobile/Web UI — tab filtering, list display, badge for "Approved" state

---

## Merchant Notification Details (from Liz + Seth — 2026-06-16)

### Implementation (all existing patterns):

**1. Silent sync push (data)**
- `isMsg.pushNotification.sync({ accountId, full: false })` — one-liner
- Triggers `syncStore.sync()` on mobile → new invoice appears in list, estimate status updates
- Pattern: used by bookkeeping, stripe, paypal, recurring invoices, document restore, expense OCR
- No visible notification — just data refresh

**2. Visible ALERT push notification (user-facing)**
- `isMsg.pushNotification.alert({ accountId, title, body, data })` or similar
- Shows banner/notification tray: "Your estimate EST0001 was approved by Tomás!"
- Tap → deep link to the converted invoice
- Pattern: used by "Payment received" notifications today
- Payload needs: `invoiceId` (to navigate to), `estimateId` (to show context)

**3. In-app message linking estimate → invoice**
- Show in a notifications/activity feed or as a banner on the estimate itself
- "This estimate was approved → View Invoice INV0014"
- Pattern: existing `is-messages` / notification infrastructure
- Could be implemented as:
  - A notification row in existing notifications list
  - A status banner on the estimate detail screen
  - Both (notification to attract attention, banner for persistence)
- Product decision: which approach? (Ask Seth for designs)

### Technical complexity: LOW

All three mechanisms exist today. New work is:
- Define new event type/copy for "estimate approved"
- Define deeplink payload (which screen to open, which document)
- Wire the trigger in Parse Cloud Function (after successful conversion)

---

## Connection to Deposit Vaulting Flow

The "Save Payment Method" checkbox + `setup_future_usage` in Seth's prototype (screens 4-5) is the **entry point to the deposit vaulting flow** documented in [deposit-vaulting-flow-summary.md](./deposit-vaulting-flow-summary.md).

### What happens if client checks the checkbox (vault + both charge paths)

Mechanics are identical to Recurring Payments:

**At checkout (deposit payment):**
- `PaymentIntent` with `setup_future_usage: 'off_session'`
- Stripe charges the deposit AND stores the payment method in one shot
- Returns a `SetupIntent` / `PaymentMethod` ID → saved to the account/invoice record

**Merchant-initiated charge (off-session — same as RP):**
- Merchant taps "Charge Remaining Balance" in the app
- `PaymentIntent.create({ payment_method: savedPmId, off_session: true, confirm: true })`
- Same Lambda/SQS pattern as RP scheduled charges
- Client doesn't need to do anything

**Buyer-initiated charge (on-session):**
- Client returns to `/checkout/{invoiceId}`
- Vaulted card pre-filled
- Client confirms and pays the remaining balance
- Standard checkout flow — no off-session needed

**What's new vs RP:**
- Trigger for merchant-initiated charge is manual ("Charge Now" button) vs RP's scheduled firing
- Authorization copy ties to a specific invoice balance, not a recurring subscription
- Vaulting happens at deposit payment time (not a separate consent step like RP's consent flow)

RP already proves the vaulting + off-session charge pattern works in production. This is a new entry point into the same Stripe machinery.

---

## Approval Lifecycle (simplified — no token table, updated 2026-06-17)

**Decision: Skip the token table.** The estimate's own state (`approvedAt`) serves as the single-use guard. No separate token infrastructure needed.

### How it works

The link in the email is simply `/v/{documentId}/approve` — the `documentId` is the existing MongoDB ObjectId (same as current share links use).

| Event | Estimate state | What client sees |
|-------|---------------|-----------------|
| Merchant sends estimate | `approvedAt: undefined` | — |
| Client clicks "Review Estimate" | (unchanged — viewing is read-only) | Estimate view page + "Approve" button |
| Client clicks "Approve Estimate" | `approvedAt: timestamp`, `approvedBy: { email?, timestamp }` | Redirect to checkout |
| Client clicks link again after approval | `approvedAt` already set → detected | "Already approved" page + link to checkout |
| Estimate deleted / converted manually | Estimate doesn't exist or already has invoice | Error page: "contact sender" |

### Guards (replaces token validation)

| Concern | How it's handled without a token |
|---------|----------------------------------|
| **Single-use** | Check `approvedAt` — if set, reject. Atomic update with a condition (e.g., `findOneAndUpdate` where `approvedAt` is null) |
| **Expiry** | Check estimate's `sentAt` or `updatedAt` — if older than threshold (30 days?), show "link expired" |
| **Invalidation on edit** | If merchant edits and re-sends, new email has same link (same documentId). Page always shows latest estimate version — no stale data risk |
| **Double-approve race** | Atomic `findOneAndUpdate` with `approvedAt: { $exists: false }` condition — only one caller wins |
| **Enumeration** | DocumentId is a 24-char hex MongoDB ObjectId — not guessable (same security as current share links) |

### v2 token alternative (Juan)

If we ever need expiry without a DB table, use an **encryption key** approach — the link contains an encrypted payload with a timestamp. Server decrypts and checks the timestamp at request time. No DB lookup, no token table, no cleanup jobs. Simpler than a Postgres token table for the expiry use case.

---

## Effort Estimate (T-shirt)

| Component | Size | Notes |
|-----------|------|-------|
| ~~Token table~~ | — | Removed — using estimate's own `approvedAt` as guard |
| ~~`is-estimates` microservice~~ | — | Removed — using Parse Cloud Function instead |
| ~~`@is/common` shared package~~ | — | Removed — conversion logic lives in cloud function |
| Parse Cloud Function `approveEstimate` | S-M | Validates state, converts, saves invoice, returns invoiceId. Logic extracted from client code. |
| `/v/{documentId}/approve` route in is-web-app | S-M | New Next.js page, reuses existing public document components, adds "Approve" button + email input |
| Email template update | S | Add deposit amount + "Review Estimate" CTA to existing estimate email |
| "Approved" state on estimates | S | New field on estimate object, sync to clients, UI indicator |
| Merchant notifications (push + in-app) | S | New ALERT event type + in-app message, leverage existing `is-msg` infra |
| After-approval state (show invoice) | XS | If `approvedAt` set, show public invoice page — works out of the box |
| Checkout deposit mode (is-unifiedxp) | S-M | May already work, or needs a flag to show deposit vs full balance |
| **Total** | **M-L** | ~2-3 sprints with one dev (reduced from 3-4 with microservice approach) |

Note: This estimate covers only the estimate-accept-link flow (this doc). The deposit vaulting + future charge work (documented in [deposit-vaulting-flow-summary.md](./deposit-vaulting-flow-summary.md)) is additional effort on top of this.

---

## Open Questions

### Technical (ordered by severity)

1. **Conversion logic divergence** — mobile and web already differ (items, notes, fields). Do we align platforms, or just pick one for server-side?
2. **Checkout deposit mode** — does is-unifiedxp checkout already support showing just the deposit amount (not full balance)? Or does it need a new mode/flag?

### Product/Design (ordered by importance)

1. **What if estimate was already converted?** Client clicks link but merchant already did it manually. Show existing invoice? Error page? Redirect to checkout for the already-created invoice?
2. **"Approved" state UI** — how does merchant see this? New tab? Badge? Filter? Liz flagged this as needed, Seth hasn't designed it yet
3. **In-app message design** — notification row in activity feed? Banner on estimate detail? Both? (Need Seth's designs)
4. **Abandoned checkout follow-up** — if client approves but never pays, should we remind them? Email? How long to wait? (Probably v2)
5. **Signatures** — if estimate requires signature, does client still need to sign before the link converts? Or does clicking "Approve" count as acceptance?
6. **Multiple links per estimate** — can merchant regenerate? Does old link auto-invalidate?
7. **Link delivery** — embedded in existing estimate email? New email template? Both?
8. **PDF/Print on view page** — Seth shows these buttons. Do we have server-side PDF generation for estimates? Can we defer to v2?
9. **Analytics events needed** — `estimate_link_generated`, `estimate_link_clicked`, `estimate_approved_via_link`, `estimate_checkout_started`, `estimate_checkout_completed`, `estimate_checkout_abandoned`, etc.
10. **Email input on approval page** — Should the "Approve Estimate" page ask for the buyer's email before approving? Helps record `approvedBy` on the estimate and signals to the merchant if a different person approved. Not a hard identity gate — just metadata. (Liz leans yes)

---

## Ownership

**Ownership split (proposed — simplified 2026-06-17):**
- **Core team** — Parse Cloud Function `approveEstimate` (conversion logic + state management), "Approved" state on estimate model
- **PGrowth** — is-web-app approve page UI, redirect to checkout, card vaulting + deposit config on converted invoice
- **Shared** — merchant notifications (could be either team, uses existing infra)

**Key simplification:** No new microservice ownership question — cloud function lives in is-parse-server (Core-owned). Approve page lives in is-web-app (shared).
