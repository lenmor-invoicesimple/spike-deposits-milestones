# Long Press Menu on Payment Rows (Mobile)

## Summary

Add a long press gesture on scheduled payment rows (mobile) that reveals an action sheet with Delete, Edit, and Mark Paid options. This replaces the current pattern of navigating into the payment detail screen to perform these actions — giving users faster access to common actions without leaving the Payment Schedule screen.

[Figma: Long Press Interaction](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5798-13744&t=k9ome08sjjiZvdSj-0)

---

## Feature Flag

- Gated behind a client-side feature flag (Optimizely)
- When FF is off, long press does nothing (current behavior)
- **Do NOT add any feature flag on is-parse-server**

---

## Design

[Figma: Long Press Interaction](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5798-13744&t=k9ome08sjjiZvdSj-0)

### Interaction Flow

1. **Long press on a scheduled payment row** → bottom action sheet appears
2. Action sheet header: `{Payment Name}` (the payment's label)
3. Actions:
   - **Delete** (red/destructive)
   - **Edit** (blue)
   - **Mark Paid** (blue)
   - **Cancel** (bottom, separate section)

### Mark Paid Flow

Tapping "Mark Paid" dismisses the action sheet and presents the existing **payment method selector** (same options already used in the Mark Paid flow). After selecting a method, the payment is marked as paid.

### Delete Flow

Tapping "Delete" shows a **confirmation alert**:
- Title: "Delete {Payment Name}?"
- Body: "Your invoice balance and due dates will be updated automatically."
- Actions: **Delete** (destructive) / **Continue Editing**

---

## Changes Required

### is-mobile

1. **Add `onLongPress` handler to payment rows** in the Payment Schedule screen's "Scheduled Payments" section
   - Add `onLongPress` prop to the `Pressable` wrapping each payment row

2. **Show action sheet on long press** using the existing `@expo/react-native-action-sheet` infrastructure
   - Use the `useActionSheet()` hook from `src/ui/action-sheet/extended.tsx` (or HOC `connectActionSheet`)
   - Same pattern as [client-list.tsx](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/clients/screens/client-list.tsx#L346) `onLongPress` → `showActionSheetWithOptions`
   - Set `destructiveButtonIndex` for Delete (renders red)
   - Set `cancelButtonIndex` for Cancel
   - Title/header: `{Payment Name}`

3. **Wire up Delete action**
   - Show confirmation alert (`Alert.alert`)
   - On confirm, call existing delete handler (same logic as `deleteUpcomingOrDepositPayment` in manage-payment screen)
   - After delete, invoice balance and due dates update automatically (backend handles this)

4. **Wire up Edit action**
   - Navigate to the existing manage-payment screen for that payment (same as current row tap behavior)

5. **Wire up Mark Paid action**
   - Show existing payment method selector
   - After method selection, call existing mark-as-paid handler with selected method
   - Payment moves from "Scheduled Payments" to "Payment History"

6. **Determine which payment types show which actions:**

| Payment State | Actions Available |
|---|---|
| Upcoming/Scheduled (unpaid) | Delete, Edit, Mark Paid |
| Deposit (unpaid) | Delete, Edit, Mark Paid |
| Completed/Paid | TBD — likely no long press, or only Edit/Delete |

---

## Existing Patterns & Code to Reuse

**Action sheet infrastructure** (already in the project):
- Library: `@expo/react-native-action-sheet`
- Wrapper: `src/ui/action-sheet/extended.tsx` — custom `useActionSheet()` hook and `connectActionSheet()` HOC
- Types: `src/ui/action-sheet/types.ts` — `ActionSheetItem` interface

**Reference implementation** — [client-list.tsx](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/clients/screens/client-list.tsx#L346):
- `onLongPress` on row → builds `ActionSheetItem[]` array → calls `showActionSheetWithOptions`
- Uses `destructiveButtonIndex` and `cancelButtonIndex`

**Action handlers** (already exist, just need to be callable from the list screen):
- **Delete:** `deleteUpcomingOrDepositPayment` in manage-payment screen
- **Edit:** Navigation to manage-payment screen (same as current row tap)
- **Mark Paid:** Existing mark-as-paid flow with payment method selector

---

## Acceptance Criteria

- [ ] Long press on a scheduled payment row shows action sheet with payment name as header
- [ ] Action sheet shows Delete (red), Edit, Mark Paid actions + Cancel
- [ ] Delete → confirmation dialog → deletes payment, updates invoice balance/due dates
- [ ] Edit → navigates to manage-payment screen for that payment
- [ ] Mark Paid → payment method selector → marks payment paid with selected method
- [ ] Cancel dismisses the action sheet without action
- [ ] Works on both iOS and Android
- [ ] Gated behind feature flag — no long press when FF is off
- [ ] Regular tap on payment row still navigates to edit (existing behavior preserved)
- [ ] Haptic feedback on long press (if platform supports)

---

## Prototype Implementation Notes

The prototype is implemented in `upcoming-payments-section.tsx` and `payment-history-section.tsx`. Key decisions made during prototyping:

- **Mark Paid** shows the payment method selector (same `paymentMethodStrings` action sheet) directly from the long press handler — no navigation to manage-payment screen needed
- After marking paid, calls `markPaidDeposit`/`markPaidUpcomingPayment` + `loadPayments` + `ensureInvoiceInSync`
- **Delete** calls `deleteUpcomingOrDepositPayment` directly (no confirmation alert in prototype — add for final)
- **Payment History rows** have a simpler long press (Edit + Cancel only)
- **Not yet implemented in prototype:** confirmation alert for Delete, payment name as action sheet title/header, haptic feedback, feature flag gating

## Open Questions — Engineering

- [ ] Confirm with core/mobile team: is the client-list pattern (`@expo/react-native-action-sheet` + `onLongPress`) still the preferred approach, or is there a newer pattern we should follow?

## Open Questions — Product/Design

- [ ] Is the long press menu for all payment types (Deposits, Milestones, Record Payment) or just Milestones?
- [ ] Should there be a visual indicator that rows are long-pressable (e.g. subtle hint on first use)?

---

## Dependencies

- Feature flag creation (separate ticket)
- Item #3 (Label field) — the action sheet header uses `{Payment Name}`, which ties into the custom label feature

## References

- [Figma: Long Press Interaction](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5798-13744&t=k9ome08sjjiZvdSj-0)
- [Confluence: D&M Phase 1/2 Plan](https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1169883137)
