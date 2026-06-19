# Payment Scheduling UI Changes

## Summary

Copy changes, layout cleanup, and display improvements across the Payment Schedule screen (mobile). Aligns current implementation with Figma designs.

[Figma: Deposits & Payments Scheduling](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5798-12743&t=k9ome08sjjiZvdSj-0)

---

## Feature Flag

- Gated behind a client-side Optimizely feature flag (per Liz: all Phase 1/2 features must be gated)
- When FF is off: current behavior
- When FF is on: new copy, layout, and display changes
- **Do NOT add any feature flag on is-parse-server**

---

## Changes Required

> Follow Figma as source of truth at all times. This list may be stale. Sync with Seth as needed.

### 1. Rename screen title

- **Current:** "Payment Scheduling"
- **Figma:** "Payment Schedule"

### 2. Conditionally display "Amount Remaining" row in summary section

- **Current:** Always shows Amount Remaining
- **Figma:** Only show when invoice is partially paid (i.e. when invoice total and amount remaining are different numbers)
- Remove "Deposit Due" row

### 3. Rename section headers and row labels

| Current | Figma |
|---------|-------|
| "Upcoming Payments" (section header) | "Scheduled Payments" |
| "Add Upcoming Payment" | "Schedule a Payment" |
| "Record Payment" | "Record a Payment" |

### 4. Replace $0.00 with + icon on action rows

- **Current:** "Request a Deposit $0.00", "Add Upcoming Payment $0.00", "Record Payment $0.00"
- **Figma:** "Request a Deposit +", "Schedule a Payment +", "Record a Payment +"
- The + should be an icon, not a dollar amount

### 5. Payment History rows: show Name instead of payment method

- **Current:** Shows payment method as the row title (e.g., "Credit Card")
- **Figma:** Should show the payment's **Name** (label) as the row title
- Subtitle remains "Paid {date}"

---

## Scope

- **Mobile (is-mobile):** All changes above
- **Web (is-web-app):** Needs web designs — or can infer from mobile if Seth prefers

---

## Acceptance Criteria

- [ ] Screen title reads "Payment Schedule"
- [ ] Summary section shows "Amount Remaining" only when invoice is partially paid (paid > 0 and remaining < total)
- [ ] Section header reads "Scheduled Payments" (not "Upcoming Payments")
- [ ] Action rows show "Schedule a Payment +", "Request a Deposit +", "Record a Payment +"
- [ ] Action rows show + icon instead of $0.00
- [ ] Payment History rows show payment Name as title (not payment method)
- [ ] All changes gated behind FF — current behavior when FF is off
- [ ] No backend/is-parse-server changes

---

## Open Questions — Product/Design

- [ ] Please confirm the list of changes above. The list was derived from visual differences between Figma and existing implementation.
- [ ] Are there any changes on the Deposit screen (Manage Deposit form)?
- [ ] Web designs: do we need separate web designs, or infer from mobile?

## Open Questions — Engineering

- [ ] (none currently)

---

## Dependencies

- Feature flag creation (separate ticket)
- Item #3 (Label field) — Payment History rows showing "Name" ties into the custom label feature

## References

- [Figma: Deposits & Payments Scheduling](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5798-12743&t=k9ome08sjjiZvdSj-0)
- [Confluence: D&M Phase 1/2 Plan](https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1169883137)
