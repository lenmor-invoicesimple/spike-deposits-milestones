# Milestone Date Validation — Exploration

> Purpose: Determine which milestone date constraints to enforce, when, and where — split across pre-vault (Phase 1/2) and vault-time (Phase 3).

---

## Current State (No Enforcement Today)

**None of the rules below are currently enforced anywhere** — the system accepts any `dueDate` on any Payment with zero validation:
- Backend (is-parse-server): `dueDate` is stored as-is, no checks
- Mobile: only validates `dueDate` is present for non-deposits; no cross-field date comparisons
- Web: no client-side ordering validation

Milestones can currently be in any order relative to each other and relative to the deposit.

---

## Validation Rules

| Rule | Enforce pre-vault? | Enforce at vault-time? | Reason |
|------|-------------------|----------------------|--------|
| Milestone `dueDate` cannot be in the past | ✅ Yes — UI guard | ✅ Yes — backstop | Merchants have Historical Payments for logging past offline payments; backdating a milestone is not a legitimate workflow |
| Milestone `dueDate` cannot be today | ✅ Yes — UI guard | ✅ Yes — backstop | Same-day scheduling unreliable for auto-pay; no legitimate pre-vault use case either |
| Milestone `dueDate` cannot be earlier than deposit's due date | ✅ Yes — UI guard | ✅ Yes — backstop | Logical constraint regardless of auto-pay; a milestone before the deposit breaks the payment order |

---

## Pre-Vault (Phase 1/2): UI Guards

All three rules enforced as UI guards on both mobile (`manage-payment.screen.tsx`) and web (upcoming-payment-form-modal).

**Why past/today dates have no legitimate use case for milestones:**
- **Already paid** → Merchant uses Historical Payments (Record Payment) to log it
- **Due now (today)** → That's the deposit, not a milestone — deposits are the immediate/first payment
- **Due in the future** → That's a milestone (the only valid case)

A milestone is by definition a future scheduled payment after the deposit. There is no scenario where a merchant needs to create an unpaid milestone with a past or today date.

**Where to enforce:**
- UI date picker on both platforms — disable or block invalid dates via `minDate` prop
- No server-side enforcement needed pre-vault (UI guard is sufficient)

**Feature Flag:**
- Gated behind a client-side Optimizely FF (per Liz: all Phase 1/2 features must be gated)
- When FF is off: current behavior (no date restriction)
- When FF is on: past/today dates blocked
- **Do NOT add any FF on is-parse-server**

### Implementation Complexity: Low

Both platforms already have the date-range plumbing — it's just not wired up for milestones.

**Web** (`upcoming-payment-form-modal.tsx`):
- `DatePicker` component (wraps `react-datepicker`) already supports `minDate` / `maxDate` props
- Recurring Invoices already uses this pattern: `<DatePicker minDate={new Date()} />` in `schedule-form.tsx`
- Change: add `minDate={tomorrow}` (or deposit's `dueDate` + 1 day for rule 3) to the existing `<DatePicker>` render

**Mobile** (`manage-payment.screen.tsx`):
- `react-native-modal-datetime-picker` natively supports `minimumDate` / `maximumDate`
- Generic `DatePicker` at `src/ui/fields/date-picker.tsx` already exposes these props
- The payment-specific wrapper (`payment-date-picker-field.tsx`) just doesn't pass them through yet
- Change: thread `minimumDate` prop through the wrapper

**Both platforms:** Date picker natively greys out / blocks dates outside the range — no custom error UI needed.

**Slightly tricky part:** Computing `minDate` for rule 3 (deposit's `dueDate`) requires reading the deposit from form context. But the deposit is already loaded in both forms, so it's accessible.

**Open question:** Needs discussion with Seth on UX — block the date in the picker (recommended, simpler), or show an error on save?

### Backbook / Existing Milestones Edge Case

Milestones created before this feature was shipped may already have past dates. When a user opens one of these for editing:

- Setting `minimumDate` on the picker alone is **not sufficient** — both mobile and web pickers will show the past date greyed out visually but retain it in form state, meaning the user can hit Save and bypass the guard silently
- A form-level validator (Yup on mobile, form validation on web) must also be added to reject past dates on submit

**Recommended UX (needs Seth confirmation):**
- Load the form with the existing past date as-is
- Show an inline validation error immediately: *"Due date must be in the future"*
- Block Save until the user updates the date
- Do NOT silently auto-correct the date on load (avoids surprising the user with a changed value)

### Deposit ↔ Invoice terms relationship

A deposit is due by default on the invoice term date. There is currently no enforcement of the relationship between the deposit's `dueDate` and the invoice term date either. Whether to add this constraint is an open question — needs discussion with Seth.

---

## Vault-Time (Phase 3): Full Validation

At vault time, enforce all three rules as a server-side check before creating the auto-pay schedule:

1. No milestone `dueDate` in the past
2. No milestone `dueDate` today
3. No milestone `dueDate` earlier than the deposit's due date

**Why vault-time covers the gap for rules 1 & 2:** A milestone could be created today with a valid future date, then become past-due by the time the client vaults. The overdue-at-vault-time case is already handled by Phase 3's overdue handling (charge immediately or prompt) — vault-time validation catches this cleanly without needing UI enforcement pre-vault.

---

## Open Questions

- [ ] UX for all three rules pre-vault: block date in picker, or error on save? → Needs Seth input
- [ ] Deposit ↔ invoice term date: should this be enforced too, and where? → Needs Seth input
- [ ] Vault-time: hard block all past-due milestones, or only the *next* upcoming one?
- [ ] What's the UX when client tries to vault but a milestone is past-due — hard block with prompt to edit, or auto-handle?

---

## Relationship to Other Features

- **Invoice dueDate ↔ Milestone Sync:** A past-due milestone makes `Invoice.dueDate` immediately "overdue" — vault-time validation prevents this state from blocking auto-pay.
- **Auto-Pay execution:** The upcoming payment scheduler (is-payments) creates an EventBridge `at()` schedule from the milestone `dueDate` — requires a future timestamp.
