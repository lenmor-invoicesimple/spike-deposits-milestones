# Invoice dueDate ↔ Milestone Sync — Technical Exploration (2026-05-21)

> Context: Seth proposed that milestone dates should drive the invoice-level dueDate. This doc explores how dueDate currently works and the feasibility of syncing it from milestones.

---

## Seth's Proposal (from Slack, May 20-21)

- Invoice due date = soonest milestone date
- Upon completion of a milestone, invoice due date shifts to the next milestone's date
- If soonest milestone is overdue → treat invoice as overdue → autopay unavailable at checkout
- Milestone → Invoice derivation is **one-way only** (not vice versa)
- Merchant can manually override due date, but it gets re-derived if they schedule a sooner milestone

### When Invoice.dueDate actually matters (Seth's clarification, May 21)

- **Client HAS vaulted** → `Invoice.dueDate` is irrelevant. Payments are fully automated from milestone dates. The milestone dates are the source of truth for the auto-pay schedule and checkout modal.
- **Client has NOT vaulted** → `Invoice.dueDate` (derived or manual) is used for:
  - Sending automatic payment reminders
  - Surfacing the "overdue" modal + flow to the merchant

Thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779309054572939?thread_ts=1778699694.576969&cid=C0B331AP0BY
Seth's reply (May 21, 17:07): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779397655.206079?thread_ts=1778699694.576969&cid=C0B331AP0BY

### Seth's follow-up on locking vs. banner (May 21, 17:07)

- Floated a "use due date as invoice due date" toggle per milestone, but noted it breaks down if the toggled milestone isn't the soonest one
- Concern with hard-locking: merchants may want to override, and forcing them to navigate Invoice Details → Payment Scheduling → find the right milestone is high cognitive load
- **Proposed middle ground: transient toast on Payment Scheduling page** (not a persistent banner on Invoice Details) — shown for ~3s when a milestone is saved and its dueDate becomes the invoice dueDate (first milestone, or sooner than all others)

### Seth's final call on locking (May 26)

> "With all the edge cases that could come up from allowing overrides though, I think just locking will make the most sense. A shortcut to payment scheduling from invoice details page when dueDate is auto-derived works in this case!"

**Confirmed approach:**
- `Invoice.dueDate` is **locked** when milestones exist
- **Shortcut link** on Invoice Details (when dueDate is auto-derived) navigates to Payment Scheduling
- **Transient toast** on Payment Scheduling page when a milestone sets/changes the invoice dueDate

---

## Agreed Approach: Always Overwrite + Lock + Banner

After exploring multiple options (see "Alternatives Considered" below), we're going with:

### Rule

```
Invoice.dueDate = soonestUnpaidMilestoneDueDate
```

Always overwrite. When milestones exist, `Invoice.dueDate` is **locked** (not manually editable) and always reflects the soonest unpaid milestone's `dueDate`.

### UI

- dueDate/Terms fields are **disabled** when milestones exist
- A **banner** in Invoice Details says: "Due date based on your payment schedule"
- Banner is **tappable** — navigates directly to the soonest milestone for editing
- This eliminates Seth's cognitive load concern ("high cognitive load of going back to Payment Scheduling")

### Why this approach

| Concern | How it's addressed |
|---------|-------------------|
| Merchant wants to change due date | Tap banner → edit milestone directly |
| Confusion from auto-deriving | Banner explains it |
| "Fighting" between termsDay and milestones | Lock eliminates conflict; `termsDay` set to CUSTOM |
| Client re-derives dueDate from termsDay on save | `termsDay = CUSTOM` means "don't compute from formula" |
| Ratchet problem (date can never move later) | Always overwrite, no `min()` logic |
| Extra state/flags needed | None — no flags, no override detection |
| Milestone marked paid → what happens? | Re-derives to next unpaid milestone |
| All milestones paid/deleted | Reset to DUE_ON_RECEIPT, clear dueDate, unlock UI |

---

## How Invoice dueDate Works Today

### Two Fields — Stored Independently

| Field | Location | Type | Example |
|-------|----------|------|---------|
| `setting.termsDay` | `Invoice.setting.termsDay` | integer | `30`, `-1`, `0`, `-2` |
| `dueDate` | `Invoice.dueDate` (top-level) | string | `"2026-06-15"` or absent |

Both are stored in MongoDB. **No server-side derivation** between them — the relationship is purely client-side.

### InvoiceTermTypes (from `@invoice-simple/common`)

| Value | Constant | Meaning | Has dueDate? |
|-------|----------|---------|-------------|
| `-2` | `NONE` | No terms set (true default, never touched) | No |
| `-1` | `CUSTOM` | Merchant picked a specific date directly | Yes |
| `0` | `DUE_ON_RECEIPT` | Due immediately | No |
| `1+` | `NEXT_DAY`, `DAYS_2`, etc. | Net N days | Yes |

### How dueDate Gets Set (Client-Side Derivation)

`dueDate` is derived from `termsDay` **only at the moment of user interaction** on the Invoice Details screen (`is-mobile/src/features/documents/screens/invoice-info.screen.tsx`):

| User action | What happens |
|-------------|-------------|
| Picks terms (e.g., Net 30) | UI calls `getDueDate(invoiceDate, 30)` → `dueDate = invoiceDate + 30` |
| Changes invoice date | UI calls `getDueDate(newDate, termsDay)` → re-derives `dueDate` |
| Picks "Custom" | UI lets them pick a date, `termsDay = -1` |
| Picks "None" or "Due on Receipt" | `dueDate = null/undefined` |

On save, both `termsDay` (inside `setting` object) and `dueDate` (top-level) are sent to Parse independently — mobile serializes both in `is-mobile/src/services/parse/models/invoice.ts` (line 59 for dueDate, line 85 for setting).

**Not re-derived on every save** — only when merchant explicitly changes Date or Terms fields.

### How a New Invoice Gets Terms

New invoice copies `setting.termsDay` from the previous invoice via `duplicateSetting()` and re-derives `dueDate` from it:

```typescript
// is-mobile/src/services/realm/entities/invoice/repository.ts:420
const invoiceTypeDueDate = getDueDate(invoiceDate, (setting && setting.termsDay) || DUE_ON_RECEIPT);
```

There is **no flag** distinguishing "merchant explicitly set terms" from "inherited from previous invoice."

### Display

- **Web**: Shown in invoice editor settings section (hidden when On Receipt)
- **Mobile**: Shown in invoice detail (shows "Past Due" in red if dueDate < today)

### "Overdue" Determination

**No stored field.** Purely UI-calculated at runtime:
- `dueDate < today` AND `balanceDue > 0` AND `dueDate != null`
- Mobile: `getOverdueInvoices()` in `invoice/repository.ts` queries Realm with this condition
- Web: same logic, compared at render time

---

## Milestone Payment dueDate — Separate System

Each Payment object (milestone) has its **own** `dueDate` field:

- Stored on the Parse Payment class (separate collection)
- Set independently by the merchant when creating/editing milestones
- **No relationship to Invoice.dueDate** — they are completely disconnected today

### Dual Storage Pattern

Payments exist in two places simultaneously:
1. **Invoice.payments[]** — embedded array on the Invoice document (denormalized, for UI — only **paid** payments)
2. **Payment collection** — separate Parse objects (normalized, for queries — all milestones including scheduled/unpaid)

They're kept in sync by Parse `beforeSave` hooks on Invoice.

**Key finding from testing:** `Invoice.payments[]` only contains paid payments. Scheduled milestones (unpaid, no `date` field) live exclusively in the Payment collection until marked paid.

---

## Complete Flow: How Milestone Changes Propagate (Verified by Testing)

### Path C-add — Add new milestone (mobile, primary flow)

Mobile calls a dedicated cloud function — it does NOT save the Invoice with an updated `payments[]`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MOBILE                                                                     │
│  manage-payment.screen.tsx → Save                                           │
│  → Parse.Cloud.run('invoiceAddPayment', { invoiceId, payment })             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PARSE SERVER — invoiceAddPayment.ts                                        │
│                                                                             │
│  1. getInvoice(invoiceId)                                                   │
│  2. new Payment() → set dueDate, paymentType, paymentMode, paymentValue     │
│  3. parse.save(paymentObj)         ← Payment saved to collection            │
│                                                                             │
│  4. completedDate = payment?.date  ← undefined for scheduled milestone      │
│                                                                             │
│  5. [completedDate = undefined] → updatePaymentDetails({ invoice })         │
│     → recalcs balanceDue, paidAmount, saves Invoice                         │
│                                                                             │
│  ⚠️  Invoice.dueDate never touched                                          │
│  ✅ HOOK POINT: after updatePaymentDetails() at line 107                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Path B — Edit existing milestone (web & mobile, same endpoint)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  WEB / MOBILE                                                               │
│  → Parse.Cloud.run('invoiceUpdatePayment', { payment })                     │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PARSE SERVER — invoicePayments.ts → [default branch]                       │
│  → updatePaymentByObjectOrTransactionId(payment)                            │
│  → updatePaymentByFieldName(...)                                            │
│  → updatePayment(paymentObj, payment)              updatePayment.ts:106     │
│                                                                             │
│  1. paymentObj.set('dueDate', payment.dueDate)     ← Payment saved          │
│  2. parse.save(paymentObj)                                                  │
│  3. getInvoiceByAccountAndRemoteId()               ← fetches parent Invoice │
│  4. updateDepositOnInvoiceSetting() (if deposit)                            │
│  5. removeHistoricalPaymentFromInvoice()                                    │
│  6. updatePaymentDetails({ invoice })              ← recalcs balanceDue     │
│  7. updatePaidDateInInvoice() (if deleted+paid)                             │
│  8. updatePaymentsInInvoice() (if has amount+date)                          │
│                                                                             │
│  ⚠️  Invoice.dueDate never touched                                          │
│  ✅ HOOK POINT: after updatePaymentDetails() at line 155                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Path C-paid — Mark milestone paid (mobile)

**Discovered via testing:** "Mark Paid" uses `invoiceUpdatePayment` with `markPaid: true`, NOT `invoiceAddPayment`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MOBILE                                                                     │
│  → Parse.Cloud.run('invoiceUpdatePayment', { markPaid: true, payment })     │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PARSE SERVER — invoicePayments.ts → [markPaid branch]                      │
│  → markPaidUpcomingPayment(payment)                 updatePayment.ts:43     │
│    → updatePaymentByFieldName('objectId', ...)      ← saves Payment w/ amount+date
│    → addPaymentToInvoicePayment({ payment, updatedPayment })                │
│      → getInvoiceByAccountAndRemoteId()                                     │
│      → addPaymentToInvoice({ invoice, payment })    addPaymentToInvoice.ts  │
│                                                                             │
│  Inside addPaymentToInvoice:                                                │
│  1. pickFields(payment)            ← STRIPS dueDate from embedded payment   │
│  2. payments.push(paymentInArray)  ← adds to Invoice.payments[] (paid only) │
│  3. updatePaymentDetails()         ← recalcs balanceDue                     │
│  4. invoice.set('paidDate', ...)                                            │
│  5. invoice.set('payments', [...])                                          │
│  6. invoice.set('partialPayment', ...)                                      │
│  7. parse.save(invoice)                                                     │
│                                                                             │
│  ⚠️  Invoice.dueDate never touched                                          │
│  ✅ HOOK POINT: after updatePaymentDetails() at line 66                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Invoice props touched by each path (today, without dueDate sync)

| Path | Props touched on Invoice |
|------|--------------------------|
| C-add (new milestone) | `balanceDue`, `paidAmount`, `updated` |
| B (edit milestone) | `balanceDue`, `paidAmount`, `updated`, optionally `paidDate`, `payments[]` |
| C-paid (mark paid) | `balanceDue`, `paidAmount`, `updated`, `paidDate`, `payments[]`, `partialPayment` |

None touch `dueDate` or `termsDay` today.

---

## Implementation Plan

### New helper function

**File:** `cloud/collections/invoice/utils/recalculateInvoiceDueDate.ts`

```typescript
import { format } from 'date-fns';
import { InvoiceTermTypes } from '@invoice-simple/common';
import * as parse from '../../../parse';

export const recalculateInvoiceDueDate = async (invoice: parse.Invoice): Promise<void> => {
  const q = new Parse.Query(parse.Payment);
  q.equalTo('invoiceRemoteId', invoice.get('remoteId'));
  q.equalTo('deleted', false);
  q.doesNotExist('date');           // no completion date = unpaid/scheduled
  q.exists('dueDate');              // has a dueDate set
  q.ascending('dueDate');
  q.limit(1);

  const [soonestMilestone] = await q.find({ useMasterKey: true });
  const setting = invoice.get('setting');

  if (soonestMilestone) {
    const dueDate = soonestMilestone.get('dueDate');
    invoice.set('dueDate', format(new Date(dueDate), 'yyyy-MM-dd'));
    setting.termsDay = InvoiceTermTypes.CUSTOM; // -1: prevents client from re-deriving
  } else {
    // No unpaid milestones remain — reset to clean slate
    invoice.set('dueDate', null);
    setting.termsDay = InvoiceTermTypes.DUE_ON_RECEIPT; // 0: unlocks UI
  }

  invoice.set('setting', setting);
};
```

### 3 hook points

| Hook point | File | Location | Code to add |
|---|---|---|---|
| C-add | `invoiceAddPayment.ts` | After `updatePaymentDetails()` at line 107 | `await recalculateInvoiceDueDate(invoice)` |
| B | `updatePayment.ts` | After `updatePaymentDetails()` at line 155 | `await recalculateInvoiceDueDate(invoice)` |
| C-paid | `addPaymentToInvoice.ts` | After `updatePaymentDetails()` at line 66 | `await recalculateInvoiceDueDate(invoice)` |

### UI changes (both platforms)

| Change | Web | Mobile |
|--------|-----|--------|
| Lock dueDate/Terms when milestones exist | Disable inputs in invoice editor | Disable date picker in invoice details |
| Banner "Due date based on payment schedule" | Show in invoice settings section | Show in invoice details |
| Banner tap → navigate to soonest milestone | Open upcoming-payment-form-modal | Navigate to ManagePaymentScreen |

---

## Edge Cases

| Scenario | `dueDate` | `termsDay` | UI |
|----------|-----------|------------|-----|
| No milestones (plain invoice) | Normal derivation | Unchanged | Editable |
| 1 milestone added | = milestone's dueDate | → CUSTOM (-1) | Locked |
| Multiple milestones | = soonest unpaid | CUSTOM (-1) | Locked |
| Soonest milestone marked paid | Shifts to next unpaid | CUSTOM (-1) | Locked |
| All milestones paid | → `null` | → DUE_ON_RECEIPT (0) | **Unlocked** |
| All milestones deleted | → `null` | → DUE_ON_RECEIPT (0) | **Unlocked** |
| Milestone deleted (others remain) | Re-derives from remaining | CUSTOM (-1) | Locked |
| Milestone date edited | Updates to new soonest | CUSTOM (-1) | Locked |
| Merchant tries to edit dueDate/Terms | Locked — must edit milestone | — | — |
| Merchant changes invoice Date | **No change** (client-side guard) | — | Date editable, dueDate locked |
| Deposit added | No effect (dueDate: null) | — | — |

---

## Alternatives Considered

### Option B — Soft auto-derive with `min()` logic

**Rule:** `Invoice.dueDate = min(currentDueDate, soonestMilestoneDueDate)`

**Problem — the ratchet:** dueDate can only ever move earlier, never later. Once a milestone sets it to Jun 10, even if that milestone is deleted/paid/edited, the old dueDate stays because `min(Jun 10, Jun 25) = Jun 10`. Merchant gets stuck and can't push due date later without deleting all milestones.

### Option C — Soft auto-derive with override flag

**Rule:** Track `isDueDateOverridden` flag. Only re-derive when flag is false or when new milestone is sooner than override.

**Problem:** Extra state, complex conditional logic ("who wrote last?"), fighting between termsDay and milestones, confusing UX where sometimes the override sticks and sometimes it doesn't.

### Why "always overwrite + lock + banner" wins

- No extra state or flags
- No ratchet problem
- No fighting between termsDay and milestones
- Eliminates entire class of edge cases
- Banner + deep link provides escape hatch for merchants
- Lowest server-side complexity (~2 SP)

---

## Warning Modal: "This will override Invoice Terms"

### Confirmed flow (Seth + Liz, May 27)

Thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779908089.296689

1. Merchant creates invoice, has terms set (e.g., Net 30)
2. Navigates to Payment Schedule, creates first milestone
3. On back button (`<`), show warning modal: "This will update your invoice due date to match the milestone"
   - **Accept** → saves milestone, `Invoice.dueDate` overridden by milestone
   - **Decline** → discards milestone, returns to Payment Schedule page

### When to show the modal (no flag needed)

Show the warning if **both** conditions are met:

1. **Real dueDate exists:** `termsDay === -1 (CUSTOM) || termsDay > 0 (Net N)`
   - Skip if `termsDay === -2 (NONE)` or `termsDay === 0 (DUE_ON_RECEIPT)` — no real dueDate to override
2. **AND** one of:
   - It's the **first milestone** being created (no existing unpaid milestones for this invoice)
   - OR the new milestone's date is **sooner than all existing milestones** (it would become the new Invoice.dueDate)

If neither condition in #2 is met (e.g., adding a later milestone), the new milestone doesn't change Invoice.dueDate — no modal needed.

### Why no `termsModifiedByUser` flag is needed

- We considered tracking whether the merchant explicitly set terms vs. inherited from previous invoice
- **Decision:** always show the warning if a real dueDate exists, even if inherited. Rationale: if it was inherited, it was still the merchant's previous workflow/preference — worth alerting
- This also naturally handles the "delete all → reset to DUE_ON_RECEIPT → create again" flow: after reset, `termsDay = 0` → condition #1 fails → no modal

### Deposits don't interfere

Deposits have `dueDate: null` (paid at checkout, no future date). Our query uses `q.exists('dueDate')` — deposits are automatically excluded. No special handling needed.

---

## Open Questions

### For Seth/Product
- [ ] Is the tappable banner → milestone navigation acceptable UX for the "override" use case?
- [x] Should deposits count as milestones for dueDate derivation?
  → **No.** Deposits have `dueDate: null` by design (paid at checkout). Already excluded by `q.exists('dueDate')` in the query.
- [x] If merchant has milestones + mark paid manually → any interaction with auto-pay gating?
  → Asked in Slack (May 25): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779742352.805339?thread_ts=1778699694.576969&cid=C0B331AP0BY
  → **Seth's answer (May 25):** Mark Paid should still be allowed for **non-vaulted** users to avoid breaking existing UX for non-payments-enabled users. For **vaulted users**, Mark Paid would be blocked by the edit modal.
  → Implication: `recalculateInvoiceDueDate` (1C hook) should still fire on manual Mark Paid for non-vaulted users — it will correctly shift `Invoice.dueDate` to the next unpaid milestone.

### For Lenmor (answered)
- [x] Are there any other paths that modify milestones? → No, all go through these 3 server paths
- [x] What happens when all milestones paid? → Keep last milestone's dueDate
- [x] `Invoice.payments[]` only has paid payments? → Yes, unpaid milestones live only in Payment collection

### Juan's feedback (May 26, DM)

Thread: https://ec-mobile-solutions.slack.com/archives/D08DUAP0J79/p1779830571913379

**Concern:** Unlike `balanceDue` (which the user can't edit), `dueDate` is user-editable. If we auto-derive it from milestones, we'd be overwriting something the user might have intentionally set.

**Alternative proposed:** Run a cron job that computes due date on-the-fly from milestones and just sends the reminder email — don't persist it on the Invoice object.

**Why the cron approach won't work:** `Invoice.dueDate` is displayed across mobile, web, and PDF. It needs to be the single source of truth on the object, not computed separately at notification time.

**Resolution:** The "always overwrite + lock" approach addresses Juan's concern:
- Once milestones exist → derive `dueDate` from soonest unpaid milestone (persist on Invoice)
- Lock the UI field so the user can't manually override while milestones are active
- If all milestones are removed → unlock, user can edit freely again
- Seth confirmed locking is acceptable (May 26)

---

## References

### Slack Threads
- Seth's initial proposal (May 20-21): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779309054572939?thread_ts=1778699694.576969&cid=C0B331AP0BY
- Seth's locking/toast clarification (May 21): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779397655.206079?thread_ts=1778699694.576969&cid=C0B331AP0BY
- Juan's DM feedback (May 26): https://ec-mobile-solutions.slack.com/archives/D08DUAP0J79/p1779830571913379
- Seth + Liz: warning modal flow (May 27): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779908089.296689
- Lenmor: modal condition confirmed by Liz (May 27): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779910077.729349
- Seth: additional guardrail for modal (May 27): https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779911187.112289

### Design
- Seth's Figma exploration: https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5285-29141
- FigJam milestone timeline (3 paths: whole invoice paid / individual milestone paid / not paid)

### Code
- Parse validation (termsDay allowed values): `is-parse-server/cloud/collections/invoice/invoiceValidation.ts:338-348`
- InvoiceTermTypes enum: `node_modules/@invoice-simple/common/dist/typings/types.d.ts`
- Invoice model (web): `is-web-app/client/src/models/InvoiceModel.ts:525-528`
- updatePayment cascade: `is-parse-server/cloud/collections/payment/updatePayment.ts:106-167`
- Invoice.beforeSave: `is-parse-server/cloud/collections/invoice/invoiceHooks.ts:36-76`
- invoiceAddPayment: `is-parse-server/cloud/collections/invoice/functions/invoiceAddPayment.ts`
- addPaymentToInvoice: `is-parse-server/cloud/collections/invoice/utils/addPaymentToInvoice.ts`
- invoicePayments (routing): `is-parse-server/cloud/collections/invoice/functions/invoicePayments.ts`
- getDueDate (mobile): `is-mobile/src/services/documents-preview/invoice-due-date.ts`
- Invoice Details screen (mobile): `is-mobile/src/features/documents/screens/invoice-info.screen.tsx`
- Invoice serialize to Parse (mobile): `is-mobile/src/services/parse/models/invoice.ts:59,85`
- New invoice creation (mobile): `is-mobile/src/services/realm/entities/invoice/repository.ts:416-436`
