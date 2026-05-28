# Payment Scheduling — Diagrams

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        is-web-app (Next.js)                      │
│                                                                  │
│  features/payment-scheduling/                                    │
│  ├── PaymentScheduling.tsx         ← Main entry component        │
│  ├── PaymentDetailsSection         ← Balance summary             │
│  ├── UpcomingPaymentsSection       ← List + action buttons       │
│  ├── CompletedPaymentsSection      ← Payment history             │
│  ├── UpcomingPaymentFormModal      ← Add deposit / upcoming      │
│  ├── CompletedPaymentFormModal     ← Record past / mark paid     │
│  ├── DeletePaymentModal            ← Confirm delete              │
│  ├── PaymentRow / PaymentMenu      ← Row + three-dot menu        │
│  └── use-date-text.ts              ← Due date display logic      │
│                                                                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTP POST
                             │ X-Parse-Session-Token
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                    Parse Backend (Cloud Functions)              │
│                                                                │
│  invoiceAddPayment       ← Create deposit or upcoming          │
│  invoiceUpdatePayment    ← Edit, mark paid, or soft-delete     │
│  invoiceGetPayments      ← Fetch all for invoice               │
│  addHistoricalPayments   ← Record a past payment               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Payment Lifecycle

```
                    PAYMENT OBJECT LIFECYCLE
                    ────────────────────────

  CREATED (Upcoming)                   COMPLETED (Paid)
  ┌───────────────────────┐            ┌───────────────────────────┐
  │ objectId              │            │ objectId                  │
  │ label    "Deposit"    │            │ label    "Deposit"        │
  │ paymentType  DEPOSIT  │            │ paymentType  DEPOSIT      │
  │ paymentMode  PERCENT  │  Mark as   │ paymentMode  PERCENT      │
  │ paymentValue 20       │ ─── Paid ─▶│ paymentValue 20          │
  │ dueDate  (optional)   │            │ amount   35.70  ← added   │
  │                       │            │ date     2025-06-01 ← added│
  │ (no amount)           │            │ method   stripe ← added   │
  └───────────────────────┘            └───────────────────────────┘

  SOFT DELETED
  ┌───────────────────────┐
  │ ...same fields...     │
  │ deleted: true   ← set │
  └───────────────────────┘
  (never hard-deleted from DB)
```

---

## Happy Path 1: Request Deposit → Mark as Paid

```
User opens "Payment Scheduling" tab
          │
          ▼
  Clicks "Request Deposit"
          │
          ▼
  UpcomingPaymentFormModal (isDeposit=true)
  ┌──────────────────────────────┐
  │ • Amount: 20% or $35.70      │
  │ • Due date: (hidden)         │
  │ • paymentType: DEPOSIT       │
  └──────────────────────────────┘
          │ submit
          ▼
  POST invoiceAddPayment
          │
          ▼
  Deposit row appears in Upcoming Payments
  "Request Deposit" button disappears
  Email button → "Email Invoice For Deposit"
  Summary → "Deposit Due: $35.70"
          │
          │  (customer is notified, time passes)
          │
          ▼
  User clicks ⋮ on Deposit row → "Mark as Paid"
          │
          ▼
  CompletedPaymentFormModal opens (pre-filled)
  ┌──────────────────────────────┐
  │ • Amount: $35.70 (pre-filled)│
  │ • Payment method: required   │
  │ • Date: today (default)      │
  └──────────────────────────────┘
          │ submit
          ▼
  POST invoiceUpdatePayment
  (payment gains amount + date + method)
          │
          ▼
  Row moves to Payment History
  Balance Due: $178.50 → $142.80 ✓
```

---

## Happy Path 2: Add Upcoming Payment

```
User clicks "Add Upcoming Payment"
          │
          ▼
  UpcomingPaymentFormModal (isDeposit=false)
  ┌──────────────────────────────┐
  │ • Label: custom text         │
  │ • Amount: % or flat          │
  │ • Due date: shown + required │
  │ • paymentType: NONE          │
  └──────────────────────────────┘
          │ submit
          ▼
  POST invoiceAddPayment
          │
          ▼
  Payment row appears: "Due Jun 15, 2025"
  (balance unchanged until marked paid)
```

---

## Happy Path 3: Record a Past Payment

```
User clicks "Record Payment"
          │
          ▼
  CompletedPaymentFormModal (isHistorical=true)
  ┌──────────────────────────────┐
  │ • Amount: required           │
  │ • Payment method: required   │
  │ • Date: required             │
  │ • Notes: optional            │
  └──────────────────────────────┘
          │ submit
          ▼
  POST addHistoricalPayments
          │
          ▼
  Appears immediately in Payment History
  Balance Due reduced ✓
```

---

## Balance Calculation

```
┌─────────────────────────────────────────────────────────┐
│                   PAYMENT DETAILS SECTION               │
│                                                         │
│  Invoice Total          $178.50                         │
│  Paid                   $ 35.70  ← sum(completed.amount)│
│  Amount Remaining       $142.80                         │
│  Balance Due            $142.80  ← or "Deposit Due"     │
│                                     if deposit pending  │
└─────────────────────────────────────────────────────────┘

Formula:
  Balance Due = Invoice Total − Σ(completed payment amounts)

  Upcoming/scheduled payments → shown as hint label only
                               → do NOT reduce Balance Due
                               → only count once "Mark as Paid"
```

---

## "Request Deposit" Button Visibility

```
  depositExists = payments.some(p => p.paymentType === DEPOSIT)
  hasCompletedPayments = completedPayments.length > 0

  Show "Request Deposit" button only when:
    !depositExists AND !hasCompletedPayments

  Otherwise: only "Add Upcoming Payment" is shown
```

---

## Due Date Display Logic (`use-date-text.ts`)

```
  useDateText(payment, isUpcoming, format)
        │
        ├─ isUpcomingPayment(payment)?  (has no `amount`)
        │     NO  → show completed date: "May 20, 2025"
        │
        └─ YES → paymentType === NONE?
                    YES → show "Due Jun 15, 2025"
                    NO  → return null  (Deposits: no date shown)
```
