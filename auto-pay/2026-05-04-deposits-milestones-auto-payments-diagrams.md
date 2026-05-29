# Deposits & Milestones — Automatic Payments: Diagrams

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

    Note over Merchant,Parse: Phase 0: Setup
    Merchant->>Parse: Enable auto-pay (set milestoneAutoPayEnabled)
    Merchant->>Parse: Create milestones with due dates

    Note over Checkout,Stripe: Phase 1: Client Vaults
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
        object setting "milestoneAutoPayEnabled: boolean"
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

---

## Execution Lambda Decision Flow

```mermaid
flowchart TD
    A[Message received from SQS] --> B[Fetch upcoming_payments row]
    B --> C{status = ACTIVE?}
    C -- No --> D[Skip - log and exit]
    C -- Yes --> E[POST is-stripe /execute\nis-stripe fetches vault internally]
    E --> G{Stripe response?}
    G -- succeeded --> H[Record success in is-payments]
    H --> I[Set INACTIVE + MILESTONE_COMPLETED]
    I --> K[Send success emails]
    K --> K2[Parse Payment marked paid via Stripe webhook]
    G -- failed --> L[POST is-payments /record-failure\nreturns shouldScheduleRetry]
    L --> M{shouldScheduleRetry?}
    M -- Yes --> N[Create EventBridge retry schedule]
    N --> O[Send retrying notification to merchant]
    M -- No --> P[Set INACTIVE + MAX_RETRIES\nCancel ALL remaining milestones]
    P --> Q[Send failure emails to merchant + client]
```

---

## Retry Schedule Timeline

```mermaid
gantt
    title Milestone Payment Retry Timeline (Card)
    dateFormat YYYY-MM-DD
    axisFormat %b %d

    section Upcoming Payment: Midpoint $2000
    Due date (initial attempt)     :milestone, m1, 2026-06-01, 0d
    Retry 1 (+1 day)               :milestone, m2, 2026-06-02, 0d
    Retry 2 (+3 days)              :milestone, m3, 2026-06-05, 0d
    Retry 3 (+6 days / final)      :milestone, m4, 2026-06-11, 0d

    section Upcoming Payment: Final $2000
    Due date (independent)         :milestone, m5, 2026-06-08, 0d
```

Note: Retries are independent per milestone. Two milestones can execute on the same day (they are separate obligations).

---

## Merchant Lifecycle Actions (Mobile) — Locked After Vault

```mermaid
flowchart TD
    subgraph "Mobile App (vault active)"
        A[Merchant edits/deletes/adds payment] --> B{Vault active?}
        B -- Yes --> C[Confirmation modal:\nThis will cancel all automatic payments]
        C -- Confirm --> D[POST /upcoming-payment/disable]
        C -- Cancel --> E[No action]
        B -- No --> F[Normal Parse edit - no auto-pay involved]
        G[Merchant edits invoice] --> C
        H[Merchant/Admin cancels auto-pay] --> D
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
```

---

## Vault Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> NoVault : Invoice created with upcoming payments

    NoVault --> Pending : Merchant enables milestoneAutoPayEnabled
    Pending --> Active : Client pays first payment + vaults at checkout
    Active --> Invalidated : Merchant edits/deletes/adds payment (cancel all)
    Active --> Invalidated : Merchant edits invoice (cancel all)
    Active --> Invalidated : Merchant cancels auto-pay
    Active --> Invalidated : Admin disables
    Invalidated --> Pending : Merchant re-enables
    Pending --> Active : Client re-vaults (sees new schedule, re-consents)

    Active --> Active : Payment charged successfully
    Active --> Active : Client pays one early (that schedule cancelled, rest stay)
    Active --> Active : Payment fails (retrying)
    Active --> Invalidated : Max retries reached\n(ALL payments cancelled)

    note right of Active : Payments LOCKED while active.\nNo edits/adds/deletes.\nVault record NEVER deleted\n(gets replaced on re-vault).
    note right of Invalidated : All schedules cancelled.\nVault record kept on is-stripe.\nClient must check out again\nto restart auto-pay.
```

---

## Checkout UI Flow (Client Side)

```mermaid
flowchart TD
    A[Client opens checkout link] --> B{milestoneAutoPayEnabled?}
    B -- No --> C[Standard checkout - pay full amount]
    B -- Yes --> D{Has future milestones?}
    D -- No --> C
    D -- Yes --> E[Show deposit amount + milestone schedule]
    E --> F{Client opts in to auto-pay?}
    F -- No --> G[Pay deposit only, no vault]
    F -- Yes --> H[Pay deposit - PaymentIntent with setup_future_usage: off_session]
    H --> I[Vault created in is-stripe]
    I --> J{Any overdue milestones?}
    J -- Yes --> K[Schedule immediately]
    J -- No --> L[Schedule on due dates]
    K --> L
    L --> M[Show confirmation: schedule summary]
```

---

## Infrastructure Architecture

```mermaid
graph TB
    subgraph "Clients"
        Mobile[Mobile App]
        Web[Web App]
        CX[Checkout is-unifiedxp]
    end

    subgraph "Parse"
        Parse[Parse Server<br/>Payment collection]
    end

    subgraph "is-stripe"
        StripeAPI[REST API]
        StripeDB[(upcoming_payment<br/>vault per invoice)]
        StripeSvc[Stripe SDK]
    end

    subgraph "is-payments"
        PayAPI[REST API]
        PayDB[(upcoming_payments<br/>per-milestone state)]
        EventsQ[payment-events-queue.fifo]
        EventsL[events-lambda<br/>topic dispatcher]
    end

    subgraph "AWS"
        EB[EventBridge Scheduler<br/>schedule per upcoming payment]
        ExecQ[upcoming-payment-execution-queue]
        ExecL[upcoming-payment-execution-lambda]
        DLQ[upcoming-payment-execution-dlq]
    end

    Mobile -->|CRUD payments| Parse
    Mobile -->|fire-and-forget| PayAPI
    Web -->|CRUD payments| Parse
    Web -->|fire-and-forget| PayAPI
    CX -->|vault + pay| StripeAPI

    StripeAPI --> StripeDB
    StripeAPI -->|SQS| EventsQ
    EventsQ --> EventsL
    EventsL --> PayDB
    EventsL --> EB

    PayAPI --> PayDB

    EB -->|at scheduled time| ExecQ
    ExecQ --> ExecL
    ExecL -->|validate| PayAPI
    ExecL -->|fetch vault + charge| StripeAPI
    ExecL -->|record result| PayAPI
    ExecL -->|mark paid| Parse
    ExecL -.->|retry| EB

    ExecQ -.->|failed 3x| DLQ
```

---

## Schedule Naming Convention

```
Charge:   upcoming-{invoiceId}-{paymentId}
Reminder: upcoming-reminder-{invoiceId}-{paymentId}
Retry:    upcoming-retry-{invoiceId}-{paymentId}-{retryCount}
RP:       payment-retry-{seriesId}-{retryCount}
```

All in shared `is-payments-retry-schedule-group`. No collisions due to distinct prefixes.

---

## Modification Impact Matrix (Locked After Vault — Decision 14)

```mermaid
flowchart LR
    subgraph "Any merchant modification (cancel all)"
        A1[Edit upcoming payment] --> D1[Cancel ALL schedule pairs]
        A2[Delete upcoming payment] --> D1
        A3[Add upcoming payment] --> D1
        A4[Edit invoice total/client] --> D1
        A5[Merchant cancels auto-pay] --> D1
        D1 --> D2[Set all rows INACTIVE]
        D2 --> D3[Client must re-vault]
    end

    subgraph "Client pays early (cancel one)"
        B1[Client pays via Pay Now / checkout] --> B2[Cancel ONE schedule pair]
        B2 --> B3[Set that row INACTIVE, MANUALLY_PAID]
        B3 --> B4[Remaining schedules stay active]
    end
```
