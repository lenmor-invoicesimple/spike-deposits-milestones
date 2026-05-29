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

## Recording Payments: Mark as Paid vs. Record Payment

Both produce a completed payment in the same `Payment` collection. The difference is the entry point and whether a prior "upcoming" document existed.

| | Mark as Paid | Record Payment (Historical) |
|---|---|---|
| **API** | `invoiceUpdatePayment` (updates existing doc) | `addHistoricalPayments` (creates new doc) |
| **Prior document** | Yes — an existing upcoming Payment row gets `amount`/`date`/`method` added | No — a fresh Payment document is created directly as completed |
| **Has scheduling fields** (`paymentType`, `paymentMode`, `paymentValue`) | Yes — inherited from the upcoming payment | No — absent |
| **Has `dueDate`** | Yes — preserved from when it was scheduled | No |
| **Use case** | Customer pays a previously scheduled milestone | Record a payment that was received outside of the scheduling flow (e.g. cash, check) |

Once completed, both look identical in the UI's "Payment History" section. The only way to distinguish them in the database is whether the scheduling fields (`paymentType`, `paymentMode`, `paymentValue`, `dueDate`) are present.

### MongoDB Field Comparison — How to Identify Payment Type in the Database

| Field | Deposit (upcoming) | Milestone (upcoming) | Deposit — Mark as Paid (manual) | Deposit — Paid via Checkout | Milestone — Mark as Paid (manual) | Milestone — Paid via Checkout | Historical/Record Payment |
|---|---|---|---|---|---|---|---|
| `paymentType` | `1` (DEPOSIT) | `0` (NONE) | `1` (DEPOSIT) | `1` (DEPOSIT) | `0` (NONE) | `0` (NONE) | **absent** |
| `paymentMode` | `1` (PERCENT) or `2` (FLAT) | `1` or `2` | `1` or `2` | `1` or `2` | `1` or `2` | `1` or `2` | `1` or `2` (user picks in form) |
| `paymentValue` | e.g. `20` (%) or `400` ($) | e.g. `50` or `200` | preserved | preserved | preserved | preserved | present (user picks in form) |
| `dueDate` | absent (deposits hide due date) | present | absent (preserved if was set) | absent (preserved if was set) | present (preserved) | present (preserved) | absent |
| `amount` | **absent** | **absent** | present | present | present | present | present |
| `date` | **absent** | **absent** | present | present | present | present | present |
| `method` | `"other"` (hook default) | `"other"` (hook default) | user selection (e.g. `"cash"`, `"check"`) | `"card"`, `"us_bank_account"` | user selection (e.g. `"cash"`, `"check"`) | `"card"`, `"us_bank_account"` | user selection (e.g. `"cash"`, `"card"`) |
| `provider` | absent | absent | absent | `"stripe"` or `"paypal"` | absent | `"stripe"` or `"paypal"` | absent |
| `transactionId` | absent | absent | auto-generated UUID | Stripe PI / PayPal order ID | auto-generated UUID | Stripe PI / PayPal order ID | auto-generated UUID |
| `label` | `"Deposit"` | `"Payment"` | `"Deposit"` | `"Deposit"` | `"Payment"` | `"Payment"` | `"Payment"` |
| `notes` | absent | absent | optional (user-entered) | auto: `"Paid online via Stripe..."` | optional (user-entered) | auto: `"Paid online via Stripe..."` | optional (user-entered) |
| `surcharge` | absent | absent | `0` | processing fee if applicable | `0` | processing fee if applicable | `0` |
| `payer`/`payerObj` | absent | absent | absent | `{ email: "buyer@..." }` | absent | `{ email: "buyer@..." }` | absent |
| `deleted` | `false` | `false` | `false` | `false` | `false` | `false` | `false` |

**The single distinguishing field for historical payments is `paymentType`: it is ABSENT.** Both deposits (`paymentType: 1`) and milestones (`paymentType: 0`) always have this field set. Historical payments share `paymentMode` and `paymentValue` (because the Record Payment form asks the user to pick Percent/Flat), but never get `paymentType` assigned.

**Quick identification rules for Mongo queries:**

| You're looking for... | Query filter |
|---|---|
| All upcoming (unpaid) payments | `{ amount: { $exists: false } }` |
| All completed payments | `{ amount: { $exists: true } }` |
| Deposits (any state) | `{ paymentType: 1 }` |
| Milestones (any state) | `{ paymentType: 0 }` |
| Historical/recorded payments (never scheduled) | `{ amount: { $exists: true }, paymentType: { $exists: false } }` |
| Paid via checkout (online) | `{ provider: { $exists: true } }` |
| Manually marked as paid (was scheduled) | `{ amount: { $exists: true }, provider: { $exists: false }, paymentType: { $exists: true } }` |
| Soft-deleted | `{ deleted: true }` |

### Lifecycle Diagram

```
[Upcoming Payment]                    [Record Payment]
       │                                     │
       │ invoiceUpdatePayment                │ addHistoricalPayments
       │ (add amount/date/method)            │ (create with amount/date/method)
       ▼                                     ▼
┌─────────────────────────────────────────────────┐
│          Completed Payment (Payment History)     │
│  - Reduces Balance Due                          │
│  - Copied into Invoice.payments[] subdocument   │
│  - Triggers bookkeeping sync (if enabled)       │
└─────────────────────────────────────────────────┘
```

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
