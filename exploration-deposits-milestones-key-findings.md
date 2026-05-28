# Key Findings: Deposits & Milestones — How It Actually Works

> Core technical findings from exploring the existing payment scheduling system (May 2026).

---

## 1. Milestones Are Payment Records

Milestones are **not** a separate object type. They are `Payment` records in Parse/MongoDB where:
- `date == null` → upcoming/unpaid
- `date != null` → completed/paid

Key fields on a Payment:
- `invoiceRemoteId` — links to the Invoice
- `paymentType` — 0=NONE, 1=DEPOSIT
- `paymentMode` — 0=NONE, 1=PERCENT, 2=FLAT
- `paymentValue` — the % or $ amount
- `amount` — calculated dollar amount
- `label` — "Deposit" or "Payment"
- `dueDate` — when this milestone is due
- `date` — null=UNPAID, set=PAID

---

## 2. `Invoice.balanceDue` = Next Milestone Amount (NOT total - paid)

This is the most important architectural finding. The shared `@invoice-simple/calculator` package implements:

```
getInvoiceBalance(invoice, payments) {
  upcomingPayment = getNearestUpcomingPayment(payments)  // date=null, dueDate exists
  
  if (upcomingPayment) {
    return upcomingPaymentAmount  // next milestone's amount
  }
  
  return total - paid  // fallback only when no milestones remain
}
```

### Example walkthrough (Invoice: $10,500 total, 3 milestones)

| State | balanceDue | Why |
|---|---|---|
| Invoice created, deposit exists | $3,000 | Deposit is nearest upcoming |
| After deposit paid | $4,000 | Milestone 1 is nearest upcoming |
| After milestone 1 paid | $3,500 | Milestone 2 is nearest upcoming |
| After milestone 2 paid | $0 | No upcoming → fallback: total - paid |

### Call chain when a payment completes:

```
Stripe payment succeeds
  → invoiceAddPayment (Parse cloud function)
    → addPaymentToInvoice()
      → updatePaymentDetails()
        → fetches ALL Payment records for the invoice
        → invoice.set('balanceDue', getInvoiceBalance(universalInvoice, commonPayments))
```

### Verified in production (staging):
- Invoice `xvuIKNaaBt`, total $10,500
- Before deposit: `balanceDue: 3000`
- After deposit paid ($3,000): `balanceDue: 4000`, `paidAmount: 3000`

---

## 3. Invoice.dueDate = Payment Terms Default (Static, Does NOT Chain)

`Invoice.dueDate` is set from the merchant's account-level Payment Terms setting (e.g., DAYS_7 → creation date + 7 days). It does NOT auto-update when milestones are paid.

Each milestone has its own `dueDate` field on the Payment record, but these are independent of `Invoice.dueDate`.

### Current state (main branch):

| Field | Chains through milestones? |
|---|---|
| `balanceDue` | ✅ Yes — automatically advances to next milestone amount |
| `dueDate` | ❌ No — stays at original payment terms value |

### Spike branch exists (not merged):

Branch `spike-tie-milestone-changes-to-invoice-due-date` adds a `recalculateInvoiceDueDate()` function that auto-syncs `Invoice.dueDate` to the soonest unpaid milestone. Triggered when:
- A milestone is added (`invoiceAddPayment`)
- A milestone is edited (`updatePayment`)
- A milestone is marked paid (`addPaymentToInvoice`) → shifts to next unpaid

### UX implication:

Today, after the deposit is paid, the invoice still shows the original due date (e.g., Jun 2) — not the next milestone's actual due date (Jun 15). Reminder emails and the invoice PDF show the stale date. The spike branch fixes this gap.

---

## 4. Checkout Reads `balanceDue` Directly

Checkout (is-unifiedxp) doesn't need to know about deposits vs milestones. It reads `Invoice.balanceDue` and charges that amount. The sequential milestone chaining happens transparently through the calculator.

---

## 5. What This Means for Auto-Pay

The system already "chains" through milestones sequentially via `balanceDue`. Auto-pay's job is to:
1. Know WHEN to trigger (milestone's `dueDate`)
2. Use a vaulted payment method (instead of the client manually returning to checkout)
3. Mark the Payment record as completed (set `date`)
4. The existing calculator logic automatically advances `balanceDue` to the next milestone

The hard part isn't "what to charge" or "which milestone" — the existing system already handles that. The hard part is scheduling, vaulting, and executing without human interaction.

---

## 6. Key Code Locations

| What | Where |
|---|---|
| Balance calculation | `@invoice-simple/calculator` → `getInvoiceBalance()`, `getNearestUpcomingPayment()` |
| Backend balance update | `is-parse-server/cloud/collections/invoice/utils/updatePaymentDetails.ts` |
| Payment completion | `is-parse-server/cloud/collections/invoice/utils/addPaymentToInvoice.ts` |
| Payment creation | `is-parse-server/cloud/collections/invoice/functions/invoiceAddPayment.ts` |
| Payment → common format | `is-parse-server/cloud/collections/payment/utils/parsePaymentToCommonPayment.ts` |
| Mobile balance display | `is-mobile/src/features/upcoming-payments/section/invoice-total-section.tsx` |
| Web balance display | `is-web-app/client/src/models/InvoiceModel.ts` |
