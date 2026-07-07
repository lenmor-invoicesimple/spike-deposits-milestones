# Estimate Approval & Checkout — Specification (v2)

**Date:** 2026-07-07
**Status:** Draft — revised based on new designs + Seth/Juan direction
**Epic:** IS-11250 — Deposit Optimization v1
**Supersedes:** `../deposit-vaulting-flow/estimate-auto-accept-link-spec.md` (2026-06-17)
**PRD Reference:** [Deposits Optimizations PRD](https://everpro-tech.atlassian.net/wiki/x/JoBRS)

---

## What Changed from Prior Spec

| Aspect | Prior spec (June 2026) | This spec (July 2026) |
|--------|------------------------|----------------------|
| **Approval mechanism** | "Approve Estimate" button click | Signature (Premium) or "Approve & Pay" button (non-Premium) |
| **When conversion happens** | At approval (before payment) | **After payment succeeds** |
| **Checkout target** | Buyer pays the new invoice | **Buyer pays the estimate directly** |
| **Orphaned invoice risk** | Yes — abandoned checkout leaves unpaid invoice | **None — no invoice until payment succeeds** |
| **`approveEstimate` cloud function** | Converts estimate → invoice, returns invoiceId | Records approval/signature only; conversion triggered by payment webhook |
| **Checkout guard (`docTypeNotInvoice`)** | Untouched — buyer pays invoice | **Must be relaxed for estimates with deposits** |
| **Card vaulting** | In scope (setup_future_usage) | **Deferred** — pending legal/payment docs review (Seth, 2026-07-07) |
| **Identity capture** | Optional email input | Full name + signature (Premium) or implicit via payment (non-Premium) |
| **QR code** | Not in spec | On review page (likely for PDF/print scenario) |

---

## Overview

**Goal:** Merchant sends estimate with deposit → buyer receives email → reviews estimate → approves (via signature or payment) → pays deposit → estimate converts to invoice → merchant notified.

**Two buyer flows based on merchant tier:**

| Merchant tier | Buyer flow |
|---------------|-----------|
| **Premium** (signature enabled) | Email → Review page → "Sign" → Signature modal → "Continue To Deposit" → Checkout → Payment succeeds → Conversion |
| **Non-Premium** (no signature) | Email → Review page → "Approve & Pay" → Checkout → Payment succeeds → Conversion |

> **Key architectural decision (Juan/Seth, July 2026):** Buyer pays the deposit first, then the estimate converts to a payable invoice once the transaction is complete. This eliminates orphaned unpaid invoices from abandoned checkouts.

---

## Architecture & Service Boundaries

### Code Split

```
is-web-app/
  nextjs/app/(public)/v/[documentId]/
    page.tsx                              ← MODIFIED: public estimate review page
                                             Shows full estimate + deposit amount + QR code
                                             CTA: "Sign" (Premium) or "Approve & Pay" (non-Premium)
                                             If already converted → shows public invoice

is-parse-server/
  cloud/functions/
    approveEstimate.ts                    ← NEW (CHANGED ROLE vs prior spec)
                                             Records approval + signature data
                                             Does NOT convert to invoice (conversion is payment-triggered)
                                             Sets approvedAt + approvedBy + signature on estimate

is-services/
  packages/services/is-unifiedxp/         ← ALREADY EXISTS (Next.js customer-facing UI)
    src/app/checkout/[documentId]/
      page.tsx                            ← MODIFIED: must accept estimate documents (not just invoices)
      success/page.tsx                    ← MODIFIED: success triggers conversion

  packages/services/is-checkout/          ← ALREADY EXISTS (Express backend API)
    src/checkout/                          ← MODIFIED: payment intent creation must work for estimates

  packages/payments/payments-status/      ← ALREADY EXISTS
    src/payments-status/
      invoice-payments-status.ts          ← MODIFIED: relax docTypeNotInvoice guard for estimates with deposits
```

### Service Roles

| Service | Type | Role | Change from prior spec |
|---------|------|------|----------------------|
| `is-web-app` (exists, modified route) | Next.js | `/v/{documentId}` — estimate review page + CTA | **Changed:** No separate `/approve` route; CTA is "Sign" or "Approve & Pay" on existing public page |
| Parse Cloud Function (NEW) | Cloud Function | Records approval/signature; does NOT convert | **Changed:** No conversion logic here anymore |
| `is-unifiedxp` (exists, modified) | Next.js | Checkout UI — must accept estimates | **Changed:** Previously only invoices could reach checkout |
| `is-checkout` (exists, modified) | Express | Payment intent creation for estimate deposits | **Changed:** Must create PaymentIntent for estimate documents |
| `is-payments` (exists) | Express | Payment methods | Unchanged |
| Conversion trigger (NEW) | Webhook/callback | On payment success → convert estimate → invoice | **New:** This logic didn't exist before |

---

## Request Flow

### Premium Merchant (Signature Enabled)

```
Email "Review Estimate" CTA (shows DEPOSIT DUE: $500)
  → is-web-app: /v/{documentId} (public estimate review page)
    → fetch estimate from Parse
    → if already converted → show public invoice
    → otherwise → render estimate view + "Sign" button + QR code

"Sign" button tap
  → Signature modal (client-side):
    → Enter full name → cursive signature generated
    → "Sign" button confirms
    → Calls Parse Cloud Function: approveEstimate({ documentId, name, signature })
      → Sets approvedAt + approvedBy { name, signature, timestamp }
      → Returns success
    → Modal shows "All Done! Continue To Deposit"

"Continue To Deposit" button
  → Redirect to is-unifiedxp: /checkout/{documentId} (THE ESTIMATE — not an invoice)
    → Shows deposit amount + online fee
    → Payment methods (Card, Bank, PayPal, Apple Pay, Link, Google Pay)
    → "Pay Now" → is-checkout creates PaymentIntent for estimate

Payment success webhook/callback
  → Convert estimate → invoice (NEW conversion trigger)
    → Create invoice from estimate data
    → Record deposit as paid on new invoice
    → Set estimate as converted (link to invoice)
    → Send push notifications to merchant
    → Send invoice email to buyer with paid amount
```

### Non-Premium Merchant (No Signature)

```
Email "Review Estimate" CTA (shows DEPOSIT DUE: $500)
  → is-web-app: /v/{documentId} (public estimate review page)
    → Same as above but CTA is "Approve & Pay" (no sign step)

"Approve & Pay" button tap
  → Redirect directly to is-unifiedxp: /checkout/{documentId}
    → Same checkout flow as above

Payment success webhook/callback
  → Same conversion logic as above
```

---

## Estimate Lifecycle (Revised)

| State | Trigger | What merchant sees | Fields |
|-------|---------|-------------------|--------|
| **Open** | Estimate created / sent | In document list, deposit configured | `depositType` set, no `approvedAt` |
| **Approved** (Premium only) | Buyer signs | Badge/indicator on estimate | `approvedAt: timestamp`, `approvedBy: { name, signature }` |
| **Converted** | Deposit payment succeeds | Banner: "converted into invoice(s)" | `convertedTo: [invoiceId]` or equivalent link |

> **Note:** For non-Premium flow, there's no separate "Approved" state — the estimate goes directly from Open → Converted when payment succeeds. The act of paying IS the approval ("Approve & Pay").

---

## Guards Table (Revised — no token, updated for payment-first)

| Concern | How it's handled |
|---------|-----------------|
| **Single conversion (buyer flow)** | Check if estimate already has a linked invoice from buyer flow. If yes, redirect to existing invoice checkout or show "already converted" |
| **Double-payment race** | Atomic check in payment webhook — if conversion already triggered, skip. Stripe's idempotency key also helps |
| **Expiry** | Check estimate's `sentAt` or `updatedAt` — if older than threshold (30 days?), show "link expired" |
| **Invalidation on edit** | Page always shows latest estimate version — no stale data. If merchant edits after buyer signed but before payment, show warning? (TBD) |
| **Enumeration** | DocumentId is 24-char hex MongoDB ObjectId — not guessable (same as current share links) |
| **Doc limit bypass** | Conversion logic MUST check merchant's subscription doc limit before creating the invoice. Ref: `is-web-app/nextjs/app/(authenticated)/(core)/(documents)/actions/convert-document.action.tsx#L45` |
| **Estimate not payable (general)** | Only estimates with `depositType !== NONE` AND merchant has IS Payments enabled can reach checkout. payments-status guard relaxed ONLY for this case |

---

## What's New vs What Exists (Revised)

**Build new:**
1. ~~Parse Cloud Function converts estimate → invoice~~ → **Revised:** Cloud function only records approval/signature. Conversion is triggered by payment success webhook.
2. **Payment-triggered conversion logic** — on deposit payment success: create invoice from estimate, record deposit as paid, link estimate↔invoice, notify merchant
3. `/v/{documentId}` route modifications — estimate review page with "Sign" or "Approve & Pay" CTA + QR code (uses existing public page infra)
4. **Relax `docTypeNotInvoice` guard** in payments-status — allow estimates with deposits into checkout
5. **is-checkout modifications** — PaymentIntent creation must accept estimate documents
6. **is-unifiedxp checkout** — must render for estimates (show deposit amount, not full balance)
7. Email template update — add deposit amount + "Review Estimate" CTA
8. Push notifications — "estimate approved" + "invoice created"
9. Bidirectional banner (estimate→invoice(s), invoice→source estimate)
10. History events — Delivered, Opened, Approved, Converted to Invoice
11. Signature modal (client-side) — name input, cursive generation, confirm

**Already exists (reuse as-is):**
- `is-unifiedxp` checkout UI (payment form, all methods, success page)
- All payment methods (Card, Bank, PayPal, Apple Pay, Google Pay, Link)
- Online payment fee calculation
- Silent sync push infrastructure
- Push notification infra (ALERT type)
- "Request Client Signature" toggle + signature storage (Premium)
- QR code generation (if already exists on invoices — TBD confirm)

**Deferred to v2:**
- Card vaulting / `setup_future_usage` (pending legal/payment docs review — Seth, 2026-07-07)
- Merchant-initiated off-session charge for remaining balance
- ACH handling (delayed settlement)

---

## Effort Estimate (Revised)

| Component | Size | Notes |
|-----------|------|-------|
| Parse Cloud Function `approveEstimate` | **XS** (reduced) | Only records approval/signature — no conversion logic |
| Payment-triggered conversion logic | **M** | New — triggered by webhook. Creates invoice, records payment, links docs, notifies |
| Relax `docTypeNotInvoice` guard | **S-M** | Careful scoping — only estimates with deposits, not all estimates |
| is-checkout: accept estimate documents | **S-M** | PaymentIntent creation, eligibility checks for estimates |
| is-unifiedxp: checkout renders for estimates | **S** | May need deposit mode flag / different copy |
| `/v/{documentId}` review page modifications | **S-M** | CTA logic (Sign vs Approve & Pay), QR code, deposit display |
| Signature modal (buyer-side) | **S** | Client-side only — name → cursive → API call |
| Bidirectional banners + linking | **S** | UI + data model (convertedTo / createdFrom fields) |
| History events | **S** | Opened, Approved, Converted events in timeline |
| Push notifications | **XS** | Existing infra, new event type + copy |
| Email template update | **XS** | Add deposit amount + CTA |
| **Total** | **L** | ~3-4 sprints with one dev (larger than prior spec due to checkout pipeline changes) |

> **vs prior spec:** Effort increased from M-L to L because relaxing the checkout pipeline is deeper work than a standalone cloud function. The tradeoff is cleaner architecture (no orphaned invoices).

---

## Open Questions (Carried + New)

### Technical

1. **Conversion logic location** — Where does payment-success → create-invoice live? Options: (a) is-checkout webhook handler, (b) new Lambda triggered by payment event, (c) is-payments pipeline, (d) Juan's debounced SQS pipeline proposal. This is the most critical architecture decision.
2. **Checkout for estimates** — How much of is-checkout/is-unifiedxp assumes the document is an invoice? Need to audit. The `docTypeNotInvoice` guard is just one of potentially many invoice-only assumptions.
3. **Race condition: merchant edits estimate after buyer signs but before payment** — What happens? Block edits? Show buyer a stale version? Invalidate approval?
4. **Conversion logic divergence** — Mobile and web "Make Invoice" differ (items, notes, fields). Which logic does the payment-triggered conversion use?
5. **QR code generation** — Does this already exist for invoices? If so, reuse. If not, new work.

### Product/Design

6. **Multiple conversions** — Can one estimate spawn multiple invoices via this flow? Or is it one-and-done (first payment converts, subsequent visits show the invoice)?
7. **Estimate state after conversion** — Does estimate stay editable/re-sendable, or lock into a "Converted" state?
8. **Abandoned approval (Premium)** — Buyer signs but never pays. Estimate shows "Approved" but no invoice exists. Is this a valid permanent state? Should we remind the buyer?

---

## Ownership (Proposed)

| Component | Team | Notes |
|-----------|------|-------|
| Parse Cloud Function (approval/signature) | Core | Existing is-parse-server ownership |
| Payment-triggered conversion | Core + PGrowth | Critical path — needs both teams |
| Checkout pipeline changes (is-checkout, payments-status) | PGrowth | Existing checkout ownership |
| is-unifiedxp estimate rendering | PGrowth | Existing checkout UI ownership |
| Review page + CTA logic | Shared | is-web-app public routes |
| Banners, history, notifications | Core | Document lifecycle |
