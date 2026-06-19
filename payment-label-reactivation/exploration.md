# Re-activate Label Field on Deposits, Milestones, Record Payments

## Background

The `label` field exists on Payment documents in MongoDB (values: `"Deposit"` or `"Payment"`), and the form field is already rendered in the UI — but it's **disabled**. Users cannot edit it.

This doc evaluates what it takes to make it editable.

---

## Current State

### Where label is locked

| Platform | File | How |
|---|---|---|
| Web (upcoming) | `is-web-app/nextjs/components/invoice-editor/invoice-payment/payment-scheduling/upcoming-payment-form-modal.tsx:193` | `disabled={true}` |
| Web (completed/record) | `is-web-app/nextjs/components/invoice-editor/invoice-payment/payment-scheduling/completed-payment-form-modal.tsx:247` | `disabled={true}` |
| Mobile (upcoming/deposit) | `is-mobile/src/features/documents/screens/manage-payment.screen.tsx:401` | `editable={false}` |
| Mobile (historical) | `is-mobile/src/features/documents/screens/manage-historical-payment.screen.tsx` | **Field does not exist** — needs to be added |

### How label is auto-assigned today

- Deposits: `"Deposit"` (from i18n key `invoice.multiplePayment.deposit`)
- Milestones/Historical: `"Payment"` (from i18n key `invoice.multiplePayment.payment`)
- Backend fallback in `invoiceAddPayment.ts:84-86`: `payment?.label || (isDeposit ? PaymentLabel.DEPOSIT : PaymentLabel.PAYMENT)`

---

## Risk Assessment

### Calculator (`@invoice-simple/calculator`): NO RISK

The calculator never uses `label` for logic. All payment classification is based on `paymentType`, `paymentMode`, `paymentValue`, and `date`. Label is purely display.

### Mobile: NO RISK

Mobile uses `label` only for rendering text in payment rows. No branching on label value — payment type is always determined by `paymentType` enum.

### Backend: 2 ISSUES TO FIX

#### Issue 1: `updatePayment.ts:130` — label overwrite on edit

```typescript
// Current — wipes label if client doesn't send it
paymentObj.set('label', payment?.label);

// Fix — only update if explicitly provided
if (payment?.label !== undefined) {
  paymentObj.set('label', payment.label);
}
```

**Why this matters:** When "Mark as Paid" or any edit is called without `label` in the payload, the existing custom label gets overwritten with `undefined`.

#### Issue 2: `invoiceHooks.ts` — hardcoded labels (lines 334, 369, 430)

These 3 locations hardcode `label: PaymentLabel.PAYMENT` when creating Payment documents from the `Invoice.payments[]` subdocument sync path.

```typescript
// Current
label: PaymentLabel.PAYMENT,

// Fix
label: payment.label || PaymentLabel.PAYMENT,
```

**Why this is LOW risk:** The `Invoice.payments[]` subdocument doesn't store `label` today. So `payment.label` is always `undefined` and the fallback applies — making this a no-op for current behavior but future-proof.

#### Not an issue: `handleDepositChange` (line 539)

Hardcodes `PaymentLabel.DEPOSIT` when creating a new deposit via the legacy Invoice.setting path. This is correct — it's creating a brand-new deposit where no user label exists.

---

## Implementation Plan

### Minimal (web only, ~1 hour)

| Step | File | Change |
|---|---|---|
| 1 | `upcoming-payment-form-modal.tsx:193` | Remove `disabled={true}` |
| 2 | `completed-payment-form-modal.tsx:247` | Remove `disabled={true}` |
| 3 | `updatePayment.ts:130` | Guard: only set label if provided |
| 4 | `invoiceHooks.ts:334,369,430` | Fallback: `payment.label \|\| PaymentLabel.PAYMENT` |

### Full (web + mobile, ~2-3 hours)

| Step | File | Change |
|---|---|---|
| 5 | `manage-payment.screen.tsx:401` | Remove `editable={false}` |
| 6 | Mobile validation schema | Ensure label is optional (already `string().trim().required()` — may want to relax to optional with default) |

### Rendering surfaces to verify

Custom labels must display properly on all surfaces where "Deposit"/"Payment" currently renders:

| Surface | Where | Concern |
|---|---|---|
| Public invoice (`/v/` route) | "UPCOMING PAYMENTS" section — label is rendered inline left-aligned with amount right-aligned | Long labels could overlap or wrap awkwardly |
| PDF export | Same "UPCOMING PAYMENTS" section in generated PDF | Fixed-width layout — long strings may overflow or get truncated |
| Mobile Payment Scheduling screen | Payment row label text | Already renders `label` as plain text — should handle custom values |
| Web Payment Scheduling tab | Payment row in upcoming/completed lists | Same as mobile |
| Email template (if applicable) | Check if payment breakdown appears in any transactional emails | TBD — verify during implementation |

### Validation

- Max length: **30 characters** (recommended default — prevents overflow on PDF/mobile row). Mark for discussion during grooming.
- Allowed characters: alphanumeric + common punctuation (no special/control characters)
- Empty/whitespace-only: fall back to auto-generated default ("Deposit"/"Payment")
- Server-side: add validation in `invoiceValidation.ts`
- Client-side: add `maxLength` attribute on web input, character limit on mobile input

---

## Open Questions

### Engineering concerns

- [ ] Verify label rendering on public invoice (`/v/` route) with long strings (e.g. 30 chars) — does it wrap, truncate, or overflow?
- [ ] Verify PDF rendering with long labels — same check for the "UPCOMING PAYMENTS" section
- [ ] Check if any email templates (checkout confirmation, payment receipt) include the payment label
- [ ] Confirm `invoiceValidation.ts` is the right place for server-side validation (vs inline in `updatePayment.ts`)

### Product/Design concerns (discuss during grooming)

- [ ] Max character limit — is 30 chars the right cap? Should we allow more on web but truncate on mobile/PDF?
- [ ] Should label remain editable after a payment is marked as paid, or lock once completed?
- [ ] Should the default "Deposit"/"Payment" be a placeholder hint or a pre-filled value?
- [ ] Does the unlocked field need a design review for styling, or is current field appearance acceptable?
- [ ] Any analytics event needed when a user customizes the label? (e.g. track adoption)

---

## Files Referenced

| File | Role |
|---|---|
| `is-web-app/nextjs/components/.../upcoming-payment-form-modal.tsx` | Web form for creating/editing upcoming payments |
| `is-web-app/nextjs/components/.../completed-payment-form-modal.tsx` | Web form for mark-as-paid and record payment |
| `is-mobile/src/features/documents/screens/manage-payment.screen.tsx` | Mobile payment form |
| `is-parse-server/cloud/collections/payment/updatePayment.ts` | Backend payment update (label overwrite risk) |
| `is-parse-server/cloud/collections/invoice/invoiceHooks.ts` | Invoice beforeSave hooks (hardcoded labels) |
| `is-parse-server/cloud/collections/invoice/functions/invoiceAddPayment.ts` | Payment creation (already has fallback — safe) |

---

## Summary

**Current:** The "Name" field on deposit, milestone, and record payment forms is visible but locked — always auto-assigned as "Deposit" or "Payment".

**Proposed:** Allow users to type a custom label (e.g. "Foundation Work", "Final Delivery") when creating or editing a payment.

**Complexity Guess:** Low

**Details:**

is-web-app
- Remove `disabled={true}` on `upcoming-payment-form-modal.tsx:193` (upcoming payments)
- Remove `disabled={true}` on `completed-payment-form-modal.tsx:247` (mark-as-paid + record payment)
- Keep auto-generated default ("Deposit"/"Payment") as initial value so blank labels aren't possible

is-mobile
- Remove `editable={false}` on `manage-payment.screen.tsx:401`

is-parse-server
- Fix `updatePayment.ts:130` to not overwrite label with `undefined` when clients omit it
- Add fallback in `invoiceHooks.ts:334,369,430` so legacy sync path doesn't clobber custom labels
- `invoiceAddPayment.ts` already falls back correctly — no change needed

Rendering verification
- Public invoice (`/v/` route): "UPCOMING PAYMENTS" section — currently shows "Deposit" / "Payment" hardcoded; must handle custom labels up to 30 chars
- PDF export: same section — verify long labels don't overflow fixed-width layout
- Email templates: check if payment label appears in any transactional emails

No change needed
- Calculator (`@invoice-simple/calculator`) — label-agnostic, uses `paymentType` enum for all logic
- Mobile display — renders label as plain text, no branching on value
