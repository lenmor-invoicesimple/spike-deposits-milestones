# Deposits & Milestones — Automatic Payments: Design Decisions

> Spike exploration for integrating automatic payments into the existing Payment Scheduling feature (deposits & milestones), reusing the recurring payments backend on is-services.

## Context

Payment Scheduling lets merchants split a single invoice into multiple payment milestones (deposits + upcoming payments) with specific due dates and varying amounts. Today, due dates are informational only — no automated charging. This spike explores adding automatic payments: the client vaults a payment method, and milestones are charged automatically on their scheduled dates.

The recurring payments backend (is-stripe, is-payments, EventBridge) already handles vaulting, scheduled execution, and retries for recurring invoice series. The goal is to reuse this infrastructure for milestone-based payments on single invoices.

### Key Differences from Recurring Payments

| Aspect | Recurring Payments | Deposits & Milestones |
|---|---|---|
| Scope | Recurring invoice series (many invoices) | Single invoice (many milestones) |
| Schedule | Regular interval (monthly, weekly, etc.) | Irregular, pre-determined dates |
| Amounts | Same amount each cycle | Different amounts per milestone |
| End condition | Defined (count or end date) | Open-ended (merchant can always add more) |
| Identifier | `recurringInvoiceSeriesId` | `invoiceId` + individual milestone IDs |
| Trigger | is-recurring-invoices generates invoice → SQS | EventBridge schedule fires per milestone |
| Amount source | From generated invoice | From milestone's paymentValue (resolved at schedule time) |

---

## Decision 1: Vaulting Flow

**Question:** When a client vaults their payment method for an invoice with milestones, how does the flow work?

### Options

**A) Vault during first payment (deposit)** — Client pays the deposit at checkout, vaults their method as part of that transaction. Future milestones auto-charge. Closest to how recurring payments work today.

**B) Vault without immediate charge** — Client receives invoice, vaults via a "Set up automatic payments" flow. All milestones (including the first) charge on their scheduled dates. No money moves at vaulting time.

**C) Hybrid — merchant chooses per invoice** — Merchant toggles whether the first milestone requires manual payment or auto-charges like the rest.

**D) Progressive consent** — Client pays first milestone manually, then gets prompted to set up automatic payments for remaining milestones. Lower friction since trust is established.

**E) Vault via separate link** — Invoice includes a "Set up automatic payments" CTA. Client can pay manually or opt into auto-pay at any point. Decouples vaulting from payment events.

**F) Merchant pre-vaults from prior relationship** — Attach an existing vaulted method from a previous transaction. Requires a customer-level vault registry.

### Decision: A (vault during first payment) with decoupled backend

**Reasoning:** A and D are simplest to build because they use the existing checkout flow where Stripe Elements are already rendered. B and E require a new standalone vaulting UI. F requires a customer-level vault registry (current system is per-series). We design for A as the primary flow but architect the backend so vaulting and scheduling are decoupled (same as recurring payments, where the schedule is created with the series but vaulting happens later). This keeps the door open for B/D/E later without rearchitecting.

---

## Decision 2: Behavior on Modification

**Question:** When an invoice with milestones is modified after vaulting, what should happen to scheduled payments?

### Options (invoice-level)

**A) Cancel and done** — All future schedules cancelled, vault invalidated, client must re-vault. Clean break. How recurring payments works today.

**B) Cancel and re-create** — Future schedules cancelled then re-created with updated amounts/dates. Vault stays valid. Risks: charging amounts client didn't consent to.

**C) Cancel, notify, and require re-consent** — Future schedules cancelled, vault stays on file but paused. Client must re-consent before auto-payments resume.

### Decision: Granular per-milestone + invoice-level reset

**Reasoning:** Milestones are an **open set** — merchants can add new upcoming payments at any time. Cancelling everything when a single milestone changes is too aggressive.

**Per-milestone changes (granular):**
- **Adding a new milestone** → create a new schedule for it (if vault exists). No impact on existing schedules.
- **Editing a milestone (amount/date)** → cancel and re-create just that milestone's schedule. No impact on others. Light confirmation modal shown to merchant. Client gets email notification.
- **Deleting a milestone** → cancel just that milestone's schedule.

**Invoice-level changes (full reset):**
- **Editing the invoice itself (total, client, etc.)** → confirmation modal ("This will cancel automatic payments"), cancel all schedules. Vault record is NOT deleted — gets replaced when client re-vaults (same as RP pattern). Client must check out again to restart auto-pay.

**Vault deletion rule (applies to all disable reasons):**
Vault is **never deleted** — not on max retries, not on merchant cancel, not on admin disable, not on invoice modification. Only action is cancelling EventBridge schedules and setting `milestone_payments` rows to INACTIVE. Vault gets replaced via upsert on re-vault. Matches RP behavior.

### Decision 2a: Amount Resolution Timing

**Question:** For PERCENT-mode milestones, when is the percentage resolved to cents?

**Context:** Today, upcoming payments with `paymentMode: PERCENT` resolve dynamically against the current invoice total (50% at $100 total shows $50; if total changes to $150, it shows $75). The dollar amount is only frozen when marked as paid.

**Options:**
- **A) Resolve at schedule creation time (snapshot)** — Store resolved `amountCents` when schedule is created. If invoice total changes, the full reset (Decision 2) cancels everything anyway.
- **B) Resolve at execution time (live)** — Lambda fetches current total and calculates at charge time. Risk: if reset trigger is missed, client gets charged unexpected amount.

**Decision: A (snapshot at schedule creation time)**

**Reasoning:** Consistent with the reset model. Merchant sees exactly what will be charged. Simplifies execution Lambda (no need to fetch invoice and calculate). The `milestone_payments` table stores `amountCents` directly.

### Decision 2b: Milestone Edit UX

**Question:** When merchant edits a milestone that has an active schedule, what confirmation is needed?

**Options:**
- **A) No modal, silently re-schedule**
- **B) Light confirmation modal** — "This milestone has automatic payment scheduled. Updating will reschedule the charge to [amount] on [date]. Continue?"
- **C) Same as invoice-level, require re-consent**

**Decision: B (light confirmation, no vault reset)**

**Reasoning:** Proportional to the change. Client gets email notification that their upcoming charge changed, but doesn't need to re-consent. Vault stays valid.

---

## Decision 3: Retry Logic for Failed Payments

**Question:** Should retry logic for failed milestone payments work the same as recurring payments, or differently given milestones can be close together?

### Options

**A) Same retry logic, independent per milestone** — Each milestone retries on its own schedule (1/3/6 days for cards, no retries for ACH) regardless of other milestones. If retries overlap with another milestone's date, both execute. Simple, matches existing infra.

**B) Same retry logic, cap retries before next milestone** — Skip retries that would fall on or after the next milestone's due date. Prevents overlapping charges.

**C) Shorter retry intervals** — e.g. 1/2/3 days instead of 1/3/6. Accounts for milestones being closer together.

**D) No retries** — Fail once, notify merchant. Simplest, avoids overlap entirely.

### Decision: A (independent retries)

**Reasoning:** B creates an awkward limbo — skipped retries and exhausted retries both end in the same place (merchant intervention), so the complexity doesn't buy anything. Two charges on the same day for different milestones isn't a problem — they're separate obligations the client agreed to. If a milestone keeps failing, it hits MAX_RETRIES and stops on its own. Same email notification flow as recurring payments.

---

## Decision 4: Vault Lifecycle (Open Set)

**Question:** When all milestones are paid, should the vaulted payment method be automatically cleaned up?

### Options

**A) Vault persists until explicitly disabled** — Merchant or client disables it manually. If merchant adds a new milestone later, auto-pay is already set up.

**B) Auto-cleanup when all milestones are paid** — Disable the vault with reason like `ALL_MILESTONES_COMPLETED`. Clean, no orphaned vaults.

### Decision: A (vault persists)

**Reasoning:** Milestones are an open set — the merchant can add another upcoming payment at any time. There's no moment where the system can confidently say "we're done here." Auto-cleanup would mean the client has to re-vault if the merchant adds a new milestone, which is unnecessary friction. Keeping the vault alive matches the open-ended nature and the client can always disable it themselves.

---

## Decision 5: Payment Provider

**Question:** Should this support both Stripe and PayPal vaulting from the start?

### Options

**A) Stripe-only initially** — Match the path recurring payments took. PayPal added later.

**B) Both from day one** — Roughly doubles the vaulting/execution surface area.

### Decision: A (Stripe-only, provider abstracted)

**Reasoning:** PayPal integration with recurring payments is currently in progress as a separate effort. Starting with Stripe keeps scope manageable. The `provider` field in is-payments already abstracts over payment providers, so PayPal slots in later without rearchitecting. PayPal will be a near-future requirement.

---

## Decision 6: Platform Priority

**Question:** Which platforms need this first — web, mobile, or both?

### Options

**A) Web first** — Payment scheduling UI is well-documented on web.

**B) Mobile first** — Recurring payments launched mobile-first. Payment scheduling exists on both platforms.

**C) Both simultaneously** — Parallel development.

### Decision: B (mobile first, high-level plan for web)

**Reasoning:** Recurring payments launched mobile-first, so this follows the same precedent. The backend work (is-payments, is-stripe, EventBridge) is shared regardless. Mobile merchant UI gets built first, with a high-level plan for web to follow. Client-side checkout (is-unifiedxp) is shared across platforms.

---

## Decision 7: Checkout (is-unifiedxp) Scope

**Question:** Is the checkout/client side in scope for this spike?

### Decision: Yes, in scope

**Reasoning:** Checkout needs changes for milestone-specific UI and copy (showing the milestone schedule and amounts rather than "future invoices will be charged automatically"). The vaulting logic itself (Stripe Elements, SetupIntent confirmation) can be reused from the recurring payments checkout flow. Different UI/copy, same underlying vaulting mechanism.

---

## Decision 8: Overdue Payments at Vault Time

**Question:** How should the system handle upcoming payments whose due date has already passed when the client vaults?

### Example

Merchant creates payments on April 1:
- Payment 1 (deposit): $500, no date
- Payment 2: $1,000, due May 1
- Payment 3: $1,500, due May 15

Client pays deposit + vaults on May 10. Payment 2 is overdue.

### Options

**A) Schedule overdue payments for immediate execution**

Each overdue payment gets its own EventBridge schedule (`at(now + 1min)`) → fires through the normal Lambda pipeline → its own PaymentIntent → its own Stripe webhook → marks one Parse Payment as paid.

*UX:*
- Client pays $500 (deposit/first payment) at checkout
- Payment 2 ($1,000) charges as a separate transaction within minutes
- Client sees multiple charges on their card statement in rapid succession
- Multiple charge notification emails arrive quickly

*Technical:*
- Zero special handling — overdue is just a schedule with an earlier date
- Reuses the exact same Lambda pipeline as future payments (one charge = one webhook = one Parse Payment)
- Each charge is independently retryable (if one fails, retry 1/3/6 days kicks in)
- Clear refund granularity (one PaymentIntent per obligation)

*Cons:*
- Client gets rapid-fire charges + emails (poor UX)
- Extra infrastructure hops for something that could be immediate
- Multiple Stripe processing fees

**B) Bundle overdue payments into the checkout charge**

Checkout calculates: `firstPayment + sum(overdue payments)` → single PaymentIntent. Only FUTURE payments get EventBridge schedules.

*UX:*
- Client pays $1,500 at checkout ($500 deposit + $1,000 overdue Payment 2)
- Single charge on card statement
- Checkout shows: "Amount due today: $1,500 (includes 1 past-due payment)"
- One email confirmation

*Technical:*
- **New code path in checkout:** Must calculate overdue total, show breakdown to client
- **Parse write-back problem:** One PaymentIntent covers multiple Parse Payment objects. Stripe webhook fires once → `addPaymentToDocument()` marks one payment. Need new mechanism to mark N payments from a single charge:
  - Option: Checkout explicitly calls `Parse.Cloud.run('invoiceAddPayment')` for each overdue payment after charge success
  - Or: is-payments marks them via the `createUpcomingPayment` SQS message (include list of already-paid payment IDs)
- Retry is all-or-nothing: bundled charge fails → all overdue payments fail (no partial success)
- Refund granularity: single PaymentIntent covers multiple obligations — partial refund is ambiguous

*Cons:*
- Breaks the clean "one charge = one Parse Payment" invariant
- New checkout logic to identify and bundle overdue amounts
- Ambiguous refund/dispute handling

### Comparison

| Aspect | Option A (schedule immediately) | Option B (bundle into checkout) |
|--------|---|---|
| Code complexity | None — reuses pipeline | New: multi-payment write-back, checkout overdue logic |
| Parse write-back | Automatic (one webhook = one payment) | Manual (iterate and mark N payments) |
| Client experience | Multiple rapid charges + emails | Single charge, cleaner |
| Retry granularity | Per-payment (independent) | All-or-nothing |
| Refund clarity | One PaymentIntent per obligation | Ambiguous partial refunds |
| Stripe fees | N separate charges | 1 charge (cheaper) |
| Pipeline consistency | Same path for all payments | Special path for overdue |

### Decision: ⚠️ REVISITING — to confirm with team (2026-05-11)

Original decision was A (schedule immediately). Re-evaluating because:
- Option B is cleaner UX (single charge at checkout)
- But Option A is architecturally simpler (no special paths, one-to-one charge-to-payment)

**Key question for team:** Is the multi-payment write-back complexity in Option B worth the UX improvement? Or is rapid-fire charging (Option A) acceptable given the client just consented moments ago?

**Note — `dueDate` semantic change:** This decision is tightly connected to the fact that `dueDate` shifts from informational ("I'd like payment by then") to actionable ("the card will be charged on this date"). The checkout UI must show what will happen — including any overdue payments — so the client consents with full visibility.

---

## Decision 9: Auto-Pay Eligibility Signal

**Question:** How does checkout know whether to show the auto-pay opt-in for a milestone invoice? Today, all invoices with milestones have due dates — we can't use "has milestones" alone as the signal.

**Context:** In RP, the signal is `recurringInvoiceSeriesId` existing on the invoice — an inherent property of how the invoice was created. For milestones, the invoice already exists with milestones before auto-pay enters the picture, so we need an explicit signal added after the fact. Feature flags (Flagsmith/Optimizely) will gate rollout regardless, but we still need a per-invoice indicator.

### Options

**A) Feature flag only** — All milestone invoices eligible once flag is on for that account. No per-invoice control.

**B) Field on the Parse Invoice** — e.g. `invoice.setting.milestoneAutoPayEnabled: true`. Merchant explicitly enables per invoice. Checkout reads it from existing invoice data (no extra API call).

**C) Record in is-payments** — Lightweight config table: `{ invoiceId, autoPayEnabled: true }`. Checkout queries is-payments (new endpoint).

**D) Derive from merchant action** — Auto-pay offered only if milestones created via a special "auto-pay enabled" flow. Milestone objects carry a flag.

### Complexity Comparison: B vs C

| Aspect | B (Parse Invoice field) | C (is-payments record) |
|--------|------------------------|------------------------|
| Services touched | 3 | 4 |
| Files modified | ~10-15 | ~18-25 |
| Database changes | None (Parse is schemaless) | New table + Drizzle migration |
| New API endpoints | 0 | 1 |
| Extra API calls at checkout | 0 | 1 (+50-150ms latency) |
| Regression risk | Low-Medium | Medium-High |
| Codebase precedent | `invoice.setting.paymentSuppressed` controls checkout behavior | RP queries is-payments for status |

**Why B is simpler:**
- Parse is schemaless — no migration needed
- Checkout already fetches the full invoice — zero latency impact
- Direct precedent: `invoice.setting.paymentSuppressed` is already a boolean controlling checkout behavior
- Mobile/web already know how to update invoice fields

**Why C might be better long-term:**
- Keeps all payment automation state in is-payments (single source of truth)
- Doesn't pollute Parse Invoice with payment infra concerns
- Scales better if config grows (preferences, rules, etc.)
- Same pattern as RP's checkout → is-payments query

### Decision: B (Parse Invoice field) — with option to migrate to C later

**Reasoning:** `invoice.setting.paymentSuppressed` is direct precedent for a flag on the invoice controlling checkout behavior. For a boolean "is this invoice offering auto-pay," a Parse field is sufficient. No new infrastructure, no extra API call, no latency hit. If we later need richer config, we can migrate to C — but for now B is pragmatic.

**Field:** `invoice.setting.milestoneAutoPayEnabled: boolean`

### Alternative Under Consideration: E (Implicit — no per-invoice toggle)

> **Source:** Designer (Seth) conversation, 2026-05-07. Design intent is that auto-pay checkout adapts automatically based on whether milestones exist — no explicit merchant toggle needed.

**E) Implicit eligibility** — Any invoice with scheduled milestones is eligible for auto-pay at checkout (gated by feature flag for rollout). No per-invoice field. Vault record in is-stripe becomes the source of truth for "auto-pay is active."

**How it would work:**
- Checkout checks: does this invoice have upcoming milestones? + is feature flag on? → show opt-in
- Client vaults → vault record created → auto-pay is now active
- Merchant cancels → vault record deleted → auto-pay is inactive
- Client can re-vault at next checkout (opt-in reappears since milestones still exist)

**How it would work at checkout (confirmed, no hidden cost):**

`ExtendedCheckoutData.payments[]` already contains all Payment objects for the invoice — fetched via `invoiceGetPayments` Parse Cloud function in `getDocumentData()`. Filtering for upcoming milestones requires no new API call:

```ts
// payments[] is already in ExtendedCheckoutData — zero extra fetch
const upcomingMilestones = checkoutData.payments?.filter(
  p => p.paymentType !== PaymentTypes.DEPOSIT && !p.date && p.dueDate
)
const showAutoPayOptIn = upcomingMilestones.length > 0 && featureFlag.isOn('milestone_auto_pay')
```

This is **equally simple to Option B at the checkout layer** — both are a single field/array check on data that's already loaded.

**Comparison: B (explicit toggle) vs E (implicit):**

| Aspect | B (explicit toggle) | E (implicit) |
|--------|--------------------:|-------------:|
| Parse changes | New field | None |
| Checkout gate | Read `milestoneAutoPayEnabled` (already in `ExtendedCheckoutData`) | Filter `payments[]` for upcoming milestones (already in `ExtendedCheckoutData`) |
| Extra API call at checkout | None | None |
| Checkout implementation complexity | Trivial boolean check | Trivial array filter — equally simple |
| Merchant enables | Flips toggle per invoice | Automatic (milestones exist = eligible) |
| Merchant cancels | Flips toggle / deletes vault | Deletes vault only |
| Re-enable after cancel | Flip toggle + client re-vaults | Client just re-vaults at next checkout |
| Per-invoice opt-out (pre-vault) | Merchant doesn't flip toggle | Not possible — all milestone invoices show opt-in |
| Rollout control | Feature flag + per-invoice field | Feature flag only |
| Mobile UI | Toggle + status indicator + cancel | Status indicator + cancel only |

**Pros of E:**
- No new Parse field — no schema concern
- Matches Seth's design intent (auto-pay is "how milestones work," not a separate feature to enable)
- Vault record as source of truth is clean — single place to check "is auto-pay active?"
- Easier rollout: feature flag gates everything, no merchant education needed
- Checkout implementation is equally simple — milestone data already in `ExtendedCheckoutData.payments[]`

**Cons of E:**
- No per-invoice opt-out before client vaults (merchant can't exclude a specific invoice)
- All milestone invoices show auto-pay at checkout once feature flag is on — could surprise some merchants
- Less granular rollout control — can't enable for "some invoices" for the same merchant

**Ticket impact if E is adopted:**
- Ticket 5a shrinks (no toggle, just status + cancel action): 3 SP → 2 SP
- Ticket 4a simplifies slightly (filter payments[] instead of reading a field): no SP change
- Decision 9 replaced entirely
- Total SP drops by ~1

**Status:** Discussing with tech lead 2026-05-08. May adopt E for V1 simplicity with option to add per-invoice control later if merchants request it.

---

## Decision 10: Schedule Execution Timezone

**Question:** What timezone should milestone EventBridge schedules fire in?

**Context:** Milestone due dates are date-only (no time component). EventBridge needs a specific UTC instant to schedule at. This requires both a time-of-day and a timezone. Investigation findings:

- **Timezone is not captured anywhere in the checkout/vault flow today** — not in RP, not in is-unifiedxp, not in is-stripe. RP's schedules default to UTC (`ScheduleExpressionTimezone` falls back to `'UTC'` if not set on the `RecurringInvoiceSeries` Parse object, and the web app doesn't populate it).
- Adding browser timezone capture at checkout (`Intl.DateTimeFormat().resolvedOptions().timeZone`) would be new work.

### Options

**A) UTC (default, same as RP)** — Schedule fires at `dueDate @ time-of-day(consent_granted_at)` in UTC. Simplest. Consistent with RP behavior today. Milestones fire at the UTC equivalent of vault time, which may be off for non-UTC merchants.

**B) Capture browser timezone at checkout** — `Intl.DateTimeFormat().resolvedOptions().timeZone` in is-unifiedxp, passed through to is-stripe → SQS. Fires at the correct local time for the client, but client timezone may not match merchant's intent.

**C) Use merchant account timezone** — Read from Parse account (similar to how `is-recurring-invoices` reads from the series object). Most correct — merchant set the due date so their timezone is most meaningful. Requires identifying where merchant timezone is stored.

### Decision: A (UTC) for V1 — to confirm before ship

**Reasoning:** Matches RP's current behavior. No new checkout or account changes needed. Due dates are typically set well in advance — a few hours of offset is acceptable for V1. Option C is the right long-term answer; revisit before shipping if merchant timezone is easily accessible.

**To confirm:** Before V1 ship, verify whether using merchant account timezone (Option C) is feasible without significant extra work.

---

## Decision 12: Reminder Email Lead Time

**Question:** How many days before the due date should the "Upcoming Scheduled Payment" reminder email fire?

**Context:** Designer confirmed a pre-charge reminder email with "Pay Now" CTA. Implementation uses a per-milestone EventBridge reminder schedule (`milestone-reminder-{invoiceId}-{milestoneId}`) that fires N days before the charge schedule. Same pattern as the charge schedule — created/cancelled as a pair.

### Options

**A) 3 days** — Short notice. Minimizes the window where client might pre-pay and we need to cancel the charge schedule. Common for bill reminders.

**B) 7 days** — Full week notice. Gives client time to ensure funds are available or pre-pay if they prefer. Matches credit card statement cycle patterns.

**C) Configurable per account** — Merchant sets reminder lead time. Most flexible but adds complexity (stored where? default?).

### Decision: TBD — confirm with design

**Edge case:** If a milestone due date is < N days from vault time, the reminder would need to fire in the past. Options: skip the reminder entirely, or send it immediately at vault time (alongside the "Automatic Payments Enabled" email — potentially confusing).

---

## Decision 13: Naming — "upcoming_payment" not "milestone"

**Question:** What should tables, schedule names, topics, and API paths use as their noun?

**Context:** The product term is "upcoming payments" — not "milestones." The word "milestone" appears nowhere in user-facing copy or designer screens. Parse calls them "Payment" objects with `paymentType: NONE` (upcoming) vs `paymentType: DEPOSIT`. The feature is "Deposits & Upcoming Payments" with auto-pay.

### Decision: Use `upcoming_payment` / `upcoming-payment` everywhere (pending team confirmation)

**Naming convention:**
- Tables: `upcoming_payment` (vault, is-stripe), `upcoming_payments` (schedules, is-payments)
- EventBridge schedules: `upcoming-{invoiceId}-{paymentId}`, `upcoming-reminder-{invoiceId}-{paymentId}`
- SQS topics: `createUpcomingPayment`, `upcomingPaymentSuccess`, `upcomingPaymentFailure`, `upcomingPaymentReminder`, `cancelUpcomingPaymentSchedules`
- API paths: `/backend/upcoming-payment/...`
- Lambda names: `upcoming-payment-execution-lambda`

**⚠️ To confirm with team** before implementation. "Milestone" may have internal meaning that justifies keeping it in code even if not user-facing. But current assumption: rename everything to `upcoming_payment`.

---

## Decision 14: Only Next Scheduled Payment Locked (revised — supersedes Decisions 2, 2a, 2b)

**Question:** What can the merchant do to upcoming payments after the client vaults?

**Context (tech lead sync 2026-05-11):** The client consented at checkout to be charged specific amounts on specific dates. Any modification (edit amount, edit date, add payment, delete payment) changes what the client agreed to — this is a consent problem, not just a UX problem.

**Revised context (PM/design sync 2026-05-12):** With successive scheduling (Decision 15 revised), only one payment has an active EventBridge schedule at any time. Future payments haven't been scheduled yet. This naturally allows editing future payments without breaking the auto-pay chain.

### Options

**A) Per-payment granular edits (original design — Decisions 2, 2a, 2b)**
- Edit amount/date → cancel + reschedule that payment
- Requires "rescheduled" email to client, trust that merchant isn't abusing
- Legally questionable: charging different amount than client consented to

**B) Lock all, cancel-all on any modification** *(strong alternative — simplest, safest)*
- After vault: upcoming payments are frozen. Merchant cannot edit amount, date, add, or delete individual payments.
- If merchant needs to change anything → confirmation modal: "This will cancel all automatic payments. Client will need to authorize again at checkout."
- `POST /upcoming-payment/disable` → cancel all schedules, invalidate vault
- Merchant can then edit freely (payments revert to manual), re-send invoice, client re-vaults with new schedule

**C) Allow deletes only (forgive), lock edits/adds**
- Merchant can cancel individual payments (forgive obligation) without full reset
- Adds/edits still require cancel-all

**D) Only next payment locked, future payments editable (chosen)**
- The payment with an active EventBridge schedule (the next one to fire) is immutable
- Future payments (status: `QUEUED`, no schedule yet) can be edited/added/deleted freely in Parse
- When the chain reaches a future payment, it reads the current values from Parse at that time
- Merchant can cancel the next scheduled payment without killing the entire auto-pay relationship — chain advances to the one after

### Decision: D (only next payment locked, future editable)

**Reasoning:**
1. **Real-world flexibility** — projects change. Merchant can adjust future payment amounts/dates without disrupting the client's auto-pay setup.
2. **Natural fit with succession (Decision 15)** — only one schedule exists at a time, so only one payment needs to be immutable.
3. **Lightweight cancel** — "skip this payment" is a valid merchant action that doesn't require a full reset.
4. **Consent is still respected for the next charge** — the imminent charge (with an active schedule) matches what the client agreed to. Future changes are reflected next time the client gets a reminder.

**Why "lock all" remains a strong alternative:**
- **Consent integrity** — client agreed to the full schedule at checkout. Editing future payments means the charges may not match what was shown during consent.
- **Simplicity** — single code path, no partial-lock state machine.
- **Aligns with tech lead's concern** — avoids complications of partial edits.
- **If succession is reverted to all-at-once:** lock-all becomes the natural and simpler choice.

**Supersedes:** Decisions 2, 2a, 2b (which assumed granular per-payment edits were allowed). Those decisions are now void.

**Impact on API surface:**
- `POST /backend/upcoming-payment/disable` — cancel all (full reset, invalidate vault)
- `POST /backend/upcoming-payment/cancel-one` — cancel/skip the next scheduled payment; chain advances to following payment
- Future payments edited directly in Parse (no is-payments API needed — system reads from Parse when creating the next schedule)

**Mobile UX:**
- Next scheduled payment row: locked (no edit/delete). Shows "Scheduled for [date]" badge. Action available: "Skip this payment" → cancels that schedule, chain advances.
- Future payment rows (QUEUED): fully editable as normal (edit amount, date, add new, delete)
- "Disable All Automatic Payments" still available as nuclear option → confirmation modal → full reset
- If merchant edits a future payment: no confirmation needed (no schedule exists for it yet)

---

## Decision 15: Schedule in Succession (revised from "all at once")

**Question:** Should all upcoming payments be scheduled at vault time, or one-at-a-time in succession?

**Context (tech lead sync 2026-05-11):** Tech lead suggested scheduling only the next payment, then chaining — on success, schedule the one after. Rationale: if payment 3 fails, payments 4 and 5 "will fail for sure."

**Revised context (PM/design sync 2026-05-12):** PM raised concern about merchants losing control over individual payments. With all-at-once + locked-after-vault, any change requires a full cancel + re-vault cycle. Succession gives per-payment flexibility: only the next scheduled payment is immutable, future ones remain editable.

### Options

**A) All at once** *(strong alternative — simpler infrastructure)*
- At vault time: create all `upcoming_payments` rows + all EventBridge schedule pairs
- All payments are independently scheduled and independently fire
- Cancel is heavier (delete N schedules) but creation is simpler (one batch)
- Reminder emails for all payments exist immediately
- No chain-break risk — schedules exist or they don't

**B) Successive chaining** *(chosen)*
- At vault time: create all `upcoming_payments` rows (status: `QUEUED`), but only create one EventBridge schedule pair (next unpaid payment, status: `ACTIVE`)
- On success: handle-success creates the next schedule pair before marking current as COMPLETED
- On max retries: chain stops naturally — no future schedule gets created
- Merchant can freely edit future payments (only the next one is locked)

### Decision: B (successive chaining)

**Reasoning:**
1. **Per-payment merchant control.** Only the next scheduled payment is locked. Future payments remain editable (amount, date, add, delete) without cancelling auto-pay entirely.
2. **Lightweight cancel.** Merchant can skip/cancel the next payment without nuking the entire auto-pay relationship. No re-vault needed to continue with subsequent payments.
3. **Natural failure handling.** If max retries hit, the chain simply stops. No cleanup of 4 other EventBridge schedules. Clean end state.
4. **Merchant flexibility in real-world scenarios.** Project gets delayed? Just edit the future payment's date in Parse — the schedule for it hasn't been created yet, so the new date is used when the chain reaches it.
5. **Aligned with PM/design vision.** Liz's original framing implied per-payment merchant awareness, not batch-and-forget.

**Chain-break recovery (SQS handles this):**
- handle-success ordering: (1) create next schedule, (2) mark current as COMPLETED, (3) return success to SQS
- If step 1 fails → Lambda throws → SQS retries (charge already succeeded, Stripe idempotency protects)
- If all retries fail → DLQ → CloudWatch alarm → team redrives message from DLQ back to SQS
- Same pattern already used in RP for retry schedule creation in handle-failure

**Why "all at once" remains a strong alternative:**
- Simpler infra: no chaining logic in handle-success
- No chain-break risk (all schedules pre-exist)
- Reminder emails for all payments exist from day one
- Better observability (full schedule visible in EventBridge, not just DB)
- If Decision 14 stays as "lock all," all-at-once is the natural fit

**Decision depends on:** If Decision 14 is "only next locked" → succession makes sense. If Decision 14 stays as "lock all" → all-at-once is simpler and equivalent.

---

## Decision 16: "Pay Now" in Reminder Email — Cancel-One vs Checkout Flow

**Question:** When the client clicks "Pay Now" in the reminder email and pays early, does the system need a dedicated `cancel-one` endpoint, or does it just open the normal checkout (triggering the re-vault / manual payment flow)?

**Context:** The "Upcoming Scheduled Payment" reminder email (new — RP doesn't have this) includes a "Pay Now" CTA. If the client pays before the scheduled charge date, we need to avoid double-charging. RP doesn't have this scenario because it has no pre-charge reminder emails.

### Options

**A) Dedicated `cancel-one` endpoint — pay early without touching other schedules**
- "Pay Now" links to a checkout that pays this one payment only
- Backend: `POST /upcoming-payment/cancel-one { invoiceId, milestoneId }` → cancels that one schedule pair, remaining stay active
- Client doesn't see auto-pay checkbox (not a full checkout — just paying this one)
- Simplest client UX: click, pay, done. Other payments continue automatically.

*Pros:*
- Client pays early without disruption to remaining auto-pay
- No re-consent needed (they already consented to the full schedule)
- Simple UX: "Pay Now" feels like a one-click action

*Cons:*
- New endpoint (but very simple — just cancel one row + one schedule pair)
- New checkout variant: "pay single payment without showing auto-pay checkbox"
- What if client's card changed? Old vault still used for remaining — no chance to update

**B) "Pay Now" opens normal checkout — triggers re-vault / manual flow**
- "Pay Now" links to the same checkout as any other payment link
- Client sees auto-pay checkbox again
- If they check it: re-vault (cancel all, fresh schedules for remaining minus this one)
- If they don't: cancel all (remaining revert to manual)

*Pros:*
- No new endpoint — reuses the re-vault / manual flow we already designed
- Gives client the chance to update their card
- Single consistent checkout experience

*Cons:*
- Disruptive: client just wants to pay one early, but gets forced into a re-consent decision
- If they don't check the box (maybe they don't notice), all auto-pay is cancelled — bad surprise
- Feels heavy for what should be a simple "pay now" action
- Odd UX: "I'm auto-paying everything... I just want to pay this one early... why is it asking me to re-authorize?"

### Decision: ⚠️ OPEN — to confirm with team

**Recommendation:** Option A (cancel-one). The "pay early" use case is fundamentally different from "recover from failure." Paying early shouldn't disrupt the rest of the schedule. The re-vault flow is for recovery (card changed, max retries). Forcing re-consent for an early payment is confusing.

If Option A: need to confirm with design whether "Pay Now" opens a simplified checkout (no auto-pay checkbox) or something else entirely.

---

## Decision 17: Offline Behavior — Block Editing When Vault Active

**Question:** How should the mobile app handle offline scenarios when auto-pay is active and the merchant tries to edit/add/delete upcoming payments?

**Context:** Mobile is offline-first. Payment scheduling edits use `Parse.Cloud.run()` (Cloud Functions cannot be queued offline). If vault is active, any edit triggers `POST /upcoming-payment/disable` (fire-and-forget to is-payments). If that call fails due to no connectivity, Parse may reflect the edit but schedules remain active → data inconsistency (client charged on old schedule).

**Current behavior:**
- **Recurring Payments:** Uses `OfflineSectionCover` + `RecurringInvoiceInternetConnectionBanner` on the recurring invoice edit screen. UI is frozen (non-interactive) when offline. Proven pattern.
- **Payment Scheduling (today):** No offline protection. Edits call `Parse.Cloud.run()` directly with no connectivity check. Fails silently or errors if network unavailable.

### Options

**A) Block editing when vault active + offline (RP pattern)** — Same as recurring invoices: `OfflineSectionCover` on the payment scheduling section when `hasActiveVault && !isConnected`. Blocks all touch interaction, shows banner.

*Pros:*
- Consistent with existing RP behavior
- Simple, proven pattern already in codebase
- Zero risk of data inconsistency

*Cons:*
- Blocks entire payment scheduling UI (even "safe" actions like viewing)
- Payment scheduling is a primary feature — blocking it offline may frustrate merchants
- More restrictive than strictly necessary (only need to block modifications, not reads)

**B) Block only inside confirmation modal** — Let merchant browse/view freely. When they tap edit/add/delete and the confirmation modal appears, check connectivity before the "Confirm" action. If offline → inline error: "Internet connection required to cancel automatic payments."

*Pros:*
- Less restrictive — merchant can still view schedule offline
- Targeted: only blocks the dangerous action (disable call)
- Better UX for offline-first value prop

*Cons:*
- Slightly more implementation effort (connectivity check per action vs one cover)
- Inconsistent with RP pattern (different offline handling for similar features)

**C) Queue the disable call** — Store pending disable locally, replay when online.

*Pros:*
- Best offline UX — merchant edits freely, system syncs later

*Cons:*
- Risk: merchant thinks it's cancelled but client still gets charged before sync
- Significant complexity (local queue store, conflict resolution, retry-on-reconnect)
- Dangerous if sync takes minutes/hours — client could be charged in the gap

### Decision: A (block editing when vault active + offline)

**Reasoning:** Consistency with the existing RP pattern. The risk of data inconsistency (schedules staying active while Parse reflects edits) is too high for Options B/C. Payment scheduling view is still accessible — only interactive edits are blocked by the `OfflineSectionCover`. This is the simplest path for Phase 1. Can revisit with Option B if user feedback shows offline blocking is a significant pain point.

**Note:** When vault is NOT active (no auto-pay), editing remains unrestricted (same as today's behavior — just Parse writes with no backend side effects).

---

## Decision 18: Payment Scheduling UI Redesign — Ownership

**Question:** The designer has a broader Payment Scheduling UI redesign (beyond auto-pay-specific changes). Who owns this work — Payments Growth (auto-pay) or Payments Core (Payment Scheduling feature owner)?

**Figma:** https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5094-3466

**Context:** The redesign covers updates to the deposit/upcoming payment management screens on mobile that go beyond what's strictly needed for auto-pay (Tickets 5a/5b/5c). This is a broader UX refresh of the Payment Scheduling feature itself.

**Sub-questions:**
- Does this need to ship with Phase 1 auto-pay, or can it follow?
- What's the scope boundary — which changes are "auto-pay prerequisite" vs "nice-to-have redesign"?
- If Payments Core owns it, how do we coordinate timing with Phase 1?

### Decision: ⚠️ OPEN — to confirm with team

Placeholder ticket (Ticket 8, 8 SP) created. Blocking questions: team ownership, scope boundary, timing relative to Phase 1.

---

## Decision 19: Unscheduled Balance — What Happens to Remaining Amount?

**Question:** If a merchant creates a deposit (20%) + one upcoming payment (20%), the remaining 60% has no scheduled payment. With auto-pay active, what happens to that unscheduled balance?

**Current behavior:** Milestones are an open set. Merchant can add more at any time. If they don't, the remaining balance sits there indefinitely — merchant collects manually (send link, mark paid). No enforcement.

**With auto-pay:** Only explicitly scheduled upcoming payments get auto-charged. The 60% would NOT be charged automatically. But now the client has vaulted — is there an expectation that the full invoice will be handled? Or is the merchant expected to add more upcoming payments to cover the rest?

**Sub-questions:**
- Should we warn/block the merchant if scheduled payments don't cover 100% of the invoice total?
- Should we prompt the merchant to schedule the remaining balance?
- What if the merchant never adds more upcoming payments — does the vault eventually expire or just sit indefinitely?
- Is this a Phase 1 concern, or fine to leave as-is (same as today's manual behavior) for now?

### Decision: Automate only explicitly scheduled milestones; remaining balance reverts to manual payment ✅ RESOLVED (2026-05-29)

> Slack thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779994686.931629
> Seth: "I'm gonna lean towards automating only milestones that have been explicitly scheduled and revert back to manual payments if there is a remaining invoice balance."

**Reasoning:** Merchants may intentionally schedule only milestones with hard deadlines, without the goal of automating the full invoice balance. The remaining balance is the merchant's discretion — they can send a checkout link or mark as paid via other methods.

**Behavior:**
1. Auto-pay fires for all scheduled milestones on their dates
2. After the last scheduled milestone succeeds, the schedule completes (no more ACTIVE rows or EventBridge schedules)
3. `invoice.balanceDue` already reflects only the unscheduled remainder (updated naturally by each successful payment via Stripe webhook → Parse `invoiceAddPayment`)
4. Client pays remaining balance manually via checkout (sees current `balanceDue`, pays normally)
5. No re-vault needed for manual payment — client just goes to checkout

**No special logic needed:**
- Schedule system is purely milestone-driven — doesn't track or reason about `invoice.total`
- No "auto-generate remaining balance milestone" logic
- No vault-aware balance tracking
- Existing checkout already handles partial payment (shows current `balanceDue`)

**If merchant wants to auto-pay new milestones added after schedule completes:**
- Client would need to re-vault (new consent for new amounts/dates)
- This is correct from a consent perspective — buyer consented to specific payments, not open-ended future ones

---

## Decision 20: Exceeding 100% Total — Payment Scheduling Validation (Overpayment Gate)

**Question:** Currently, Payment Scheduling allows creating payments that exceed the invoice total (e.g., 60% + 60% = 120%). If both are marked paid, the balance goes negative. Should we add validation to prevent this?

**Current behavior:** No validation. Merchant can add multiple payments that sum to > 100%. Marking them all paid results in a negative `balanceDue`. This is an existing Payment Scheduling quirk.

**With auto-pay:** This becomes more dangerous. If both 60% upcoming payments are scheduled and auto-charged, the client gets overcharged automatically with no intervention. Today at least someone manually marks paid — with auto-pay, the system does it silently.

**Options:**
- **A) Add validation (block > 100%)** — Payment scheduling form prevents adding payments that would exceed total. This is a change to core Payment Scheduling functionality (affects all users, not just auto-pay).
- **B) Add validation only when auto-pay is active** — Allow exceeding 100% for manual payments (backwards-compatible), but block it when auto-pay will be triggered.
- **C) Warn but don't block** — Show warning when > 100%, let merchant proceed if they want.
- **D) Leave as-is for Phase 1** — Document the risk, fix in a later phase. Same as today.

### Decision: B — Block vaulting if milestones sum > invoice total ✅ RESOLVED (2026-05-29)

> Slack thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779994686.931629
> Seth: "My initial take is that we should block vaulting if the sum of the scheduled milestones exceed the invoice total."

**Reasoning:**
- Merchants wouldn't intentionally schedule more than the total (typo/error prevention)
- Mitigates chargebacks — buyer never auto-charged beyond invoice balance
- Surfaces an actionable warning to the merchant rather than silently allowing overcharge

**Behavior:**
- At vault time (checkout), validate: `sum(unpaid milestone amounts) <= invoice.total`
- If sum exceeds total: auto-pay opt-in is hidden/disabled at checkout
- Merchant sees inline warning on Payment Scheduling screen: "Automatic payments unavailable — scheduled payments exceed the invoice total. Adjust payment amounts to enable."
- Manual payments still allowed (same as today) — only auto-pay is blocked

**Implementation:**
- Validation lives in checkout (is-unifiedxp) where auto-pay opt-in is rendered
- Simple comparison: `sumMilestoneAmounts > invoiceTotal` → hide/disable opt-in
- No backend (is-payments, is-stripe) involvement — pure UI/checkout gate
- Does NOT change core Payment Scheduling forms (milestones can still exceed total for manual use)
- This is Option B: existing behavior preserved for manual payments, only auto-pay gated

**Note:** This does NOT fix the underlying Payment Scheduling quirk (no max validation on forms). That remains a separate concern for Payments Core (Decision 18). This decision only prevents the auto-pay system from auto-overcharging.

---

## Decision 21: Single Row Creation (no upfront batch)

**Question:** Should is-payments create `upcoming_payments` rows for ALL milestones at vault time, or only create one row at a time (the next payment)?

**Context:** Original Decision 15 (successive scheduling) already limited EventBridge schedules to one-at-a-time, but still created all DB rows upfront as QUEUED. This decision goes further: don't create the rows at all until the chain reaches them.

### Options

**A) Create all rows upfront (QUEUED) but only schedule one** *(original design)*
- At vault: insert N rows (1 ACTIVE, rest QUEUED)
- On success: find next QUEUED row, create its schedule, set it ACTIVE
- Merchant edits in Parse; system reads row to match at chain time

*Pros:*
- Full schedule visible in is-payments DB from day one (queryable)
- Skip/cancel targets a pre-existing row

*Cons:*
- Duplicates Parse milestone data (amounts, dates) into is-payments at vault time
- Drift risk if merchant edits milestone in Parse before its turn — row has stale snapshot
- Write amplification: N rows created on vault even though only 1 is needed now
- Disable/cancel cleanup: must set N QUEUED rows to INACTIVE (minor, but unnecessary work)

**B) Create one row at a time — only when the chain reaches it** *(chosen)*
- At vault: insert 1 row (ACTIVE) for the next unpaid milestone + create its schedule pair
- On success: read next milestone from Parse, create 1 new row (ACTIVE) + schedule pair
- No QUEUED rows exist — future milestones live only in Parse until their turn

*Pros:*
- No data duplication: Parse is the source of truth for future milestones (amounts, dates)
- No drift risk: row is created with current Parse values when the chain reaches it
- Simpler vault handler (create 1 row, not N)
- Matches RP pattern more closely (each execution schedules the next)
- Disable is trivial: at most 1 ACTIVE row to cancel (no QUEUED rows to clean up)
- "Add/edit/delete future payment" is a pure Parse operation — no is-payments awareness needed

*Cons:*
- No is-payments DB visibility into future schedule (only next payment visible)
- Status endpoint must read from Parse for future milestones (or mobile/web just shows Parse data for future rows)
- If Parse is down when chain advances, next schedule isn't created (SQS retries mitigate)

### Decision: B (single row creation)

**Reasoning:**
1. **Parse already owns the schedule.** Milestones (amounts, dates, order) live in Parse. Creating QUEUED rows in is-payments duplicates this data with no clear benefit.
2. **No skip/cancel needed for QUEUED rows.** Merchant skip/cancel only applies to the NEXT payment (which has a row and a schedule). Future milestones are edited directly in Parse — no is-payments interaction needed.
3. **No drift.** Row is created with fresh Parse values when the chain reaches it — merchant edits are always picked up.
4. **Simpler disable.** Only 1 row to set INACTIVE + 1 schedule pair to delete. No batch cleanup.
5. **Status endpoint is still simple.** Returns the single ACTIVE row (next charge). Future schedule comes from Parse (already loaded by mobile/web).

**Impact on design doc:**
- `createUpcomingPayment` SQS handler: creates 1 row (ACTIVE) + 1 schedule pair (not N rows)
- handle-success: reads next milestone from Parse (via SQS payload or direct fetch), creates 1 new row + schedule
- Status endpoint: returns current ACTIVE row only; client reads Parse for future milestones
- QUEUED status is removed from the system (rows are either ACTIVE or INACTIVE)
- Disable: cancel 1 schedule pair + set 1 row INACTIVE (no QUEUED cleanup)

**Supersedes:** The "create all rows upfront" part of Decision 15. Successive chaining remains (one schedule at a time), but now it's also one row at a time.

---

## Summary of All Decisions

| # | Topic | Decision |
|---|---|---|
| 1 | Vaulting flow | Vault during first payment, backend decoupled |
| ~~2~~ | ~~Modification behavior~~ | ~~Superseded by Decision 14~~ |
| ~~2a~~ | ~~Amount resolution~~ | ~~Superseded by Decision 14~~ |
| ~~2b~~ | ~~Milestone edit UX~~ | ~~Superseded by Decision 14~~ |
| 3 | Retry logic | Independent per milestone, same as RP (1/3/6 days, cards only) |
| 4 | Vault lifecycle | Persists until explicitly disabled (open set) |
| 5 | Payment provider | Stripe-only first, provider abstracted for PayPal |
| 6 | Platform priority | Mobile first, high-level plan for web |
| 7 | Checkout scope | In scope — different UI/copy, reuse vaulting logic |
| 8 | Overdue payments at vault | ⚠️ REVISITING: (A) schedule immediately via pipeline vs (B) bundle into checkout charge. Tradeoff: pipeline consistency vs cleaner UX. To confirm with team. |
| 9 | Auto-pay eligibility signal | Parse Invoice field (`setting.milestoneAutoPayEnabled`), may migrate to is-payments record later |
| Q1 | Merchant→is-payments communication | Mobile calls is-payments directly (fire-and-forget REST), matching bookkeeping sync pattern. No Parse changes. |
| Q2 | Milestone data source | Client sends all data in request payload (self-contained). No is-payments→Parse fetch. |
| 10 | Schedule execution timezone | UTC for V1 (same as RP). Timezone not captured anywhere in checkout/vault flow today. Revisit Option C (merchant account timezone) before ship. |
| 11 | Milestone scheduler Lambda | Removed — EventBridge fires directly to SQS execution queue (matches RP). Execution Lambda validates `status = ACTIVE`. Extra Lambda hop adds overhead with no meaningful benefit. |
| Q3 | Vault info display | Denormalize last4/type onto is-payments (stored, not exposed in V1). Show "Auto-pay enabled" only. |
| Q4 | Schedule timezone/time | Use vault time as execution time, in client's timezone. Same infra as RP. Design may add time picker later. |
| 12 | Reminder email lead time | TBD — how many days before due date to send "Upcoming Scheduled Payment" email (3? 7?). Per-payment EventBridge reminder schedule, created/cancelled as a pair with charge schedule. |
| 13 | Naming convention | Use `upcoming_payment` not `milestone` in tables, schedules, topics, API paths. "Milestone" not in product copy. Pending team confirmation. |
| 14 | Only next payment locked (revised) | Only ACTIVE payment (has is-payments row + schedule) is immutable. Future payments remain in Parse only (editable). Lock-all is strong alternative. Supersedes Decisions 2, 2a, 2b. |
| 15 | Schedule in succession (revised) | Only next payment scheduled; on success, handler creates next row + schedule. All-at-once is strong alternative. Chain-break handled by SQS retries + DLQ. |
| 21 | Single row creation (no batch) | Create one `upcoming_payments` row at a time (ACTIVE only). No upfront QUEUED rows. Parse is source of truth for future milestones. Supersedes batch-creation part of Decision 15. |
| 16 | "Pay Now" in reminder email | ⚠️ OPEN: Does "Pay Now" need a `cancel-one` endpoint, or does it just open normal checkout (re-vault/manual flow)? See Decision 16. |
| 17 | Offline behavior | Block editing when vault active + offline (same `OfflineSectionCover` pattern as RP). No edits queued offline. |
| 18 | Payment Scheduling UI redesign ownership | ⚠️ OPEN: Broader redesign beyond auto-pay — Payments Growth or Payments Core? Scope boundary and timing TBD. |
| 19 | Unscheduled balance | Automate only scheduled milestones; remaining balance reverts to manual. No special logic — `balanceDue` updated naturally, client pays remainder via checkout. |
| 20 | Exceeding 100% total (overpayment gate) | Block vaulting if `sum(milestones) > invoice.total`. Checkout hides/disables auto-pay opt-in. Merchant sees warning. Manual payments unaffected. |

