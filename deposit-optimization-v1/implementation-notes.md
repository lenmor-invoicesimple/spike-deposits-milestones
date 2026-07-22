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

**Fix:** Skip the deposit field overwrite for Estimates only (`!isEstimate` guard). Invoice behavior is unchanged.

#### Why this was never a bug before

- **On Invoices:** Deposits are configured on a **separate screen** (`ManagePaymentScreen`). That screen calls `addDeposit()` / `updateDeposit()` in `services/parse/models/payment.ts` — these call a **Parse Cloud Function directly**, bypassing the local Realm→Parse sync entirely. By the time the user navigates back to the editor and `handleSave()` fires, Parse already has the correct deposit values. So fetching remote here returns up-to-date (non-stale) data.
- **On Estimates:** Deposits were never enabled before, so these fields were always `null`/`NONE` — overwriting null with null was a no-op.

#### Why the race only affects Estimates

The new `RequestDepositSection` edits deposit fields **inline on the same screen** that triggers `handleSave()`. The write goes to local Realm, but `updatePaymentsAndDeposit()` fetches Parse before the Realm→Parse sync has pushed the change. Remote is stale by definition.

#### Alternatives considered

| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| **Separate screen for Estimate deposits** (like Invoice's ManagePaymentScreen) | Eliminates race entirely — same proven pattern | Design explicitly chose inline toggle for simpler estimate UX | Rejected — contradicts design |
| **Call Parse directly from inline UI** (like `addDeposit()`) | Eliminates race, data hits Parse immediately | Invoice's `addDeposit()` creates a Payment record directly; Estimate deposits write to `setting.*` fields and rely on Parse Server's `handleDepositChange` hook to create the Payment record. Would need a new Cloud Function. Would make deposits the only inline editor field that bypasses Realm→Parse sync — diverges from how items, notes, tax, client all save. | Rejected — architectural outlier |
| **Skip overwrite for Estimates** (chosen) | Simplest, one-line conditional. Consistent with how all other inline editor fields work. Regular sync cycle still pulls remote via `createInvoiceFromParseInvoice()`'s `...setting` spread. | Slightly different code path for Invoice vs Estimate | **Chosen** |

#### The two save paths (Invoice vs Estimate deposits)

```
INVOICE (existing — Path A, direct Cloud Function):
  Editor → navigate → PaymentSchedulingScreen → ManagePaymentScreen
  → addDeposit() → Parse Cloud Function `invoiceAddPayment` → Payment record created directly
  → navigate back → handleSave() → fetchParse → remote is UP-TO-DATE ✓

ESTIMATE (new — Path B, setting fields + beforeSave hook):
  Editor → inline RequestDepositSection → write setting.depositType to Realm
  → handleSave() fires immediately → fetchParse → remote is STALE ✗
  → Realm→Parse sync pushes later → Parse Server `handleDepositChange` hook
    detects setting change → creates/updates/deletes Payment record automatically
```

#### How deposit Payment records get created (two paths, same result)

Both paths end up creating a Payment record in the `Payment` collection with `paymentType: DEPOSIT`. The difference is WHO creates it:

| | Path A (Invoice) | Path B (Estimate) |
|---|---|---|
| **Trigger** | User taps "Save" on ManagePaymentScreen | Setting fields sync to Parse via Realm→Parse |
| **Mechanism** | `addDeposit()` → `Parse.Cloud.run('invoiceAddPayment')` | `handleDepositChange` hook in `invoiceHooks.ts:480` fires on `beforeSave` |
| **Who creates Payment** | Client explicitly via Cloud Function | Parse Server hook reactively |
| **Timing** | Immediate — Payment exists before user returns to editor | Deferred — Payment created when sync pushes setting changes |
| **Result** | Payment record: `{paymentType: DEPOSIT, paymentMode: depositType, paymentValue: rate/amount}` | Same Payment record, same fields |

The `handleDepositChange` hook (`is-parse-server/cloud/collections/invoice/invoiceHooks.ts:480`):
1. Compares old vs new `setting.depositType/Rate/Amount`
2. If deposit was added/updated: finds existing DEPOSIT Payment → updates it, or creates new one via `addPayment()`
3. If deposit was deleted: calls `deleteUnPaidDeposit()` to remove the Payment record

**Why Estimate uses Path B (not Path A):**
- Path A (`addDeposit()`) calls a Cloud Function that directly creates a Payment record — this is the "separate screen" approach where the client manages the Payment lifecycle explicitly
- Estimate writes deposit config to `setting.*` fields inline (same screen as other fields like tax, discount, notes). All setting fields sync via the same Realm→Parse mechanism. The Parse Server hook picks up the change and handles the Payment record automatically.
- This means the Estimate deposit handlers (`onToggleDeposit`, `onChangeDepositType`, `onChangeDepositAmount`) are correct — they write to `setting.*` and let the existing hook do the rest. No need to call `addDeposit()` directly.

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

### Why Surcharge (Online Payment Fee) Works Out of the Box for Estimates

The only change needed to enable surcharge on estimates was removing the `docType !== DOCTYPE_INVOICE` early return in `invoice-payments-passing-fees-section.tsx:88`. Everything else in the surcharge stack is already docType-agnostic:

| Layer | What it checks | docType-aware? |
|-------|---------------|----------------|
| `passingFeesEligible` (Payments.store.ts:230) | PayPal/Stripe eligible with fees configured | No — provider state only |
| `isPaypalEligibleForPassingFees` | PayPal provider + invoice fees | No — looks at provider + setting |
| `isStripeEligibleAndOptedInForPassingFees` | Stripe provider + invoice fees | No — same |
| `setFeeRate` / `setInvoiceFeesType` (invoice.screen.tsx:456-485) | Writes to `setting.feeRate` / `setting.feesType` | No — writes to setting.* like any other field |
| Parse Server `settingSchema(isEstimate)` | Validates `feesType` + `feeRate` | No — both fields have `presence: false` with no isEstimate conditional |
| `paymentsStore.surchargeAmount()` | Calculates fee from `setting.feeRate` | No — works on any document with feeRate |
| `createParseInvoiceFromInvoice()` write path | Spreads `...settingWithoutNulls` to Parse | No — inclusive spread sends all fields |
| `createInvoiceFromParseInvoice()` read path | Spreads `...setting` from Parse | No — inclusive spread reads all fields |

The `docType !== DOCTYPE_INVOICE` guard was purely a UI render gate — it prevented the component from showing on estimates, but no underlying logic depended on doc type.

**Note:** `use-track-surcharge-awareness.ts:23` still has a `docType !== DOCTYPE_INVOICE` check. That hook is only for analytics tracking (fires a `trackScreenView` event). Left as invoice-only for now — estimate-specific tracking can be added later if needed.

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

## Design Decisions

### Provider Guard: `isAnyPaymentsProviderAccepting` (not `Eligible`)

The Estimate deposit section uses `isAnyPaymentsProviderAccepting` — meaning the user has actually connected a payment provider and can receive money. This is stricter than `isAnyPaymentsProviderEligible` (which only checks if they're in a supported country / opted in).

Rationale: showing a deposit toggle to users who can't collect payments yet is confusing. "Accepting" = Stripe or PayPal is live and ready.

**TODO for Invoice:** The existing Invoice `PaymentSchedulingSection` and `InvoicePaymentsSection` still use `isPaymentProviderEligible`. Consider tightening to `isAnyPaymentsProviderAccepting` when working on Invoice deposit improvements — evaluate whether Invoice has edge cases (e.g. mid-onboarding flows) that need the looser check.

---

## Architecture Pivot: Write to Payment Collection Directly

**Date:** 2026-07-08
**Source:** Slack thread with Musashi + Mark-Olivier (C062Z4LA84E/p1783528529503799)

### Context

Musashi confirmed that `setting.depositType/depositRate/depositAmount` are **deprecated fields**. The `handleDepositChange` hook that syncs these to the Payment collection is being removed. The recommended approach is to write deposits directly to the Payment collection via Parse Cloud Functions.

### Old Approach (deprecated — kept as reference)

Our initial spike wrote to `setting.*` fields:
```
User toggles deposit → write setting.depositType to Realm
→ handleSave() fires → Realm→Parse sync pushes setting changes
→ Parse Server `handleDepositChange` hook detects setting change
→ hook creates/updates/deletes Payment record reactively
```

**Problems:**
1. Relies on deprecated fields + hook that's being removed
2. Race condition with `updatePaymentsAndDeposit()` (required skip-for-estimates fix)
3. Mark-Olivier's bug (PR #4252): deleted deposits set fields to `0` instead of `null`, leaking during estimate↔invoice conversion
4. Inconsistent with how Invoice manages deposits (via Cloud Function)

### New Approach: `estimateSetDeposit` Cloud Function

Single idempotent Parse Cloud Function that handles all deposit CRUD:

```typescript
// Params:
{
  invoiceId: string;        // Parse objectId of the Estimate
  paymentMode: 0 | 1 | 2;  // NONE=0 (delete), PERCENT=1, FLAT=2
  paymentValue?: number;    // rate or flat amount (required if mode !== NONE)
}
```

**Logic (server-side):**
1. Validate user permissions + access
2. Fetch Estimate by `invoiceId`
3. Find existing deposit Payment (`paymentType: DEPOSIT, deleted: false`) for this document's `remoteId`
4. If `paymentMode === NONE`: soft-delete existing deposit Payment
5. If `paymentMode === PERCENT` or `FLAT`:
   - Existing deposit → update `paymentMode` + `paymentValue`
   - No existing deposit → create new Payment record
6. Call `updateDepositOnInvoiceSetting()` — keeps `setting.*` in sync for backward compat
7. Call `updatePaymentDetails()` — recompute balanceDue/paidAmount
8. Return `{ paymentId, success }`

**Reusable helpers (already in is-parse-server):**
- `findPayment()` — `cloud/collections/payment/findPayment.ts`
- `addPayment()` — `cloud/collections/payment/addPayment.ts`
- `updatePaymentByFieldName()` — `cloud/collections/payment/updatePayment.ts`
- `deleteUnPaidDeposit()` — `cloud/collections/payment/deletePayment.ts`
- `updateDepositOnInvoiceSetting()` — `cloud/collections/payment/updateDepositOnInvoiceSetting.ts`
- `updatePaymentDetails()` — `cloud/collections/invoice/utils/updatePaymentDetails.ts`
- `checkUserPermissions()` / `checkUserAccess()` — `cloud/collections/payment/utils/userValidation.ts`

### Why this is better for inline UI

| | Old (setting.* + hook) | New (Cloud Function) |
|---|---|---|
| Deprecated? | Yes — fields + hook being removed | No — same pattern as Invoice |
| Race condition | Yes — needs skip-for-estimates hack | No — writes to Parse directly |
| Idempotent | No — rapid toggles can create duplicates via hook | Yes — find-or-create on server |
| Client complexity | Write to Realm + hope sync works | Single async call, clear success/failure |
| objectId tracking | Not needed (hook handles internally) | Not needed (server finds existing) |

### How Invoice does it today (reference — separate screen)

```
ManagePaymentScreen → addDeposit() → resolves invoiceId from remoteInvoiceId
→ Parse.Cloud.run('invoiceAddPayment', { invoiceId, payment: { paymentType: DEPOSIT, paymentMode, paymentValue, label } })
→ returns { invoice, paymentId }

ManagePaymentScreen → updateDeposit() → needs objectId from existing Payment
→ Parse.Cloud.run('invoiceUpdatePayment', { payment: { objectId, paymentMode, paymentValue } })

ManagePaymentScreen → delete → needs objectId
→ Parse.Cloud.run('invoiceUpdatePayment', { payment: { objectId, deleted: true } })
```

**Why we can't just reuse `invoiceAddPayment` for inline UI:**
- `addDeposit` requires resolving `invoiceId` from `remoteInvoiceId` (extra API call)
- `updateDeposit` / `deleteDeposit` require `objectId` of the existing Payment record
- From inline UI, we'd need to fetch the Payment objectId first, then route to add/update/delete — 3 different code paths
- Multiple rapid taps could create duplicate deposits (no idempotency)

The new `estimateSetDeposit` consolidates all of this into one idempotent call.

### Concerns Raised (Slack thread 2026-07-08, C062Z4LA84E/p1783528529503799)

**Musashi:** "we would have to make requests to payments collection whenever user opens estimate screen... that would increase the request load on parse/mongo"

**Response:** We already call `invoiceGetPayments` on componentDidMount for both doc types, so no new call expected there. The only new call is `estimateSetDeposit` itself — user-initiated, low frequency.

```typescript
// invoice.screen.tsx:160 — already fires for estimates today
loadPayments(invoice.remoteId);
```

**Musashi:** "you might have to sync to local realm after create/remove requests"

**Response:** New Parse Cloud function accepts `invoiceId` and is idempotent — no need to track `objectId` in UI state (like Payment Scheduling does). For inline UI that might be toggled rapidly, server checks if deposit exists already, then proceeds to create/update/delete. Call `loadPayments()` after the cloud function returns to sync Realm.

**Mark-Olivier (PR #4252):** Deleted deposits set `setting.depositType/Rate/Amount` to `0` instead of `null`, leaking during estimate↔invoice conversion.

**Response:** If we write to Payment collection directly via cloud function, the client won't touch `setting.*` fields — so the stale-zeros bug won't be triggered from our flow. Cloud function still syncs back to `setting.*` for backward compat, but via masterKey so the `handleDepositChange` hook is skipped.

---

## Implementation Steps

### Step 1: Parse Server — Create `estimateSetDeposit` Cloud Function

**Where:** `/Users/lenmor/is5/is-parse-server/cloud/collections/invoice/functions/`

Create new file `estimateSetDeposit.ts`:
- Define the cloud function with params validation
- Reuse existing helpers (findPayment, addPayment, updatePayment, deleteUnPaidDeposit)
- Call `updateDepositOnInvoiceSetting()` for backward compat
- Call `updatePaymentDetails()` to recompute totals
- Register in `defineCloudFunctions.ts`

**Test:** Call from Parse Dashboard or curl to verify Payment record is created/updated/deleted

### Step 2: Mobile — Create API wrapper

**Where:** `/Users/lenmor/is/is-mobile/src/services/parse/models/payment.ts`

Add new function:
```typescript
export const setEstimateDeposit = async ({
  invoiceId,
  paymentMode,
  paymentValue,
}: {
  invoiceId: string;
  paymentMode: PaymentModes;
  paymentValue?: number;
}) => {
  return Parse.Cloud.run('estimateSetDeposit', { invoiceId, paymentMode, paymentValue });
};
```

### Step 3: Mobile — Rewire `RequestDepositSection` handlers

**Where:** `/Users/lenmor/is/is-mobile/src/features/documents/screens/invoice.screen.tsx`

Replace the current handlers that write to `setting.*`:

```typescript
// Old (deprecated):
onToggleDeposit = (enabled) => {
  invoice.setting.depositType = enabled ? DepositTypes.FLAT : DepositTypes.NONE;
  handleSave();
}

// New:
onToggleDeposit = async (enabled) => {
  await setEstimateDeposit({
    invoiceId: invoice.objectId,
    paymentMode: enabled ? PaymentModes.FLAT : PaymentModes.NONE,
    paymentValue: enabled ? defaultAmount : undefined,
  });
  // Refresh local state from Parse
}
```

Add loading state + error handling (like ManagePaymentScreen does with try/catch + alert).

### Step 4: Mobile — Remove `updatePaymentsAndDeposit` skip-for-estimates hack

**Where:** `/Users/lenmor/is/is-mobile/src/features/documents/navigators/invoice.navigator.tsx:1140-1146`

The `!isEstimate` guard is no longer needed because:
- We're not writing to `setting.*` locally anymore
- Deposit is saved directly to Parse via cloud function
- Remote will be up-to-date when `updatePaymentsAndDeposit` fetches

Can either remove the guard entirely or leave it (harmless).

### Step 5: Mobile — Handle local state refresh after cloud function call

After `estimateSetDeposit` returns, the local Realm invoice still has stale deposit info. Options:
- **Option A:** Call `loadPayments(remoteInvoiceId)` + update local setting from response (ManagePaymentScreen pattern)
- **Option B:** Optimistic UI — update local Realm immediately, let next sync confirm
- **Option C:** Re-fetch the invoice from Parse after the call

Recommend **Option A** — matches existing pattern, gives immediate feedback.

### Step 6: Verify end-to-end

- [ ] Toggle on → Payment record created in MongoDB
- [ ] Change type (FLAT↔PERCENT) → Payment record updated
- [ ] Toggle off → Payment record soft-deleted
- [ ] `setting.depositType/Rate/Amount` stay in sync (backward compat)
- [ ] Rapid toggles don't create duplicates
- [ ] App kill + reopen → deposit state survives
- [ ] balanceDue recomputed correctly

---

## What's Left for This Ticket

- [ ] **Step 1:** Create `estimateSetDeposit` cloud function (is-parse-server)
- [ ] **Step 2:** Create mobile API wrapper (`setEstimateDeposit`)
- [ ] **Step 3:** Rewire `RequestDepositSection` handlers to use cloud function
- [ ] **Step 4:** Remove or update `updatePaymentsAndDeposit` skip-for-estimates hack
- [ ] **Step 5:** Handle local state refresh after cloud function call
- [ ] **Step 6:** Verify end-to-end (Payment record CRUD, no duplicates, sync)
- [ ] Totals section: Add "Payments" row to estimates (guard at `invoice-totals-section.tsx:173`)
- [ ] Totals section: Show "Deposit Due" instead of "Balance Due" when deposit is configured
- [ ] Feature flag gating (Flagsmith flag for gradual rollout)
- [ ] Verify estimate PDF/preview shows deposit amount
