# Payment Scheduling — Concepts

## What It Is

Payment Scheduling lets invoice senders split a single invoice into multiple payment milestones. Instead of one lump payment, you and your customer agree on a timeline: deposit upfront, milestone payments during the project, final payment on delivery.

---

## Key Concepts

### Payment States

A payment is either **Upcoming** or **Completed** — determined by its data shape, not a status field.

| State | Has `amount`? | Has `date`? | Has `method`? |
|---|---|---|---|
| Upcoming (scheduled) | No | No | No |
| Completed (paid) | Yes | Yes | Yes |

There is no `status: "pending"` flag. The calculator (`@invoice-simple/calculator`) checks: does this payment have an `amount` field? If yes → completed. If no → upcoming.

> **Gotcha**: If a bug ever writes an empty `amount` to an upcoming payment, it silently appears as a $0 completed payment in history.

### Payment Types

| Type | `paymentType` value | Notes |
|---|---|---|
| Deposit | `PaymentTypes.DEPOSIT` | Only one allowed per invoice |
| Upcoming Payment | `PaymentTypes.NONE` | Unlimited |

Despite the name, `PaymentTypes.NONE` is the type for a regular upcoming payment — not an error.

### Payment Mode

How the amount is expressed:

| Mode | Meaning | Example |
|---|---|---|
| `PERCENT` | % of invoice total | 50% of $1000 = $500 |
| `FLAT` | Fixed dollar amount | $89.00 |

---

## Deposit vs. Upcoming Payment

Functionally almost identical — both are scheduled, unpaid milestones stored the same way. The only differences:

| | Deposit | Upcoming Payment |
|---|---|---|
| `paymentType` | `DEPOSIT` | `NONE` |
| Due date shown on row | No | Yes |
| How many allowed | 1 per invoice | Unlimited |
| Balance label | "Deposit Due" | "Balance Due" |
| Email button label | "Email Invoice For Deposit" | "Email Invoice" |

---

## Balance Calculation

```
Balance Due = Invoice Total − sum(completed payment amounts)
```

Upcoming/scheduled payments do **not** reduce Balance Due. They only appear in the summary section as a hint ("Deposit Due: $35.70") but the actual balance doesn't move until the payment is marked as paid.

Marking as paid = recording `amount` + `date` + `method` on the payment object.

---

## Due Date

The due date on an Upcoming Payment is **purely informational**:

- Displayed as `"Due Jun 15, 2025"` under the payment row
- Defaults to today when creating a payment
- Optional field — can be left blank
- No overdue alerts, no automatic reminders, no enforcement
- Does not affect balance or payment status

---

## API Endpoints (Parse Cloud Functions)

All calls authenticated via `X-Parse-Session-Token`.

| Endpoint | Purpose |
|---|---|
| `invoiceAddPayment` | Create a deposit or upcoming payment |
| `invoiceUpdatePayment` | Edit, mark as paid, or soft-delete a payment |
| `invoiceGetPayments` | Fetch all payments for an invoice |
| `addHistoricalPayments` | Record a past payment directly into history |

Soft delete: payments are never hard-deleted. `invoiceUpdatePayment` with `{ deleted: true }` hides them.

---

## Analytics Events

| Action | Event |
|---|---|
| Add upcoming payment | `payment-added` |
| Record past payment | `historical-payment-added` |
| Edit past payment | `historical-payment-updated` |
| Mark upcoming as paid | `payment-marked-paid` |
| Delete payment | `payment-deleted` |

---

## Value vs. Alternatives

| | Payment Scheduling | Regular Checkout | Recurring Invoices |
|---|---|---|---|
| Use case | Split one invoice into milestones | Pay invoice in full now | Same amount, same cadence forever |
| Customer consent needed? | No | No | Yes (ACH mandate) |
| Payment processor required? | No (can record manual payments) | Yes | Yes (Stripe/ACH) |
| Flexibility | High — % or flat, any dates | None | Low — fixed schedule |
| Best for | Project-based work (contractors, freelancers) | One-time purchases | Subscriptions, retainers |

---

## Key Files

| File | Purpose |
|---|---|
| `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/upcoming-payments-section.tsx` | Upcoming payments list + deposit/upcoming buttons |
| `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/upcoming-payment-form-modal.tsx` | Add/edit deposit or upcoming payment |
| `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/payment-details-section.tsx` | Balance summary (Invoice Total / Paid / Balance Due) |
| `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/payment-rows.tsx` | Renders each payment row |
| `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/use-date-text.ts` | Due date display logic |
| `nextjs/components/invoice-editor/invoice-payment/payment-scheduling/payment-menu.tsx` | Three-dot menu (Mark Paid / Edit / Delete) |
