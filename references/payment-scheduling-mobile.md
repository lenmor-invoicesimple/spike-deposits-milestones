# Payment Scheduling — Mobile (is-mobile)

## Navigation

Payment Scheduling is accessed from the **invoice edit screen** via a tappable row (`PaymentSchedulingSection`) with a chevron. It navigates to a dedicated full-screen: `PaymentSchedulingScreen`.

Only shown when:
- Payments feature is enabled (`isPaymentsEnabled()`)
- Invoice type is `DOCTYPE_INVOICE` (not estimates)
- Invoice is not a recurring invoice series

```
Invoice Edit Screen
  └── PaymentSchedulingSection (tappable row)
        └── navigates to → PaymentSchedulingScreen
```

---

## Screen Structure

`PaymentSchedulingScreen` is a single scrollable full-screen view with 4 sections:

| Section | Component | Purpose |
|---|---|---|
| Balance Summary | `InvoiceTotalSection` | Invoice total, paid, remaining, next due date |
| Upcoming Payments | `UpcomingPaymentsSection` | Scheduled payments list + action buttons |
| Payment History | `PaymentHistorySection` | Completed payments + "Record Payment" button |
| Check Deposits | `CheckDepositsSection` | Stripe check capture status (mobile-only) |

---

## Payment Management Screens

### ManagePaymentScreen
Add or edit an upcoming payment or deposit.

**Form fields:**
- Payment name (readonly for deposits)
- Payment mode: flat or percentage
- Amount or rate
- Due date (upcoming payments only — not shown for deposits)
- Delete button (edit mode only)

**Submissions:**
- **Save** → creates/updates via `invoiceAddPayment` or `invoiceUpdatePayment`
- **Mark as Paid** → shows action sheet to select payment method, then calls `invoiceUpdatePayment` with `markPaid: true`

### ManageHistoricalPaymentScreen
Record or edit a completed/past payment.

**Form fields:** mode, amount/rate, payment method, date, notes

**Backend:** `addHistoricalPayments` (new) or `invoiceUpdatePayment` (edit)

---

## Parse Cloud Functions Called

Same functions as web-app:

| Operation | Cloud Function |
|---|---|
| Fetch payments | `invoiceGetPayments` |
| Add upcoming / deposit | `invoiceAddPayment` |
| Edit / mark as paid / delete | `invoiceUpdatePayment` |
| Add historical payment | `addHistoricalPayments` |
| Mark invoice paid/unpaid | `markPaymentsPaidOrUnpaid` |

Soft delete: `invoiceUpdatePayment` with `{ deleted: true }` — same as web.

---

## State & Data Layer

| Layer | Technology | Purpose |
|---|---|---|
| UI state | MobX (`PaymentsStore`) | Observable store, reactive UI |
| Form state | Formik | Form validation in payment screens |
| Local cache | Realm | Offline-first local storage, real-time listeners |
| Backend | Parse Cloud Functions | Source of truth |

**Realm listeners** drive live UI updates — when a payment changes in Realm, the section re-renders automatically without an explicit fetch.

---

## Mobile-Only Features

### Check Deposit Scanning (Stripe)
If Stripe check capture is enabled and accepting payments, "Mark as Paid" shows an intermediate action sheet:
- **Deposit Check** → triggers camera-based check scanning flow
- **Record Payment** → standard payment method selection

This entire flow (`CheckDepositsSection`, `depositCheckConsentPromptHandler`) does not exist on web.

### "Past Due" indicator
`InvoiceTotalSection` highlights overdue payments in red and shows "Past Due" next to the next due date. Web shows due date text only with no overdue styling.

### Offline-first
All payments are cached in Realm. The app works offline and syncs when reconnected. Web has no local cache — it always requires a live Parse connection.

---

## Key Differences vs Web-App

| | Web (is-web-app) | Mobile (is-mobile) |
|---|---|---|
| **Navigation** | Tab inside invoice editor ("Payment Scheduling" tab) | Full-screen pushed from invoice edit screen |
| **State management** | Zustand + react-hook-form | MobX + Formik |
| **Local storage** | None (Parse-only) | Realm (offline-first) |
| **Form pattern** | Modal overlays (`UpcomingPaymentFormModal`, `CompletedPaymentFormModal`) | Dedicated screens (`ManagePaymentScreen`, `ManageHistoricalPaymentScreen`) |
| **Mark as Paid** | Opens `CompletedPaymentFormModal` in place | Action sheet → `ManagePaymentScreen` |
| **Check deposit scanning** | Not available | Available (Stripe check capture) |
| **Overdue styling** | No | Red "Past Due" label |
| **Bookkeeping sync** | Optimizely-gated, client-side | Same — triggers after every payment operation |
| **Parse functions** | Identical | Identical |
| **Payment types/modes** | Identical | Identical |
| **Balance formula** | Identical | Identical |

---

## File Locations

```
is-mobile/src/
├── features/documents/screens/
│   ├── payment-scheduling.screen.tsx    ← Main screen
│   ├── manage-payment.screen.tsx        ← Add/edit upcoming or deposit
│   └── manage-historical-payment.screen.tsx  ← Record/edit historical
├── features/documents/navigators/
│   └── invoices.navigator.tsx           ← PaymentSchedulingScreen registration
├── features/upcoming-payments/section/
│   ├── payment-scheduling-section.tsx   ← Nav button on invoice screen
│   ├── upcoming-payments-section.tsx    ← Upcoming list + buttons
│   ├── invoice-total-section.tsx        ← Balance summary
│   ├── payment-history-section.tsx      ← Completed payments
│   └── check-deposits-section.tsx       ← Stripe check status (mobile-only)
├── features/upcoming-payments/components/manage-payment/
│   ├── mark-paid-button.tsx
│   ├── payment-mode-field.tsx
│   ├── payment-date-picker-field.tsx
│   ├── payment-rate-field.tsx
│   └── payment-method-field.tsx
├── services/parse/models/payment.ts     ← All Parse Cloud Function calls
├── services/realm/entities/payments/
│   └── repository.ts                    ← Realm queries + local cache
└── stores/Payments.store.ts             ← MobX store
```
