# Payment Scheduling — Mermaid Diagrams

## Component Architecture

```mermaid
graph TD
    PS[PaymentScheduling.tsx<br/>Main Entry]

    PS --> PDS[PaymentDetailsSection<br/>Balance Summary]
    PS --> UPS[UpcomingPaymentsSection<br/>Scheduled List]
    PS --> CPS[CompletedPaymentsSection<br/>Payment History]

    UPS --> UPFM[UpcomingPaymentFormModal<br/>Add Deposit / Upcoming]
    UPS --> PR[PaymentRow]
    PR --> PM[PaymentMenu ⋮<br/>Mark Paid / Edit / Delete]

    PM --> CPFM[CompletedPaymentFormModal<br/>Mark as Paid / Edit]
    PM --> DPM[DeletePaymentModal<br/>Confirm Delete]

    CPS --> PR2[PaymentRow]
    PR2 --> PM2[PaymentMenu ⋮<br/>Edit / Delete]
```

---

## Data Flow: Add a Payment

```mermaid
sequenceDiagram
    actor User
    participant UI as UpcomingPaymentFormModal
    participant Hook as usePayments / syncInvoicePayments
    participant Parse as Parse Cloud Functions

    User->>UI: Click "Request Deposit" or "Add Upcoming Payment"
    UI->>UI: Open modal (isDeposit flag set)
    User->>UI: Fill amount, mode, due date
    User->>UI: Submit
    UI->>Hook: onSubmit(values)
    Hook->>Parse: POST /functions/invoiceAddPayment<br/>{ invoiceId, payment }
    Parse-->>Hook: { success, payment }
    Hook->>Hook: syncInvoicePayments() → refetch
    Hook-->>UI: Updated payments list
    UI->>UI: Close modal
```

---

## Data Flow: Mark as Paid

```mermaid
sequenceDiagram
    actor User
    participant Row as PaymentRow / PaymentMenu
    participant Modal as CompletedPaymentFormModal
    participant Hook as usePayments / syncInvoicePayments
    participant Parse as Parse Cloud Functions

    User->>Row: Click ⋮ → "Mark as Paid"
    Row->>Modal: Open (pre-filled with upcoming payment data)
    User->>Modal: Confirm amount, select method, set date
    User->>Modal: Submit
    Modal->>Hook: onSubmit(values)
    Hook->>Parse: POST /functions/invoiceUpdatePayment<br/>{ payment: { ...existing, amount, date, method } }
    Parse-->>Hook: { success }
    Hook->>Hook: syncInvoicePayments() → refetch
    Hook-->>Row: Payment moves to Completed section
```

---

## Payment Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> Upcoming : invoiceAddPayment\n(no amount field)

    Upcoming --> Completed : Mark as Paid\n(invoiceUpdatePayment adds amount + date + method)
    Upcoming --> Deleted : Delete\n(invoiceUpdatePayment sets deleted: true)

    Completed --> Completed : Edit\n(invoiceUpdatePayment)
    Completed --> Deleted : Delete\n(invoiceUpdatePayment sets deleted: true)

    Deleted --> [*] : Soft deleted\n(never removed from DB)
```

---

## Balance Calculation

```mermaid
flowchart LR
    IT[Invoice Total\n$178.50]
    CP[Completed Payments\nsum of amount fields]
    BD[Balance Due]

    IT --> BD
    CP -->|subtract| BD

    UP[Upcoming Payments\nscheduled only]
    UP -. does NOT affect .-> BD
```

---

## "Request Deposit" Button Visibility

```mermaid
flowchart TD
    A[Render UpcomingPaymentsSection] --> B{depositExists?}
    B -- Yes --> D[Show only\nAdd Upcoming Payment]
    B -- No --> C{completedPayments\n.length > 0?}
    C -- Yes --> D
    C -- No --> E[Show both\nRequest Deposit\nAdd Upcoming Payment]
```

---

## Due Date Display Logic

```mermaid
flowchart TD
    A[useDateText called] --> B{isUpcomingPayment?\nno amount field}
    B -- No\nCompleted --> C[Show paid date\ne.g. May 20 2025]
    B -- Yes\nUpcoming --> D{paymentType === NONE?}
    D -- Yes\nRegular upcoming --> E[Show due date\ne.g. Due Jun 15 2025]
    D -- No\nDeposit --> F[Return null\nno date shown]
```
