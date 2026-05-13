# Diagrams: Payment Scheduling — Current State (Before Auto-Pay)

> **Purpose:** Baseline "before" diagrams for the Deposits & Milestones feature as it exists
> today — no automatic charging, no vaulting, no EventBridge. Due dates are purely informational.
> These diagrams pair with the "after" diagrams in:
> - `2026-05-04-deposits-milestones-auto-payments-diagrams.md` (with explicit toggle)
> - `2026-05-07-diagrams-implicit-eligibility-variant.md` (with implicit eligibility)

---

## Current Flow Overview

```mermaid
sequenceDiagram
    participant Merchant as Merchant (Mobile)
    participant Parse as Parse Server
    participant Checkout as Checkout (is-unifiedxp)
    participant Client as Client

    Note over Merchant,Parse: Merchant sets up invoice
    Merchant->>Parse: Create invoice (total, client)
    Merchant->>Parse: Set deposit (depositType, depositRate/depositAmount on invoice.setting)
    Merchant->>Parse: Add upcoming payments (label, paymentValue, paymentMode, dueDate)

    Note over Merchant,Client: Merchant shares invoice
    Merchant->>Client: Share invoice link

    Note over Client,Checkout: Client pays deposit at checkout
    Client->>Checkout: Open invoice link
    Checkout->>Client: Show deposit amount due
    Client->>Checkout: Pay deposit (card/PayPal/ACH)
    Checkout->>Stripe: confirmPayment() → return URL hits is-stripe
    Stripe->>Parse: Parse.Cloud.run('invoiceAddPayment') — records Payment object
    Parse->>Parse: updatePaymentDetails() → recalculates invoice.balanceDue

    Note over Merchant,Parse: Future milestones — manual only
    Client->>Merchant: Pays milestone (outside system: bank transfer, cash, check)
    Merchant->>Parse: Manually mark upcoming payment as paid (select method, enter amount)
    Note over Merchant,Parse: OR merchant requests card payment separately

    Note over Merchant,Parse: Due dates pass — no automatic action
    Note over Merchant,Parse: Overdue = red indicator only. Merchant must follow up manually.
```

---

## Data Model (Current)

```mermaid
erDiagram
    INVOICE ||--o{ PARSE_PAYMENT : "has payments"

    INVOICE {
        string remoteId PK
        string accountId
        number totalCents
        string clientId
        object setting "depositType, depositRate, depositAmount"
    }

    INVOICE_SETTING {
        enum depositType "controls deposit type"
        number depositRate "% value if PERCENT mode"
        number depositAmount "flat value if FLAT mode"
    }

    PARSE_PAYMENT {
        string objectId PK
        string invoiceRemoteId FK
        string label "e.g. '50% Deposit', 'Milestone 2'"
        number paymentType "DEPOSIT=1, NONE=0"
        number paymentMode "PERCENT=1, FLAT=2"
        number paymentValue "amount or % to collect"
        date dueDate "informational only — nothing fires"
        number amount "null = unpaid, set = paid"
        date date "null = unpaid, set = paid"
        string method "card, cash, check, bank..."
        string transactionId "Stripe/PayPal ID if card"
    }
```

Note: A payment is **unpaid** when `amount` and `date` are null. It is **paid** when both are set.
There is no explicit `status` field — state is inferred from data shape.

---

## Milestone States (Current)

```mermaid
stateDiagram-v2
    [*] --> Upcoming : Merchant adds upcoming payment\n(label, amount/%, dueDate)

    Upcoming --> Overdue : dueDate passes\n(no automatic action)
    Overdue --> Upcoming : Merchant edits dueDate\nto future date

    Upcoming --> Paid : Merchant marks as paid\n(sets amount, date, method)
    Overdue --> Paid : Merchant marks as paid\n(sets amount, date, method)

    note right of Upcoming : dueDate is informational.\nNothing happens automatically.\nShown in "Upcoming Payments" list.

    note right of Overdue : Same as Upcoming but\ndisplayed in red.\nMerchant must follow up.

    note right of Paid : Moves to "Payment History".\namount + date + method all set.
```

---

## Merchant Mobile UI Structure (Current)

```mermaid
flowchart TD
    A[Invoice Screen] --> B[PaymentSchedulingScreen]

    B --> C[InvoiceTotalSection]
    B --> D[UpcomingPaymentsSection]
    B --> E[PaymentHistorySection]
    B --> F[CheckDepositsSection]

    C --> C1[Total / Paid / Balance Due]
    C --> C2[Nearest due date\nred if past due]

    D --> D1[Deposit row if unpaid]
    D --> D2[Milestone rows, each with\nlabel + amount + dueDate]
    D --> D3[Add Upcoming Payment button]

    E --> E1[Paid deposits]
    E --> E2[Paid milestones\nmethod + date shown]

    D1 --> G[ManagePaymentScreen]
    D2 --> G

    G --> G1[Edit label / amount / dueDate]
    G --> G2[Mark as Paid\nselect method]
    G --> G3[Delete]
```

---

## Checkout Flow (Current — Deposit Only)

```mermaid
flowchart TD
    A[Client opens invoice link] --> B[Checkout loads invoice]
    B --> C{Deposit configured?}
    C -- No --> D[Full invoice amount shown]
    C -- Yes --> E[Deposit amount shown\ne.g. 20% of total]
    D --> F[Client selects payment method\ncard / PayPal / ACH / etc.]
    E --> F
    F --> G{Existing saved card?}
    G -- Yes --> H[Option: Save my payment info\nfor faster checkout next time]
    G -- No --> H
    H --> I[Client pays]
    I --> J[Parse Payment record created\namount + date + method + transactionId]
    J --> K[Invoice balance updated]
    K --> L[Upcoming milestones still shown\nas manual items — no scheduling]

    note1[Convenience vaulting checkbox is\nindependent of milestone scheduling.\nSaved card is for faster checkout\nnext time, not for auto-charging.]
    note2[ExtendedCheckoutData.payments already\ncontains ALL Payment objects for the invoice\nincluding upcoming milestones — fetched via\ninvoiceGetPayments at page load.]
```

---

## Data Flow Summary (Current)

```mermaid
graph TB
    subgraph "Merchant (Mobile)"
        M1[Create invoice + milestones]
        M2[Share invoice link]
        M3[Manually mark milestones paid]
        M4[Edit / delete milestones]
    end

    subgraph "Parse Server"
        P1[Invoice object]
        P2[Payment objects\ndeposit + milestones]
    end

    subgraph "Checkout is-unifiedxp"
        C1[Show deposit amount]
        C2[Collect payment]
        C3[Record payment to Parse]
    end

    subgraph "Client"
        CL1[Pays deposit at checkout]
        CL2[Pays future milestones\noutside system or manually]
    end

    M1 -->|Cloud.run invoiceCreate| P1
    M1 -->|Cloud.run invoiceAddPayment| P2
    M2 -->|invoice link| CL1
    CL1 --> C1
    C1 --> C2
    C2 -->|Stripe / PayPal charge| C3
    C3 -->|update Payment| P2
    CL2 -->|bank / cash / check| M3
    M3 -->|Cloud.run markPaid| P2
    M4 -->|Cloud.run invoiceUpdatePayment| P2
```

---

## Before vs. After: Key Differences

| Aspect | Before (today) | After (with auto-pay) |
|--------|---------------|----------------------|
| Due dates | Informational only | Trigger real charges |
| Client at checkout | Pays deposit, nothing else | Pays deposit + optionally vaults card |
| Future milestones | Manual follow-up by merchant | Auto-charged on due date |
| Payment method storage | Convenience vault (optional, for faster checkout) | Auto-pay vault (explicit consent to charge) |
| Overdue milestones | Red indicator only | Charged immediately at vault time |
| Merchant action needed | Follow up for every milestone | Only needed on failure |
| New backend services | None | is-stripe vault table, is-payments schedules, EventBridge, SQS, Lambdas |
| Parse changes | None | None (auto-pay state lives in is-stripe + is-payments) |
| Checkout milestone data | `payments[]` already in `ExtendedCheckoutData` (fetched via `invoiceGetPayments`) | Same — no new fetch needed for eligibility check or post-payment schedule cancellation |

---

## Implementation Notes

### How Checkout Records a Payment to Parse (verified in code)

Checkout does **not** call Parse directly. The full chain:

1. **is-unifiedxp** — client pays via Stripe Elements (`confirmPayment()`). The return URL points to is-stripe: `{IS_STRIPE_SERVICE_URL}/payment-intent/confirm/:documentId`
2. **is-stripe** (`routes/public/payment-intent-confirm.ts:159`) — detects success, calls `addPaymentToDocument()`
3. **is-stripe** (`services/document-add-payment.ts:17` → `services/invoice-payments.ts:40`) — calls `invoiceAddPayment()` on Parse
4. **is-stripe** (`utils/parse/public-invoice.ts:164`) — `Parse.Cloud.run('invoiceAddPayment', { invoiceId, payment }, { useMasterKey: true })`
5. **Parse Server** (`cloud/collections/invoice/functions/invoiceAddPayment.ts:111`) — creates Payment object, calls `addPaymentToInvoice()`
6. **Parse Server** (`cloud/collections/invoice/utils/updatePaymentDetails.ts:13`) — recalculates and saves `invoice.balanceDue` via `getInvoiceBalance()` from `@invoice-simple/calculator`

**Key point:** `invoice.balanceDue` is updated automatically as a side effect of `invoiceAddPayment` — there is no separate "update balance" call. The same chain fires whether payment is recorded via checkout (is-stripe → Parse) or by merchant marking paid manually (mobile → Parse.Cloud.run directly).
