# Where to store the estimate→invoice conversion pointer (`convertedTo`)

**Status:** Recommendation — leaning to a dedicated Mongo mapping collection (Juan's proposal). Not yet locked.
**Date:** 2026-07-17
**Session:** fc009aa1 (continuation)
**Drivers:** Juan (proposed the separate-collection idea + sync-complexity concern), Lenmor.

Related: `confluence-plan-2-proposed-edits.md` items 1/2/13/14/22, `session-handoffs.md`, memory `project_deposit_optimization_v1.md`.

---

## The question

The 1:1 conversion lock needs a durable record that estimate `E` was converted to invoice `I`. Items 1/2/13/14 assume this lives in an indexed `convertedTo` field on the document. Juan raised: given mobile sync complexity, is a **field on the synced document** the right home, or should this be a **dedicated store outside the sync pipeline**?

Three candidate homes:

1. **Field on the synced document** — `estimate.convertedTo` / `invoice.convertedFrom`
2. **`invoice.setting.*`** — inside the setting subdoc
3. **A dedicated Mongo collection** — a payments-agnostic mapping, outside Realm↔Parse sync

---

## Key finding that reframed the decision: mobile is pull-based, not reactive

Before comparing storage, we verified how is-mobile actually receives **server-originated** changes (agent investigation, session fc009aa1). This turned out to change which tradeoffs are real.

**is-mobile has no LiveQuery, no polling, and no reactive subscription on invoice/estimate documents.** Fresh server data lands only when something explicitly calls `syncStore.sync()` (`src/stores/Sync.store.ts:136`), which does a delta pull keyed on `updatedAt` (`src/services/sync/utils.ts:163`).

What triggers a sync of Invoice/Estimate from the server:

| Trigger | Syncs? | Evidence |
|---|---|---|
| App cold start | ✅ Yes | `src/stores/App.store.ts:83` (`init()`→`sync()` if authed) |
| App background→foreground | ❌ **No** | `src/features/core/hooks/use-app-state-listener.ts:18-26` — only `checkForUpdates()`+Intercom, **no sync** |
| Pull-to-refresh on a list | ❌ No such control | no `RefreshControl`/`onRefresh` in `invoice-list.screen.tsx` |
| Open invoice/estimate detail | ✅ Conditionally | `invoice.navigator.tsx:415-433` — only if opened by `remoteId` and not yet in Realm; if already local, **no fetch** |
| Timer / polling | ❌ No | none |
| Push notification (`SYNC` type) | ✅ Yes | `src/services/notifications/handlers.ts:139-154` → `sync(data.full)`; silent-push background sync `headless-fetch-task.ts:27` |
| LiveQuery real-time | ❌ No | not used anywhere |
| Local edit/save | ✅ Yes | save calls `sync()`/`syncFireAndForget()` (e.g. `invoice/repository.ts:64`) |

### The "invoice left open the whole time" scenario

If a user has an **invoice detail screen mounted** and the server flips the conversion pointer (e.g. checkout-triggered convert-first), **nothing updates the open screen in real time.** The detail container reads `this.invoice` as an in-memory reference set once at `componentDidMount` (`invoice.navigator.tsx:387,415`), and its only Realm listener is for **payments**, not the invoice/setting document (`invoice.navigator.tsx:422-425`).

So even a `SYNC` push that freshens Realm won't repaint the button on the open screen. **It takes a navigate-away-and-back** (re-runs `getInvoiceByRemoteId` in the container constructor) **or a save** to reflect a server-side conversion pointer.

### Why this reframes the storage decision

The two properties that intuitively favor "field on the document" — **"works offline for free"** and **"propagates faster"** — turn out to be properties of the mobile sync engine, **not** the storage location:

- **Propagation to an open screen is a wash across all three options.** A synced field still needs re-nav to repaint (the screen doesn't observe Realm for the invoice doc); a separate collection would need a deliberate focus-time network query (which doesn't exist today either). Neither gives silent in-place hide.
- **The offline button-hide doesn't depend on `convertedTo` at all.** "Does this estimate have a deposit" already lives on the synced `invoice.setting` (`depositType/depositRate/depositAmount`, `invoice-setting/schema.ts:42-44`), so the offline hide rides that flag regardless of where the conversion pointer lives.

Net effect: the propagation/offline advantages that used to prop up option 1 evaporate, leaving the storage decision to be made on **consistency, uniqueness, and coupling** — where the separate collection wins.

---

## Options with tradeoffs (post-verification)

### Option 1 — Field on the synced document (`estimate.convertedTo` / `invoice.convertedFrom`)

- ✅ Zero extra lookups — already in the doc being rendered.
- ➖ Offline: works, but **no longer an advantage** — the button-hide rides the deposit flag, not this field.
- ➖ Propagation to an open screen: **same as every option** (needs re-nav); a synced field does not repaint a mounted detail screen.
- ❌ Rides the Realm↔Parse sync pipeline: Realm schema bump, migration, backfill, and **conflict resolution** — last-write-wins can clobber or flap the pointer across two offline devices.
- ❌ No server-side uniqueness guarantee — a synced field cannot stop a double-convert; two offline clients can both "win."
- ❌ Reverse lookup (estimate→invoice) still needs an indexed collection query anyway, so it doesn't fully escape indexing.

### Option 2 — `invoice.setting.*`

- ❌ **Off the table**, for two concrete reasons:
  1. Inherits all of option 1's sync cost (pipeline, migration, conflict resolution).
  2. The convert flow **already strips** `depositType/depositRate/depositAmount` from `setting` during conversion (mobile `use-cases.ts:61,121`). Parking the conversion pointer in the exact subdoc the convert logic is actively mutating is the wrong home.
- ⚠️ `setting.*` is also flagged for deprecation elsewhere — worth a quick code verify before citing, but the sync + colocation arguments already kill it.

### Option 3 — Dedicated Mongo mapping collection (Juan's proposal) ⭐ recommended

- ✅ **Payments-agnostic** — estimate→invoice conversion is a core-domain concept, not a payments one. Keeping it in Mongo (not the payments Postgres DB) avoids coupling a core feature to payments infra and avoids a cross-store write on every conversion.
- ✅ **Server-authoritative + a unique index on `estimateId`** → a hard, idempotent double-convert guard that no synced field can offer. This is where the 1:1 lock actually lives.
- ✅ Index both directions (`estimateId`, `invoiceId`) → clean lookups either way.
- ✅ **Out of the sync pipeline entirely** — no Realm migration, no conflict handling.
- ➖ Offline: N/A, but **doesn't matter** — offline button-hide rides the deposit flag (the old "not available offline" ❌ downgrades to a non-issue).
- ➖ Propagation to an open screen: same as every option (needs re-nav) — not a strike against it specifically.
- ❌ **Only real remaining cost:** needs an atomic-ish write with the conversion. Mongo cross-collection writes aren't transactional without replica-set transactions, so define ordering + reconciliation for the crash-between-steps case (see open questions).

---

## Recommendation

**Go with option 3 (dedicated Mongo mapping collection).** Once the propagation/offline advantages of the field-on-doc approach are shown to be a wash, option 3's remaining merits stand alone: payments-agnostic, server-authoritative uniqueness guard, no sync migration/conflict handling. Its one real cost is the atomic-write/reconciliation work.

## Scope facts that hold regardless of storage choice

1. **The "Make Invoice" button is gated purely on `docType === ESTIMATE` today** (`invoice.screen.tsx:400`). There is no server-driven "already converted?" check anywhere. So converted/deposit gating is **net-new logic** in every option.
2. **Prompt hide on an already-open screen is missing for all options.** If we want it, that is separate work: make the detail container observe Realm for the invoice document (not just payments), or add a `useFocusEffect` re-fetch on the detail screen. Not required for v1 — v1 leans on the deposit-flag offline hide + re-nav (see item 22 in the plan).

## Open questions before locking

1. **Write ordering + reconciliation** — create invoice → write mapping. Define what happens if the mapping write fails (orphan invoice, no mapping): unique index on `estimateRemoteId` + a reconciliation sweep, or a replica-set transaction if the deployment supports it.
2. **Make `estimateRemoteId` a unique constraint, not just a lookup index** — that constraint *is* the double-convert guarantee.
3. **Confirm the `setting.*` deprecation status in code** so it can be stated as fact, not assumption.

---

## Update (2026-07-17, later same session): read-path cost rebalances the decision

Verified `findNotPayableReason` (the item-2 payment gate) and its callers (agent investigation, session fc009aa1).

**Finding:** the gate takes a slim `CheckoutData` DTO (`packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:42`; type `types.ts:86`, a 6-field projection), **not** the Mongo document — so `convertedTo` is not on what the function receives today regardless of storage choice. BUT every real caller already loads the **full `Invoice` document from Mongo with no projection** before calling it:
- Pay-button eligibility `is-checkout/src/eligibility/service.ts:38-42` (**hottest** — per buyer eligibility check / pay-button render, router `eligibility/router.ts:61`)
- is-unifiedxp `PaymentsSection` render `is-unifiedxp/src/utils/document-payable.ts:28` → `PaymentsSection.tsx:157` (**hot** — per buyer checkout page render)
- Checkout redirect `is-checkout/src/checkout/service.ts:65,110` (warm)
- `isInvoicePayable` abandoned-cart `is-abandoned-cart/src/utils/is-invoice-payable.ts:22`, scheduled Lambda `send-reminder/scheduled-handler.ts:43` (**batch loop** — N× multiplier)

**Consequence — first concrete point FAVORING field-on-doc:**
- **Field-on-doc:** `document.convertedTo` is already in memory at every call site (full doc, no projection) → thread it into `CheckoutData` + formatters → **zero extra queries.**
- **Separate collection:** requires an extra `findOne` on the mapping collection at each call site → lands on 2 hot buyer paths + a batch Lambda. Small next to the Stripe/PayPal status call already on those paths, but real and avoidable.

**Rebalanced tradeoff:** it's **read-path convenience (field) vs. write-path integrity + decoupling (collection)** — read cost on hot buyer paths, write integrity on a once-per-conversion action. Closer than the original recommendation implied; not settled.

### ⭐ Hybrid option (now the leading candidate)
Keep the authoritative **unique-indexed mapping collection** for the write-side double-convert guard AND **denormalize `convertedTo`** (or a boolean) onto the Invoice document for the hot read gate. Costs a little consistency discipline (two writes) but buys both the fast zero-query gate and the hard uniqueness constraint. Compatible with either "collection-only" or "collection + denormalized read field."

---

## Proposed collection name + schema (code-verified against is-mongo conventions)

Conventions confirmed (agent, session fc009aa1) in `packages/data-sources/is-mongo/src/collections/`: collections are **Parse Server-backed classes**, PascalCase names (`'Invoice'`, `'Payment'`), camelCase fields, **string** ids (not ObjectId), cross-refs via `remoteId` strings (not `_id`), account via `_p_account: "Account$<id>"` pointer string. **No index creation in-repo** — indexes are an infra/Parse action. Type pattern: `Domain & WithRemoteId & WithAccountId & WithId`.

**Server-only isolation comes free:** a new Parse class is NOT synced to mobile unless added to the mobile `syncTable` list (`Setting, Photo, Invoice, Item, Client, Msg, Expense, RecurringInvoiceSeries`). Don't add it → never touches Realm, no migration, no conflict handling. Add a comment noting it's intentionally sync-excluded.

**Proposed name:** `EstimateConversion` (alternatives: `DocumentConversion` if generalizing beyond estimates; `EstimateInvoiceLink`).

```ts
// packages/data-sources/is-mongo/src/collections/estimate-conversion.ts
import { collection, WithAccountId, WithId, WithRemoteId } from './collection';

export type EstimateConversion = {
  estimateRemoteId: string;   // source estimate's remoteId — UNIQUE (double-convert guard)
  invoiceRemoteId: string;    // resulting invoice's remoteId
  convertedAt: Date;          // when conversion committed (charge-time, convert-first)
} & WithRemoteId & WithAccountId & WithId;

export const getEstimateConversionCollection =
  collection<EstimateConversion>('EstimateConversion');
```

**Indexes (infra action — NOT in-repo, blocking, must exist in every env):**
1. `{ estimateRemoteId: 1 }` **UNIQUE** — the double-convert guarantee lives in `unique: true`, not the `1` (which is just ascending index direction). Without the unique flag it's just a lookup index and the guarantee evaporates.
2. `{ invoiceRemoteId: 1 }` non-unique — reverse lookup (items 3/22).
3. Optional `{ _p_account: 1 }` — per-account listing.

**In-repo files to change (mechanical):** add `'EstimateConversion'` to the `Collection` union (`collection.ts:3-13`); new `estimate-conversion.ts`; `export *` from `collections/index.ts`; optional `estimateRemoteId` filter helper in `filters.ts`. Do NOT add to mobile `syncTable`.

## Parse/Mongo implementation checklist (must-not-forget)

- **Dual-access contract:** written via Parse Cloud Function (masterKey + Parse SDK), but read on the hot path via **raw Mongo** (`is-checkout` is Mongo-native, bypasses Parse). Raw reads skip Parse CLP/ACL/`beforeFind` — Parse-side security does NOT protect the is-checkout read path. The `is-mongo` TS type is the only cross-boundary contract.
- **Parse auto-fields:** Parse always stamps `_created_at`/`_updated_at`/`_id`/`ACL` at DB level even though no mixin declares them. Explicit `convertedAt` is partially redundant with `_created_at` — keep it anyway (clearer, decoupled from Parse internals).
- **`_p_account` pointer format:** must be written as a real Parse pointer → serializes to `"Account$<id>"` (`filters.ts:17-21`, stripped by `transformers.ts:9`). Writing a bare id string breaks Parse-side queries. Use the helpers on both ends.
- **Duplicate-key handling = the idempotency (NOT optional):** a `beforeSave` check-then-insert cannot prevent a concurrent double-insert — only the DB unique index does. The cloud fn MUST catch the dup-key error (Mongo `E11000` / Parse error 137 `DUPLICATE_VALUE`) and treat it as "already converted → return existing mapping." Retry story (items 13b/14/18) leans on this.
- **Class-then-index ordering:** Parse lazily creates a class on first write, which can race with index creation on an empty class. Define the class/schema explicitly first → create unique index → then start writing.
- **Deletion / GDPR cascade:** decide what happens to the mapping on estimate/invoice delete and on account-delete/GDPR purge. Check whether the account-delete flow enumerates Parse classes explicitly (new class won't be purged unless added) or wipes generically by account pointer.
- **Env parity:** the unique index must be created in sandbox + staging + prod. Infra steps outside migrations are the classic "worked in staging, missing in prod" bug.
- **No soft-delete here:** other collections carry a `deleted` flag; conversions are append-only unless we decide they can be undone.
