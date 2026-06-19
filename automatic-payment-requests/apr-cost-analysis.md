# APR — Cost & Scale Analysis

> Last updated: 2026-06-05
> Based on AWS on-demand pricing (US East). Can be recalculated with reserved/savings plan rates.

---

## Pricing Used

| Service | Unit | Price |
|---|---|---|
| Lambda requests | per 1M invocations | $0.20 |
| Lambda compute | per GB-second | $0.0000166667 |
| SQS Standard | per 1M messages | $0.40 |
| Step Functions (Standard) | per 1K state transitions | $0.025 |

---

## Architectures Compared

### Design A — 2 Lambdas, L1 loops (current plan with p-limit)

```
L1 (Sweep + PG insert + Email, p-limit(10)): 512MB, runs once/day
L2 (Parse Writer, ×50 concurrent via SQS): 256MB, per invoice
```

- L1 processes all invoices in a concurrent loop (10 at a time)
- L2 handles Parse Msg writes in parallel via SQS
- **Hard ceiling: ~80k matched invoices** (L1 hits 15-min Lambda limit)

### Design B — 3 Lambdas, all per-invoice work via SQS

```
L1 (Sweep + Publish): 512MB, ~5s, runs once/day
L2 (PG insert + Email Worker, ×50 concurrent): 256MB, ~110ms per invocation
L3 (Parse Writer, ×50 concurrent): 256MB, ~350ms per invocation
```

- L1 just queries Mongo and dumps messages into SQS — always ~5s
- L2 and L3 are short-lived workers (sub-second each), no timeout risk
- **No ceiling** — scale by increasing maxConcurrency

### Design C — Step Functions + pagination (Fundbox pattern)

```
Step Function pagination loop (500 invoices/page, cursor-based)
+ same L2/L3 workers as Design B
```

- Used by Fundbox because DynamoDB scans require cursor pagination across multiple Lambda invocations
- APR doesn't need this — a single indexed Mongo query returns all matched invoices
- Adds Step Function cost with no architectural benefit over Design B

---

## Cost Tables

### Design A — Daily cost (2 Lambdas, p-limit)

| Matched invoices | L1 duration | L1 cost | L2 cost | SQS cost | **Daily total** |
|---|---|---|---|---|---|
| 1,000 | ~11s | $0.000094 | $0.0006 | $0.0004 | **$0.001** |
| 10,000 | ~1.9 min | $0.001 | $0.006 | $0.004 | **$0.011** |
| 50,000 | ~9.2 min | $0.005 | $0.030 | $0.020 | **$0.055** |
| 80,000 | ~14.7 min ⚠️ | $0.008 | $0.049 | $0.032 | **$0.089** |
| 100,000 | **fails** | — | — | — | — |

### Design B — Daily cost (3 Lambdas, SQS fan-out)

| Matched invoices | L1 cost | L2 cost | L3 cost | SQS cost (2 queues) | **Daily total** |
|---|---|---|---|---|---|
| 1,000 | $0.00004 | $0.0007 | $0.002 | $0.0008 | **$0.003** |
| 10,000 | $0.00004 | $0.007 | $0.015 | $0.008 | **$0.030** |
| 50,000 | $0.00004 | $0.033 | $0.075 | $0.040 | **$0.148** |
| 100,000 | $0.00004 | $0.067 | $0.150 | $0.080 | **$0.297** |
| 500,000 | $0.00004 | $0.330 | $0.750 | $0.400 | **$1.480** |
| 1,000,000 | $0.00004 | $0.670 | $1.500 | $0.800 | **$2.970** |

### Design C — Daily cost (Step Functions + SQS fan-out)

| Matched invoices | Step Function cost | Lambda costs | SQS cost | **Daily total** |
|---|---|---|---|---|
| 1,000 | $0.00015 | $0.003 | $0.0008 | **$0.004** |
| 10,000 | $0.0015 | $0.030 | $0.008 | **$0.040** |
| 50,000 | $0.0075 | $0.148 | $0.040 | **$0.196** |
| 100,000 | $0.015 | $0.297 | $0.080 | **$0.392** |
| 500,000 | $0.075 | $1.480 | $0.400 | **$1.960** |
| 1,000,000 | $0.150 | $2.970 | $0.800 | **$3.920** |

---

## Side-by-Side Comparison (daily)

| Matched invoices/day | Design A | Design B | Design C |
|---|---|---|---|
| 1,000 | $0.001 | $0.003 | $0.004 |
| 10,000 | $0.011 | $0.030 | $0.040 |
| 50,000 | $0.055 | $0.148 | $0.196 |
| 80,000 | $0.089 ⚠️ ceiling | $0.237 | $0.314 |
| 100,000 | **fails** | $0.297 | $0.392 |
| 500,000 | **fails** | $1.480 | $1.960 |
| 1,000,000 | **fails** | $2.970 | $3.920 |

## Monthly cost (Design B vs C, assuming daily run × 30)

| Matched invoices/day | Design B (monthly) | Design C (monthly) |
|---|---|---|
| 1,000 | $0.09 | $0.12 |
| 10,000 | $0.90 | $1.20 |
| 50,000 | $4.44 | $5.88 |
| 100,000 | $8.91 | $11.76 |
| 1,000,000 | $89.10 | $117.60 |

---

## Real Production Data

**Periscope query (2026-06-02):** 12,600 invoices due today across all production accounts.
This is the total pool before APR filtering (opt-in setting, valid email, `paymentRequestSentDate: null`).

At realistic adoption rates:

| Adoption | Matched invoices | Design B daily | Design B monthly |
|---|---|---|---|
| 5% | 630 | $0.002 | $0.06 |
| 10% | 1,260 | $0.004 | $0.12 |
| 25% | 3,150 | $0.009 | $0.27 |
| 50% | 6,300 | $0.019 | $0.57 |
| 100% | 12,600 | $0.037 | $1.11 |

**Bottom line: Even at 100% adoption of today's pool, Design B costs ~$1/month.**

---

## Verdict

| Concern | Design A | Design B | Design C |
|---|---|---|---|
| Cost | Cheapest | Middle | Most expensive |
| Ceiling | 80k (fails beyond) | None | None |
| Complexity | 2 Lambdas | 3 Lambdas | 3 Lambdas + Step Function |
| Lambda duration | Up to 15 min ⚠️ | <1s each ✅ | <1s each ✅ |
| Needed for APR? | Simple but limited | Right fit | Overkill (pagination not needed) |

**Recommendation: Design B.** No ceiling, each Lambda is short-lived (PCore concern addressed), costs are negligible at any realistic scale, and Step Functions adds cost with no benefit since APR's Mongo query doesn't need pagination.

---

## Why Not Temporal?

[Temporal](https://temporal.io/) is a durable workflow orchestration platform. Its [Schedules](https://docs.temporal.io/evaluate/development-production-features/schedules) feature replaces EventBridge cron with built-in pause/resume, overlap policies, backfill, and workflow visibility.

### What Temporal would look like for APR

```
Temporal Schedule (daily 14:00 UTC)
  → Workflow: SweepAndSend
    → Activity: queryMongo (matched invoices)
    → for each invoice (concurrent via task queue workers):
        → Activity: insertMsg (PG)
        → Activity: queueSWUJob (email via is-messages)
        → Activity: updateInvoice (Mongo)
        → Activity: writeParse (Parse Msg)
```

Temporal handles per-activity retries, state persistence, and progress inspection out of the box.

### Why it doesn't fit APR today

| Concern | Temporal | Design B (Lambda + SQS) |
|---|---|---|
| Infrastructure cost | ~$200+/mo (Temporal Cloud) or self-hosted cluster | ~$1/mo at current scale |
| Team knowledge | New SDK, concepts (workflows, activities, task queues, signals) | Existing Lambda+SQS patterns (abandoned-cart, Fundbox) |
| Activity workers | Still need ECS/EC2/Lambda to run activities | Already on Lambda |
| Fan-out concurrency | `maxConcurrentActivityExecutionSize` on worker | `maxConcurrency` on SQS event source |
| Overlap protection | Built-in overlap policies | Not needed — L1 finishes in ~5s |
| Backfill | Built-in schedule backfill | Not needed — re-trigger is idempotent (`paymentRequestSentDate: null` filter) |
| Visibility | Temporal UI shows workflow state | CloudWatch + Datadog |

### When Temporal would make sense

- If Invoice Simple moves **all scheduled jobs** (APR, abandoned-cart, is-messages cron, Fundbox) onto one platform — becomes a strategic investment
- If APR grows into a **multi-step saga** with human-in-the-loop, branching logic, or multi-day retry escalation
- If the team already operated Temporal for other workloads

**Verdict:** Temporal is a strong platform but adopting it for APR alone doesn't justify the cost and ramp-up. Design B solves the same fan-out problem with existing team knowledge at a fraction of the cost.

---

## Notes

- `queueSWUJob()` latency (~100ms) is an estimate for internal HTTP round-trip to is-messages. Needs staging benchmark to confirm.
- `maxConcurrency: 50` on L2/L3 SQS event sources is a safe starting point. Can be increased without code changes if throughput needs to grow.
- Step Functions comparison uses Standard Workflows ($0.025/1K transitions). Express Workflows ($1.00/1M requests + duration) would be slightly cheaper at high volume but adds operational complexity.
- Fundbox uses Step Functions specifically because it paginates DynamoDB scans via `LastEvaluatedKey` cursor across multiple Lambda invocations. APR doesn't need this pattern.
