> **PRD:** [Deposits + Milestones - Stripe Automatic Payments + Automatic Payment Request Notifications](https://everpro-tech.atlassian.net/wiki/x/MYBePg)

---

## Background

Payment Scheduling lets merchants split a single invoice into multiple payment milestones — a deposit upfront, plus upcoming payments with specific due dates and varying amounts. Today, due dates are purely informational: nothing happens automatically when a date arrives. Merchants must manually follow up with clients to collect each payment.

This spike explores adding **automatic payments** to the existing Payment Scheduling feature, allowing clients to vault a payment method at checkout so that future milestones are charged automatically on their scheduled dates.

### Why Now

Adoption is low (3% MAU schedule, 1% MPM complete) with a steep early decline — users discover the manual limitations quickly and abandon. The root causes: no reminders, no auto-charge, and checkout friction on every subsequent payment. Automating payments makes Stripe more attractive for multi-payment scenarios and increases platform stickiness. See PRD for full metrics.

---

## Feature Eras — Where We Are

Payment Scheduling has three major capability milestones. We are designing for Era 3 (payment automation), but Era 1 is not yet complete.

| Era | What | Status |
|-----|------|--------|
| **1. Unified Scheduling System** | Consolidate Payments Schedule + Invoice Terms + Abandoned Cart Automation into one system. Enforce required/future dates, guard rails around payment amounts (> 100% validation, unscheduled balance). Currently these three don't speak to each other and dates can be set in the past. Prerequisite for automation. | **In progress — not done yet** |
| **2. Automatic Payment Request (Email Automation)** | Automated checkout link emails sent on due date (non-vaulted, works for all users including non-Stripe). Requires Era 1 (can't automate emails around dates that aren't enforced). | Not started |
| **3. Payment Automation** | Client vaults, upcoming payments auto-charge on due dates. This is the bulk of this spike. Requires Era 1 + 2. | Spike complete, design in progress |

**Key implication:** Era 1 changes are a prerequisite for safe automation. We can't auto-charge on due dates if due dates aren't validated (no past dates, required fields, no > 100% overcharge). This means Era 1 work — which is a unification of three separate scheduling systems (Payments Schedule, Invoice Terms, Abandoned Cart) — likely needs to happen first or in parallel. *(Note: we initially scoped Era 1 narrowly as "due date enforcement / guard rails" only. The PRD clarifies it's the broader "Unified Scheduling System" — consolidating all three into one coherent system.)*

**Ownership (TBD):** We (Payments Growth) may end up owning Era 1 and Era 2 in addition to Era 3, since they're prerequisites and it's unclear if Payments Core will take them on. This would significantly expand our scope. To be confirmed. See Decision 18 in decisions doc (UI redesign ownership) and Decisions 19/20 (balance validation).

---

## How Payment Scheduling Works Today

A merchant creates a single invoice and adds:

- **Deposit** — a one-time upfront payment (flat or % of total). No due date. Paid at checkout (online) or manually recorded by merchant.
- **Upcoming Payments** (also called "milestones") — one or more future payments, each with a label, amount (flat or %), and a due date.

**Key characteristics:**
- Due dates are informational only — no reminders, no enforcement, no automated charging.
- Milestones are an **open set** — the merchant can add more upcoming payments at any time.
- Amounts can be expressed as a percentage of the invoice total (resolved dynamically) or a fixed amount.
- State is inferred from data shape: a payment with `amount + date + method` is completed; without them, it's upcoming.
- Both mobile and web are supported. Mobile is the primary platform.
- A milestone can be paid in two ways: (1) merchant marks it paid manually in the app, or (2) client pays online via a checkout link (deposit email / "View online payment").
- Each paid deposit or upcoming payment reduces the invoice balance. `invoice.balanceDue` (stored in Parse/MongoDB) reflects the **soonest upcoming payment amount** (deposit if unpaid, otherwise nearest upcoming milestone), capped at the full remaining balance. It is not simply `total − paid` — it is `min(nearest upcoming payment, remaining balance)`, recalculated on every payment event via `getInvoiceBalance()` in `@invoice-simple/calculator`.

### Today (manual)

<img src="https://raw.githubusercontent.com/lenmor-invoicesimple/spike-deposits-milestones/main/images/image-current-payment-scheduling.webp" width="800" alt="Diagram 1: Today (manual)" />

Due dates are informational. No automation, no reminders, no scheduled charges. Merchant follows up manually or sends checkout links.

### With Automatic Payments (Phase 1)

<img src="https://raw.githubusercontent.com/lenmor-invoicesimple/spike-deposits-milestones/main/images/image-proposed-auto-pay.webp" width="800" alt="Diagram 2: With Automatic Payments" />

**Flow (successive scheduling):**
1. Merchant creates invoice with milestones/upcoming payments (stored in Parse)
2. Client opens checkout, sees payment schedule, opts in to auto-pay
3. Client pays first payment — is-stripe vaults the card (`setup_future_usage: 'off_session'`)
4. is-stripe publishes SQS message to is-payments: "vault confirmed, here are the milestones"
5. is-payments creates rows for all milestones (QUEUED) but only schedules the **next** one (ACTIVE)
   - Only one EventBridge schedule pair (charge + Automatic Payment Request (reminder)) exists at any time
   - If that payment fails after max retries, remaining QUEUED payments are set INACTIVE (chain stops)
6. On due date: EventBridge fires → SQS → Execution Lambda → charges vaulted card via is-stripe
7. On success: Lambda creates the schedule for the **next** QUEUED payment (chain advances)
8. Stripe webhook automatically marks the Parse Payment as paid
9. Merchant can skip the next scheduled payment (chain advances) or disable all (full reset)

---

## Compared to Recurring Payments

We already have an automatic payments backend (Recurring Payments) built on is-stripe, is-payments, and EventBridge. However, milestones and recurring payments are fundamentally different:

| Aspect | Recurring Payments | Deposits & Milestones |
|---|---|---|
| Scope | Many invoices, one series | One invoice, many milestones |
| Schedule | Regular interval (weekly, monthly) | Irregular, pre-determined dates |
| Amounts | Same each cycle | Different per milestone |
| End condition | Defined (count or end date) | Open-ended |

This means we can **reference the existing RP patterns** (Stripe vaulting, EventBridge scheduling, retry logic) but the implementation is mostly new — separate tables, separate handlers, separate Lambda. RP serves as an architectural blueprint, not a shared library.

**Vaulting mechanism:** Same as RP — `PaymentIntent` with `setup_future_usage: 'off_session'`. Single call that charges and vaults simultaneously. The resulting `vaultedToken` (Stripe payment method ID) is stored for future off-session charges.

### Recurring Payments (current architecture, for reference)

<img src="https://raw.githubusercontent.com/lenmor-invoicesimple/spike-deposits-milestones/main/images/image-current-recurring-payment.webp" width="800" alt="Diagram 3: Recurring Payments" />

Key difference: RP is driven by `is-recurring-invoices` generating new invoices on a regular interval. Upcoming payments has no invoice generation — it's one invoice with multiple pre-set charge dates.

---

## Key Concerns & Open Questions

### Phasing Question: Can Automatic Payment Request (reminder emails) ship independently?

The PRD distinguishes two milestones: **Automatic Payment Request** (Era 2 — reminder emails only, no vaulting, works for all users including non-Stripe) and **Vaulted Payment Automation** (Era 3 — card vaulting + auto-charge, Stripe only). Our current Phase 1 bundles both together.

Automatic Payment Requests alone (no vaulting, no auto-charge) could move the adoption needle with significantly less infrastructure and risk. The PRD names this a Tier 2 GTM initiative — suggesting it could ship first.

**Open questions:**
1. Can Automatic Payment Request (Era 2, non-Stripe) ship as a standalone milestone before vaulting?
2. Is Automatic Payment Request automation a job for Payments Core or for the Deposits & Milestones team? (It requires a scheduling system for due-date-triggered emails — potentially shared infrastructure.)
3. If Automatic Payment Requests ship first, does it block or unlock anything for the vaulting milestone?

### Critical Semantic Change
`dueDate` today means "I'd like to be paid by this date." With auto-pay, it means **"The client's card will be charged on this date."** This is a significant product change that affects how merchants set and edit due dates, and what clients consent to at checkout.

### Design & UX Questions (for design review)
- How are overdue milestones communicated to the client before they consent at checkout?
- ~~Should we add a time-of-day field to milestone due dates, or default to vault time?~~ **Resolved:** Out of scope for V1. Defaults to vault time (same as RP). Design may revisit later.
- What does the failure state look like to the merchant after max retries?
- **Due date consolidation** — invoices today have three surfaces where dates interact: (1) invoice-level "Due on receipt", (2) per-milestone due dates on upcoming payments, (3) deposit/email sends triggered any time via "Email Invoice" / "Email Deposit." With auto-pay, milestone due dates need new validation rules: must be a future date (not past), and must be required (not optional) when auto-pay is active. Today the field is optional and no past-date check exists. This needs design and mobile form validation work.
- ~~What's the confirmation flow when the merchant edits a milestone or the invoice?~~ **Resolved:** Only next scheduled payment is locked (Decision 14 revised). Future QUEUED payments remain editable. ACTIVE payment edit → "Skip" confirmation → chain advances. Invoice-level edits → cancel all → client re-vaults.
- ~~What does the checkout opt-in look like?~~ **Resolved:** Inline checkbox on the checkout page. Email required before vault. When the client checks the box, the milestone schedule (next payment, date) is revealed inline — consistent with RP checkout behavior. PayPal is selectable but auto-pay checkbox is disabled when PayPal is selected (V1 Stripe only; PayPal vaulting in follow-up). "Pay Later" is not eligible for auto-pay.
- ~~Where does the merchant enable/disable auto-pay per invoice?~~ **Under review (as of 2026-05-08):** Two options being evaluated — see "Open Design Decision" below.

### Open Question: Auto-send toggle vs vaulted Automatic Payment Requests

The PRD has two email behaviors: (1) merchant toggles "auto-send payment request" to send checkout links on due dates (Era 2, non-vaulted), and (2) Automatic Payment Request (reminder) emails fire N days before a charge (Era 3, vaulted).

**Question:** Are these independent? If the merchant has auto-send OFF but the client vaults, does the "we're charging your card" Automatic Payment Request still fire?

**Working assumption (recommended):** Vaulted Automatic Payment Requests are **always on** when a vault is active — they're tied to client consent, not the merchant's auto-send preference. You can't charge someone's card without notice. The merchant's "auto-send" toggle only governs the non-vaulted checkout link emails (Era 2).

### Key Open Question: Overdue Payments at Vault Time (Decision 8)

If a client opens checkout after some payments are past due, how do we handle them?

**Option A — Schedule immediately (current):** Each overdue payment charges separately via the normal Lambda pipeline (~1 min after vault). Clean architecture (one charge = one Parse Payment write-back), but client gets rapid-fire charges and multiple emails.

**Option B — Bundle into checkout charge:** Overdue amounts added to the checkout PaymentIntent. Client pays one larger amount. Cleaner UX, fewer Stripe fees, but requires new Parse write-back logic (one PaymentIntent marking N payments as paid) and all-or-nothing retry.

| | Option A | Option B |
|---|---|---|
| Architecture | Same pipeline for all | New checkout code path |
| Parse write-back | Automatic (1:1) | Manual (1:N — new logic) |
| Client UX | Multiple rapid charges | Single charge |
| Retry | Per-payment | All-or-nothing |
| Stripe fees | N charges | 1 charge |

**To confirm with team (raised 2026-05-11, still open).** See Decision 8 in design decisions doc for full analysis.

### Key Open Question: "Pay Now" in Automatic Payment Request (Decision 16)

The Automatic Payment Request (reminder) email has a "Pay Now" button (new — RP doesn't have this). When client pays one payment early, what happens?

**Option A — Cancel-one (pay early, rest continue):**
- Client pays this one payment. That schedule pair cancelled. Remaining auto-pay unaffected.
- No re-consent needed. Simple UX.
- Requires: new `POST /upcoming-payment/cancel-one` endpoint + simplified checkout (no auto-pay checkbox)

**Option B — Normal checkout (re-vault / manual flow):**
- "Pay Now" opens full checkout with auto-pay checkbox
- Client must re-consent (check box) to keep remaining payments active, or remaining get cancelled
- No new endpoint. But confusing UX — paying early shouldn't force re-authorization.

**Recommendation:** Option A. Paying early is different from recovering from failure. To confirm with team/design.

### Key Decision: Offline Behavior (Decision 17)

Mobile is offline-first. When auto-pay is active, editing/adding/deleting an upcoming payment must trigger `POST /upcoming-payment/disable` to cancel schedules. If the merchant is offline, this call fails → data inconsistency (Parse reflects edit but schedules stay active, client gets charged on old schedule).

**Decision: Block editing when vault active + offline** — same `OfflineSectionCover` pattern used by recurring invoices. UI is frozen (non-interactive) with a banner when `hasActiveVault && !isConnected`. When vault is not active, editing remains unrestricted (same as today).

This matches the existing RP behavior for consistency. Can revisit if user feedback shows offline blocking is a significant pain point (Option B: connectivity check inside confirmation modal only).

### Key Open Question: Unscheduled Balance (Decision 19)

If a merchant creates a deposit (20%) + one upcoming payment (20%), the remaining 60% has no scheduled payment. With auto-pay, only explicitly scheduled upcoming payments get auto-charged — the 60% would never be collected automatically.

**Question:** Should we warn or block merchants when scheduled payments don't cover 100% of the invoice total? Or is this fine as-is (merchant adds more upcoming payments later, or collects manually)?

To confirm with team/design.

### Key Open Question: Exceeding 100% Total (Decision 20)

Currently, Payment Scheduling allows creating payments that exceed the invoice total (e.g., 60% + 60% = 120%). If both are marked paid, balance goes negative. This is an existing Payment Scheduling quirk.

**With auto-pay this becomes auto-overcharging** — the system silently charges the client more than the invoice total with no manual intervention.

**Question:** Should we add validation to prevent > 100%? This is a change to core Payment Scheduling functionality (affects all users, not just auto-pay). May require coordination with Payments Core team (see Decision 18).

To confirm with team/design.

### Lower Priority: Global Settings for Automatic Payment Requests (Era 2)

PRD specifies "store Automatic Payment Request settings at Global Settings level, not at invoice level." This is an Era 2 concern — a merchant-level toggle like "auto-send payment requests for all my invoices by default."

V1 (vaulted auto-charge) doesn't need a global setting because auto-pay is consent-driven per invoice at checkout. But if Era 2 (non-vaulted reminder emails) ships, there needs to be a settings surface. **Question:** Where does this live — existing notification preferences, a new "Automatic Payments" settings page, or per-invoice with a global default?

---

## Open Design Decision: Auto-Pay Eligibility Signal

Two approaches are under evaluation. **Backend is identical either way** — only the merchant UX and checkout gate differ.

**Option B — Explicit per-invoice toggle (current design)**
- Merchant explicitly enables auto-pay per invoice via `invoice.setting.milestoneAutoPayEnabled`
- Checkout reads this flag to decide whether to show the opt-in
- Merchant has full control: can offer auto-pay on some invoices but not others
- Requires a new Parse field and a toggle in mobile merchant UI

**Option E — Implicit (designer's intent)**
- No merchant toggle needed. If an invoice has upcoming milestones + feature flag is on → auto-pay opt-in shown at checkout automatically
- Checkout uses `ExtendedCheckoutData.payments[]` (already loaded) to check for upcoming milestones — no extra API call
- Simpler merchant flow: create milestones → send invoice → done
- Merchant can still cancel auto-pay after the client vaults
- No per-invoice opt-out before vault — all milestone invoices show opt-in once flag is on

**Both options have equal implementation cost at the checkout layer.** The tradeoff is purely product behavior: explicit merchant control vs. implicit always-on for milestone invoices.

Discussed with tech lead (raised 2026-05-08, still open — pending design input).

---

## Proposed Design (High-Level)

### What We're Building

1. **Client opts in at checkout** — pays the first payment (deposit or first upcoming payment); card is vaulted via `setup_future_usage: 'off_session'` on the PaymentIntent. They see the full payment schedule before consenting. Deposit is not required — vaulting happens on whatever the first payment is. (How the opt-in is surfaced depends on the eligibility decision above.)
2. **Upcoming payments auto-charge on their due dates** — each upcoming payment gets an EventBridge schedule pair (charge + Automatic Payment Request (reminder)). Automatic Payment Request fires N days before due date with "Pay Now" CTA. Charge fires on due date. When it fires, is-payments executes the charge via the vaulted Stripe payment method.
3. **Merchant retains per-payment control** — the next scheduled payment (ACTIVE) is locked, but all future payments (QUEUED) remain freely editable. Merchant can "skip" the next payment (chain advances) or "disable all" (full reset). Editing the invoice itself cancels all and requires the client to re-vault.
4. **Retries on failure** — same logic as Recurring Payments: 1/3/6 day retries for cards. On max retries: **chain stops** — remaining QUEUED payments are set INACTIVE (no cleanup needed since they never had schedules). Merchant tile reverts to non-vaulted state. Client must check out again to re-vault. Vault record is **never deleted** — gets replaced on re-vault (same as RP).
5. **No client self-service cancel** — client must contact merchant to cancel. Only merchant (Cancel AP button in mobile) or admin (support tool) can disable. Emails include: "To make changes to or cancel automatic payments, please contact {Merchant Business Name} at {Merchant Business Email}."
6. **Merchant send confirmation when vault is active** — when the merchant sends a subsequent invoice while auto-pay is active, a confirmation modal shows the upcoming charge details (method, amount, fee) and requires "Agree & Continue" before dispatch.

### Architecture Approach

Reuse existing RP infrastructure (EventBridge, SQS queues, Stripe vaulting utilities, retry logic) but with **separate tables and handlers** for upcoming payments to avoid coupling with the RP system.

**New data:**
- `upcoming_payment` in is-stripe — one vault record per invoice (`vaultedToken`, `customerId`, `paymentMethodType`, `consentGrantedAt`)
- `upcoming_payments` in is-payments — one execution state record per upcoming payment (`amountCents` snapshot, `scheduledDate`, `status`, `retryCount`)

**New AWS resources:**
- Upcoming payment execution Lambda (SQS → Stripe charge)
- Upcoming payment execution SQS queue
(No separate scheduler Lambda — EventBridge fires directly to SQS, same as RP)

**Communication pattern:**
- Checkout → is-stripe (vault after PaymentIntent confirmation)
- is-stripe → is-payments (SQS, self-contained message with upcoming payment data)
- Mobile → is-payments (fire-and-forget REST after Parse save, matching existing bookkeeping sync pattern)

### Scheduling Model (Decision 15: successive chaining)

Only **one payment** is actively scheduled at a time (the next one due). When it completes, the system creates the schedule for the one after it, and so on. This gives merchants per-payment flexibility without breaking the auto-pay relationship.

### Modification Behavior (Decision 14 revised: only next payment locked)

| Payment status | Merchant can... |
|---|---|
| **ACTIVE** (next scheduled — has EventBridge schedule) | "Skip" → chain advances to next. Or "Disable All" → full reset. Cannot edit amount/date. |
| **QUEUED** (future — no schedule yet) | Edit amount, date, add new, delete. No confirmation needed. Changes picked up when chain reaches it. |

| Action | Impact |
|---|---|
| Edit/delete a QUEUED (future) payment | Free — no API call, edit directly in Parse |
| Edit/delete the ACTIVE (next) payment | "Skip" confirmation → cancel that schedule → chain advances |
| Add a new upcoming payment | Added as QUEUED — chain reaches it after prior ones complete |
| Edit invoice (total, client) | Confirmation modal → cancel ALL → client must re-vault |
| Client pays via "Pay Now" / checkout link | Cancel that payment → chain advances (auto-pay continues) |
| Merchant/Admin cancels auto-pay | Cancel all (ACTIVE + QUEUED) → full reset |

**Chain-break recovery:** If the "create next schedule" step fails after a successful charge, SQS retries the Lambda automatically. If all retries fail, message goes to DLQ → CloudWatch alarm → team redrives. Same existing pattern as RP.

### Client Checkout: Manual Payment vs Re-vault (same as RP)

When a client opens a checkout link (e.g., after max retries, or merchant sends link):

| | Manual payment (no consent box) | Re-vault (consent box checked) |
|---|---|---|
| This payment | Paid | Paid |
| Retry schedules | Cancelled | Cancelled |
| Remaining payments | **All set INACTIVE** — auto-pay off | **Reset**: fresh QUEUED rows, next one set ACTIVE with new schedule |
| Vault record | Kept (unchanged) | Replaced (new card, new `consentGrantedAt`) |
| Effect | Remaining payments revert to manual | Auto-pay restarted with new card (successive chain begins fresh) |

This matches RP behavior: consent determines whether the vault is refreshed and schedules restarted.

### Email Lifecycle (5 templates)

| Email | Trigger | Content |
|-------|---------|---------|
| Automatic Payments Enabled | Client vaults at checkout | Summary (method, first amount) + full remaining schedule table |
| Upcoming Scheduled Payment | N days before due date — Automatic Payment Request (reminder) | Amount, date, method ending ####, remaining schedule, "Pay Now" CTA |
| Scheduled Payment Successful | After successful charge | Amount charged, date, method, remaining schedule, "View Invoice" CTA |
| Payment Retrying | Charge failure (retryable) | Sent to merchant — retrying on [date] |
| All Payments Cancelled | Max retries reached | Sent to merchant + client — all remaining payments cancelled |

All emails include remaining payment schedule table and "To make changes to or cancel automatic payments, please contact {Merchant Business Name} at {Merchant Business Email}."

### Naming Convention (pending team confirmation)

Technical naming uses `upcoming_payment` / `upcoming-payment` — not "milestone." Tables: `upcoming_payment` (vault), `upcoming_payments` (schedules). API paths: `/backend/upcoming-payment/...`. Schedule names: `upcoming-{invoiceId}-{paymentId}`. See Decision 13 in design decisions doc.

### Diagrams

Full flow diagrams (sequence, state machine, architecture, data model) are available on the [FigJam board](https://www.figma.com/board/tsf59ISwbdBgDs4i9oFoqp/Payment-scheduling-with-automatic-payments).

---

## Rollout Phases

| Phase | Scope | Platform | Processor |
|-------|-------|----------|-----------|
| **Phase 1** | Full feature: vault, scheduling, execution, retry, emails, lifecycle | Mobile (merchant) + Checkout (client) | Stripe only |
| **Phase 2** | Web merchant UI — status indicators, lock modals, cancel button | Web | Stripe only |
| **Phase 3** | PayPal vaulting + execution | Mobile + Web | PayPal only |
| **Phase 4** | Multi-processor support (client can vault with Stripe OR PayPal per invoice) | Mobile + Web | Stripe + PayPal |

**Phase 1 is the current design scope.** All technical design, decisions, and tickets below are Phase 1 only.

---

## What's Not In Scope (Phase 1)

- Web merchant UI (Phase 2)
- PayPal vaulting (Phase 3)
- Multi-processor vault selection (Phase 4)
- Showing card details in merchant UI
- ACH / bank account retries (cards only, same as RP)
- Time-of-day picker on upcoming payment form (defaults to vault time)
- Client self-service cancel (client contacts merchant to cancel)

---

## Implementation Tickets (Draft — Phase 1 only)

12 tickets across 5 services:

| # | Ticket | Service |
|---|--------|---------|
| 1 | is-stripe `upcoming_payment` vault table + endpoints | is-stripe |
| 2a | is-payments `upcoming_payments` table + SQS consumer + schedule management | is-payments |
| 2b | is-payments upcoming payment execution Lambda | is-payments |
| 3 | CDK infrastructure (EventBridge → SQS → Lambda) | packages/aws/cdk |
| 4a | Checkout auto-pay UI components | is-unifiedxp |
| 4b | Checkout vaulting integration + manual/re-vault | is-unifiedxp |
| 5a | Mobile auto-pay status indicator + cancel | is-mobile |
| 5b | Mobile per-payment locking (ACTIVE locked, QUEUED editable) + skip/disable + offline blocking | is-mobile |
| 5c | Mobile payment scheduling form validation | is-mobile |
| 6 | Email notifications (5 templates, all lifecycle events) | is-services |
| 7 | Feature flags, observability, E2E smoke test | cross-cutting |
| 8 | Payment Scheduling UI redesign (placeholder — owner TBD) | is-mobile |

Tickets 1, 2a, and 3 can start in parallel (different services, no cross-dependencies).

---

## Out of Scope (V2 / Future)

- **Cross-invoice vaulting** — buyer uses saved card across multiple invoices. V1 vault is per-invoice.
- **PayPal vaulting** — separate scope per PRD.
- **SMS channel for Automatic Payment Requests** — V1 is email only.
- **Merchant-initiated manual charge on vaulted card** — "charge saved card manually for subsequent payments."
- **Merchant edits payment request before sending** — non-vaulted Era 2 flow only.
- **Global settings for Automatic Payment Requests** — Era 2 scope (merchant-level toggle).

---

## Next Steps

- [x] Design review — checkout opt-in UX resolved (inline checkbox, milestone schedule, email required, PayPal disabled)
- [x] Break into Jira tickets and estimate
- [ ] Resolve eligibility signal decision (Option B vs E) — discussed 2026-05-08, pending design input
- [ ] Design review — remaining UX flows (merchant enable/disable, failure states, confirmation modals)
- [ ] Team sign-off on architecture
- [ ] Implementation (mobile first)
