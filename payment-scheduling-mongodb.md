# Payment Scheduling — MongoDB / Parse Backend Model

## Collection

**Parse class / MongoDB collection name:** `Payment`

**Defined in:** `is-parse-server/cloud/parse.ts`

```typescript
export class Payment extends Parse.Object {
  constructor(options?: any) {
    super('Payment', options);
  }
}
Parse.Object.registerSubclass('Payment', Payment);
```

---

## Dual Storage Architecture

Payments are stored in **two places**:

| Store | What's stored | Purpose |
|---|---|---|
| `Payment` collection | All payments — both upcoming AND completed | Source of truth, queried by all clients |
| `Invoice.payments[]` array (subdocument) | Completed payments only (legacy fields) | Quick access on invoice reads |

**Upcoming payments are NOT stored in `Invoice.payments[]`** — only in the `Payment` collection.

Fields stored in `Invoice.payments[]` subdocument: `amount`, `date`, `paymentMethod`, `notes`, `transactionId`, `payer`, `surcharge`.
Scheduled-payment fields (`paymentType`, `paymentMode`, `paymentValue`, `dueDate`, `label`) are excluded.

---

## Full Schema

```typescript
type Payment = {
  // Identifiers
  objectId: string           // Parse auto-generated PK
  remoteId?: string          // UUID, account-scoped identifier
  invoiceRemoteId: string    // Required — links to Invoice.remoteId
  account: Pointer           // Required — Parse pointer to Account

  // Completed payment fields (absent = upcoming)
  amount?: number            // Actual dollars paid (2 decimal places max)
  date?: Date                // When paid
  method?: PaymentMethod     // 'cash' | 'check' | 'cheque' | 'check_scan' | 'bank' |
                             // 'creditCard' | 'paypal' | 'other' | 'stripe' |
                             // 'pointOfSale' | 'peerToPeer' | 'mobilePaymentApp' |
                             // 'debit' | 'card' | 'venmo'
  transactionId?: string     // Auto-generated if date exists and not provided
  surcharge?: number         // Processing fee (defaults to 0)

  // Scheduled/upcoming payment fields
  paymentType?: 0 | 1        // 0 = NONE (regular upcoming), 1 = DEPOSIT
  paymentMode?: 0 | 1 | 2    // 0 = NONE, 1 = PERCENT, 2 = FLAT
  paymentValue?: number      // % (0-100) if PERCENT, or dollar amount if FLAT
  dueDate?: Date             // Scheduled due date
  label?: string             // Display label e.g. "Deposit", "Payment"
  paymentTriggeredDate?: Date

  // Optional metadata
  notes?: string             // Max 20,000 characters
  statementDescriptor?: string  // Bank statement text (max 100 chars)
  payerObj?: { email: string }  // Max 100 chars
  provider?: string          // 'stripe' | 'paypal'

  // System fields
  deleted: boolean           // Soft delete (defaults to false)
  updated: number            // Timestamp
  updatedAt: Date
}
```

---

## Upcoming vs Completed — State Determined by Fields

No explicit status field. State is inferred:

| State | `amount` | `date` | `method` | `dueDate` |
|---|---|---|---|---|
| Upcoming (scheduled) | absent | absent | absent | present |
| Completed (paid) | present | present | present | preserved |

When an upcoming payment is marked paid, `amount`, `date`, `method`, and `transactionId` are added — `dueDate` is preserved.

---

## Transition: Upcoming → Completed

Handled by `markPaymentsPaidOrUnpaid.ts`:

```
Before (upcoming):
  { dueDate: Date, paymentType: 1, paymentMode: 1, paymentValue: 20 }

After (marked paid):
  { dueDate: Date, paymentType: 1, paymentMode: 1, paymentValue: 20,
    amount: 35.70, date: Date, method: 'stripe', transactionId: '...' }
```

---

## Soft Delete

Payments are never hard-deleted:
```
{ deleted: true }
```
Queries filter `deleted != true` by default. `invoiceGetPayments` accepts `includeDeleted` flag.

---

## Query Patterns

**File:** `is-parse-server/cloud/collections/payment/getPayments.ts`

Indexed query fields:
- `invoiceRemoteId` — primary query key
- `account` — always scoped to account

Query limit: 5000 payments per invoice (`PAYMENTS_QUERY_LIMIT` env var).

---

## Before-Save Hooks

**File:** `is-parse-server/cloud/collections/payment/paymentHooks.ts`

| Condition | Action |
|---|---|
| Invalid `method` value | Defaults to `'other'` |
| `date` exists and no `transactionId` | Auto-generates a UUID |
| Missing/invalid `surcharge` | Defaults to `0` |
| Invalid payer email | Cleaned/removed |
| `notes` > 20,000 chars | Throws error |
| Missing `invoiceRemoteId` | Throws error |
| `statementDescriptor` > 100 chars | Throws error |

---

## Bookkeeping Sync

After any payment save, if the Optimizely `bookkeeping` flag is enabled:
```
syncPaymentToBookkeeping({ accountId, remoteId: paymentObjectId, entityType: 'PAYMENT' })
```
Triggered on both web and mobile after every add/update/mark-paid.
