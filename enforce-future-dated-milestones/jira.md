# Enforce Future-Dated Milestones

## Summary

Add date validation to milestone payment forms: milestones must have a `dueDate` in the future (not today, not past, not before the deposit's due date). Currently the system accepts any date with zero validation.

[Figma: Deposits & Payments Scheduling](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5798-13744&t=k9ome08sjjiZvdSj-0)

---

## Feature Flag

- Gated behind a client-side Optimizely feature flag (per Liz: all Phase 1/2 features must be gated)
- When FF is off: current behavior (no date restriction)
- When FF is on: past/today dates blocked in the date picker
- **Do NOT add any feature flag on is-parse-server**

---

## Validation Rules

| Rule | Reason |
|------|--------|
| Milestone `dueDate` cannot be in the past | Merchants have "Record Payment" for logging past offline payments; backdating a milestone is not a valid workflow |
| Milestone `dueDate` cannot be today | Same-day scheduling is unreliable for auto-pay; today = deposit territory, not milestones |
| Milestone `dueDate` cannot be earlier than deposit's `dueDate` | A milestone before the deposit breaks payment order |

---

## Scope

- **Milestones only** — validation applies when creating or editing a milestone payment
- **Deposits:** No date restriction (can be due today/immediately)
- **Record Payment (historical):** No date restriction (explicitly for logging past offline payments — must allow past dates)

---

## Changes Required

### is-web-app

**File:** [upcoming-payment-form-modal.tsx](https://github.com/invoice-simple/is-web-app/blob/master/nextjs/components/invoice-editor/invoice-payment/payment-scheduling/upcoming-payment-form-modal.tsx#L314)

The `DatePicker` currently has no `minDate` prop:

```tsx
<DatePicker
  id={'dueDate'}
  selected={field.value}
  ...
/>
```

**Change:** Add `minDate` when FF is on and payment is not a deposit:

```tsx
<DatePicker
  id={'dueDate'}
  selected={field.value}
  minDate={tomorrow}  // or deposit.dueDate + 1 day for rule 3
  ...
/>
```

The `DatePicker` component (wraps `react-datepicker`) already supports `minDate`. Recurring Invoices already uses this pattern in `schedule-form.tsx`.

Also add a form-level validator (react-hook-form) to reject past dates on submit (needed for backbook edge case — see below).

---

### is-mobile

**File:** [payment-date-picker-field.tsx](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/upcoming-payments/components/manage-payment/payment-date-picker-field.tsx)

The `DateTimePicker` (react-native-modal-datetime-picker) currently has no `minimumDate` prop:

```tsx
<DateTimePicker
  isVisible={datePickerVisible}
  mode='date'
  date={...}
  onConfirm={...}
  onCancel={...}
/>
```

**Change:** Add `minimumDate` when FF is on and payment is not a deposit:

```tsx
<DateTimePicker
  isVisible={datePickerVisible}
  mode='date'
  date={...}
  minimumDate={tomorrow}  // or deposit.dueDate + 1 day for rule 3
  onConfirm={...}
  onCancel={...}
/>
```

`react-native-modal-datetime-picker` natively supports `minimumDate` — dates outside range are greyed out / unselectable.

Also add a Yup validation rule on `dueDate` to reject past dates on submit (needed for backbook edge case — see below).

---

## Acceptance Criteria

- [ ] When FF is on, milestone date picker blocks past dates and today (greyed out / unselectable)
- [ ] When FF is on, milestone date picker blocks dates earlier than deposit's `dueDate`
- [ ] When FF is off, current behavior (no date restriction)
- [ ] Works on both iOS and Android (mobile)
- [ ] Works on web
- [ ] Form-level validation rejects past dates on submit (prevents silent save of stale dates)
- [ ] Existing milestones with past dates: no changes unless user opens and saves (exact behavior per Seth's decision above)
- [ ] No backend/is-parse-server changes

---

## Open Questions — Product/Design

### Backbook: Existing Milestones with Past Dates

Milestones created before this feature ships may already have past dates. When FF is on and a user edits one of these:

| Scenario | Behavior |
|----------|----------|
| User never opens that milestone | Nothing changes. Past date stays on backend as-is. |
| User opens it, doesn't touch date, hits Back/Cancel | Nothing changes. No warning. |
| User opens it, doesn't touch date, hits **Save** | **Decision needed — see options below** |
| User opens it, taps the date field | Picker shows future-only range. User must pick a valid date. |

**Why `minDate` alone is not enough:**

Both mobile and web pickers will visually grey out past dates but **retain the existing past date in form state**. Without a form-level validator, the user can hit Save and bypass the guard silently.

| | Option A: Always validate | Option B: Validate only when date is touched |
|---|---|---|
| **Behavior** | Block save if `dueDate` is past, even if user didn't touch it | Only validate date field if user interacted with it |
| **Pros** | Prevents bad data from being re-saved; eventually cleans backbook | Non-intrusive — user can edit label/amount without being forced to fix date |
| **Cons** | User who only wants to edit the label gets blocked by a date they didn't touch | Leaves bad data in system until user explicitly touches the date field |
| **Complexity** | Same | Same (both platforms track touched/dirty state already) |

Both options are trivially different to implement — a one-liner in the validation schema. This is purely a product decision.

### Other

- [ ] Should there be a visual indicator (banner/warning) when opening a milestone with a past date, even if we don't block save?
- [ ] Deposit ↔ invoice term date: should we also enforce that the deposit's `dueDate` can't be after the invoice term date?

## Prototype Implementation Notes

The prototype is implemented in:
- `payment-date-picker-field.tsx` — added `minimumDate` prop, passed through to `DateTimePicker`
- `manage-payment.screen.tsx` — passes `minimumDate={new Date()}` for milestone due dates (allows today)
- `validation-schema.ts` — Yup `.test('future-date', ...)` rejects past dates on submit using `startOfDay` comparison

**Prototype allows today** (uses `>=` today, not `>` today). The spec says "cannot be today" — confirm with Seth which is correct.

**Not yet implemented in prototype:** Rule 3 (milestone can't be before deposit's dueDate), web changes, feature flag gating.

## Open Questions — Engineering

- [ ] Rule 3 (`dueDate > deposit.dueDate`): confirm deposit is always accessible from form context on both platforms (believed yes but needs verification during implementation)

---

## Existing Patterns to Reuse

- **Web:** Recurring Invoices `schedule-form.tsx` already uses `<DatePicker minDate={new Date()} />`
- **Mobile:** `react-native-modal-datetime-picker` natively supports `minimumDate` / `maximumDate`
- **Both:** Date picker natively greys out / blocks dates outside the range — no custom error UI needed for the picker itself

---

## Dependencies

- Feature flag creation (separate ticket)

## References

- [Confluence: D&M Phase 1/2 Plan](https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1169883137)
