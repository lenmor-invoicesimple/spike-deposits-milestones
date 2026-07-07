# Implementation Notes — Estimate Deposit CRUD (Mobile)

**Date:** 2026-07-07
**Branch:** IS-10432-spike-deposits-and-milestones
**Codebase:** /Users/lenmor/is/is-mobile

---

## What Was Implemented

### 1. New Component: `RequestDepositSection`

**File:** `src/features/documents/components/invoice-sections/request-deposit-section.tsx`

A new inline UI component for the estimate editor:
- "Request Deposit" toggle (Switch)
- Type selector: Flat Amount / Percentage (styled button group)
- Amount input: numeric entry with `$` prefix (flat) or `%` suffix (percentage)
- 50% max cap enforced on percentage via `MAX_PERCENTAGE` constant

Props take deposit values + callbacks — component is docType-agnostic for future Invoice reuse.

### 2. Wired into Invoice Screen

**File:** `src/features/documents/screens/invoice.screen.tsx`

- Import added for `RequestDepositSection` and `DepositTypes`
- Rendered between `InvoiceTotalsSection` and `PaymentSchedulingSection`
- Gated by: `docType === DOCTYPE_ESTIMATE && paymentsEnabled && paymentProviderEligible`
- Three handlers added:
  - `onToggleDeposit` — sets `depositType` to FLAT (on) or NONE (off)
  - `onChangeDepositType` — switches between FLAT/PERCENT, resets the other field
  - `onChangeDepositAmount` — writes to `depositRate` (percent) or `depositAmount` (flat)

### 3. Surcharge / Online Payment Fee Enabled for Estimates

**Files:**
- `src/features/payments/payments-option/components/invoice-payments-passing-fees-section.tsx:88`
- `src/features/payments/payments-option/hooks/use-track-surcharge-awareness.ts:23`

Both had `docType !== DocTypes.DOCTYPE_INVOICE` guards — relaxed to also allow `DOCTYPE_ESTIMATE`.

### 4. Fixed `updatePaymentsAndDeposit()` Race Condition

**File:** `src/features/documents/navigators/invoice.navigator.tsx` (line ~1112)

**Problem:** Every `handleSave()` called `updatePaymentsAndDeposit()` which fetched the remote Parse invoice and overwrote local deposit fields with the (stale) remote values — before the local save had a chance to sync. This caused:
- Toggle closing immediately on tap
- Amount resetting to 0 on every keystroke

**Fix:** Removed the deposit field overwrite from `updatePaymentsAndDeposit()` entirely. The regular Realm↔Parse sync cycle handles pulling deposit fields from remote (via `...setting` spread in `createInvoiceFromParseInvoice`).

---

## Key Findings

### Data Model
- Deposit fields (`depositType`, `depositRate`, `depositAmount`) already exist on the shared `Invoice` Parse class `setting` object — no schema migration needed.
- Realm schema already has them as optional fields (`int`/`double`).
- `@invoice-simple/calculator`'s `getDeposit()` works for any document regardless of docType.

### Parse Server Validation
- `settingSchema(isEstimate)` in `invoiceValidation.ts` accepts `depositType`, `depositRate`, `depositAmount`, `feesType`, and `feeRate` for **both** invoices and estimates — no Parse Server changes needed.
- `handleDepositChange` hook creates a Payment record in the `Payment` collection when deposit is set — works for estimates too.

### Sync Architecture (Mobile)
- **Push:** `saveInvoice()` → Realm write → `syncFireAndForget()` → `performSync()` → `RInvoice.updateParseFromRealm()` → `createParseInvoiceFromInvoice()` which spreads `...settingWithoutNulls` to Parse.
- **Pull:** `RInvoice.updateRealmFromParse()` → `createInvoiceFromParseInvoice()` → `{ ...setting }` spread copies all fields from Parse to Realm.
- **Gotcha:** `updatePaymentsAndDeposit()` ran on every `handleSave()`, fetching stale remote values and overwriting local edits before sync could push them.

### Type System Issues
- `DepositRate` type from `@invoice-simple/common` is a preset literal union (0|10|20|30|40|50) — but Seth wants free-entry up to 50%. Used `as any` cast since Realm stores it as `int` regardless.
- `DepositType` is `0|1|2` literal — cast needed when assigning from a `number` variable.

### What Already Worked on Estimates (no changes needed)
- IS Payments section (Stripe/PayPal tiles, Configure Payment Methods, Manage Payment Settings)
- Request Client Signature toggle
- Show Financing Options / Acorn toggle
- Parse Server accepting deposit/surcharge fields

### is-packages Guards (NOT addressed — future ticket)
These affect the **web app** read/write path, not mobile:
- `is-packages/packages/parse-domain/src/mapping-functions/parseInvoiceToDocument.ts` — `getEstimateSettings()` does NOT extract deposit fields (only affects web app)
- `is-packages/packages/domain-invoicing/src/document/estimate.ts` — `EstimateSettings` type excludes deposit fields (only affects web app)
- These need fixing for web parity but mobile bypasses this layer entirely.

### Estimate ↔ Invoice Conversion (NOT addressed — future ticket)
- `src/services/realm/entities/invoice/use-cases.ts` line 61 and 121 — deposit fields are omitted when converting estimate↔invoice. This is intentional for manual "Make Invoice" but will need revisiting for the automated payment-triggered conversion.

---

## Files Changed (Summary)

| File | Change |
|------|--------|
| `src/features/documents/components/invoice-sections/request-deposit-section.tsx` | NEW — deposit toggle + type + amount component |
| `src/features/documents/screens/invoice.screen.tsx` | Added deposit section render + handlers |
| `src/features/documents/navigators/invoice.navigator.tsx` | Removed deposit field overwrite in `updatePaymentsAndDeposit()` |
| `src/features/payments/payments-option/components/invoice-payments-passing-fees-section.tsx` | Relaxed docType guard for estimates |
| `src/features/payments/payments-option/hooks/use-track-surcharge-awareness.ts` | Relaxed docType guard for estimates |

---

## What's Left for This Ticket

- [ ] Totals section: Add "Payments" row to estimates (currently invoice-only guard at `invoice-totals-section.tsx:173`)
- [ ] Totals section: Show "Deposit Due" instead of "Balance Due" when deposit is configured
- [ ] Feature flag gating (Flagsmith flag for gradual rollout)
- [ ] Verify deposit data survives app kill + full sync (Parse round-trip)
- [ ] Verify estimate PDF/preview shows deposit amount
