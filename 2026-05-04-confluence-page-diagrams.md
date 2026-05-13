# Confluence Page Diagrams (Mermaid Source)

> Render these to PNG for embedding in Confluence.
> These correspond to the diagram placeholders in `2026-05-04-confluence-page.md`.

---

## Diagram 1: Today (manual)

```mermaid
graph LR
    subgraph Merchant
        M[Mobile App]
    end
    subgraph Parse
        P[(MongoDB<br/>Invoice + Payments)]
    end
    subgraph Client
        C[Checkout<br/>is-unifiedxp]
    end

    M -->|create invoice +<br/>milestones/upcoming payments| P
    M -->|mark paid manually| P
    C -->|pay via link| P

    style P fill:#f9f9f9
```

---

## Diagram 2: With Automatic Payments (Phase 1 — successive scheduling)

```mermaid
graph LR
    subgraph Merchant
        M[Mobile App]
    end
    subgraph Parse
        P[(MongoDB<br/>Invoice + Payments)]
    end
    subgraph Client
        C[Checkout<br/>is-unifiedxp<br/>shows schedule when vaulting]
    end
    subgraph is-stripe
        S[(upcoming_payment<br/>vault)]
    end
    subgraph is-payments
        IP[(upcoming_payments<br/>QUEUED + ACTIVE)]
    end
    subgraph AWS
        EB[EventBridge<br/>Scheduler]
        SQS[SQS Execution Queue]
        ESQS[SQS Events Queue]
        L[Execution<br/>Lambda]
        EL[Events<br/>Lambda]
    end

    M -->|create milestones<br/>aka upcoming payments| P
    C -->|pay + vault| S
    S -->|vault stored| S
    S -->|SQS: vault confirmed,<br/>create rows + schedule NEXT| IP
    IP -->|schedule PAIR<br/>for next payment only<br/>charge + reminder| EB
    EB -->|on due date<br/>charge| SQS
    EB -->|N days before<br/>reminder| ESQS
    SQS --> L
    ESQS --> EL
    EL -.->|Automatic Payment Request<br/>reminder email| C
    L -->|charge card| S
    L -->|on success:<br/>schedule NEXT payment| IP
    L -.->|webhook marks paid| P
    M -->|skip/cancel-one/disable| IP
```

---

## Diagram 3: Recurring Payments (current architecture, for reference)

```mermaid
graph LR
    subgraph Merchant
        M[Mobile/Web App]
    end
    subgraph is-recurring-invoices
        RI[Scheduler<br/>generates invoices]
    end
    subgraph Client
        C[Checkout<br/>is-unifiedxp<br/>vaults at first invoice]
    end
    subgraph is-stripe
        S[(recurring_payment<br/>vault)]
    end
    subgraph is-payments
        IP[(recurring_payments<br/>execution state)]
    end
    subgraph AWS
        EB[EventBridge<br/>Scheduler]
        SQS[SQS Queue]
        L[Execution<br/>Lambda]
    end

    M -->|create recurring series| RI
    RI -->|generate invoice on schedule| SQS
    C -->|pay first invoice + vault| S
    S -->|SQS: create RP| IP
    RI -->|on interval| EB
    EB -->|on due date| SQS
    SQS --> L
    L -->|charge card| S
    L -.->|webhook marks paid| RI
    M -->|disable RP| IP
```
