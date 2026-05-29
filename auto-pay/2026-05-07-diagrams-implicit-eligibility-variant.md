# Diagrams: Implicit Eligibility Variant (Option E)

> **Context:** These diagrams reflect the design variant where auto-pay eligibility is derived
> implicitly from the presence of milestones, rather than an explicit `milestoneAutoPayEnabled`
> flag on the invoice. See Decision 9 (Alternative E) in decisions.md.
>
> **What changes vs. base diagrams:**
> - Phase 0 (merchant enables flag) is removed — no merchant action needed to offer auto-pay
> - Checkout eligibility check is "has future milestones?" not "is flag set?"
> - Vault state machine has no "Pending" state — vault goes directly NoVault → Active
> - ER diagram removes `milestoneAutoPayEnabled` from Invoice
> - Merchant lifecycle: no "enable" action, only "disable" (cancel after vault)
>
> **What stays the same:** All backend infrastructure, execution flow, retry logic, emails,
> AWS resources, data model tables, and API contracts are identical.

---

## End-to-End Flow Overview

```mermaid
sequenceDiagram
    participant Merchant as Merchant (Mobile)
    participant Parse as Parse Server
    participant Checkout as Checkout (is-unifiedxp)
    participant Stripe as is-stripe
    participant Payments as is-payments
    participant EB as EventBridge
    participant Lambda as Execution Lambda

    Note over Merchant,Parse: Phase 0: Merchant creates invoice
    Merchant->>Parse: Create milestones with due dates
    Note over Merchant,Parse: No "enable auto-pay" step needed.<br/>Milestones existing = auto-pay offered at checkout.

    Note over Checkout,Stripe: Phase 1: Client Vaults
    Checkout->>Checkout: Has milestones + feature flag on?
    Checkout->>Stripe: Pay deposit (PaymentIntent, setup_future_usage: off_session)
    Stripe->>Stripe: Store vault (upcoming_payment table)
    Stripe->>Payments: SQS: createUpcomingPayment {invoiceId, milestones[], consentGrantedAt}
    Payments->>Payments: Create upcoming_payments rows
    Payments->>EB: Create schedules per milestone (at dueDate in UTC)

    Note over EB,Lambda: Phase 2: Execution (on due date)
    EB->>Lambda: Fire at scheduled time (direct to SQS — no scheduler Lambda)
    Lambda->>Payments: Validate status = ACTIVE
    Lambda->>Stripe: Execute charge (is-stripe fetches vault internally)
    Lambda->>Payments: Record success/failure
    Note over Lambda,Parse: Parse Payment marked paid via Stripe webhook (automatic)

    Note over Merchant,Payments: Phase 3: Lifecycle
    Merchant->>Parse: Edit/add/delete milestone
    Merchant->>Payments: Fire-and-forget: update/add/cancel schedule
```

---

## Data Model Relationships

```mermaid
erDiagram
    INVOICE ||--o{ PARSE_PAYMENT : "has milestones"
    INVOICE ||--o| UPCOMING_PAYMENT_VAULT : "has vault (is-stripe)"
    INVOICE ||--o{ UPCOMING_PAYMENTS : "has schedules (is-payments)"
    PARSE_PAYMENT ||--o| UPCOMING_PAYMENTS : "milestoneId = remoteId"

    INVOICE {
        string remoteId PK
        string accountId
        object setting "no milestoneAutoPayEnabled field needed"
    }

    PARSE_PAYMENT {
        string objectId PK
        string remoteId "milestoneId in is-payments"
        string invoiceRemoteId FK
        number paymentType "0=NONE, 1=DEPOSIT"
        number paymentMode "1=PERCENT, 2=FLAT"
        number paymentValue
        date dueDate "trigger date for auto-pay"
        number amount "present = completed"
        date date "present = completed"
        string method "present = completed"
    }

    UPCOMING_PAYMENT_VAULT {
        serial id PK
        string invoice_id FK "UNIQUE"
        string account_id
        string customer_id "Stripe cus_xxx"
        string vaulted_token "Stripe pm_xxx"
        string payment_method_type "card or us_bank_account"
        string currency_code
        timestamp consent_granted_at
    }

    UPCOMING_PAYMENTS {
        serial id PK
        string invoice_id FK
        string milestone_id FK "Parse Payment remoteId"
        string account_id
        integer amount_cents "snapshot at creation"
        string status "ACTIVE or INACTIVE"
        string disabled_reason "nullable"
        date scheduled_date
        integer retry_count
        timestamp last_executed_at
    }
```

Note: `INVOICE.setting.milestoneAutoPayEnabled` is removed. Auto-pay eligibility is determined
entirely by: (1) invoice has upcoming milestones with due dates, and (2) feature flag is on for
the account. Vault record existing = auto-pay is active.

---

## Checkout UI Flow (Client Side)

```mermaid
flowchart TD
    A[Client opens checkout link] --> B{Feature flag on for account?}
    B -- No --> C[Standard checkout - deposit only]
    B -- Yes --> D{ExtendedCheckoutData.payments\nhas upcoming milestones?}
    D -- No --> C
    D -- Yes --> E[Show deposit amount + milestone schedule]
    E --> F{Client opts in to auto-pay?}
    F -- No --> G[Pay deposit only, no vault]
    F -- Yes --> H{PayPal selected?}
    H -- Yes --> I[Disable checkbox - show helper text]
    I --> G
    H -- No --> J[Pay deposit - PaymentIntent with setup_future_usage: off_session]
    J --> K[Vault created in is-stripe]
    K --> L{Any overdue milestones?}
    L -- Yes --> M[Schedule immediately]
    L -- No --> N[Schedule on due dates]
    M --> N
    N --> O[Show confirmation: Automatic Payments Enabled]
```

---

## Vault Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> NoVault : Invoice created with upcoming payments

    NoVault --> Active : Client pays first payment + opts in at checkout
    Note: No "Pending" state — vault is either absent or active.\nMerchant has no enable action. Auto-pay is offered\nwhenever upcoming payments exist + feature flag is on.

    Active --> Invalidated : Merchant edits/deletes/adds payment (cancel all)
    Active --> Invalidated : Merchant edits invoice (cancel all)
    Active --> Invalidated : Merchant cancels auto-pay (via Disable button)
    Active --> Invalidated : Admin disables
    Invalidated --> Active : Client re-vaults at next checkout (sees new schedule)

    Active --> Active : Payment charged successfully
    Active --> Active : Client pays one early (that schedule cancelled, rest stay)
    Active --> Active : Payment fails (retrying)
    Active --> Invalidated : Max retries reached\n(ALL payments cancelled)

    note right of Active : Payments LOCKED while active.\nNo edits/adds/deletes.\nVault record NEVER deleted\n(gets replaced on re-vault).
    note right of Invalidated : All schedules cancelled.\nVault record kept on is-stripe.\nClient must check out again\nto restart auto-pay.
```

---

## Merchant Lifecycle Actions (Mobile) — Implicit Variant, Locked After Vault

```mermaid
flowchart TD
    subgraph "Mobile App (vault active)"
        A[Merchant edits/deletes/adds payment] --> B{Vault active?}
        B -- Yes --> C[Confirmation modal:\nThis will cancel all automatic payments]
        C -- Confirm --> D[POST /upcoming-payment/disable]
        C -- Cancel --> E[No action]
        B -- No --> F[Normal Parse edit - no auto-pay involved]
        G[Merchant edits invoice] --> C
        H[Merchant cancels auto-pay] --> D
    end

    subgraph "is-payments"
        D --> I[Delete ALL EventBridge schedule pairs]
        I --> J[Set all rows INACTIVE]
    end

    subgraph "Checkout (client pays early)"
        K[Client pays via Pay Now / checkout link] --> L[POST /upcoming-payment/cancel-one]
        L --> M[Delete ONE schedule pair]
        M --> N[Set that row INACTIVE, MANUALLY_PAID]
    end

    subgraph "is-stripe"
        Note2[Vault record is never deleted.\nGets replaced on re-vault.]
    end

    Note1[No 'Enable Auto-pay' action.\nMerchant can only Cancel after\nclient has vaulted.\nPayments LOCKED while vault active.]
```

---

## Eligibility Signal Comparison

```
OPTION B (explicit toggle — base design):
─────────────────────────────────────────
Checkout gate:  invoice.setting.milestoneAutoPayEnabled === true
                AND feature flag on
                AND has milestones

Merchant flow:  Create invoice → add milestones → flip toggle → send invoice
                Client sees auto-pay opt-in at checkout

Mobile UI:      Toggle in invoice settings + status indicator + cancel button

Parse changes:  New field (invoice.setting.milestoneAutoPayEnabled)


OPTION E (implicit — this variant):
─────────────────────────────────────────
Checkout gate:  checkoutData.payments.filter(upcoming milestones).length > 0
                AND feature flag on
                (payments[] is ALREADY in ExtendedCheckoutData — no extra fetch needed)

Merchant flow:  Create invoice → add milestones → send invoice
                Client sees auto-pay opt-in at checkout automatically

Mobile UI:      Status indicator + cancel button only (no toggle)

Parse changes:  None
Checkout cost:  None — milestone data already in ExtendedCheckoutData.payments[]
```

> **Implementation note (updated 2026-05-08):** Earlier analysis flagged a hidden cost for
> Option E (milestone data not available at checkout). This was incorrect. `ExtendedCheckoutData`
> already includes `payments[]` — the full list of Payment objects for the invoice, fetched via
> `invoiceGetPayments` in `getDocumentData()`. Filtering for upcoming milestones is a trivial
> array filter on already-loaded data. Option E and Option B are equally simple at the checkout
> layer. The real tradeoff is product behavior (all milestone invoices opt-in vs. per-invoice
> merchant control), not implementation cost.
