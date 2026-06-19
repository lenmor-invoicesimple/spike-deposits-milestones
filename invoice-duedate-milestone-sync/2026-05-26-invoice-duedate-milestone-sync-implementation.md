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
export const recalculateInvoiceDueDate = async (
  invoice: parse.Invoice,
  { resetOnEmpty = true }: { resetOnEmpty?: boolean } = {},
): Promise<void> => {
  const q = new Parse.Query(parse.Payment);
  q.equalTo('invoiceRemoteId', invoice.get('remoteId'));
  q.equalTo('deleted', false);
  q.doesNotExist('date'); // no completion date = still unpaid/scheduled
  q.exists('dueDate');
  q.ascending('dueDate');
  q.limit(1);

  const [soonestMilestone] = await q.find({ useMasterKey: true });
  const setting = invoice.get('setting');

  if (soonestMilestone) {
    const dueDate = soonestMilestone.get('dueDate');
    invoice.set('dueDate', format(new Date(dueDate), 'yyyy-MM-dd'));
    setting.termsDay = InvoiceTermTypes.CUSTOM;
  } else if (resetOnEmpty) {
    // Check if milestones ever existed on this invoice (including deleted/paid ones)
    const historyQ = new Parse.Query(parse.Payment);
    historyQ.equalTo('invoiceRemoteId', invoice.get('remoteId'));
    historyQ.exists('dueDate');
    historyQ.limit(1);
    const hadMilestones = (await historyQ.count({ useMasterKey: true })) > 0;

    if (hadMilestones) {
      // All milestones deleted — reset to clean slate, unlock UI
      invoice.set('dueDate', null);
      setting.termsDay = InvoiceTermTypes.DUE_ON_RECEIPT;
    }
  }

  invoice.set('setting', setting);
};
```

**What it sets:**

| Condition | `Invoice.dueDate` | `setting.termsDay` |
|-----------|-------------------|--------------------|
| Unpaid milestone exists | Soonest milestone's dueDate | `-1` (CUSTOM) |
| No unpaid milestones (all paid/deleted) | `null` (cleared) | `0` (DUE_ON_RECEIPT) |

**`resetOnEmpty` parameter:**
- `true` (default — used by 1A/1B): when no unpaid milestones remain, resets to DUE_ON_RECEIPT and clears dueDate. Used on add/edit/delete paths where "nothing left" means milestones were removed.
- `false` (used by 1C): skips the reset entirely when all milestones are paid. All milestones paid = invoice complete; preserve original termsDay so the next invoice inherits the merchant's preferred terms. Only fires the history check + reset when `resetOnEmpty: true`.

**`hadMilestones` history check (inside `resetOnEmpty` branch):**
The `else if (resetOnEmpty)` branch doesn't unconditionally reset — it first checks whether any Payment with `exists('dueDate')` ever existed on this invoice (including deleted/paid ones). Without this guard, a plain invoice with no milestones would get reset to DUE_ON_RECEIPT on every `invoiceAddPayment` call for a non-milestone payment (e.g., a regular payment without a dueDate). The history query is a `count` with `limit(1)` — fast, index-friendly.

**Why `termsDay` must change too:**
- Mobile sends both `dueDate` and `setting.termsDay` on every invoice save
- If `termsDay` stays as `30` (Net 30), the client's `getDueDate()` could re-derive `dueDate = invoiceDate + 30` on the next save, overwriting the milestone-derived value
- Setting `termsDay = -1` (CUSTOM) means: "dueDate is set directly, don't compute from formula"
- Setting `termsDay = 0` (DUE_ON_RECEIPT) on "all milestones deleted" means: clean slate, no dueDate

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
- [ ] Show info banner on Invoice Details → navigates to Payment Scheduling when dueDate is milestone-derived (see below)
- [ ] Toast on Payment Scheduling page (~3s) when a saved milestone becomes the invoice dueDate (first milestone, or sooner than all others)

#### Info Banner on Invoice Details (mobile)

**What:** Green success banner below Due Date row: "Due date updated to match your soonest payment date" with chevron → navigates to Payment Scheduling screen.

**Complexity: Low (~1-2 hours)**

Existing building blocks:
- `<Banner variant="success">` component (`src/ui/banner/banner.tsx`) — supports `leftIcon`, `rightIcon`, colored background
- `check-circle` icon pattern already used elsewhere (green `#22C55E`)
- `NavigationRow` component with built-in chevron support (`src/ui/views/section.tsx:222`)
- Route `PaymentSchedulingScreen` already wired in `invoices.navigator.tsx:229`
- `hasUnpaidMilestones` flag already available on the screen (controls lock state)

**Implementation:**
1. Add banner conditionally after `renderInvoiceDueDate()` when `hasUnpaidMilestones === true`
2. On press → `navigation.navigate('PaymentSchedulingScreen', { invoice, onSubmit: this.onChangePayments })`
3. Deep-link to specific milestone (optional, Phase 2) — pass `scrollToMilestoneId` param, use `scrollToIndex` on list

**File:** `is-mobile/src/features/documents/screens/invoice-info.screen.tsx`

### Web (`is-web-app`)
- [ ] Disable dueDate/Terms inputs in invoice editor when milestones exist
- [ ] Skip dueDate re-derivation on invoice date change when milestones exist
- [ ] Warning modal on milestone creation (same conditions as mobile)
- [ ] Show info banner → navigates to payment scheduling section when dueDate is milestone-derived (same pattern as mobile)
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
| **Old client changes termsDay or invoiceDate** | Server guard corrects (see below) | Server guard corrects | Old UI has no lock |

---

## Backwards Compatibility: Server Guard in `beforeSaveInvoice`

**Problem:** Old mobile/web clients (deployed before the UI lock changes) can still:
1. Change `invoiceDate` → old client calls `getDueDate(newDate, CUSTOM)` → returns `null` → overwrites milestone-synced dueDate
2. Change `termsDay` → old client picks e.g. `DAYS_7` → re-derives `dueDate = invoiceDate + 7` → overwrites milestone-synced dueDate and termsDay

**Why this doesn't bounce:** Mobile only re-derives dueDate when user explicitly changes termsDay or invoiceDate (confirmed via audit of `invoice-info.screen.tsx`). On normal saves, it passes through whatever dueDate is in local state. So the server-corrected value persists locally until the user actively touches those fields.

**Fix:** Add a guard in `beforeSaveInvoice` (`invoiceHooks.ts`) — checks the *original* (pre-save) `termsDay`. If it was CUSTOM (meaning milestones own this invoice), re-run `recalculateInvoiceDueDate()` to overwrite whatever the client sent.

```typescript
// Backwards-compat guard: old clients (without the UI lock for terms/dueDate fields)
// can overwrite the milestone-synced dueDate when the user changes invoiceDate or termsDay.
// If unpaid milestones exist, re-run recalculate to ensure server always wins.
// Safe: no bounce because mobile only re-derives dueDate on explicit user interaction,
// not on every save — the corrected value persists locally until the next explicit change.
// Can be removed once old client versions are fully phased out (force-update threshold).
// Uses req?.original (pre-save snapshot) to check if termsDay WAS CUSTOM before this save —
// catches the case where an old client overwrites CUSTOM with a day-based value.
const originalTermsDay = req?.original?.get('setting')?.termsDay;
if (originalTermsDay === InvoiceTermTypes.CUSTOM) {
  await recalculateInvoiceDueDate(invoice);
}
```

**Why `req?.original` instead of current value:** If an old client overwrites `termsDay` from `CUSTOM` → `DAYS_7`, checking the *current* value would see `DAYS_7` and skip the guard. Checking the *original* catches exactly this case.

**Performance:** Only fires for invoices where `termsDay` was already CUSTOM (milestones exist). Non-milestone invoices skip entirely — zero extra queries.

**PR:** Committed to [is-parse-server#594](https://github.com/invoice-simple/is-parse-server/pull/594) (`spike-tie-milestone-changes-to-invoice-due-date` branch).

### Why a feature flag doesn't replace this

A feature flag could gate the **new UI lock** (roll out gradually to new clients), but it **doesn't protect against old clients** — they don't know about the flag and will keep sending stale dueDate/termsDay regardless.

The server guard is needed for the window where:
- Server has `recalculateInvoiceDueDate` deployed
- Old clients haven't updated yet (no UI lock, no flag check)

**Feature flag is optional on top** for gradual rollout of the new client UI lock. Server guard is the safety net regardless.

### Removal criteria

This guard can be removed once old client versions are fully phased out (force-update threshold). Document heavily on the PR branch.

---

## Musashi's Suggestion: Piggyback on `updatePaymentDetailsWithPayments`

**Context:** Musashi asked (Confluence comment Jun 3): "why don't we update dueDate the way paidAmount, balanceDue is updated? That way, no extra call or save is happening" — referencing `updatePaymentDetailsWithPayments` in `cloud/collections/invoice/utils/updatePaymentDetails.ts`.

### What `updatePaymentDetailsWithPayments` does

```typescript
// updatePaymentDetails.ts (private function, not exported directly)
const updatePaymentDetailsWithPayments = async ({ invoice }: { invoice: parse.Invoice }) => {
  const payments = await getPayments({ params: { invoiceId: invoice.id, sortOrder: 'desc', sortField: 'date' } });
  const commonPayments = payments.map(parsePaymentToCommonPayment);
  const universalInvoice = getUniversalInvoice({ invoice });
  invoice.set('balanceDue', +getInvoiceBalance(universalInvoice, commonPayments));
  invoice.set('paidAmount', +getPaidAmount({ payments: commonPayments }));
};
```

Two exported wrappers:
- `updatePaymentDetails(invoice)` — fetches fresh invoice, calls above, then `parse.save()`
- `updatePaymentDetailsForInvoiceHook(invoice)` — calls above, does NOT save (caller handles it)

### Verification: Same 3 hook points

Both `updatePaymentDetails` and `recalculateInvoiceDueDate` are called at the **exact same 3 locations**, right next to each other. Actual code as implemented:

```typescript
// 1A: invoiceAddPayment.ts
const updatedInvoice = await updatePaymentDetails({ invoice });
await recalculateInvoiceDueDate(updatedInvoice);
await parse.save(updatedInvoice);
return { invoice: updatedInvoice, paymentId };

// 1B: updatePayment.ts
await updatePaymentDetails({ invoice });
await recalculateInvoiceDueDate(invoice);  // resetOnEmpty: true (default)
await parse.save(invoice);

// 1C: addPaymentToInvoice.ts
await updatePaymentDetails({ invoice });
invoice.set('paidDate', ...); invoice.set('payments', ...); // existing fields
await recalculateInvoiceDueDate(invoice, { resetOnEmpty: false });
return parse.save(invoice);
```

No extra save is happening — `recalculateInvoiceDueDate()` only mutates the in-memory object, same as `updatePaymentDetailsWithPayments`. The `save()` call at the end persists both changes atomically.

### Could we merge them?

**Technically yes**, but with caveats:

1. **1C passes `{ resetOnEmpty: false }`** — merging would require a parameter on the combined function
2. **Separation of concerns** — "financial summary" (balance, paidAmount) vs "schedule sync" (dueDate) are conceptually different
3. **The 4th call site** — `recalculateInvoiceDueDate` is also called from `beforeSaveInvoice` (backwards-compat guard, line 91 of `invoiceHooks.ts`). That's NOT a payment operation and shouldn't recalculate balanceDue.

### Verdict

**What we did already satisfies Musashi's concern.** There's no extra save — both functions mutate the same invoice object before a single `save()`. The only difference from literally putting it inside `updatePaymentDetailsWithPayments` is one extra line at each call site. A combined wrapper is possible but cosmetic:

```typescript
// Possible refactor (optional, no behavior change):
export const updatePaymentDetailsAndDueDate = async (
  invoice: parse.Invoice,
  { resetOnEmpty = true }: { resetOnEmpty?: boolean } = {},
) => {
  await updatePaymentDetailsWithPayments({ invoice });
  await recalculateInvoiceDueDate(invoice, { resetOnEmpty });
};
```

**Decision:** Keep as-is for now (explicit at each call site). Can refactor to combined wrapper later if team prefers — no behavior change either way.

---

## Feature Flag Gating

Per Liz: all milestone-sync features must be gated behind a feature flag.

### Constraint: No Optimizely/Flagsmith SDK in is-parse-server

The team confirmed there's no way to add Optimizely SDK to is-parse-server for now. This means the server cannot independently check whether a feature flag is enabled for a given account.

### How clients currently call Parse cloud functions

**No feature flag state is ever passed today:**

| Client | Call pattern | FF context? |
|--------|-------------|-------------|
| Mobile | `Parse.Cloud.run('invoiceAddPayment', { invoiceId, payment })` | None — Optimizely/Flagsmith checked client-side only |
| Web | `Parse.Cloud.run('markPaymentsPaidOrUnpaid', { invoiceRemoteId, ... })` | None — Flagsmith checked client-side only |
| Checkout webhooks (is-bookkeeping) | `Parse.Cloud.run('invoiceAddPayment', params, { useMasterKey: true })` | None — no user session, no FF context |

### The webhook/auto-pay problem

When a payer completes checkout:
```
Stripe webhook → is-bookkeeping → Parse.Cloud.run('invoiceAddPayment', { useMasterKey: true })
→ afterSave hook fires → recalculateInvoiceDueDate()
```

There is **no client** in this flow. No one to check Optimizely. Same for future auto-pay (Phase 3). If the recalc is gated by a client-passed flag, these paths would never trigger it.

### Options evaluated

#### Option A: Client passes FF in params

```typescript
// Mobile/web would send:
Parse.Cloud.run('invoiceAddPayment', { invoiceId, payment, featureFlags: { milestone_sync: true } })

// Server checks:
if (params.featureFlags?.milestone_sync) await recalculateInvoiceDueDate(invoice);
```

| Pro | Con |
|-----|-----|
| Per-account gradual rollout | Webhook/auto-pay paths have no client to pass the flag |
| Client already has Optimizely/Flagsmith | Need fallback for masterKey paths: `if (flag || isMasterKey) recalc()` — but then webhooks always recalc regardless of flag |
| | Every cloud function signature needs updating |

**Problem:** If webhooks always recalc (because no client to check), then the flag only gates *client-triggered* milestone changes. A merchant could be "off" the flag but still get their dueDate synced when their client pays via checkout. Inconsistent.

#### Option B: ENV kill-switch on Parse + client FF for UI only (RECOMMENDED)

```
Server: MILESTONE_SYNC_ENABLED=true (ENV in Porter, kill-switch)
  → recalculateInvoiceDueDate always runs when ENV is on
  → all triggers (mobile, web, webhook, auto-pay) treated equally

Client: Optimizely/Flagsmith gates UI changes only
  → Lock fields, show banner, show warning modal
  → Gradual rollout of UI experience
```

| Pro | Con |
|-----|-----|
| Simple — no cross-layer FF coordination | Server is all-or-nothing (no per-account gradual rollout on server) |
| Webhook/auto-pay paths work identically | Edge case: merchant sees dueDate change "magically" before UI rolls out to them |
| Recalc is idempotent and harmless when no milestones exist | |
| Kill-switch available for emergencies | |

**Why the "magic dueDate" edge case is acceptable:** The recalc only fires when milestones exist AND a milestone is added/edited/deleted/paid. If the merchant isn't interacting with milestones, nothing changes. If they are, the new dueDate is *correct* — they just don't see the explanatory banner yet. Once the UI flag rolls out to them, they get the full experience.

#### Option C: Account-level flag in MongoDB

```typescript
// New field on Setting collection:
Setting.find({ accountId, remoteId: 'milestoneSyncEnabled', valBool: true })

// Server hooks check before recalcing:
const enabled = await isMilestoneSyncEnabled(invoice.get('accountId'));
if (enabled) await recalculateInvoiceDueDate(invoice);
```

| Pro | Con |
|-----|-----|
| Per-account gradual rollout on server | Requires schema addition + migration |
| Webhook paths can look it up | Extra query on every payment save (cacheable) |
| Consistent across all trigger paths | More complex — need to sync this flag with Optimizely/Flagsmith |

#### Option D: Dip's suggestion — new cloud function called by client

```typescript
// Client calls explicitly when FF is on:
Parse.Cloud.run('syncMilestoneDueDate', { invoiceId })
```

| Pro | Con |
|-----|-----|
| No changes to existing hooks | Webhook/auto-pay paths STILL need server-side recalc |
| Client has full FF control | Duplicates logic (client calls + hooks would both need recalc) |
| | Race condition: client calls recalc, then hook also runs |

### Decision: Option B (ENV kill-switch + client UI flag)

**Server layer:**
- `recalculateInvoiceDueDate()` gated by ENV: `MILESTONE_SYNC_ENABLED`
- All trigger paths (mobile, web, webhook, auto-pay) behave the same
- Kill-switch: set ENV to `false` in Porter → stops all recalculation
- beforeSave guard also gated by same ENV

**Client layer (gradual rollout):**
- Optimizely (mobile) / Flagsmith (web) flag: `milestone_due_date_sync`
- Gates: field locking, info banner, warning modal, skip getDueDate re-derivation
- Rollout: 0% → 10% → 50% → 100% over days/weeks

**Checkout/webhook layer:**
- No changes needed — recalc fires from existing hooks, gated by same ENV

### Changes needed per platform

| Platform | What | Lines to change | Effort |
|----------|------|----------------|--------|
| **Mobile** | Info banner (locking already works via `hasUnpaidMilestones`) | ~15-20 lines | Low (~2h) |
| **Web** | Disable Terms + DueDate fields, add banner, wire `hasUnpaidMilestones` prop through component tree | ~30-40 lines | Medium (~4h) |
| **Checkout/webhooks** | Nothing — doesn't touch dueDate, recalc is in the cloud function | 0 | None |
| **Parse server** | Add ENV check around recalc calls (3 hook points + beforeSave guard) | ~8 lines | Low (~30min) |

**Mobile detail:** `invoice-info.screen.tsx` already has the lock logic:
- Line 188-190: skips `getDueDate()` re-derivation when `hasUnpaidMilestones` — already done
- Line 204-213: DueDate field locked when `hasUnpaidMilestones` — already done
- Line 247-252: Terms field locked when `hasUnpaidMilestones` — already done
- Only missing: info banner (~15 lines of JSX)

**Web detail:** `InvoiceTerms.tsx` (lines 107-146):
- Add `isDisabled={hasUnpaidMilestones}` on Terms `<Select>` and DueDate `<DatePickerWrapper>`
- Wire `hasUnpaidMilestones` prop from parent (`InvoiceBody.tsx` or store)
- Add conditional info banner component

### Rollback risk: orphaned state

If ENV is flipped to `false` after being `true`:

1. Invoice already has `termsDay = CUSTOM`, `dueDate = milestone date` (written while enabled)
2. ENV off → server stops recalculating on milestone edits
3. Client FF off → UI unlocks Terms/DueDate fields
4. Merchant edits a milestone → dueDate **doesn't update** (stale)
5. Merchant changes Terms to Net 30 → client re-derives dueDate from invoiceDate → milestone link silently broken

No crash, no error — just **silent data divergence** between milestones and invoice dueDate. Low blast radius if rollout is incremental.

### Rollback mitigation

If ENV is disabled, run a manual script to fix affected invoices:
- Query invoices with `setting.termsDay = CUSTOM` that have unpaid milestones
- Re-run `recalculateInvoiceDueDate()` on each to re-sync, OR reset to `DUE_ON_RECEIPT` if sync is being permanently abandoned

Acceptable for Phase 1 — the "damage" from rollback is a stale dueDate, not money moving incorrectly. Script only needed if we actually roll back.

### Dip's concern: "if we turn off after releasing, dueDate updates would start having a wrong value"

Same as our rollback risk above. Mitigated by the manual script. Also, "wrong value" is really "stale value" — no incorrect money movement, just an out-of-sync dueDate that the merchant can manually fix by editing Terms.

### Dip's concern: "you can do all of this from client side"

**Our counter-argument for server-side:**
1. **Checkout webhook** — no client exists when payer pays via link; who recalculates?
2. **Auto-pay (Phase 3)** — is-payments Lambda triggers payment; no client
3. **Single source of truth** — prevents drift between mobile, web, and webhook paths
4. **Old clients** — can't update them; server guard catches stale writes regardless

Client-side recalc would work for the happy path but breaks for paths 1-2, which are critical for Phase 3.

### Dip's observation: "To create a new payment you need to be online"

**Confirmed.** This kills the offline edge case entirely. Milestone creation requires network (Parse cloud function call). We can remove offline sync concerns from our risk list. The Realm-to-Parse sync question is about *invoice saves* (which do queue offline), not payment creation.

---

## Realm-to-Parse Sync: Why Server-Owned dueDate Works

Mobile uses a bidirectional, event-driven sync (NOT real-time) between local Realm and Parse Server.

### Sync mechanism

| Direction | How | When triggered |
|-----------|-----|----------------|
| Push (Realm → Parse) | `Parse.Object.saveAll()` — sends **full Invoice** including dueDate | On save, after payment ops, user pull-to-refresh |
| Pull (Parse → Realm) | Queries objects where `updatedAt > lastSyncTime`, rewrites full Realm object | After each push cycle completes |
| Conflict resolution | Last-writer-wins by `updatedAt` timestamp — server usually wins | Automatic during sync |

No polling, no LiveQuery, no push notifications. Sync is triggered by user actions.

### Happy path for milestone-sync

```
1. User creates milestone on mobile
2. Mobile pushes Payment to Parse (cloud function call — requires network)
3. afterSave hook fires → recalculateInvoiceDueDate() → sets new dueDate on Invoice
4. Parse updates Invoice.updatedAt
5. Mobile pull phase: fetches Invoice (updatedAt changed) → overwrites local Realm with server dueDate
6. UI updates ✓
```

The push → pull happens in the **same sync cycle**. By the time the next push could occur, local Realm already has the correct server-derived dueDate.

### Why stale dueDate doesn't re-push incorrectly

The concern: "What if mobile pushes its old dueDate back to the server?"

- After step 5, local Realm has the server's dueDate. Next push sends the correct value.
- Even if timing is off and a stale push arrives, the **beforeSave guard** catches it: if `req.original.termsDay === CUSTOM`, it re-runs `recalculateInvoiceDueDate()` and overwrites whatever the client sent.
- Double safety: hook recalc + guard recalc both produce the same result (idempotent).

### Key files (mobile sync layer)

| Purpose | File |
|---------|------|
| Sync orchestration | `services/sync/utils.ts` — `performSync()`, `syncTable()` |
| Bidirectional conflict logic | `repository/Syncable.database.ts` — `updatedAt` comparison |
| Invoice Parse model (push) | `services/parse/models/invoice.ts` — `createParseInvoiceFromInvoice()` |
| Invoice Realm model (pull) | `services/realm/entities/invoice/models.ts` — `createInvoiceFromParseInvoice()` |
| Ensure-in-sync before critical ops | `services/sync/ensure-invoice-insync.ts` |

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

## Status

**Server code status:** Fully implemented in is-parse-server on branch `spike-tie-milestone-changes-to-invoice-due-date`. `yarn build` passes with zero TypeScript errors.

**What's implemented:**
- [x] `recalculateInvoiceDueDate.ts` — sets both `dueDate` and `setting.termsDay`, `resetOnEmpty` param, `hadMilestones` history guard
- [x] Hook 1A (`invoiceAddPayment.ts`) — recalc on new milestone
- [x] Hook 1B (`updatePayment.ts`) — recalc on edit/delete
- [x] Hook 1C (`addPaymentToInvoice.ts`) — recalc on mark-paid, `resetOnEmpty: false`
- [x] `beforeSaveInvoice` guard (`invoiceHooks.ts`) — backwards-compat for old clients

**Not yet implemented:**
- [ ] ENV kill-switch (`MILESTONE_SYNC_ENABLED`) — documented in "Feature Flag Gating" above but not yet added to the 4 call sites. Needed before production deploy.
- [ ] Mobile UI changes (lock fields, info banner, warning modal)
- [ ] Web UI changes (disable Terms/DueDate inputs, info banner)

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
