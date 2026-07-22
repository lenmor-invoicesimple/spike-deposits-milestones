# Enable Online Payments on Estimates (End-to-End)

**Status:** Design walkthrough done (2026-07-14); 13 decisions locked, several superseded by the design meeting. Open engineering spikes + 2 product questions remain before implementation planning.
**Epic:** IS-11250 Deposit Optimization v1
**Branch:** IS-10432-spike-deposits-and-milestones

> **⚠️ CONVERT-FIRST REOPENED (2026-07-20):** Juan/Sonya proposed reverting to convert-last (pay → charge → convert). The L2/L3 premise that justified convert-first was **falsified in code** (see [`checkout-flow-current.md`](./checkout-flow-current.md) for the current-state trace + trade-off table). This doc still describes convert-first as decided — left intentionally while options stay open. Treat the decision below as under review, not final.

> **⚠️ Reading order:** The **"Intended Design (from Figma flow)"** section is the current source of truth. Start with the **"SUPERSEDING UPDATE — Design Meeting Decisions"** block (it overrides several earlier numbered decisions and the original architecture assumptions). The final flow is **convert-first at "Pay Now"** — NOT the pay-first/webhook model described in some earlier sections. Earlier "Resolved Questions" and the "Proposed Fix"/"Files to Change" sections predate the design and are annotated where superseded but not fully rewritten.

---

## Summary

The "Add Payments" tile/toggle already renders on Estimates in the mobile app with no restrictions. Users can enable Stripe/PayPal on an Estimate today. **However, actual payment collection is blocked** — when a customer receives the estimate and tries to pay, every backend service rejects it because of a single docType gate.

**Design overlay:** the shipped design is deposit-driven — merchant sets a deposit on the estimate → buyer hits Pay Now → estimate synchronously converts to an invoice → buyer pays the invoice (deposit now, remaining balance later). See "Intended Design" for the authoritative flow and locked decisions.

---

## Current State

### What already works (no changes needed)

| Layer | Status | Notes |
|-------|--------|-------|
| Mobile UI — payments tile/toggle | Works | `InvoicePaymentsSection` has no docType gate |
| Mobile UI — surcharge/passing fees | Works | Guard removed in this spike |
| Payments.store.ts | Works | All eligibility checks are account-level, zero docType logic |
| Provider enable/disable APIs | Works | `PATCH /api/v1/accounts/me/enable-provider` is account-level |
| Payment method updates | Works | `updateDocumentPaymentMethods` has no docType check |
| Parse Server — deposit hook | Works, but ⚠️ **being deprecated** | `handleDepositChange` creates Payment records for both doc types. **Note (Musashi, 2026-07-08):** `setting.depositType/Rate/Amount` + this hook are deprecated; deposit CRUD is moving to a direct `estimateSetDeposit` cloud function. See [`implementation-notes.md`](./implementation-notes.md) "Architecture Pivot". Payability logic here should not build new dependencies on the hook. |
| Parse Server — setting validation | Works | `settingSchema(isEstimate)` accepts feesType/feeRate for both |
| Provider-specific checks (PayPal/Stripe SDKs) | Works | Only check provider state, currency, suppression — no docType |

### What's blocked

**Single gate:** `findNotPayableReason()` in `payments-status` package

```
is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58
```

```typescript
if (documentType !== 0) {
  return GeneralNotPayableReason.docTypeNotInvoice;
}
```

This blocks ALL downstream payment processing for estimates (docType=1).

---

## `findNotPayableReason` — Full Analysis

### Function signature

```typescript
export function findNotPayableReason({
  checkoutData,
  documentStatus,
  shouldSkipDeletedCheck,
  surchargeCents,
}: {
  checkoutData: CheckoutData | undefined;
  documentStatus?: PaymentsDocumentStatus;
  shouldSkipDeletedCheck?: boolean;
  surchargeCents?: number;
}): GeneralNotPayableReason | null
```

### All conditions checked (in order)

| # | Condition | Reason returned | Applies to Estimates? |
|---|-----------|-----------------|----------------------|
| 1 | No checkout data | `noCheckoutData` | Yes |
| 2 | `documentType !== 0` | `docTypeNotInvoice` | **This is the blocker** |
| 3 | Document deleted | `deleted` | Yes |
| 4 | `totalCents <= 0` | `totalZero` | Yes |
| 5 | `balanceDueCents <= 0` | `balanceDueZero` | Yes — need to verify estimates have balanceDueCents |
| 6 | `(balanceDueCents + surcharge) / 100 >= 1_000_000` | `balanceDueTooLarge` | Yes |
| 7 | Has pending/ACH orders | `pendingOrders` | Yes |

### `GeneralNotPayableReason` enum

```typescript
// is-services/packages/payments/payments-status/src/types/types.ts:126-134
export enum GeneralNotPayableReason {
  deleted = 'Document is deleted',
  totalZero = 'Document total is zero',
  balanceDueZero = 'Document balance due is zero',
  balanceDueTooLarge = 'Document balance due is too large',
  docTypeNotInvoice = 'Document is not an invoice',
  noCheckoutData = 'No checkout data',
  pendingOrders = 'Document has pending orders',
}
```

### Provider-specific reasons (checked separately, AFTER general check passes)

```typescript
export enum PaypalNotPayableReason {
  notAccepting = 'Document owner is not accepting payments yet',
  currencyMismatch = 'Paypal currency mismatch',
  paymentSuppressed = 'Document owner disabled payments for this document',
}

export enum StripeNotPayableReason {
  notAccepting = 'Document owner is not accepting payments yet',
  currencyMismatch = 'Stripe currency mismatch',
  paymentSuppressed = 'Document owner disabled payments for this document',
  recurringSeriesAmountMismatch = 'Recurring series amount does not match recurring payment amount',
}
```

---

## All Callers of `findNotPayableReason`

| Service | File | How it handles rejection | API Route |
|---------|------|--------------------------|-----------|
| **is-checkout** | `src/document/document-payable.ts:47` | Throws `InvalidRequestError` | `GET /checkout/eligibility/:accountId/:documentRemoteId` |
| **is-checkout** | `src/checkout/service.ts:65` | Returns `null` (no redirect URL) for `docTypeNotInvoice` | `GET /checkout/:accountId/:documentRemoteId/report` |
| **is-unifiedxp** | `src/utils/document-payable.ts:28` | Throws `InvalidStateError(DOCUMENT_NOT_AN_INVOICE)` → 500 error page | `/checkout/[documentId]` (Next.js page) |
| **is-stripe** | `src/services/payments-status/document-payable.ts:41` | Throws `InvalidRequestError` | Payment intent create route |
| **is-stripe** | `src/services/payments-status/document-payable.ts:13` | Throws `InvalidRequestError` (skips deleted check) | Payment intent confirm route |
| **is-abandoned-cart** | `src/utils/is-invoice-payable.ts:22` | Returns `false` | Internal scheduling logic |

### The checkout flow (how a customer reaches payment)

```
1. Mobile sends estimate → email to customer includes checkout link

2. Customer clicks link:
   GET /checkout/:accountId/:documentRemoteId
   → is-checkout fetches document from DB
   → calls findNotPayableReason (BLOCKED here for estimates)
   → if passes: redirects to is-unifiedxp

3. is-unifiedxp renders checkout page:
   /checkout/[documentId]
   → fetches document from Parse
   → calls findNotPayableReason AGAIN (redundant for estimates)
   → renders Stripe/PayPal payment buttons

4. Customer submits payment:
   → is-stripe payment-intent-create
   → calls findNotPayableReason AGAIN
   → creates Stripe PaymentIntent
   → returns client secret to frontend
```

**The docType check happens 3 times** (eligibility, redirect, and payment creation). All from the same shared function.

---

## Proposed Fix

> ⚠️ **PRE-DESIGN — scope likely smaller under convert-first (block A).** This section (and "Files to Change" below) was written for the pay-first model where the estimate must be payable through the *entire* pipeline (all 3 docType checks). Under the final convert-first flow, the PaymentIntent is created against the **invoice** (docType=0), so the is-stripe/is-paypal *payment-time* gates pass naturally — only the estimate **checkout-page eligibility** (`findNotPayableReason`, used by is-checkout/is-unifiedxp to render the page) needs to allow docType=1 up to the Pay Now click. The PayPal `validateDocumentPayable` and inline-payments validation may not need changes at all. **The "Convert-first gate scope" spike will confirm the exact minimal set** before this is finalized. Treat the options below as the shared-function mechanics, not the confirmed scope.

### Option A: Allow docType 0 and 1 (Invoice + Estimate)

```typescript
// invoice-payments-status.ts:58
if (documentType !== 0 && documentType !== 1) {
  return GeneralNotPayableReason.docTypeNotInvoice;
}
```

**Pros:** Minimal change, single line, all downstream services unblocked automatically.
**Cons:** Enum value name `docTypeNotInvoice` becomes slightly misleading. Need to rename or add a new reason.

### Option B: Allowlist approach

```typescript
const PAYABLE_DOC_TYPES = [0, 1]; // Invoice, Estimate
if (!PAYABLE_DOC_TYPES.includes(documentType)) {
  return GeneralNotPayableReason.docTypeNotPayable; // renamed
}
```

**Pros:** Cleaner, extensible, self-documenting.
**Cons:** Slightly more code. Need to rename enum value (breaking change for anyone checking string value).

### Option C: Feature-flag the estimate allowance

```typescript
const isEstimate = documentType === 1;
if (documentType !== 0 && !(isEstimate && featureFlags.estimatePaymentsEnabled)) {
  return GeneralNotPayableReason.docTypeNotInvoice;
}
```

**Pros:** Safe rollout, can disable if issues arise.
**Cons:** Feature flag dependency in a shared package. Needs flag infrastructure passed into `findNotPayableReason`.

### Recommendation: Option B + feature flag at the caller level

Use the clean allowlist in the shared function, but gate at the **caller** level (is-checkout eligibility endpoint) behind a feature flag. This way:
- Shared function is correct and extensible
- Rollout is controlled at the API layer
- No flag plumbing into the payments-status package

---

## Other Considerations

### `balanceDueCents` for Estimates

Need to verify that estimates produce a valid `balanceDueCents` in `CheckoutData`. If `balanceDueCents` is 0 or undefined for estimates (because they haven't been "accepted" / no payments recorded), the function would return `balanceDueZero` even after removing the docType gate.

**Check:** How does `formatDocumentData()` in is-checkout compute `balanceDueCents` for an estimate?

### Parse Server inline payments validation

```typescript
// invoiceValidation.ts:724-738
payments: {
  length: (value, attributes) => {
    const isEstimate = attributes['docType'] === DocTypes.DOCTYPE_ESTIMATE;
    return { is: isEstimate ? 0 : undefined };
  }
}
```

This enforces `payments` array length === 0 on the **inline** payments field of the Parse Invoice object. This is the **legacy** inline payments array — NOT the separate `Payment` collection. Online payments (Stripe/PayPal) and deposits both write to the separate `Payment` collection via `invoiceAddPayment` / `handleDepositChange`, so this restriction doesn't block them.

**However:** if online payment confirmation writes a record to the inline `payments` array (as a "payment received" entry), that would fail validation. Need to verify whether the payment confirmation flow writes to inline `payments` or only to the `Payment` collection.

### Estimate acceptance flow

The payment system has **no concept of "accepting" an estimate**. Estimate acceptance (client signs/approves) is handled by the web app's public document view (`/v/{documentId}`), not through the payment checkout system.

**Product question:** Should paying a deposit on an estimate implicitly "accept" it? Or are these separate actions?

### `is-abandoned-cart`

Currently returns `false` (not payable) for estimates. If we enable estimate payments, abandoned cart reminders would start firing for estimates too. May need a separate docType check in abandoned-cart to avoid spamming estimate recipients.

---

## Resolved Questions

### 1. `balanceDueCents` for Estimates — WORKS CORRECTLY

**Finding:** The `@invoice-simple/calculator` package computes `balanceDueCents` correctly for estimates.

- `formatDocumentData()` in `is-checkout/src/document/document-payable.ts` calls `calculate(document, payments)`
- For an estimate with no payments: `balanceDue = getInvoiceTotal(doc) + 0 - 0 = full total`
- The calculator operates on `items`, `setting` (tax/discount/shipping) — no docType dependency
- External payments are fetched from the `Payment` collection by `invoiceRemoteId` (works for any document)

**However:** The Estimate type does NOT have `depositType`/`depositRate`/`depositAmount` in its domain settings (only `InvoiceSettings` has those). So `isDepositEligible` returns false and balanceDue is always the full total. **To support deposits on estimates, the Estimate settings type must be extended.**

### 2. Payment Confirmation → Inline Payments Array — CONFIRMED PROBLEM

**Finding:** YES, both Stripe and PayPal payment confirmation write to the inline `payments` array.

**The flow (both providers):**
1. Webhook fires (e.g., `payment_intent.succeeded`)
2. `addPaymentToDocument()` → `tryAddingPaymentToInvoice()` → `addPaymentToInvoice()`
3. Calls `Parse.Cloud.run('invoiceAddPayment', ...)` with `useMasterKey: true`
4. `invoiceAddPayment` creates a **Payment collection record** (always succeeds)
5. THEN pushes payment into the **inline `payments` array** on the document
6. Calls `parse.save(invoice)` → triggers `beforeSaveInvoice` → `fixAndValidateParseObj()`

**For Estimates (docType=1):**
- **In production** (`NODE_ENV=production`): `fixPayments()` **silently resets** `payments = []` and `partialPayment = 0`. The Payment collection record persists, but the inline array is wiped.
- **In non-production** (staging/dev): Validation **throws an error**, causing the save to fail entirely.

**Impact:** Data inconsistency — Payment record exists in the collection but Estimate never reflects it inline. The inline array is used for calculating `paidAmount` in some contexts, so the estimate would always show as unpaid via inline logic.

**Fix needed:** Either (a) remove the `payments.length === 0` validation for estimates in Parse Server, or (b) skip the inline array write for estimates and rely solely on the Payment collection (which the checkout page already does via `getPayments()`).

> ⚠️ **LIKELY OBVIATED by block A (convert-first).** Because conversion happens on the Pay Now click *before* payment, the payment is recorded against the **invoice** (docType=0), not the estimate — so `fixPayments()` never sees a paid estimate. This inline-payments problem only bites if a payment can land while the doc is still docType=1, which the final flow avoids. Keep on the radar only for the (blocked) case of a payment arriving pre-conversion. Verify in the "Convert-first gate scope" spike.

### 3. Estimate Acceptance vs Payment — DECOUPLED IN CODE, COUPLED BY DESIGN (via Pay Now)

**Current code:** Estimate acceptance (client signs/approves via the public document viewer `/v/{documentId}`) is completely separate from the payment system. The checkout flow has no concept of "accepting" an estimate.

**Design decision (RESOLVED — block A):** Approval/conversion is triggered by the buyer's **"Pay Now" click** (synchronous conversion before payment), not by payment success. Reaching Pay Now still requires intent to pay, so payment remains the effective gate (no free approval). On conversion the estimate is marked approved/`convertedTo` and a push fires to the merchant. The code decoupling is the gap to close.

### 4. Abandoned Cart — WOULD START FIRING

If estimates become payable, `is-abandoned-cart` would start sending reminder emails for unpaid estimates. Currently returns `false` (not payable) for non-invoices. Decision: either exclude estimates via explicit docType check, or embrace it as a feature.

### 5. Feature Flag — FLAGSMITH AT CALLER LEVEL

Existing pattern in the codebase uses Flagsmith. Recommend gating at the `is-checkout` eligibility endpoint (the entry point for checkout URLs) rather than inside the shared `payments-status` package.

### 6. Deposit-Only vs Full Payment — REQUIRES TYPE EXTENSION (domain type only; Realm already has fields)

To support "Pay Deposit" on estimates, `getDeposit()` must return a non-null deposit, which requires `depositType`/`depositRate`/`depositAmount` on the estimate settings. The state of these fields is **layer-dependent**:

- **Shared domain type** (`@invoice-simple/domain-invoicing`): `EstimateSettings` = `BaseDocumentSettings & { estimateSignatureRequired, estimateSignedText, estimateSignedAt }` — does **NOT** include deposit fields. Only `InvoiceSettings` declares `depositType`/`depositRate`/`depositAmount`/`feesType`/`paymentSuppressed`/`stripePaymentSuppressed`. **This is the type that must be extended.**
- **Mobile Realm schema** (`is-mobile/src/services/realm/entities/invoice-setting/schema.ts`): a **single unified** `InvoiceSetting`/`RInvoiceSetting` is used for both docTypes and **already includes** `depositType` (L42/81), `depositRate` (L43/82), `depositAmount` (L44/83), `feesType` (L45/84), `feeRate` (L46/85). So mobile persistence can already physically hold deposit fields on an estimate — the gap is purely the shared TS type + intentional stripping (see #8).

The checkout page already has deposit UI (`PaymentDepositDetails`) that activates automatically once the calculator returns a deposit amount.

### 7. Estimate→Invoice Conversion Creates a NEW Document — Payment Records Orphaned

**Correction to a prior assumption:** conversion does NOT flip `docType` in place. There is **no server-side conversion cloud function** in `is-parse-server` (confirmed against `cloud/defineCloudFunctions.ts`; despite its name, `invoiceConversion.ts` is validation auto-fix, not doc-type conversion). Conversion is entirely **client-driven** and produces a **brand-new invoice document**:

**Mobile** — `is-mobile/src/services/realm/entities/invoice/use-cases.ts:22-82` (`makeInvoiceFromEstimate`):
- Starts from `defaultInvoice(...)` which mints a **fresh `remoteId: uuid()`** and `id: null` (`models.ts:220-249`). Nested `client`/`company`/`items`/`payments` all get **new remoteIds** via `copyWithNewRemoteId`.
- The **original estimate is preserved** and marked closed (`setting.fullyPaid = true`), then the new invoice is saved separately (`invoice.navigator.tsx:1057-1063`).

**Web (legacy client)** — `InvoiceModel.ts:1296-1312` (`_convertEstimateData`) + `estimateToInvoice.ts`: same pattern — new document with regenerated remoteIds, original estimate marked `fullyPaid = true`.

**Web (Next.js)** — `convert-document.action.tsx:45` → `createDocumentInParse(...)`: same — new invoice number/id, original marked closed via `markEstimateAsClosed`.

**Impact on Payment records:** Payment collection records — including deposit records from `handleDepositChange` — are keyed by `invoiceRemoteId` (`cloud/collections/payment/addPayment.ts:24,79`). Because conversion mints a **new remoteId** for the invoice, any Payment records collected against the estimate would remain pointed at the **estimate's** remoteId.

> ✅ **RESOLVED / OBVIATED by block A (convert-first at Pay Now).** In the final flow, conversion happens **before** any payment, so there is never a payment sitting on the estimate to orphan. The new-remoteId invoice is created first, then the PaymentIntent is issued against that invoice — the deposit is recorded on the invoice from the start. The "carry payments over" (option a) work is **not needed**. This section documents the pre-design finding and why the naive "pay-first" model would have required carryover; the convert-first decision sidesteps it. The cloud-function conversion may keep minting a new remoteId (consistent with current client behavior) — that's fine since no payment precedes it.

### 8. Payment/Fee/Deposit Settings Are Inconsistently Carried on Conversion

The three client conversion paths disagree on which payment-related settings survive:

| Setting | Mobile (`use-cases.ts:45-62`) | Legacy client (`InvoiceModel.ts`) | Next.js (`document-transformation-utils.ts:64-114`) |
|---------|-------------------------------|-----------------------------------|------------------------------------------------------|
| `paymentSuppressed` | Carried | Carried | Carried |
| `stripePaymentSuppressed` | Carried | Carried | Carried |
| `feeRate` | Carried | Carried | Carried |
| `feesType` | **Dropped** | Carried | **Reset to account default** |
| `depositType/Rate/Amount` | **Dropped** (explicit `omit`) | Carried | **Carried** (only stripped in `duplicate` branch) |

**Notable bugs/inconsistencies:**
- **Mobile drops `feesType` but keeps `feeRate`** → surcharge amount survives while surcharge mode is lost (the bug flagged in the gap analysis).
- **Deposit fields:** mobile explicitly strips them; legacy client and Next.js carry them. Once deposits-on-estimates ships, these three paths must be aligned — otherwise a deposit configured on an estimate silently vanishes on mobile conversion but persists on web.
- Any deposit fields carried over on web are currently inert because the estimate can't compute a deposit until #6 lands — but post-#6 the divergence becomes user-visible.

**Recommendation:** define a single canonical "convert settings" transform (which payment/fee/deposit fields carry) and apply it consistently across all three clients — ideally server-side alongside the payment-carryover decision in #7.

---

## Intended Design (from Figma flow)

The designs define a **deposit-triggered auto-conversion** flow that changes the target architecture. It is *not* the "manual convert" path the code currently implements — deposit payment itself drives the conversion.

### ⚠️ SUPERSEDING UPDATE — Design Meeting Decisions (Liz, 2026-07-14)

Liz posted the design-meeting decision ([Slack](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869)) which **overrides several decisions below**. Read this block first; the numbered decisions that follow are annotated where superseded.

**A. Conversion = CONVERT-FIRST at "Pay Now" (reverses decision #3; obviates #4 and #5).**
- Conversion happens the moment the buyer hits **"Pay Now"**, *before* payment is attempted. **Invoice ID (not estimate ID) is passed to Stripe/PayPal** — required to supply L2/L3 interchange fields.
- Consequences: payment is against the **invoice `remoteId` from the first PaymentIntent**, so:
  - **§7 payment-carryover problem disappears** — decision #4 (re-key Payment record) is **no longer needed**.
  - **ACH-timing (decision #5) is moot** — the invoice already exists before payment; settlement timing doesn't gate conversion.
  - Conversion is **synchronous** (Parse Cloud Function call on Pay Now) → needs a **loading state on "Pay Now"** to cover latency. Not a payment-success webhook.
  - Likely **shrinks the gate work**: the PaymentIntent is created against the invoice (docType 0), so is-stripe/is-paypal *payment-time* docType gates pass naturally. Only the estimate **checkout-page eligibility** (`findNotPayableReason`) needs to allow docType 1 so the deposit checkout can render. ⚠️ verify.

**B. GLOBAL payments settings, not document-level (confirms decision #9's reversal). ✅ RESOLVED — GLOBAL (Liz-confirmed, 2026-07-14).**
- Estimate is payable if **Deposit set AND PayPal/Stripe enabled globally** — no document-level toggle, surcharge inherits global, existing deposit-estimates payable retroactively.
- **Full decision record, comparison scorecard, technical findings, and the chronological back-and-forth are in [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md).**
- **The choice reduced to one question:** is a deposit online-fee locked at 100% (no per-estimate discount) acceptable? Liz: *"I'm comfortable with that trade off."* → **Global.** Global wins on ~7 dimensions (less UI work, no conversion-transform work, cleaner model, uniform retroactive surcharge, near-useless per-estimate toggle under the 1:1 lock); document-level's only advantage was per-estimate fee tunability, which Liz gave up.
- **One work item this adds to the global path:** the estimate checkout fee **preview must read the live account setting** (not the estimate's stored `feesType`), so it matches the post-conversion charge — required because retroactively-payable old estimates carry a frozen `feesType`.

**C. Post-conversion link → PUBLIC ESTIMATE URL, not invoice checkout (reverses decision #10).**
- If the buyer returns via the old email link after conversion, they are redirected to the **public estimate URL** (read-only), NOT the checkout. QR code + Pay Deposit CTA are hidden. Rationale: prevents *wrong-customer-gets-wrong-invoice*. Decision #10 (redirect to child-invoice checkout) is **wrong** — do not auto-drop the buyer into paying the remaining balance.

**D. Single-conversion lock — CONFIRMED & STRENGTHENED (enhances #11, #6).**
- Applies to **manual convert too**: the "Convert to Invoice" button is **hidden after first conversion**; user is prompted to **duplicate** the estimate instead if they want to reuse it. Once converted (manually OR by buyer), the estimate is **locked: no further conversions, no further payments.**
- **1→N is explicitly off the table** for v1 (reuse = duplicate). The `convertedTo` field (decision #13) stays an array for future-proofing but v1 semantics are strictly one. The "open re-conversion fork" from #11 is **resolved: no re-conversion.**

**E. New items from the meeting (not previously captured):**
- **Retroactive payability**: all existing estimates with deposits become payable on ship → rollout/migration consideration + email-rendering check.
- **Failure handling**: (a) *payment* fails but conversion already happened → buyer retries on the invoice checkout in-session; (b) *conversion* fails (backend error) → block payment confirmation, show error UI, redirect target TBD (back to estimate checkout?).
- **Email**: same email type, **conditionally rendered** for estimate vs invoice. Current email validates payable status; estimates now qualify, so rendering must be re-checked. ⚠️ **ASSIGNED TO LENMOR to confirm.**
- **UI**: checkout tab label "Payment & Estimate" → "Payment & Invoice" once converted; success/confirmation screen reflects invoice context post-conversion.

---

### Locked Decisions (2026-07-14 walkthrough)

Reconciled against the earlier Confluence plan ([Plan - Deposits Optimization](https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1243119644/Plan+-+Deposits+Optimization)). **NOTE: several of these are superseded by the design-meeting block above — see annotations.**

1. **Trigger = deposit payment only.** Conversion fires only when a deposit is configured and the buyer pays it. No other-payment or manual-button trigger in scope for this flow.
2. **Payment is the ONLY approval gate.** There is no approve-without-pay path for buyers. This is the key departure from the old plan and it **eliminates the entire token/approval-security layer**:
   - No approval token, no expiry, no link-consumption logic.
   - No griefing risk — converting costs real money, so a bad actor won't do it for free.
   - The three identities the old plan tracked (estimate client → approver → payer) **collapse to one**: the payer, whose identity (email, name, payment method) comes authenticated for free from Stripe/PayPal. `approvedBy`/`approvedAt` are derived from the payment event.
3. **~~Architecture = pay-on-estimate, convert-after-payment.~~** ⚠️ **SUPERSEDED by block A above.** Design meeting reverted to **convert-first at "Pay Now"** (synchronous cloud function, invoice ID to Stripe/PayPal for L2/L3), NOT a payment-success webhook. The checkout still *shows* estimate context up to Pay Now, but conversion fires on the click, before payment.

### Is the old plan's Parse Cloud Function still usable? — YES, repurposed

The old plan's `approveEstimate` cloud function (estimate→invoice transform + atomic `approvedAt` guard + invoice save + push notifications) is **fully reusable as the conversion primitive**. What changes across the three candidate models is only its **caller and timing**:

| | Old plan | Interim (pay-first) — SUPERSEDED | **FINAL (block A): convert-first at Pay Now** |
|---|----------|----------------------------------|-----------------------------------------------|
| **Who calls conversion** | Public "Approve Estimate" button | Stripe/PayPal payment-success webhook | Buyer's **"Pay Now"** click (synchronous cloud fn call) |
| **When** | Before payment | After payment confirmation | **Before payment** (to mint invoice ID for L2/L3) |
| **Approval semantics** | Button click | Deposit payment success | **Pay Now click** (payment still the gate to reach checkout) |
| **Estimate payable?** | No — pays the already-created invoice | Yes — pays the estimate | Checkout *renders* on the estimate, but PaymentIntent is against the **invoice** |
| **Payment carryover** | N/A | Required (§7) | **N/A** — payment is against invoice `remoteId` from the start |

**FINAL model (block A):** convert-first at Pay Now keeps the security win of the pay-first model (no tokens — you still can't reach Pay Now without intending to pay) while **eliminating** the payment-carryover problem (payment is against the invoice from the first PaymentIntent) and the ACH-timing problem. The only estimate-payability requirement is that the **checkout page can render** for a deposit-estimate — i.e. `findNotPayableReason` must allow docType=1 up to the Pay Now click; the is-stripe/is-paypal payment-time gates then see an invoice (docType=0) and pass. ⚠️ verify (see spike "Convert-first gate scope").

**Still open (see below):** the atomic-guard mechanics now that the **Pay Now click** (potentially fired twice on double-click / retry) is the caller — the `findOneAndUpdate` where `approvedAt`/`convertedTo` is null must still ensure only one conversion wins.

### Locked Decisions — Payment Mechanics (2026-07-14 walkthrough, cont.)

4. **~~Payment carryover = re-key the deposit Payment record.~~** ⚠️ **OBVIATED by block A.** With convert-first, payment is against the invoice `remoteId` from the first PaymentIntent — there is no carryover to solve. Drop this and the "Payment record re-keying" spike.
5. **~~ACH = treat as resolved on initiation.~~** ⚠️ **MOOT under block A.** The invoice exists before payment, so settlement timing no longer gates conversion. (RP ACH behavior may still be relevant for the *remaining-balance* vaulting work, but that's deferred — decision #7.)
6. **1→many invoices = 1:1 in v1 (design is future-proofing).** One deposit payment produces one invoice (deposit paid + remaining balance on that same invoice). Build the estimate→invoices linkage as a **list** to future-proof for milestones, but only ever create one invoice in v1. The "Invoices Created" modal renders a single entry for now.

### Locked Decisions — Scope & UX (2026-07-14 walkthrough, cont.)

7. **Vaulting / remaining-balance charging = DEFERRED.** May be part of v1 eventually, but NOT in the immediate next pieces of work. This flow stops at: pay deposit → convert → invoice with remaining balance due. The buyer pays the remaining balance later via normal invoice checkout. The "Save Payment Method" checkbox, OTP/PIN, and merchant-/buyer-initiated charge (all TBD in the old plan) stay out of scope for now.
8. **Estimate edit/delete after send = allow, use latest version (v1), but KEEP AS OPEN PRODUCT QUESTION.** For now the buyer's checkout reflects the current estimate at payment time; delete → link unavailable; the "Created from a deleted estimate" invoice-history state handles delete-after-conversion. This affects amount-integrity and interacts with other cases — flagged for product/design to confirm.

### Online Payment Fee — INVESTIGATED: buyer is charged TWICE (this is current behavior)

**Answer:** The surcharge is computed **per-payment on the amount being charged now** (`getInvoiceBalance`), NOT once on the full total. So a deposit + remaining-balance lifecycle incurs **two separate surcharges**:
- Deposit $500 → fee grossed-up on $500 (≈ $17.45)
- Remaining $1,500 later → a **fresh** fee grossed-up on $1,500

**This is exactly how the existing Payment Scheduling deposit flow already behaves** — so deposit-on-estimate inherits it with **zero new fee logic**. There is no mechanism to charge the fee once on the total and split it; "fee once" would require new logic.

**Key code refs:**
- `is-packages/packages/calculator/src/paymentFees/getInvoiceSurcharge.ts:19-53` — grosses up `getInvoiceBalance(invoice, payments)`, not the total. `div(100 - percentage)` gross-up + `feeRate` fraction.
- `is-packages/packages/calculator/src/invoice/getInvoiceBalance.ts:51-87` — returns the **deposit amount** when a deposit is set; else nearest installment / remaining balance.
- `getInvoicePaidSurchargeSum.ts:15-37` — prior paid surcharge is **added back into** the remaining balance (merchant nets full principal), then a fresh surcharge is layered on the 2nd payment.
- Fee stored as `surcharge` on each **Payment record** (`is-parse-server/.../payment/addPayment.ts:22,77`), visible in history via `parsePaymentToCommonPayment.ts:16`.

**Correction to prior "drops feesType" finding:** the converted invoice **DOES** charge the online fee. Mobile `makeInvoiceFromEstimate` drops the estimate's `feesType` override, but `defaultInvoiceSetting` re-supplies `feesType` from the account default (`repository.ts:97`), and `feeRate` is carried over. Server-side surcharge additionally double-gates on both `invoice.setting.feesType === SURCHARGE` AND the live account `payment.feesType === SURCHARGE` (`stripe-fees.ts:38-45`, `paypal-fees.ts:37-43`).

**DECIDED (2026-07-14):** Accept the two-fee behavior — match current Payment Scheduling / deposit behavior. **Zero new fee logic.** Buyer pays an online fee on each online payment (deposit, then remaining balance). Revisit only if it becomes a support/trust complaint.

### Locked Decisions — Estimate Lifecycle After Conversion (2026-07-14 walkthrough, cont.)

Context: the "what happens to the estimate after pay/convert" problem left open last week. Reference [Slack thread](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1783612318324139) with Liz + Juan, and Confluence [Adding Payment Features to Estimate](https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1319108664/Adding+Payment+Features+to+Estimate).

9. **Use GLOBAL/account-level payment features for estimates. ✅ FINAL (Liz-confirmed 2026-07-14) — see [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md).** The spike initially leaned document-level, but under the 1:1 conversion lock the per-estimate toggle is near-useless, global avoids the conversion-transform correctness work, and Liz accepted the one trade-off (deposit fee locked at 100%). **No per-estimate surcharge UI, no per-estimate payment toggle.** The reasoning below (document-level parity) is retained only as historical context for the fork. ~~Estimates reuse the existing per-document payment tile/toggle, surcharge, and payment-method config — already saved to Mongo per-document (mobile). Web just needs guards removed to render them on estimates. Infrastructure is NOT rebuilt. Because payability is per-document (`paymentSuppressed`/`stripePaymentSuppressed`), not global/account-level, this is what makes the post-conversion link behavior cheap and estimate-handling "closer to how we handle invoice."
   - Known gate fixes needed (from Slack): `findNotPayableReason` (shared, blocks Stripe + checkout) and `validateDocumentPayable` in is-paypal (separate hardcoded gate). Plus places to allow docType=1 and ensure data persists through to checkout.
   - Web surcharge render is missing but the data is already wired to `invoice.settings` — low effort.~~ (end historical context)

10. **~~Post-payment link = redirect to child invoice's checkout.~~** ⚠️ **SUPERSEDED by block C.** Design meeting decided the old link redirects to the **public estimate URL (read-only)**, NOT the invoice checkout — to prevent wrong-customer-gets-wrong-invoice. QR + Pay Deposit CTA hidden post-conversion.

11. **Lock estimate to a SINGLE conversion in v1 — CONFIRMED (block D).** Once converted (manually or by buyer), no further conversions and **no further payments**. Manual "Convert to Invoice" button hidden after first conversion; user prompted to **duplicate** to reuse. 🔱 **FORK RESOLVED: no re-conversion in v1** (1→N off the table; reuse = duplicate).

12. **Double-payment guard = key non-payability off `convertedTo`/`approvedAt` (DECIDED).** Checkout treats an estimate as non-payable whenever `convertedTo` is set (or `approvedAt` exists), **independent of the toggleable `paymentSuppressed` field** — so a merchant toggling suppression back on cannot re-expose payment on an already-converted estimate. Reuses the tracking field from decision #13; no separate immutable flag needed. The atomic `approvedAt` guard at conversion still blocks double-conversion; setting `paymentSuppressed` at conversion is optional belt-and-suspenders for UI, but the checkout gate is the source of truth.

13. **Conversion tracking field = `convertedTo: string[]` on the estimate (Juan's proposal).** Add an **indexed** `convertedTo` array of invoice remoteIds on the document (Parse/Mongo — same collection, preferred over Postgres as it's a core feature). Indexed so the original estimate is searchable by invoiceId. **Requires a Realm migration** on mobile. Modeled as an array to future-proof for 1→many even though v1 is 1:1 (decision #6). This is also the field decision #12(b) can key non-payability off of.

### Needs a spike (code investigation before spec)

*(Both prior spikes — RP ACH-initiation and Payment-record re-keying — are OBVIATED by the convert-first decision, block A. Payment is against the invoice from the start; no carryover, no ACH-gated conversion.)*

Assigned to Lenmor by the design meeting:
- **~~Global vs document-level payability (block B fork).~~** ✅ **RESOLVED — GLOBAL (Liz-confirmed 2026-07-14).** Full record in [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md). "Block B findings" section below retained as the in-doc detail. Remaining global-path work: relax docType gate, **live-account surcharge preview**, `convertedTo` field + lock, do NOT expose per-estimate surcharge/payment UI, email render check.
- **Email estimate-vs-invoice rendering** — confirm the send/email flow renders correctly once estimates qualify as payable (same email type, conditional render). The email currently validates payable status; estimates will now pass.
- **Convert-first gate scope** — verify that creating the PaymentIntent against the invoice (docType 0) means only `findNotPayableReason` needs to allow docType 1 (for the estimate checkout *page* to render), and the is-stripe/is-paypal payment-time gates pass naturally. Could shrink Phase 1.
- **Synchronous conversion failure handling** — the Pay Now → cloud-function-convert → payment path needs the failure UX (conversion-fail blocks payment; payment-fail retries on invoice checkout).

### Block B findings — GLOBAL vs DOCUMENT-LEVEL payability (spike, 2026-07-14)

> ⚠️ **OUTCOME: GLOBAL (Liz-confirmed 2026-07-14).** The "Recommendation: DOCUMENT-LEVEL" below was the spike's *initial* lean and is **NOT the final decision** — it was overturned in the same-day design meeting. This subsection is retained only for its technical findings (how payability resolves today, which fields already persist, the surcharge double-gate). For the decision and full scorecard see [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md) and decision #9 above.

**~~Recommendation: DOCUMENT-LEVEL.~~** *(superseded — see note above; final = GLOBAL)* The per-document fields already work for estimates through the entire stack, and the two models are functionally closer than the design meeting assumed.

**How payability resolves today (two AND-ed gates + a separate surcharge gate):**
1. **General/document-shape gate** — `findNotPayableReason` (`is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:42-86`). The choke point: `documentType !== 0` (line 58-60) rejects **every** estimate today, before any account/document setting is even read. Also returns `balanceDueZero` when balance ≤ 0 (line 70). **This one line is what must relax for estimates — regardless of the global-vs-doc choice.**
2. **Per-provider gate** — each SDK's `findNotPayableReason(accountStatus, documentStatus, paymentSuppressed)` (`.../utils/SDKs/sdks.ts`, Stripe 90-115 / PayPal 37-55). AND-ed with layer 1: account-not-accepting → `notAccepting` (doc setting irrelevant); account-accepting-but-doc-suppressed → `paymentSuppressed`. **Either layer can independently veto; neither overrides the other.** Per-doc flags fed via `providerPaymentSuppressedAdapter` (`user-payments-status.ts:75-83`, `paymentSuppressed → PayPal`, `stripePaymentSuppressed → Stripe`).
3. **Surcharge DOUBLE-GATE (confirmed)** — surcharge applies only if **BOTH** `invoice.setting.feesType === SURCHARGE` **AND** account `payment.feesType === SURCHARGE` (`is-stripe/.../passing-fees/stripe-fees.ts:38+45`, `is-paypal/.../passing-fees/paypal-fees.ts:37+43`). Account setting stored via Parse `Setting` key `payment.feesType`/`valNum`, seeded SURCHARGE for new users.

**Per-document fields already persist for estimates (no data-model gap):**
- Realm (mobile) `invoice-setting/schema.ts` uses a **single unified** `RInvoiceSetting` for both docTypes: `paymentSuppressed` (39/78), `stripePaymentSuppressed` (40/79), `feesType` (45/84), `feeRate` (46/85).
- Parse validation `invoiceValidation.ts` `settingSchema(isEstimate)` validates all four for **both** types; the `isEstimate` branch only strips `termsDay`. Nothing payment-related is rejected for estimates.
- `defaultInvoiceSetting` (`repository.ts:27-114`) writes for estimates too: `paymentSuppressed = null` / `stripePaymentSuppressed = null` (91-92), `feesType = getDefaultPassingFeesType()` **unconditionally** (97). `feeRate` left undefined until edited.

**Existing-estimate edge cases (the crux of the fork):**
- `paymentSuppressed` / `stripePaymentSuppressed` are **`null` on all pre-feature estimates** — no estimate UI ever wrote them (surcharge/payment tiles are docType-gated to invoices, see §"surcharge render gap"). So **both models default existing estimates to payable identically** → the feared "silently ignore a user's suppression choice" risk is near-zero. Retroactive-payability exposure is the same either way.
- `feesType` **is populated** on existing estimates (account default at creation). This is the field most likely to be "stale," and it's non-uniform across old estimates (SURCHARGE / MARKUP / undefined depending on `deprecate_markup` flag + account state when created). Under either model, once payable, an old estimate whose stored `feesType === SURCHARGE` will surcharge as soon as the account gate also matches.

**Surcharge render gap (web) — the only real UI work, and it's a gate removal:** two leaf components hard-gate the fees UI to invoices — `surcharge-fees.tsx:205` (`if (docType !== DOCTYPE_INVOICE) return null`) and `payment-markup-fees.tsx:37`. The data (`feesType`/`feeRate` on `invoice.setting`) is already wired for estimates; only the render is gated. The payment tile itself (`invoice-simple-payments.tsx`) reads `paymentSuppressed`/`stripePaymentSuppressed` with **no docType branch** — its gating is upstream in `findNotPayableReason`.

**Double-pay lock is orthogonal to this choice:** `findNotPayableReason` is the single point every payability path funnels through (checkout URL, report URL, payment-intent confirm), so anchoring the lock there enforces it regardless of global-vs-doc. **Caveat: no `convertedTo`/`approvedAt` field exists in any schema today** (grep-confirmed). Today's de-facto conversion lock is `markEstimateAsClosed` setting `setting.fullyPaid = true` → `findNotPayableReason` returns `balanceDueZero`. Decision #13's `convertedTo` must be a **new persisted field** added to Realm schema + Parse validation + surfaced into `CheckoutData` (`formatDocumentData`, `types.ts:86`), with a new `GeneralNotPayableReason` check. `estimateSignedAt` already exists but is signature-specific, not conversion.

| Aspect | GLOBAL (account-level) | DOCUMENT-LEVEL (per-doc) — **recommended** |
|---|---|---|
| Core change | Relax docType gate (`invoice-payments-status.ts:58`); payable when deposit set + account Stripe/PayPal enabled | Same relaxation, PLUS reuse per-estimate `paymentSuppressed`/`stripePaymentSuppressed`/`feesType`/`feeRate` (already persisted) |
| Surcharge | Would "inherit global" — but code **still double-gates** on `invoice.setting.feesType` (stripe-fees.ts:38). True global surcharge needs an **extra bypass** of the doc gate for estimates | Already wired end-to-end; only un-gate web UI (`surcharge-fees.tsx:205`, `payment-markup-fees.tsx:37`) |
| Web UI work | Less (can leave surcharge tile invoice-only) | Un-gate surcharge tile + fee-rate for estimates |
| Existing estimates | `paymentSuppressed` null → payable (same as doc). `feesType` populated → retroactive, **un-toggleable** surcharge if double-gate kept | Same payability default; `feesType` non-uniform, **but user can toggle per doc to fix** |
| Per-estimate off-switch | **None** — can't disable payment on a single estimate | Full parity with invoices |
| Double-pay lock | New `convertedTo`/`approvedAt` in `findNotPayableReason` — works identically | Same (orthogonal) |
| Net | Fewer touchpoints, but surcharge is inherently doc-gated so "pure global" needs an extra bypass; no safety toggle | Reuses proven invoice machinery (schema/validation/SDK all already handle estimates); gives merchants a per-doc safety toggle |

**Load-bearing facts:** docType block is one line (`invoice-payments-status.ts:58`); per-doc payment fields already persist for estimates (`schema.ts:39-46`, `invoiceValidation.ts:383-475`, `repository.ts:91-97`); surcharge is double-gated (`stripe-fees.ts:38+45`); web surcharge UI is the only real gap (`surcharge-fees.tsx:205`); double-pay lock belongs in `findNotPayableReason` but needs a NEW `convertedTo`/`approvedAt` field (none exists — current de-facto lock is `fullyPaid` → `balanceDueZero`).

### End-to-end flow (FINAL — convert-first at Pay Now, block A)

1. **Merchant sets a deposit on the estimate** (`Request Deposit` toggle → Type: Flat Amount / Percentage → Amount). Estimate editor shows a **`Deposit Due $500`** line. Estimate is sent to the buyer. (Payability comes from deposit + globally-enabled Stripe/PayPal per block B — pending confirmation.)
2. **Buyer opens the public estimate viewer** and sees **`Pay Deposit`** + a QR code, with the note *"An online payment fee will be charged if this estimate is paid online."* The checkout tab reads **`Payment | Estimate`** (still an estimate at this point).
3. **Buyer clicks "Pay Now" → SYNCHRONOUS conversion fires first.** A Parse Cloud Function converts the estimate → invoice (atomic guard on `approvedAt`/`convertedTo`), *before* any payment is attempted. A **loading state** on Pay Now covers the cloud-function latency.
   - If **conversion fails** (backend error): block payment confirmation, show error UI (redirect target TBD — back to estimate checkout?).
4. **PaymentIntent/Order is created against the INVOICE id** (not the estimate) — supplies L2/L3 interchange fields. Charges `deposit + online payment fee` (e.g. $500 + $17.45). Checkout tab flips **`Payment | Estimate` → `Payment | Invoice`**; success screen reflects invoice context.
   - If **payment fails** (conversion already done): buyer stays in-session and retries on the invoice checkout.
5. **On success:** merchant gets push (*"Your estimate EST0001 has been approved by {Client}"* / *"New Invoice Created INV0023…"*). Invoice shows **`Paid (Jan 1) $500`** and **`Balance Due $1,500`** — the deposit is recorded against the invoice directly (no carryover needed, block A). Invoice shows green banner *"This invoice was created from estimate EST0001"* + a **`Manage Payment Schedule`** entry.
6. **Post-conversion estimate is LOCKED** (block D): green banner *"This estimate has been converted into an invoice(s)"*; manual "Convert to Invoice" button hidden (prompt to **duplicate** instead); QR + Pay Deposit CTA hidden. **Old email link → redirected to the read-only public estimate URL, NOT checkout** (block C). Tapping the banner opens an **"Invoices Created"** modal (v1: single entry; array-backed for future). Estimate History gains **`Converted to Invoice INV####`** / **`Approved - {client}`**. If the source estimate was later deleted, the invoice history reads **"Created from a deleted estimate"**.

> **Note on ACH:** the Figma mockup shows an ACH "deferred invoice creation" step. Under block A this is **moot for conversion** — the invoice exists before payment regardless of method. ACH's pending/settlement state only affects when the *deposit payment* shows as cleared, not when the invoice is created.

### What the design confirms about the open questions

- **Acceptance = deposit payment (Q3).** Paying the deposit is the approval event; conversion is automatic, not a manual merchant action.
- **Payment carryover is mandatory (Q4).** The $500 deposit paid on the estimate must land on the invoice as a recorded payment, leaving `Balance Due = total − deposit`.
- **Deposit fields belong on the estimate (Q6).** Confirmed — the estimate editor owns the deposit config and `Deposit Due` line.

### NEW gaps this design surfaces (not yet in the gap analysis)

> ⚠️ **Severities below predate the convert-first decision (block A).** Several "Critical/High" rows are now **obviated** because conversion happens *before* payment (invoice minted on Pay Now); annotations added inline. The still-live gaps are auto-conversion (now a synchronous cloud fn, not a webhook), the "approved" event, labels, and bidirectional linkage.

| Gap | Severity | Details |
|-----|----------|---------|
| **Auto-conversion on deposit payment** | Critical → **still live (re-scoped)** | ⚠️ Under block A this is a **synchronous Parse Cloud Function on the Pay Now click**, NOT a payment-success webhook. Still net-new code, but the trigger changed. |
| **Payment carryover to new invoice** | ~~Critical~~ → **OBVIATED (block A)** | Convert-first mints the invoice *before* payment, so the deposit Payment record is keyed to the invoice from the first PaymentIntent. No re-keying/copying needed. See §7 + decision #4. |
| **"Approved" event on deposit payment** | High | Estimate must transition to an *approved* state and emit history + push. No such coupling exists today (§3). Now fires on the Pay Now conversion (approved/`convertedTo` set), not on payment success. |
| **ACH deferred conversion** | ~~High~~ → **MOOT (block A)** | Invoice exists before payment regardless of method; settlement timing no longer gates conversion. ACH's pending state only affects when the deposit *shows cleared*. See decision #5. |
| **One estimate → many invoices** | ~~High~~ → **1:1 in v1 (block D)** | Locked to strict 1:1; reuse = duplicate. `convertedTo` stays an array to future-proof, but only one invoice is ever created in v1. The "Invoices Created" modal renders a single entry. |
| **Bidirectional conversion linkage** | Medium | Invoice banner "created from estimate EST0001" (has `estimateId` today) AND estimate banner listing its invoices (does NOT exist today). Estimate needs to track child invoice ids. |
| **"Created from a deleted estimate"** | Medium | Invoice history must gracefully render when the source estimate is gone — implies the invoice stores enough estimate identity (number/id) to show this, and doesn't hard-depend on the estimate record existing. |
| **Estimate checkout labels say "Estimate"** | Medium | Checkout tab shows `Payment | Estimate` and CTA is `Pay Deposit` — reinforces the dynamic-label work in Phase 4, plus a deposit-specific CTA. |
| **Remaining-balance invoice creation (ACH)** | Medium | The ACH copy promises "an invoice covering the remaining balance will be created" — confirms the invoice represents the *remaining* balance after deposit, consistent with the `Balance Due $1,500` display. |

### Open product/eng questions still remaining

> ⚠️ **Most of these are now RESOLVED by the design meeting (blocks A–D) — kept with answers inline for traceability.**

- **~~Conversion timing/ownership~~ → RESOLVED (block A):** conversion is a **synchronous Parse Cloud Function on the Pay Now click** (server-driven), before payment. Not a payment-success webhook, not client-on-next-sync.
- **~~Multiple invoices semantics~~ → RESOLVED (block D):** strictly **1:1 in v1** (one deposit payment → one invoice carrying deposit-paid + remaining balance). Array-backed for future milestones; no 1→N in v1.
- **Deposit fee handling → RESOLVED (see "Online Payment Fee" section):** the online fee is a surcharge grossed-up on the amount charged now (the deposit), stored per Payment record; the buyer is charged a **fresh** fee on the remaining balance later. Two-fee behavior accepted (matches Payment Scheduling). Zero new fee logic.
- **~~Non-deposit estimates~~ → out of scope (decision #1):** trigger is **deposit payment only**. No auto-conversion for a non-deposit estimate in this flow.

---

## Complete Gap Analysis: Estimate vs Invoice Payments

### How It's Similar (Would Work With No/Minimal Changes)

| Aspect | Notes |
|--------|-------|
| Payment toggle on mobile | `InvoicePaymentsSection` has no docType gate — already works |
| Surcharge on mobile | `passing-fees-section.tsx` allows both docTypes |
| Provider enable/disable APIs | Account-level, no docType logic |
| Payment method config | Country-based, no docType filtering |
| `paymentSuppressed` / `stripePaymentSuppressed` | Per-document fields, work on estimates |
| Surcharge calculation (`getSurchargeCents`) | DocType-agnostic |
| Webhook handlers (post-payment) | Operate on payment record IDs, no docType re-check |
| Deposit UI in checkout page | `PaymentDepositDetails` is docType-agnostic |
| `balanceDueCents` computation | Calculator works for any document |
| Payment method ordering/blocking | No docType dependency |

### How It's Different (Gaps That Need Work)

| Gap | Severity | Details |
|-----|----------|---------|
| **`findNotPayableReason` docType gate** | Critical | `documentType !== 0` blocks ALL payment processing. Single fix unlocks Stripe + is-checkout + is-unifiedxp |
| **PayPal separate docType gate** | Critical | `is-paypal/packages/server/src/services/invoice-payable.ts:29` has its own `documentType !== 0` — independent of shared package |
| **Inline payments array validation** | High | Parse `fixPayments()` silently wipes the array in prod; throws in staging. Needs fix for payment recording to be consistent |
| **Email template — no "Pay Now" button** | High | Estimates use `swuEstimateTemplateV2` which has no checkout link. Need new template or conditional `pay_url` |
| **Email `docType != DOCTYPE_INVOICE` guard** | High | `is-api/src/controllers/invoice-sending.ts:1010` explicitly skips payment URL generation for non-invoices |
| **10+ hardcoded "Invoice" strings in checkout UI** | Medium | Tab labels, toasts, Apple Pay/Google Pay labels, iframe title all say "Invoice" |
| **Web app payments widget blocked** | Medium | `settings-sidebar.tsx` gates `WidgetPayments` to `DOCTYPE_INVOICE` only |
| **Web app surcharge blocked** | Medium | `surcharge-fees.tsx:205` returns null for non-invoices |
| **Deposit fields not on Estimate type** | Medium | `EstimateSettings` lacks `depositType`/`depositRate`/`depositAmount` — needed for deposit-first payment |
| **`feesType` lost on estimate→invoice conversion** | Low | Mobile omits `feesType` when converting — user loses surcharge setting |
| **PayPal double-gate at capture** | Low | `capturePaypalOrder` re-validates docType even after order creation |
| **is-checkout router error handling** | Low | `docTypeNotInvoice` shows generic "Go to Invoice Simple" page, no estimate-friendly message |
| **No docType in payment records** | Low | `payment_intent` / `paypal_order` tables don't store docType — analytics requires joining to Parse |

### Payment Lifecycle Coverage

| Stage | Stripe | PayPal | Estimate Status |
|-------|--------|--------|-----------------|
| Checkout page load | `findNotPayableReason` blocks | `validateDocumentPayable` blocks | **BLOCKED** |
| Payment intent/order creation | `findNotPayableReason` blocks | `validateDocumentPayable` blocks | **BLOCKED** |
| Payment confirmation/capture | `findNotPayableReason` blocks | `validateDocumentPayable` blocks | **BLOCKED** |
| Webhook: payment success | No docType check | No docType check | Would work IF payment created |
| Webhook: refund/dispute | No docType check | No docType check | Would work IF payment exists |
| Add payment to Parse (inline) | No docType check in write | PayPal re-validates! | Stripe: works but silently wiped; PayPal: blocked |
| Abandoned cart reminders | Returns false for non-invoices | N/A | Would start firing if enabled |

### Toggle Architecture (Local vs Global)

| Level | Mechanism | Estimates? |
|-------|-----------|------------|
| **Global (account-level)** | `isPaypalEnabled` / `isStripeEnabled` via provider account status | Applies to all docs — same for estimates |
| **Local (per-document)** | `setting.paymentSuppressed` (PayPal) / `setting.stripePaymentSuppressed` (Stripe) | Fields exist, mobile UI works, web UI blocked |
| **Interaction** | Both must be enabled for document to be payable | Works for estimates once `findNotPayableReason` allows them |

---

## Email Flow Details

**Same system handles both:** `POST /api/v2/email-invoice` in is-api → `send()` in `email.ts`

**Template selection logic (is-api `email.ts` ~line 654-657):**
- Estimates always get `swuEstimateTemplateV2` (no "Pay Now" button)
- Only invoices get `paymentsSwuInvoiceTemple` (with checkout URL)

**3 layers currently block payment links in estimate emails:**
1. Template selection — separate template without pay button
2. `findNotPayableReason` — rejects docType=1 during eligibility check
3. `invoice-sending.ts:1010` — explicit `if (docType != DOCTYPE_INVOICE) return;` skips pay URL generation

**Checkout URL pattern:** `{IS_SERVICES_BASE_URL}/checkout/{accountId}/{documentRemoteId}`

**Estimate-signed email:** Separate endpoint `POST /api/v2/email-estimate-signed` — sends notification when client signs. Has estimateUrl (public viewer) but no payment link. Could potentially add checkout URL here.

---

## Files to Change (Comprehensive)

### Phase 1: Core Gate Removal (Makes Estimates Payable)

| File | Change | Risk |
|------|--------|------|
| `is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58` | Allow docType 0 and 1 in `findNotPayableReason` | Low — single condition, unlocks Stripe path |
| `is-paypal/packages/server/src/services/invoice-payable.ts:29` | Allow docType 1 in PayPal's separate `validateDocumentPayable` | Low — single condition, unlocks PayPal path |
| `is-services/packages/payments/payments-status/src/types/types.ts` | Rename `docTypeNotInvoice` → `docTypeNotPayable` (optional) | Low — cosmetic |

### Phase 2: Parse Server Inline Payments Fix

| File | Change | Risk |
|------|--------|------|
| `is-parse-server/cloud/collections/invoice/invoiceValidation.js:683-688` | Remove `payments.length === 0` enforcement for estimates OR allow payments array | Medium — validation change, needs careful testing |
| `is-parse-server/cloud/collections/invoice/utils/invoiceConversion.js:117-122` | Remove `fixPayments()` silent wipe for estimates | Medium — production behavior change |

### Phase 3: Email Flow

| File | Change | Risk |
|------|--------|------|
| `api/src/services/email.ts:~654-657` | Update template selection to use payments template for estimates with payments enabled | Medium — email template change |
| `api/src/controllers/invoice-sending.ts:1010-1011` | Remove `docType != DOCTYPE_INVOICE` guard | Low |
| New email template (or modify existing) | Add "Pay Now" button to estimate email | Medium — SWU template work |

### Phase 4: Checkout Page UI

| File | Change | Risk |
|------|--------|------|
| `is-unifiedxp/src/components/ResponsiveWrapper/ResponsiveWrapper.tsx:9` | Make tab label dynamic ("Invoice"/"Estimate") | Low |
| `is-unifiedxp/src/components/InvoiceSection/InvoiceSection.tsx:61` | Make iframe title dynamic | Low |
| `is-unifiedxp/src/components/PaypalApplePay/PaypalApplePay.tsx:27` | Make paymentLabel dynamic | Low |
| `is-unifiedxp/src/components/PaypalGooglePay/PaypalGooglePay.tsx:31` | Make paymentLabel dynamic | Low |
| `is-unifiedxp/src/components/PaymentButtons/PaymentButtons.tsx:80` | "Invoice Paid" → "Estimate Paid" | Low |
| `is-unifiedxp/src/components/PaymentToast/PaymentToast.tsx:7,15` | "This invoice is overdue" → dynamic | Low |
| `is-unifiedxp/src/components/ReportsPageInvoiceInfo/ReportsPageInvoiceInfo.tsx:24,35` | "Invoice Number"/"Invoice Amount" → dynamic | Low |
| `is-unifiedxp/src/utils/document-payable.ts` | Handle removed reason gracefully | Low |

### Phase 5: Web App UI (If Desired)

| File | Change | Risk |
|------|--------|------|
| `is-web-app/nextjs/.../settings-sidebar-migrated.tsx:249` | Remove `DOCTYPE_INVOICE` gate on `WidgetPayments` | Low |
| `is-web-app/nextjs/.../surcharge-fees.tsx:205` | Remove `DOCTYPE_INVOICE` guard | Low |

### Phase 6: Domain Model (For Deposit Support)

| File | Change | Risk |
|------|--------|------|
| `@invoice-simple/domain-invoicing` (EstimateSettings type) | Add `depositType`/`depositRate`/`depositAmount` | Medium — shared type change |
| `is-mobile/src/services/realm/entities/invoice-setting/schema.ts` | Add deposit fields for estimates | Medium |
| Mobile/Web estimate editors | Add deposit configuration UI | Medium |

### Phase 7: Housekeeping

| File | Change | Risk |
|------|--------|------|
| `is-services/packages/services/is-abandoned-cart/src/utils/is-invoice-payable.ts` | Decide: exclude estimates or allow reminders | Low |
| `is-checkout/src/checkout/router.ts` | Update error handling for estimates | Low |
| `is-mobile/src/services/realm/entities/invoice/use-cases.ts` | Preserve `feesType` on estimate→invoice conversion | Low |

---

## Effort Estimate

| Phase | Work | T-shirt |
|-------|------|---------|
| Phase 1: Gate removal | 2 files, ~4 lines changed | XS (half day) |
| Phase 2: Parse inline fix | 2 files, validation logic | S (1 day) |
| Phase 3: Email flow | Template + 2 guards | M (2-3 days) |
| Phase 4: Checkout UI strings | 7+ files, string conditionals | S (1 day) |
| Phase 5: Web app UI | 2 files, remove guards | XS (half day) |
| Phase 6: Domain model + deposit | Type extension + UI | L (1 week) |
| Phase 7: Housekeeping | 3 files, minor | XS (half day) |
| **Total (without Phase 6)** | | **M (4-5 days)** |
| **Total (with deposits)** | | **L (2 weeks)** |

---

## Recommended Implementation Order

1. **Phase 1** (gate removal) — immediately unblocks end-to-end testing
2. **Phase 2** (inline payments fix) — ensures payment recording works for estimates
3. **Phase 4** (UI strings) — checkout page shows correct labels
4. **Phase 3** (email) — customers can receive payment links
5. **Phase 5** (web UI) — web app users can configure payments on estimates
6. **Phase 7** (housekeeping) — clean up edge cases
7. **Phase 6** (deposits) — enables "pay deposit" vs "pay full" for estimates

Phases 1-4 give you a working estimate payment flow end-to-end. Phase 6 is optional and only needed if you want deposit-specific checkout behavior on estimates.

---

## Key Files (Reference)

| File | Role |
|------|------|
| `is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts` | Shared `findNotPayableReason` — **primary gate** |
| `is-paypal/packages/server/src/services/invoice-payable.ts` | PayPal's **separate gate** (independent of shared package) |
| `is-services/packages/payments/payments-status/src/types/types.ts` | Enums + `CheckoutData` type |
| `is-services/packages/payments/payments-status/src/utils/SDKs/sdks.ts` | Provider-specific not-payable checks |
| `is-services/packages/services/is-checkout/src/document/document-payable.ts` | `formatDocumentData` → builds CheckoutData |
| `is-services/packages/services/is-checkout/src/document/get-payments.ts` | Fetches Payment collection records |
| `is-services/packages/services/is-checkout/src/eligibility/service.ts` | Eligibility API (mobile calls this) |
| `is-services/packages/services/is-checkout/src/checkout/service.ts` | Checkout redirect logic |
| `is-services/packages/services/is-unifiedxp/src/utils/document-payable.ts` | is-unifiedxp's validateDocumentPayable |
| `is-services/packages/services/is-unifiedxp/src/components/PaymentsSection/PaymentsSection.tsx` | Checkout page component |
| `is-services/packages/services/is-unifiedxp/src/components/PaymentDetails/PaymentDepositDetails.tsx` | Deposit UI (already exists) |
| `is-services/packages/services/is-stripe/src/services/payments-status/document-payable.ts` | is-stripe's validateDocumentPayable |
| `is-services/packages/services/is-stripe/src/services/document-add-payment.ts` | Payment confirmation → Parse write |
| `is-services/packages/services/is-stripe/src/utils/parse/public-invoice.ts` | Calls `invoiceAddPayment` cloud function |
| `is-paypal/packages/server/src/services/paypal-create-order.ts` | PayPal order creation (calls validateDocumentPayable) |
| `is-paypal/packages/server/src/services/paypal-capture-order.ts` | PayPal capture (double-validates docType) |
| `is-paypal/packages/server/src/utils/parse/public-invoice.ts` | PayPal's `invoiceAddPayment` call |
| `is-parse-server/cloud/collections/invoice/functions/invoiceAddPayment.js` | Creates Payment record + pushes to inline array |
| `is-parse-server/cloud/collections/invoice/invoiceValidation.js:683-688` | `payments.length === 0` enforcement for estimates |
| `is-parse-server/cloud/collections/invoice/utils/invoiceConversion.js:117-122` | `fixPayments()` silent wipe for estimates |
| `api/src/services/email.ts` | Email template selection + pay_url assignment |
| `api/src/controllers/invoice-sending.ts` | V1/V2 email handlers, docType guard at line 1010 |
| `is-services/packages/services/is-abandoned-cart/src/utils/is-invoice-payable.ts` | Abandoned cart payability check |
| `is-mobile/src/features/payments/api.ts:44-48` | Mobile eligibility API call |
| `is-mobile/src/features/payments/payments-option/components/invoice-payments-passing-fees-section.tsx` | Mobile surcharge toggle (no docType gate) |
| `is-web-app/nextjs/app/(authenticated)/(core)/(documents)/components/widget-payments/surcharge/surcharge-fees.tsx` | Web surcharge (docType gated) |
