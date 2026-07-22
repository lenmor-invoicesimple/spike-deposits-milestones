# Simple Deposit CRUD on Invoice (Mobile)

**Status:** Planning — no designs yet
**Epic:** IS-11250 Deposit Optimization v1
**Depends on:** Estimate deposit work (same branch, IS-10432)

---

## Context

Invoice already has deposit configuration via Payment Scheduling → ManagePaymentScreen. This ticket is about potentially simplifying or redesigning that flow to be more consistent with the new Estimate inline deposit UX.

No designs exist yet. Open questions below capture what needs to be decided before implementation.

---

## Key Product/Design Question: Coexistence with Payment Scheduling

**This is the main blocker.** No designs exist yet for how "Request Deposit" (inline toggle) should coexist with the existing Payment Scheduling page on Invoice.

The web estimate deposit ticket (`/docs/superpowers/specs/web-estimate-deposit/`) also depends on this answer — if we build web estimate deposit before this is decided, we risk building the wrong pattern and refactoring later.

**Options being considered:**

| Option | Description | Impact |
|--------|-------------|--------|
| **Replace** | Inline toggle replaces the deposit portion of Payment Scheduling. Milestones stay on the separate screen. | ManagePaymentScreen becomes "Milestones Only". Simplest UX but requires refactoring what that screen does. |
| **Shortcut + Advanced** | Inline toggle is a quick-access shortcut. Payment Scheduling page remains the full-featured view for deposits + milestones. | Two places to configure the same field — sync issues, user confusion. |
| **Full inline** | Remove Payment Scheduling entirely. Move deposit + milestones all inline. | Big refactor. Milestones inline might be too complex for the editor screen. |
| **Keep separate** (status quo for Invoice) | Don't add inline toggle to Invoice. Keep Payment Scheduling page as-is. | Estimate and Invoice have different UX — inconsistent, but simpler to ship. |

**Recommendation:** Wait for designs before starting Invoice deposit work (mobile or web). Estimate's inline toggle is shipped and validated — that gives product data to inform the Invoice decision.

---

## Open Questions

### UX Approach

1. **Same inline toggle as Estimate?** Or keep the separate ManagePaymentScreen approach?
   - Estimate uses `RequestDepositSection` inline on the editor screen
   - Invoice currently navigates: Editor → PaymentSchedulingScreen → ManagePaymentScreen → addDeposit()
   - If we go inline for Invoice too, we hit the same race condition that Estimate had (see below)

2. **How does this interact with Payment Scheduling?**
   - Payment Scheduling already has deposit + milestone configuration
   - If we add an inline deposit toggle, does it replace ManagePaymentScreen's deposit section? Coexist? Override?
   - Risk of two places to configure the same field

3. **Free-entry percentage vs preset?**
   - `DepositRate` type from `@invoice-simple/common` is a preset literal union (0|10|20|30|40|50)
   - Estimate uses free-entry up to 50% (cast with `as any`)
   - Should Invoice match, or keep presets?

### Technical Approach

4. **Provider guard: `isAnyPaymentsProviderAccepting` vs `isAnyPaymentsProviderEligible`?**
   - Estimate uses `isAnyPaymentsProviderAccepting` (stricter — provider is live)
   - Invoice's existing `PaymentSchedulingSection` and `InvoicePaymentsSection` use `isPaymentProviderEligible` (looser — user is in supported country / opted in)
   - Evaluate whether Invoice has edge cases (e.g. mid-onboarding flows) that need the looser check
   - If we tighten to `Accepting`, users mid-setup won't see the section

5. **Race condition if going inline:**
   - Invoice currently avoids the race because `addDeposit()` calls a Parse Cloud Function directly (bypasses Realm→Parse sync)
   - If we move to inline editing (like Estimate), we'd need the same `!isEstimate` → `!isInvoice` skip in `updatePaymentsAndDeposit()`, or a different solution
   - Alternative: keep the direct Parse call approach but trigger it from inline UI (new Cloud Function needed)

6. **Refactoring ManagePaymentScreen:**
   - If inline replaces the deposit section of ManagePaymentScreen, what happens to the remaining milestones UI on that screen?
   - ManagePaymentScreen handles: deposit type/amount, milestones, payment mode (percent/flat/none)
   - Splitting out deposit from milestones may require rethinking that screen's purpose

### Data Model

7. **Same fields, same hooks:**
   - `setting.depositType`, `setting.depositRate`, `setting.depositAmount` — shared across Invoice and Estimate
   - Parse Server `handleDepositChange` hook creates Payment records for both doc types
   - No schema migration needed regardless of UX approach

8. **Estimate ↔ Invoice conversion:**
   - `src/services/realm/entities/invoice/use-cases.ts` line 61 and 121 — deposit fields are currently omitted when converting estimate↔invoice
   - Intentional for manual "Make Invoice" but needs revisiting if deposit should carry over

---

## Technical Notes (from Estimate implementation)

### The two save paths

```
INVOICE (existing — separate screen):
  Editor → navigate → PaymentSchedulingScreen → ManagePaymentScreen
  → addDeposit() → Parse Cloud Function (direct) → Payment record created
  → navigate back → handleSave() → fetchParse → remote is UP-TO-DATE ✓

ESTIMATE (new — inline):
  Editor → inline RequestDepositSection → write setting.depositType to Realm
  → handleSave() fires immediately → fetchParse → remote is STALE ✗
  → Realm→Parse sync pushes later → Parse Server handleDepositChange hook
    creates Payment record
```

### Key files for Invoice deposit (current)

| File | Role |
|------|------|
| `src/features/documents/screens/manage-payment.screen.tsx` | ManagePaymentScreen — configures deposit type/amount |
| `src/services/parse/models/payment.ts:363` | `addDeposit()` — Parse Cloud Function call |
| `src/features/documents/navigators/invoice.navigator.tsx:~1112` | `updatePaymentsAndDeposit()` — fetches remote, overwrites local |
| `src/features/documents/screens/invoice.screen.tsx` | Editor screen — renders PaymentSchedulingSection |

### Reusable from Estimate work

- `RequestDepositSection` component — already docType-agnostic (takes props, no docType assumption)
- Action sheet dropdown pattern (Type selector)
- 50% max cap logic
- `isPaymentProviderAccepting()` method on invoice.screen.tsx

### is-packages guards (web app — NOT mobile)

These affect the web app read/write path and need fixing for web parity regardless of mobile approach:
- `is-packages/packages/parse-domain/src/mapping-functions/parseInvoiceToDocument.ts` — `getEstimateSettings()` does NOT extract deposit fields
- `is-packages/packages/domain-invoicing/src/document/estimate.ts` — `EstimateSettings` type excludes deposit fields

---

## What's Left (once designs arrive)

- [ ] Design decision: inline vs separate screen vs hybrid
- [ ] Decide interaction with existing Payment Scheduling screen
- [ ] Decide provider guard level (`Accepting` vs `Eligible`)
- [ ] Implement UI changes
- [ ] Handle race condition (if going inline)
- [ ] Update/refactor ManagePaymentScreen if needed
- [ ] Feature flag gating
- [ ] E2E test: deposit round-trip on Invoice
