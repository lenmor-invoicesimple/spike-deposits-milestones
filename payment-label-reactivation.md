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
| Mobile | `is-mobile/src/features/documents/screens/manage-payment.screen.tsx:401` | `editable={false}` |

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

### Optional enhancements

- Add max length validation (e.g. 30 chars) to prevent UI overflow
- Keep the auto-generated default ("Deposit"/"Payment") as placeholder text so users know what it defaults to
- Consider whether label should remain editable after mark-as-paid (or lock once completed)

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

No change needed
- Calculator (`@invoice-simple/calculator`) — label-agnostic, uses `paymentType` enum for all logic
- Mobile display — renders label as plain text, no branching on value

**Confirm:**
- Should there be a max character limit on label? (UI overflow risk on mobile/PDF)
- Should label remain editable after a payment is marked as paid, or lock once completed?
- Should the default "Deposit"/"Payment" be a placeholder hint or a pre-filled value?
- Does this need design review for the unlocked field styling, or is current field appearance acceptable?
- Any analytics event needed when a user customizes the label?
- Does the PDF/email template render `label` — if so, does it handle long strings gracefully?
