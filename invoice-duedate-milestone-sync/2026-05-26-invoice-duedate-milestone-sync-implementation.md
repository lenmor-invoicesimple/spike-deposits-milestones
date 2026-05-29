# Invoice dueDate ↔ Milestone Sync — Implementation (2026-05-26)

> Companion to: `2026-05-21-invoice-duedate-milestone-sync-exploration.md`
> Status: Implemented in is-parse-server, pending UI changes on web/mobile

---

> **⚠️ CRITICAL for is-payments team:**
> The upcoming payment auto-pay execution handler **MUST** use `Parse.Cloud.run('invoiceUpdatePayment', { markPaid: true, payment })` — NOT `invoiceAddPayment`.
> This updates the existing milestone's `date` field so that `Invoice.dueDate` correctly shifts to the next unpaid milestone.
> If `invoiceAddPayment` is used instead (like RP does today), the original milestone stays "unpaid" in queries and `Invoice.dueDate` never advances.
> See "Trigger Audit" section below for full details.

---

## Rule

```
Invoice.dueDate = soonestUnpaidMilestoneDueDate
Invoice.setting.termsDay = CUSTOM (-1)       // when milestones set dueDate
Invoice.setting.termsDay = DUE_ON_RECEIPT (0) // when all milestones deleted
```

Always overwrite on every milestone change. `Invoice.dueDate` is locked in the UI when milestones exist — merchant must edit the milestone to change it. `termsDay` is set to CUSTOM to prevent client-side re-derivation.

---

## New File

### `cloud/collections/invoice/utils/recalculateInvoiceDueDate.ts`

Shared helper called from all 3 hook points. Queries the Payment collection directly (not `Invoice.payments[]` — that only contains paid payments) for the soonest unpaid milestone.

```typescript
// Queries Payment collection for unpaid milestones (no date = not yet paid)
// Sets Invoice.dueDate to the soonest one's dueDate + termsDay to CUSTOM
// If no unpaid milestones remain → resets termsDay to DUE_ON_RECEIPT, clears dueDate
export const recalculateInvoiceDueDate = async (invoice: parse.Invoice): Promise<void>
```

**What it sets:**

| Condition | `Invoice.dueDate` | `setting.termsDay` |
|-----------|-------------------|--------------------|
| Unpaid milestone exists | Soonest milestone's dueDate | `-1` (CUSTOM) |
| No unpaid milestones (all paid/deleted) | `null` (cleared) | `0` (DUE_ON_RECEIPT) |

**Why `termsDay` must change too:**
- Mobile sends both `dueDate` and `setting.termsDay` on every invoice save
- If `termsDay` stays as `30` (Net 30), the client's `getDueDate()` could re-derive `dueDate = invoiceDate + 30` on the next save, overwriting the milestone-derived value
- Setting `termsDay = -1` (CUSTOM) means: "dueDate is set directly, don't compute from formula"
- Setting `termsDay = 0` (DUE_ON_RECEIPT) on "all milestones deleted" means: clean slate, no dueDate

**How to set `termsDay` (existing pattern in `invoiceHooks.ts`):**
```typescript
const setting = invoice.get('setting');
setting.termsDay = InvoiceTermTypes.CUSTOM; // or DUE_ON_RECEIPT
invoice.set('setting', setting);
```

**Why Payment collection and not `Invoice.payments[]`:**
- `Invoice.payments[]` only contains **paid** payments
- Unpaid/scheduled milestones live exclusively in the Payment collection until marked paid
- Must query Payment collection to see future milestones

---

## Hook Points

### 1A — New milestone added
**File:** `cloud/collections/invoice/functions/invoiceAddPayment.ts`
**When:** Mobile calls `invoiceAddPayment` cloud function with a scheduled milestone (no `date` = not yet paid)

```
invoiceAddPayment
  └─ [!completedDate branch] ← this is the scheduled milestone path
       updatePaymentDetails()
       recalculateInvoiceDueDate()  ← 1A NEW
       parse.save(invoice)          ← 1A NEW
```

**Before this change:** returned early after `updatePaymentDetails()` without touching `dueDate`.

---

### 1B — Milestone edited (due date, amount, or any field)
**File:** `cloud/collections/payment/updatePayment.ts`
**When:** Web or mobile calls `invoiceUpdatePayment` cloud function (default branch — not mark-paid, not online payment)

```
updatePayment()
  └─ updatePaymentDetails()
     recalculateInvoiceDueDate()   ← 1B NEW
     parse.save(invoice)           ← 1B NEW
     updatePaidDateInInvoice()     (existing, only if deleted+paid)
     updatePaymentsInInvoice()     (existing, only if has amount+date)
```

**Note:** `recalculateInvoiceDueDate` runs before `updatePaidDateInInvoice` and `updatePaymentsInInvoice`. Both of those do their own `parse.save(invoice)` but only touch `paidDate` and `payments[]` — they will not overwrite `dueDate`.

---

### 1C — Milestone marked paid
**File:** `cloud/collections/invoice/utils/addPaymentToInvoice.ts`
**When:** Mobile calls `invoiceUpdatePayment` with `markPaid: true` → `markPaidUpcomingPayment` → `addPaymentToInvoice`

```
addPaymentToInvoice()
  └─ updatePaymentDetails()
     invoice.set('paidDate', ...)
     invoice.set('payments', [...])   ← paid payment appended (dueDate stripped by pickFields)
     invoice.set('partialPayment', ...)
     invoice.set('updated', ...)
     recalculateInvoiceDueDate()      ← 1C NEW (shifts to next unpaid milestone)
     parse.save(invoice)
```

**Why this matters:** When a milestone is marked paid, its `date` field gets set — `doesNotExist('date')` in the helper query now excludes it, so `dueDate` shifts to the next unpaid milestone automatically.

---

## Full Call Chain Reference

```
MOBILE: add milestone
  Parse.Cloud.run('invoiceAddPayment', { invoiceId, payment: { dueDate, paymentType, ... } })
    → invoiceAddPayment.ts
        → [!completedDate] updatePaymentDetails() → recalculateInvoiceDueDate() [1A]

MOBILE/WEB: edit milestone
  Parse.Cloud.run('invoiceUpdatePayment', { payment })
    → invoicePayments.ts [default branch]
        → updatePaymentByObjectOrTransactionId()
            → updatePayment()
                → updatePaymentDetails() → recalculateInvoiceDueDate() [1B]

MOBILE: mark milestone paid
  Parse.Cloud.run('invoiceUpdatePayment', { markPaid: true, payment })
    → invoicePayments.ts [markPaid branch]
        → markPaidUpcomingPayment()
            → updatePaymentByFieldName()     ← saves Payment with amount+date
            → addPaymentToInvoicePayment()
                → addPaymentToInvoice()
                    → updatePaymentDetails() → recalculateInvoiceDueDate() [1C]
```

---

## Files Changed

| File | Type | Change |
|------|------|--------|
| `cloud/collections/invoice/utils/recalculateInvoiceDueDate.ts` | **NEW** | Shared helper |
| `cloud/collections/invoice/functions/invoiceAddPayment.ts` | Modified | Import + 1A hook |
| `cloud/collections/payment/updatePayment.ts` | Modified | Import + 1B hook |
| `cloud/collections/invoice/utils/addPaymentToInvoice.ts` | Modified | Import + 1C hook |

---

## Pending: UI Changes

The server always overwrites `Invoice.dueDate` and `setting.termsDay` — the UI needs to reflect that these fields are managed.

### Warning Modal (on first milestone creation)

**When to show** (both conditions must be true):
1. `termsDay === -1 (CUSTOM) || termsDay > 0` — a real dueDate exists to override
2. AND one of:
   - First milestone being created (no existing unpaid milestones)
   - OR new milestone's date is sooner than all existing milestones

**Trigger:** Back button (`<`) on milestone creation screen

**Actions:**
- Accept → save milestone, server overrides `Invoice.dueDate` + sets `termsDay = CUSTOM`
- Decline → discard milestone, return to Payment Schedule page

**No flag needed** — derivable from existing `termsDay` value and milestone count.

### Client-Side Guards (`invoice-info.screen.tsx`)

All guards use the same condition: **`hasUnpaidMilestones`** — derived from Realm using `getUpcomingPayments(invoice.remoteId).length > 0`.

**How `hasUnpaidMilestones` is passed:**
- **Call site:** `invoice.screen.tsx` → `onPressInvoiceInfo()` computes the flag via `getUpcomingPayments(invoice.remoteId).length > 0` and passes it as a nav param
- **Function:** `getUpcomingPayments` (`services/realm/entities/payments/repository.ts:32`) queries Realm for `date == nil && deleted == false` — same semantics as the server query
- **Synchronous:** Realm queries are sync, no async needed at the call site
- **Why not `termsDay === CUSTOM`:** CUSTOM can be set manually by the merchant (custom due date without milestones). Using the actual Payments data is the only accurate signal.

| Line | User action | Guard |
|------|-------------|-------|
| 188 | Changes invoice Date | `if (!hasUnpaidMilestones) { setFieldValue('invoiceDueDate', getDueDate(...)) }` |
| 229 | Picks custom Due Date | **Lock field** — disable interaction entirely |
| 264, 281 | Picks Terms | **Lock field** — disable interaction entirely |

**Why the Date guard matters:** Unlike Terms/DueDate which are locked, the invoice Date field stays editable (merchant needs to change when invoice was issued). But today, changing Date triggers `getDueDate()` which re-derives dueDate. When milestones manage dueDate, this re-derivation must be skipped.

**How `getDueDate()` works** (`is-mobile/src/services/documents-preview/invoice-due-date.ts`):
```typescript
export const getDueDate = (date: string, terms: number): string | undefined => {
  switch (terms) {
    case NONE:           return undefined;   // -2
    case DUE_ON_RECEIPT: return undefined;   // 0
    case CUSTOM:         return date;        // -1: returns invoiceDate as-is
    default:             return addDays(parseISO(date), terms); // Net N
  }
};
```
- CUSTOM returns the invoiceDate — but this only runs when merchant **explicitly changes** Date or Terms
- It does NOT run on every save. The Realm/local `dueDate` is preserved between interactions.
- On save, mobile just sends whatever `dueDate` is currently in local state (`is-mobile/src/services/parse/models/invoice.ts:59`)

**Key confirmation:** Parse Server's `beforeSave` hook does NOT re-derive dueDate from termsDay. Verified in `invoiceHooks.ts` — `beforeSave` touches `dueDate` nowhere. So our server-side writes to `dueDate` and `termsDay` will not be overwritten by the hook pipeline.

**All places that set dueDate on mobile (exhaustive audit):**

| File:Line | Trigger | Guard needed? |
|-----------|---------|---------------|
| `invoice-info.screen.tsx:188` | Merchant changes invoice Date | **Yes** — skip `getDueDate()` if milestones exist |
| `invoice-info.screen.tsx:229` | Merchant picks custom Due Date directly | **Yes** — lock/disable field |
| `invoice-info.screen.tsx:264` | Merchant picks Terms (picker modal) | **Yes** — lock/disable field |
| `invoice-info.screen.tsx:281` | Merchant picks Terms (action sheet) | **Yes** — lock/disable field |
| `invoice.screen.tsx:428` | Saving from info screen → writes to Realm | No — reads whatever was already set |
| `realm/entities/invoice/models.ts:206` | Creating brand new invoice | No — no milestones at creation |
| `realm/entities/invoice/repository.ts:420` | Duplicating invoice | No — duplicated invoice has no milestones |
| `stores/quick-invoice.store.ts` | AI quick invoice | No — edge case, no milestones |
| `parse/models/invoice.ts:217` | Syncing FROM Parse | No — reading server value (correct) |

All guards are on **one screen** (`invoice-info.screen.tsx`). Locking Terms + DueDate fields handles lines 229/264/281. The only code guard is on line 188 (Date change).

**Verified (2026-05-28):** All `getDueDate()` call sites on mobile audited — the 3 in `invoice-info.screen.tsx` are guarded, and the 2 in `repository.ts:420` and `models.ts:206` are invoice-creation paths where milestones can't exist. No gaps.

### Mobile (`is-mobile`)
- [ ] Lock dueDate field in invoice details when milestones exist
- [ ] Lock Terms field in invoice details when milestones exist
- [ ] Skip `getDueDate()` re-derivation on invoice Date change when milestones exist
- [ ] Warning modal on back button from milestone creation (conditions above)
- [ ] Show shortcut link on Invoice Details → navigates to Payment Scheduling when dueDate is auto-derived
- [ ] Toast on Payment Scheduling page (~3s) when a saved milestone becomes the invoice dueDate (first milestone, or sooner than all others)

### Web (`is-web-app`)
- [ ] Disable dueDate/Terms inputs in invoice editor when milestones exist
- [ ] Skip dueDate re-derivation on invoice date change when milestones exist
- [ ] Warning modal on milestone creation (same conditions as mobile)
- [ ] Show shortcut link → navigates to payment scheduling section when dueDate is auto-derived
- [ ] Toast/snackbar on Payment Scheduling when a saved milestone becomes the invoice dueDate

**Web re-derivation audit (2026-05-28):**

All re-derivation paths are in `InvoiceModel.ts` and `InvoiceTerms.tsx`:

| Method | File:Line | Guard needed? | Notes |
|--------|-----------|---------------|-------|
| `setInvoiceDate()` | `InvoiceModel.ts:525` | **No — already safe** | Condition is `termsDay > 0`; server sets `termsDay = CUSTOM (-1)` when milestones exist, so re-derivation never fires |
| `setTerms()` | `InvoiceModel.ts:536` | **Yes** — disable Terms `<Select>` | Called from `InvoiceTerms.tsx:116` |
| `setDueDate()` | `InvoiceModel.ts:559` | **Yes** — disable DueDate `<DatePickerWrapper>` | Called from `InvoiceTerms.tsx:140` |

Simpler than mobile — `setInvoiceDate` is already safe. Only need `isDisabled={hasUnpaidMilestones}` on the two inputs in `InvoiceTerms.tsx`.

---

## Edge Cases

| Scenario | `Invoice.dueDate` | `setting.termsDay` | UI state |
|----------|-------------------|--------------------|----------|
| No milestones (plain invoice) | Unchanged (client-derived) | Unchanged | Editable |
| First milestone added | → milestone's dueDate | → `-1` (CUSTOM) | Locked + warning modal |
| Multiple milestones | = soonest unpaid | `-1` (CUSTOM) | Locked |
| Milestone due date edited | Updates to new soonest | `-1` (CUSTOM) | Locked |
| Milestone marked paid | Shifts to next unpaid | `-1` (CUSTOM) | Locked |
| All milestones paid | → `null` (cleared) | → `0` (DUE_ON_RECEIPT) | **Unlocked** |
| All milestones deleted | → `null` (cleared) | → `0` (DUE_ON_RECEIPT) | **Unlocked** |
| Single milestone deleted | Re-derives from remaining | `-1` (CUSTOM) | Locked |
| Deposit milestone | Excluded (has `dueDate: null`) | — | — |
| Merchant changes invoice Date while milestones exist | No change (guard skips re-derivation) | No change | Date editable, dueDate locked |

---

## Milestone Lifecycle Timeline (from Seth's FigJam, May 27)

Example setup: Invoice with 3 milestones: M1=$100 (May 30), M2=$200 (Jun 30), M3=$300 (Jul 30).
`Invoice.dueDate` = May 30 (derived from M1, the soonest).

### Path A: Merchant marks whole invoice paid

```
May 30: M1 due
        Merchant marks entire invoice as paid → Invoice paid in full, done.
        No milestone logic fires further.
```

### Path B: Merchant marks individual milestone paid

```
May 30: M1 due
        Merchant marks M1 paid ($100)
        → recalculateInvoiceDueDate fires [1C hook]
        → query excludes M1 (now has date), finds M2
        → Invoice.dueDate shifts to Jun 30 (M2)
        → balanceDue drops by $100

Jun 30: M2 due
        Merchant marks M2 paid → dueDate shifts to Jul 30 (M3)
        ...repeats until balance is $0 (all milestones paid → dueDate = null, termsDay = DUE_ON_RECEIPT)
```

### Path C: Milestone NOT paid by due date

```
May 30: M1 due
May 31: M1 still unpaid
        → Nothing triggers (no hook fires on time passing)
        → Invoice.dueDate stays May 30
        → UI shows "overdue" (dueDate < today && balanceDue > 0) — purely calculated, no stored flag
        → If auto-reminders enabled: payment request sent to client on May 30
        → Invoice stays overdue until M1 is eventually marked paid,
          at which point dueDate shifts to Jun 30 (M2) via [1C hook]
```

**Key insight:** `Invoice.dueDate` only shifts when a milestone is *actively marked paid* — not when time passes. We don't "watch" milestones vs. current date. The "overdue" state is just the UI noticing `dueDate < today` with balance remaining.

---

## Milestone Delete: Soft Delete Path (verified)

Deleting a milestone is a **soft delete** — sets `payment.deleted = true` via the regular `updatePayment()` flow. This goes through our 1B hook.

**Sequence:**
1. Client calls `invoiceUpdatePayment` with `payment.deleted = true`
2. `updatePayment()` → `paymentObj.set('deleted', true)` → `parse.save(paymentObj)` (line 117/133 in `updatePayment.ts`)
3. `updatePaymentDetails()` runs (recalcs balanceDue)
4. `recalculateInvoiceDueDate()` runs [1B hook] — the Payment is already saved with `deleted: true` at this point
5. Query has `q.equalTo('deleted', false)` → deleted milestone excluded
6. If other unpaid milestones remain → dueDate shifts to next soonest, `termsDay` stays CUSTOM
7. If **no** unpaid milestones remain → `else` branch → `dueDate = null`, `termsDay = DUE_ON_RECEIPT`, UI unlocks

**No special handling needed** — delete flows through the same code path as edit. The query conditions naturally exclude deleted milestones.

---

## How New Invoices Inherit Terms (Context for Warning Modal)

When a new invoice is created on mobile, terms are copied from the previous invoice:

```typescript
// is-mobile/src/services/realm/entities/invoice/repository.ts
export async function duplicateInvoice(invoice: Invoice) {
  const invoiceDate = format(new Date(), 'yyyy-MM-dd');
  const invoiceTypeDueDate = getDueDate(
    invoiceDate,
    (setting && setting.termsDay) || InvoiceTermTypes.DUE_ON_RECEIPT,
  );
  const dueDate = docType === DOCTYPE_INVOICE && invoiceTypeDueDate ? invoiceTypeDueDate : null;
  return {
    ...newInvoice,
    dueDate,                                    // re-derived from termsDay + today's date
    setting: duplicateSetting(setting, docType), // termsDay copied as-is
  };
}

function duplicateSetting(setting, docType) {
  // copies termsDay, taxRate, color, template, etc. — no modification
  return { ...setting, termsDay };
}
```

**Implication for warning modal:** There is no stored signal distinguishing "merchant explicitly set terms on this invoice" from "inherited from previous invoice." The `termsDay` value looks identical in both cases.

**Decision:** Always show the warning if `termsDay === -1 || termsDay > 0` — even if inherited. Rationale: if it was their previous workflow, it's still worth alerting them that milestones will override it. Seth and Liz confirmed this approach (May 27).

---

## Design Rationale: Why Lock (from Juan's Review)

Juan flagged (May 26 DM) that unlike `balanceDue` (computed, not user-editable), `dueDate` is a user-editable field — auto-deriving it could silently overwrite the merchant's intent.

His alternative: compute dueDate on-the-fly via cron at email-send time, not on the Invoice object. This won't work because `Invoice.dueDate` is displayed across mobile, web, and PDF — it must be the persisted source of truth.

**Resolution:** "Always overwrite + lock" addresses both concerns:
- Milestones exist → dueDate is derived and the UI field is locked (no silent overwrites — merchant sees it's managed)
- Milestones removed → UI unlocks, merchant can edit freely
- Seth confirmed locking is acceptable (May 26)

---

## Trigger Audit: All Paths That Modify Invoice.dueDate

| # | Scenario | Trigger path | Hook | Invoice.dueDate updated? |
|---|----------|-------------|------|--------------------------|
| 1 | New milestone added | `invoiceAddPayment` → `!completedDate` branch | 1A | ✅ |
| 2 | Milestone dueDate edited | `invoiceUpdatePayment` → `updatePayment` | 1B | ✅ |
| 3 | Milestone manually marked paid | `invoiceUpdatePayment` → `markPaid` → `addPaymentToInvoice` | 1C | ✅ |
| 4 | Milestone deleted | `invoiceUpdatePayment` → `updatePayment` (sets deleted=true) | 1B | ✅ |
| 5 | **Auto-pay charges milestone** | Stripe webhook → must use `markPaid` path → `addPaymentToInvoice` | 1C | ✅ |
| 6 | Due date passes, unpaid | Nothing triggers — no shift needed | — | N/A (UI shows overdue) |

### Requirement for is-payments: auto-pay must use `markPaid` path

The upcoming payment execution handler (not yet built) **must** mark the existing milestone Payment as paid via `invoiceUpdatePayment` with `markPaid: true` — NOT create a new Payment via `invoiceAddPayment`.

**Why:** `recalculateInvoiceDueDate` queries for milestones with `doesNotExist('date')`. If auto-pay creates a separate paid Payment instead of setting `date` on the original milestone, the original milestone remains "unpaid" in the query and `Invoice.dueDate` never shifts.

**Correct flow:**
```
Upcoming payment execution Lambda
  → Stripe charges off-session PaymentIntent
  → Stripe webhook fires
  → payment-intent-updated.ts
  → must call Parse.Cloud.run('invoiceUpdatePayment', { markPaid: true, payment })
  → markPaidUpcomingPayment() → sets date on EXISTING milestone Payment
  → addPaymentToInvoice() → recalculateInvoiceDueDate() [1C]
  → Invoice.dueDate shifts to next unpaid milestone ✅
```

**Note:** Current RP webhook path uses `invoiceAddPayment` (creates new Payment). The upcoming payment handler must diverge here — use `invoiceUpdatePayment` with `markPaid: true` instead, to update the existing milestone record.

---

## Key Code References

### is-parse-server (this repo)
| File | Purpose |
|------|---------|
| `cloud/collections/invoice/utils/recalculateInvoiceDueDate.ts` | **NEW** — shared helper (this feature) |
| `cloud/collections/invoice/functions/invoiceAddPayment.ts` | 1A hook — add milestone |
| `cloud/collections/payment/updatePayment.ts` | 1B hook — edit/delete milestone |
| `cloud/collections/invoice/utils/addPaymentToInvoice.ts` | 1C hook — mark paid |
| `cloud/collections/invoice/invoiceValidation.ts:338-348` | `allowedTermsDays` array (what's valid for termsDay) |
| `cloud/collections/invoice/invoiceHooks.ts` | beforeSave — confirmed does NOT touch dueDate/termsDay |
| `cloud/collections/invoice/utils/updatePaymentDetails.ts` | Existing helper (recalcs balanceDue) |

### is-mobile
| File | Purpose |
|------|---------|
| `src/features/documents/screens/invoice-info.screen.tsx` | Invoice Details screen — where Terms/DueDate/Date are edited. Client-side guards go here. |
| `src/services/documents-preview/invoice-due-date.ts` | `getDueDate()` function — derives dueDate from invoiceDate + termsDay |
| `src/services/parse/models/invoice.ts:59,85` | Serializes `dueDate` (line 59) and `setting` incl. termsDay (line 85) to Parse on save |
| `src/services/realm/entities/invoice/repository.ts:416-436` | `duplicateInvoice()` — how new invoices inherit termsDay |
| `src/services/realm/entities/invoice/repository.ts:460` | `duplicateSetting()` — copies termsDay from previous invoice |

### Shared packages
| Package | Purpose |
|---------|---------|
| `@invoice-simple/common` | Exports `InvoiceTermTypes` enum: NONE=-2, CUSTOM=-1, DUE_ON_RECEIPT=0, NEXT_DAY=1, etc. |

---

## Slack Decision Threads

| Date | Thread | Decision |
|------|--------|----------|
| May 20-21 | [Seth's proposal](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779309054572939?thread_ts=1778699694.576969&cid=C0B331AP0BY) | dueDate = soonest milestone, one-way derivation |
| May 26 | [Juan's DM](https://ec-mobile-solutions.slack.com/archives/D08DUAP0J79/p1779830571913379) | Raised editability concern → resolved by locking |
| May 27 | [Seth + Liz flow confirmation](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779908089.296689) | Warning modal on back button, accepted/declined flow |
| May 27 | [Lenmor's modal condition](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1779910077.729349) | Show if termsDay > 0 or CUSTOM; Liz approved |

---

## Status & TS Error

**Server code status:** Implemented in is-parse-server on branch `spike-tie-milestone-changes-to-invoice-due-date`.

**Known TS error in `recalculateInvoiceDueDate.ts`:** `Argument of type 'typeof Payment' is not assignable to parameter of type 'string | (new (...args: any[]) => Object<Attributes>)'` and `.get()`/`.set()` not found on `parse.Invoice`. To fix: check how `updatePaymentDetails.ts` uses parse types and mirror that pattern (e.g., it may use `Parse.Object` directly instead of the typed wrappers).

**Updated code needed:** The current implementation only sets `dueDate`. It needs to be updated to also set `termsDay` (as shown in the updated code snippet in this doc). The `else` branch also needs to clear `dueDate` and reset `termsDay = DUE_ON_RECEIPT`.

---

## Feature Summary (2026-05-28)

**Current:** Invoice terms (`dueDate` and `setting.termsDay`) are set by the merchant directly via the Terms picker or custom date on mobile/web; milestones have their own `dueDate` field with no connection to the invoice-level terms.

**Proposed:** Invoice terms are automatically derived from the soonest unpaid milestone — `dueDate` reflects that milestone and `termsDay` is locked to `CUSTOM (-1)`; when all milestones are paid/deleted, terms reset to `DUE_ON_RECEIPT (0)` and the UI unlocks.

**Details:**
- New shared helper `recalculateInvoiceDueDate.ts` queries Payment collection for soonest unpaid milestone (`doesNotExist('date')`, `exists('dueDate')`, `equalTo('deleted', false)`)
- Sets `Invoice.dueDate` to that milestone's dueDate + `setting.termsDay = CUSTOM (-1)` to prevent client re-derivation
- When no unpaid milestones remain, clears `dueDate = null` and resets `setting.termsDay = DUE_ON_RECEIPT (0)`
- Called from 3 hook points: 1A (add milestone), 1B (edit/delete milestone), 1C (mark paid)
- Mobile/web UI locks Terms + DueDate fields when milestones exist, using `getUpcomingPayments(invoice.remoteId).length > 0` as the signal (not `termsDay === CUSTOM`, since CUSTOM can be set manually by the merchant)
- Auto-pay execution handler (is-payments, not yet built) **must** use `invoiceUpdatePayment` with `markPaid: true` to correctly advance the terms

**Confirm:**
- [ ] **PM:** Warning modal shows even for inherited terms (not explicitly set on this invoice) — Seth/Liz confirmed May 27, re-confirm if UX team raises concerns
- [ ] **Engineering (is-payments):** Auto-pay handler requirement is documented but not enforced — add integration test or code comment?
- [ ] **Risk — race condition:** Concurrent invoice save from mobile while a milestone is being edited — last-writer-wins on `dueDate`/`termsDay`. Low probability.
- [ ] **Risk — deposits:** Excluded by `q.exists('dueDate')` since deposits have `dueDate: null`. If deposits ever gain a dueDate, consider adding `q.notEqualTo('paymentType', 'deposit')` guard.
- [ ] **Unknown:** "All milestones deleted → DUE_ON_RECEIPT" loses the merchant's original terms (e.g., Net 30). Is this intentional? (Likely yes — milestones represent a deliberate workflow change.)
