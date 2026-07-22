# Proposed edits — Confluence "Plan - Deposits Optimization 2"

**Source page:** https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1333690981 (v6, owner: Liz)
**Purpose:** section-by-section content to fill the skeleton page, grounded in the verified specs in this folder. **Do NOT paste blindly** — this is Liz's page; propose as inline comments / suggested edits, or hand her this doc.

> ⚠️ **DECISION FLIPPED (2026-07-15): DOCUMENT-LEVEL, not global.** Liz agreed to move to **document-level** payment/surcharge settings on estimates. This reverses [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md) (records RESOLVED = GLOBAL) and the `project_dm_architecture_pivot` memory — **both now stale** (flagged at the bottom). Everything below is written for the document-level path.

> **Legend:** ➕ add · ✏️ reword existing line · ⚠️ existing line needs correction · 📁 implementation reference (file:line).

---

## Walkthrough progress (item-by-item review with Lenmor)

Reviewing all ~24 work items one-by-one; each verified against code before presenting, then locked into the section below. **No `📁` prefix, plain text file:line as sub-bullets, no "Item N" cross-refs, plain wording ("based on", not "key off").**

**Done (locked into doc):**
1. ✅ `convertedTo` indexed field — Merchant Pre-conv → Eligibility
2. ✅ Server payment gate based on `convertedTo` — Merchant Pre-conv → Eligibility
3. ✅ `convertedTo` backfill — Merchant Pre-conv → Requirements — **SKIPPED for release (Liz OK)**; documented as known gap + feasible-if-wanted via `estimateId` reverse link
4. ✅ Retroactive `feesType` — Merchant Pre-conv → Requirements — **code fallback at conversion** (null → account default); backfill noted as alternative
5. ✅ Per-estimate surcharge UI (Mobile) — Merchant Pre-conv → Flow — essentially done (gate already widened, uncommitted); remaining = commit + tests + tracking-gate check
6. ⏭️ Per-estimate surcharge UI (Web) — Merchant Pre-conv → Flow — **SKIPPED for now** (revisit later)
7. ✅ Payment-toggle visibility — Merchant Pre-conv → Flow — **decision only**: keep visible as doc-level knob; no net-new code, mount folds into item 5
8-10. ✅ Email changes (bunched) — Merchant Pre-conv → Flow — **single edit in `email.ts` covers both send paths** (web v2 + mobile v3 share `send()`; v1 dead); new flag-selected template (precedented), subject-line docType fix (real bug), `deposit_due` field via `getDeposit().amount`

11. ✅ "Pay Deposit" button + on-page payment block — Buyer Pre-conv → Review Page — Medium–Large; shared invoice-viewer, relax docType-hardcoded QR (`useHidePaymentsQRCode.ts:10`) + balance (`InvoiceBalance.tsx:11`) gates, bespoke DEPOSIT-DUE rule, Seth top+bottom refactor; blocked on payability engine
12. ✅ Surcharge double-gate — Buyer Pre-conv → Checkout — **DECISION: relax gate 2 (honor per-doc) behind a FF**; XS code (2 lines × stripe+paypal) + one new Optimizely/Flagsmith flag; wrappers already in is-stripe, `getSurchargeCents` already has `accountId`

**Next:** 13. Convert-first server function (foundational) — Buyer Pre-conv → Parse

**Full 24-item list (heading → item):**
1. Eligibility → `convertedTo` indexed field ✅
2. Eligibility → server payment gate based on `convertedTo` ✅
3. Requirements → backfill `convertedTo` on already-converted estimates
4. Requirements → retroactive `feesType` (backfill vs null→account fallback)
5. Flow → per-estimate surcharge UI (Mobile)
6. Flow → per-estimate surcharge UI (Web)
7. Flow → payment-toggle visibility decision
8. Flow → email: new template vs `{% if %}`
9. Flow → email: subject-line bug fix
10. Flow → email: `deposit_due` amount field
11. Buyer Pre-conv Review Page → "Pay Deposit" button + on-page payment block
12. Buyer Pre-conv Checkout → surcharge double-gate decision
13. Buyer Pre-conv Parse → convert-first server fn (foundational)
13b. Buyer Pre-conv Checkout → provider call-site wiring (is-stripe + is-paypal) *(carved out)*
14. Buyer Pre-conv Parse → atomic conversion (mint + set-`convertedTo`)
15. Buyer Pre-conv Parse → conversion-failure error screen
15b. Buyer Pre-conv → convert/payment error matrix — "converted-but-unpaid" decision *(carved out)*
16. Buyer Pre-conv Parse → transform copies `feesType` verbatim (mobile+web+server)
17. Buyer Post-conv → success/failed labels (verify only)
18. Buyer Post-conv → payment-failure retry
19. Buyer Post-conv → old-link redirect
20. Buyer Post-conv → Pay-Now loading state
21. Buyer Post-conv → post-payment receipts (verify only)
22. Merchant Post-conv → hide manual "Make Invoice" after conversion (incl. sync-lag Layers A/B + deferred server guard — 24 folded in here)
23. Merchant Post-conv → "Duplicate & convert" combined action
24. ~~Merchant Post-conv → sync-lag handling~~ — COLLAPSED into 22 (it's the rationale for 22's Layer A/B design, not standalone work)

---

## §0 (top) — add a "Status / decisions locked" callout

➕
> **Decisions locked (2026-07-15)**
> - **Document-level settings — RESOLVED.** Estimates carry their own payment + surcharge settings (per-doc), copied to the invoice at conversion. Merchants get a per-estimate surcharge UI. *(Reversal of the earlier global decision — Liz agreed 2026-07-15.)*
> - **I2 (email) — NOT fully safe.** Body renders correctly by docType; the **subject line** mislabels a payable estimate as an "invoice" and needs a code fix.
> - **Hard release order.** convert-first + data backfills must ship **before** the payability flag flips on, or buyer payments break / already-converted estimates re-open.

---

## §"Investigation: global vs local payment toggle/surcharge" — collapse to a decision record

This section is a **decision record now, not a work list.** Keep the comparison table (it justifies the pick) and add a one-line verdict. **No work items here** — they've moved into the build sections below.

➕ Verdict line above the table:
> **Resolved → Document-level (Local), 2026-07-15.** The table is *why*; the work lives in the Merchant/Buyer sections below.

Table stays as-is with **two corrections** so it reflects reality:

- ⚠️ Row **"Checkout preview must match actual fee charged"** — Local cell *"no need; both sides read same per-doc value"* is now a genuine ✅ **win for Local**: the `/v/` disclaimer already reads the stored per-doc value, so no shared-invoice-viewer plumbing is needed (that was the *global* cost).
- ➕ Add a row we were missing — **old estimates have `feesType = null`**: Global would never read it (harmless); **Local** reads it → old estimates convert **fee-free** unless backfilled/fallback'd. (This is a *cost* of the chosen path — the work for it lives in Merchant Pre-conversion → Requirements.)

*(Full rationale + trace evidence: [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md) — pending a reversal note, see bottom.)*

---

## §"Merchant Pre-conversion" → Eligibility

- ✏️ "add `convertedTo` field on Estimate to track 1:1"
  - 📁 New **indexed** field added in **three** places that must stay in sync: Realm `invoice-setting/schema.ts` + Parse validation `invoiceValidation.ts` + `CheckoutData`. Remind to generate/run the migration + add the index.
  - ⚠️ **Net-new on mobile** — the field does not exist there today. The current "spent" signal is a boolean `estimate.setting.fullyPaid = true`, and it's set inconsistently: navigator path (`is-mobile/.../invoice.navigator.tsx:1058`) and list path (`invoice-list.component.tsx:382`) set it, but the FTUE convert modal (`convert-estimate-to-invoice-modal.tsx:46`) does **not**. If the lock is based on `convertedTo`, **all three** conversion paths must set it or the modal path leaves an unlocked estimate.
  - The **authoritative lock** is the server payment gate, which bases its check on the immutable `convertedTo` field, **not** `paymentSuppressed` (a merchant could flip that back).
    - 📁 Lock check belongs in `findNotPayableReason` (`is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:42-86`).
  - Phone-app copy of the lock is **best-effort UX only** (hide button / block obvious double-tap on a synced device); it cannot guarantee uniqueness.
    - 📁 Why: mobile conversion is **local-first** — the invoice ID is minted on-device (`uuid()` at `is-mobile/.../invoice/models.ts:221`), written to Realm, then pushed via **fire-and-forget** sync (`repository.ts:46-67` → `Sync.store.ts:94-99`). No server round-trip asks "may I convert?"
  - 🔴 **Merchant-offline duplicate-invoice edge (documented; approach TBD).** The deposit feature is online-only, but the *generic* "Make Invoice" action is a separate pre-existing offline path. A merchant whose phone hasn't synced can convert/mark-paid an estimate **after** a buyer already paid+converted online → the phone mints a **second** invoice and pushes it up → Parse ends with two invoices for one estimate.
    - **Financial risk = zero** (server payment gate already prevents double-charge / re-pay, offline or not). **Only residual = a stray duplicate invoice** — data-hygiene, manually fixable.
    - **Recommended low-effort fix (discuss later):** a **server-side ingestion guard** in the invoice-write path — reject/no-op a create whose `estimateId` points to an estimate whose `convertedTo` is already set to a *different* invoice id. Must be idempotent (same device re-syncing the same invoice must still succeed; key on "points to a **different** invoice").
    - **Full solution space (for the later discussion — no decision yet):**
      - *Server-side — prevent the bad state:*
        - **Ingestion guard** (recommended above).
        - **Unique constraint** — partial DB uniqueness index on `estimateId` (where set) so the 2nd insert fails at the data layer, whichever client sent it.
        - **Server-authoritative conversion endpoint** — phone calls a convert endpoint that allocates the invoice id + sets `convertedTo` atomically; kills the race but removes offline conversion (bigger change).
        - **Reconciliation job** — accept the dup on write, then a scheduled/triggered job merges/voids the loser.
      - *Client-side — reduce the window (UX layer, always worth doing; does NOT close the hole):*
        - **Hide-on-sync** — when the phone syncs down the now-set `convertedTo`, hide "Make Invoice"/"Mark Paid" on that estimate. Catches the common case (reconnect → sync → then act). Does **not** catch the merchant acting *while still unsynced* — that gap is where the dup is born. Needs a server guard for correctness.
        - **Force sync before convert** — require a successful sync-down immediately before allowing conversion; block when offline. Shrinks but doesn't eliminate the window.
        - **Online-only Make-Invoice for payable-deposit estimates** — disable the generic convert action when the estimate is a payable-deposit estimate and the device is offline.
      - *Detect & clean up — accept the dup, fix after:*
        - **Auto-void the loser on sync** — server detects the late duplicate, voids it, repoints merchant to the real invoice.
        - **Admin/support cleanup** — surface duplicates in an admin tool for manual merge/void.
        - **Pure accept-risk** — do nothing structural; rely on the server payment gate (already blocks double-charge) and treat the stray invoice as tolerable.
      - *Conflict-resolution / merge semantics:*
        - **Deterministic winner rule** — if two invoices share an `estimateId`, define the canonical one (the one `convertedTo` points to, or earliest server-received) and mark others superseded.
        - **Deterministic invoice id** — derive the invoice id from the estimate id instead of random `uuid()`, so two offline converts collide on the *same* id and `UpdateMode.All` merges them into one record. Attacks the root cause cheaply; edge case = the two invoices diverged in content.
    - **Zero-effort fallback:** hide button once synced + accept the rare dup + clean up manually.

- ➕ **Server payment gate based on `convertedTo`** (the correctness half of the lock; verified against `invoice-payments-status.ts`)
  - 📁 Gate lives in `findNotPayableReason` — `is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:42-86`. Every payment path (buyer checkout + any merchant-triggered payment) routes through it → this is why buyer estimate→review→checkout is airtight regardless of sync lag.
  - Base the check on the immutable `convertedTo` field, **not** `paymentSuppressed` (merchant-toggleable → could re-open a spent estimate).
  - ⚠️ **Hidden dependency — `convertedTo` is not in `CheckoutData` yet.** The fn destructures only `{ documentType, deleted, totalCents, balanceDueCents }` (`:56`). Adding the gate requires (a) adding `convertedTo` to `CheckoutData` + its populator, and (b) a new `GeneralNotPayableReason` enum value (e.g. `alreadyConverted`). Confirms this shares a story with the `convertedTo` field item above.
  - ⚠️ **Collides with the doctype check at `:58`** — `if (documentType !== 0) return docTypeNotInvoice` rejects *any estimate* today. This is the exact line the payability engine relaxes. Under **convert-first**, the buyer's payment converts estimate→invoice *before* charge, so at charge time `documentType === 0` and it passes. But the **same fn is used both to display "can this be paid?" on the pre-conversion estimate AND to gate the actual charge** — those two callers now need different answers. (This is the "two systems must agree on 'is payable?'" tension, landing on `:58`.)
  - ⚠️ **`balanceDueZero` (`:70`) already blocks *fully*-paid converts but misses partial deposits.** A fully-paid converted estimate has `balanceDue <= 0` → already returns `balanceDueZero`. But a *partially*-paid deposit leaves `balanceDue > 0` → still looks payable. **`convertedTo` is the check that covers the partial-deposit case `balanceDueZero` misses** (this is why backfill is required, not optional).
  - ⚠️ **Ordering/enum affects the buyer error message.** Put the `convertedTo` rejection **early** (right after the `!checkoutData` guard, before `totalZero`/`balanceDueZero`) so a converted estimate returns a clear `alreadyConverted` reason instead of an incidental `balanceDueZero` ("balance is zero" would be a misleading message).

---

## §"Merchant Pre-conversion" → Requirements

*(Absorbs the retroactive-payability + `feesType` backfill work — this is its real home.)*

- ⚠️ "Already converted — **need to investigate whether we need to backfill** `convertedTo` … `fullyPaid: true` … doesn't seem reliable"
  - ✅ **Decision (Liz): backfill is SKIPPED for release.** Documented here as a known gap, not a release gate.
  - **Why a backfill would otherwise be needed:** old converted estimates have no `convertedTo`. Once the payability flag flips on and the doctype gate is relaxed, those old estimates would look payable again. A backfill would set `convertedTo` on them so the server gate keeps rejecting them.
  - **The old reasoning was wrong** — this is NOT about `fullyPaid`→`balanceDueZero`. `balanceDueZero` is computed from totals-minus-payments, not derived from `fullyPaid`, and the payable check rejects estimates at the doctype gate before it ever reaches the balance check.
    - doctype gate: `is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58`
    - balance check: `invoice-payments-status.ts:70`
  - **A backfill IS feasible if we later want it** — a reverse link exists: the converted invoice stores the source estimate's id in `setting.estimateId`, so we can find which invoice an old estimate became and write it back.
    - reverse-link field: `is-parse-server/cloud/collections/invoice/invoiceValidation.ts:359` (stored as the estimate's `remoteId` UUID); set at conversion `is-mobile/src/services/realm/entities/invoice/use-cases.ts:59`, `is-web-app/nextjs/app/.../actions/convert-document.action.tsx:94`
    - would run as a standalone Mongo script in `is-parse-server/scripts/`, modeled on `fixDeletedTransactionId.ts` (batched `bulkWrite`)
  - **Residual gap even if backfilled:** `estimateId` is itself relatively new, so conversions older than that field can't be reconstructed. `fullyPaid` alone can't substitute — it's overloaded (estimate=closed / invoice=paid) and never names the target invoice.

- ➕ **Retroactive `feesType` (new, because document-level):**
  - Old estimates all have **`feesType = null`** (surcharge never existed on estimates). Preview and charge already agree on null — both read the stored per-doc value, so a null estimate previews fee-free **and** charges fee-free. No mismatch bug.
    - charge gate 1 `is-stripe/.../passing-fees/stripe-fees.ts:38` (null short-circuits to 0), PayPal mirror `is-paypal/.../passing-fees/paypal-fees.ts:37`
    - preview reads the already-computed value: `is-checkout/.../checkout/service.ts:64`, `.../eligibility/service.ts:61`
  - Conversion today re-supplies `feesType` from the account default instead of copying the estimate's value.
    - mobile strips + re-defaults: `is-mobile/.../invoice/use-cases.ts:55` → `invoice-setting/repository.ts:97`
    - web overrides: `is-web-app/nextjs/.../document-transformation-utils.ts:106`, fed by `convert-document.action.tsx:96`
  - Once the transform is changed to copy `feesType` verbatim (needed for document-level), an old null estimate converts fee-free — "copy verbatim" and "old estimates still surcharge" can't both hold without handling null explicitly.
  - ✅ **Decision: code fallback at conversion.** If the estimate's `feesType` is null, stamp the account default; otherwise copy verbatim. Small branch in the transform, applied only to the null back-catalog.
  - **Alternative (not chosen):** backfill the account's `feesType` onto every old null estimate (a real Mongo data migration), after which verbatim copy works uniformly with no null branch.

---

## §"Merchant Pre-conversion" → Flow

*(Absorbs the surcharge UI work — the merchant sets surcharge on the estimate here.)*

- ➕ **Per-estimate surcharge UI — Mobile (essentially done):**
  - The surcharge section now allows estimates — one docType gate widened, already made (uncommitted in the working tree).
    - gate: `is-mobile/src/features/payments/payments-option/components/invoice-payments-passing-fees-section.tsx:88` — now `docType !== DOCTYPE_INVOICE && docType !== DOCTYPE_ESTIMATE`
    - matching gate in awareness hook: `use-track-surcharge-awareness.ts:23`
    - `git status` confirms exactly these two files dirty (2 insertions / 2 deletions), nothing else
  - Mount point never blocked estimates — same screen renders both doc types, mounts the section with no docType condition (only flag + not-recurring):
    - `is-mobile/src/features/documents/screens/invoice.screen.tsx:330` and `:363`
  - Leaf components docType-agnostic, render as-is: `configure-surcharge.tsx`, `add-surcharge-base.tsx`
  - Baseline removal it reverses was trivial — commit `a7a0de18e` (PAYC-315, 2025-04-29) was a single `&&`→`||` gate flip + test updates, no feature/data/backend code deleted; also edited a now-obsolete file path (live component moved to `src/features/...`).
  - Remaining to ship:
    - commit the two staged gate edits
    - update the unit tests the removal commit touched (they assert estimates hide the section)
    - awareness hook now fires surcharge-tracking on estimates too — confirm intended, or gate the tracking separately
  - 📁 **Web — medium.** `WidgetPayments` gated at the mount site, not just the leaf: `settings-sidebar-migrated.tsx:249`, `settings-sidebar.tsx:168`. Opening it exposes the full pay-online widget, so render only `<SurchargeFees>` for estimates. Leaf gates: `surcharge-fees.tsx:205`, `payment-markup-fees.tsx:37`.
- ➕ **Payment toggle (decision only — no net-new code):**
  - Keep the per-estimate payment toggle **visible** as the doc-level on/off knob. The page's "hide local Estimate payment toggle" note is stale (belongs to the abandoned global plan).
  - Toggle + per-doc fields already exist and are wired on both platforms; new docs default to on, so estimates are payable by default.
    - `paymentSuppressed` gates PayPal, `stripePaymentSuppressed` gates Stripe: `is-services/packages/payments/payments-status/src/utils/SDKs/sdks.ts:50` / `:106`
  - Toggle is NOT the re-payment lock (merchant can flip it back on) — the lock stays on immutable `convertedTo`.
  - Only work = widen the payments-section mount to estimates (`is-mobile/.../invoice.screen.tsx:310`); folds into the surcharge-UI mount change above.

**Email changes (three sub-items — land in BOTH the v2 and v3 send paths):**

- **Two live send paths, not one — both must get the change (verified: who calls what today):**
  - **Web → v2** (`POST /api/v2/email-invoice`, both invoices + estimates via the shared modal). Server-side fetch: `api/src/services/email.ts` `getEmailTemplateData:574`, invoice `:563`, fetched `:229`. Web call site: `is-web-app/nextjs/app/services/email/send-invoice-email.ts:73`.
  - **Mobile → v3** (`POST /api/v3/app/email`, estimate vs invoice is just a `docType` field on the same call). Server-side fetch too (same model as v2, NOT the v1 form-fields model): handler `AppService.email` `api/src/services/app.ts:123`, fetch `:168`. Mobile call site: `is-mobile/src/services/is-api/invoke.ts:31` + `endpoints/share.ts:18`.
  - **v1 is effectively dead in-repo** — no caller in mobile, web, is-services, or admin builds a request to `/api-im/v1/email-invoice`; it's labeled "legacy ios/android." Skip it. (Only unclosable gap: old app binaries in the field could still hit v1 — not visible from source, would need server logs. Low priority given no in-repo caller.)
  - ⚠️ **Scope both v2 and v3**, or mobile users get no deposit line + the wrong subject. Earlier "v2-only" framing was wrong — mobile is on v3.

- ✅ **One edit covers both send paths.** v2 and v3 both call the same shared `send()` → `getEmailTemplateData()` / `getSubject()` in `api/src/services/email.ts` (v2 handler `invoice-sending.ts:480`, v3 handler `app.ts:296`). So the three changes below are single-site edits in `email.ts` — no per-path duplication.

- **New template vs `{% if %}` (restyle + DEPOSIT DUE line):**
  - Prefer a **new flag-selected SWU template** over `{% if %}`. Flag-selected template selection already exists for payable invoices in the same function, so adding a payable-estimate template is a small, well-precedented branch — not new machinery.
  - Today the estimate branch short-circuits to a single hardcoded template and never reaches the deposit/payments template path: `email.ts:656-657` (`docType === ESTIMATE → swuEstimateTemplateV2`).
  - Invoice precedent to mirror (payments-eligible template selection): `email.ts:659-666` (`getPaymentsSWUTemplateId`); and the flag-gated `getInvoiceTemplate` at `email.ts:302-303`, applied `:694` (`templateId && docType === INVOICE` — INVOICE-only today, would need to admit estimates).
  - Template choice does not change the `deposit_due` scope — that field work is required either way; a new template just adds the flag-selection branch.

- 🔴 **Subject-line "Estimate" fix (real bug):**
  - Body already safe — the template branch is keyed on docType (`email.ts:656`), so the estimate body renders as an estimate.
  - Subject line NOT safe — the A/B subject override hardcodes "…sent you an invoice" and fires on **payability/eligibility, not docType**. Once estimates become payable/eligible → "invoice" subject over an "Estimate" body.
    - `api/src/util/email-experiment-service.ts:79` (`getSubject`); activated inside the shared path at `email.ts:290-291` (gated on `isEligible` + `ir_subject_email` flag).
  - Fix in **code, not the SWU template** — make `getSubject()` docType-aware, default "…sent you an estimate", confirm via prototype. One edit, both paths.

- ➕ **`deposit_due` amount field:**
  - No email shows a deposit dollar figure today. Add a `deposit_due` field to the payload — again a single edit in `email.ts`, inherited by both v2 and v3.
    - add to `SwuEmailTemplateData` (`email.ts:106`, near `balance_due` `:127`) + conditional assignment next to `email.ts:758`
    - value from `getDeposit(invoice, payments).amount` (`is-packages/.../getDeposit.ts:163`, docType-agnostic, not yet called by any email builder), format via existing `formatAmount` (`email.ts:632`)
    - companion `is_deposit_eligible` boolean already flows through (`email.ts:138`, set `:761`) — natural pairing for the line. Note the estimate branch (`:656`) short-circuits before `isEligibleForDeposit` is ever set, so estimates need that eligibility wired in too.

---

## §"Buyer Pre-conversion" → Implementation - public Review Page — fill in ("revisions on Estimate")

➕
- Adds **"Pay Deposit"** button (label by docType) + on-page payment block for estimates: **DEPOSIT DUE** line, **QR code**, **fee disclaimer** — all invoice-only today.
  - 📁 On-page document = **invoice-viewer package**, shared web + mobile. Disclaimer/surcharge gate `PaymentInstructions.tsx:60-73` (`hideSurcharge` `:71`), amount region `:196-212`.
  - ✅ Under document-level the disclaimer **already reads the correct per-doc value** — no cross-platform plumbing needed (this was the global-path cost).
- **Post-payment cleanup mostly automatic** (driven by the server "can this be paid?" check), **except the DEPOSIT DUE line** — needs an explicit rule tied to payability.
- **Shared-component blast radius:** changes → package release + mobile retest. Size Medium → Medium–Large.
- **Seth's layout:** buttons top + bottom; small refactor so buttons (and signature pop-up) don't render twice. (Reviewed clean.)
- ⚠️ **Blocked on the payability engine** — the "can this be paid?" server check is the same gate the engine relaxes (`invoice-payments-status.ts:58`). Not parallel work.

---

## §"Buyer Pre-conversion" → Implementation - checkout — fill in ("…")

*(Absorbs the double-gate decision — charge-time behavior.)*

➕
- **Inherited for free from the invoice deposit flow (verified on staging):** actual checkout + "payment successful" (incl. "Deposit Paid" + recalculated balance) needs no new work. Risk is in *pre-payment* surfaces.
- 📁 Checkout page target = is-unifiedxp (`PaymentsSection.tsx:234`), reached via `payUrl`.
- ✅ **Surcharge double-gate — DECISION: relax gate 2 (honor per-doc value), behind a feature flag.** Today the charge ANDs the per-doc value with the **account** value:
  - `is-stripe/.../passing-fees/stripe-fees.ts` — `getSurchargeCents` (`:32`): gate 1 per-doc `universalInvoice.setting.feesType===SURCHARGE` (`:38`) **AND** gate 2 account `payment.feesType===SURCHARGE` (`:45`); returns `0` otherwise. Mirror: `is-paypal/packages/server/src/services/passing-fees/paypal-fees.ts:37`+`:43`.
  - **Why relax:** the on-page disclaimer reads the per-doc value (`PaymentInstructions.tsx:71`); if gate 2 vetoes but the disclaimer showed a fee → buyer sees fee, gets charged none. Deferring to per-doc keeps display=charge. (Whole item is moot under the old global model — exists only because we went document-level.)
  - **FF-gated rollout:** wrap the gate-2 relaxation in a flag so we can fall back to the AND if it misbehaves in prod. Mechanism already exists in-service and `getSurchargeCents` already receives `accountId` (the flag key) — cheap:
    - Optimizely wrapper `is-stripe/src/utils/optimizely.ts` (`isFeatureEnabled(userId, '<flag>', attrs)`) or Flagsmith wrapper `is-stripe/src/utils/flagsmith.ts`. PayPal side has its own flag utils to mirror.
    - flag ON → charge honors per-doc `feesType` alone; flag OFF → current AND. Needs a new Optimizely/Flagsmith flag created.
  - **Size:** XS code (2-line gate change × 2 providers + flag check), plus one new flag.
- **Two systems must agree on "is payable?"** — client display vs server checkout gate use different code; must share one rule or you get a QR code with no working link.

---

## §"Buyer Pre-conversion" → Implementation - Parse — expand (2 bullets, good)

*(Absorbs the conversion-transform copy work — conversion lives here.)*

Existing bullets correct. ➕ Add:
- **convert-first is our team's work** (foundational, epic owner), not external. Release gate — flag on without it → **every buyer payment fails** (payment gate still checks "is this an invoice?" at charge time).
- **(Wiring is its own item — see "13b — Provider checkout wiring" below.)**
- Conversion must be **atomic:** mint-invoice + set-`convertedTo` commit together (block A), or partial failure leaves an unlocked/orphaned state.
- **Conversion-failure error screen:** convert-first fails *before* charge → recoverable error (buyer not charged), not "payment failed."
- ➕ **Transform must copy `feesType` verbatim (document-level).** Today both active clients re-supply from the account default — that must change:
  - 📁 **Mobile** `makeInvoiceFromEstimate` (`is-mobile/src/services/realm/entities/invoice/use-cases.ts:45-62`): today drops `feesType` — it's in the `omit(...)` list at `:54`, so `defaultInvoiceSetting` (`:47`) supplies the account default → **stop omitting `feesType` so the estimate's value survives the spread.**
  - 📁 **Web (nextjs)** `transformDocumentSettings` (`document-transformation-utils.ts:64-114`, convert branch `:101-111`): today overrides at `:106` (`feesType: options.defaultFeesType`, fed by `convert-document.action.tsx:96`) → **remove the override** so the spread's verbatim value survives.
  - 📁 `feeRate` + `paymentSuppressed` already copy verbatim — no change.
  - 📁 **convert-first server fn** must likewise copy `feesType` verbatim (net-new).

---

## §"Buyer Pre-conversion" → Implementation - checkout wiring (13b — provider call-site) — NEW ITEM

*(Separate from 13: this is the CALLER side — invoking the convert-first cloud fn from the payment providers. Lives in is-stripe + is-paypal, not the Parse fn itself.)*

➕
- **Call the convert-first cloud fn at the top of the PaymentIntent/order handler, before the docType gate** — synchronous, pre-charge.
  - Stripe: `is-stripe/src/routes/public/payment-intent-create.ts:56-73`, before `validateAccountAndDocument` (`:66`).
  - PayPal: `is-paypal/packages/server/src/services/paypal-create-order.ts:81`, before `validateDocumentPayable` (`:132`).
- **Wiring cost is near-zero — the pattern already exists.** is-stripe/is-paypal already call Parse cloud fns with the SDK + masterKey (`is-stripe/src/utils/parse/setup.ts:9`; `Parse.Cloud.run('publicInvoice'…)` `public-invoice.ts:34`, `Parse.Cloud.run('invoiceAddPayment'…)` `:164`). New `Parse.Cloud.run('convertEstimateToInvoice', …)` sits right alongside.
- ⚠️ **Two-provider duplication:** must be added to BOTH the Stripe and PayPal handlers, and both must charge against the freshly-minted invoice id the fn returns (not the estimate id).
- ⚠️ **Ordering:** convert must complete before `validateDocumentPayable`/`validateAccountAndDocument`, or the gate rejects the still-estimate doc.
- **is-checkout is the WRONG place** — it never touches Parse (reads doc from Mongo `is-checkout/src/document/get-document.ts:9` + provider status over HTTP). Hooking here needs net-new Parse wiring or a new HTTP endpoint.
- **NOT a webhook** — webhooks are post-charge, they write the payment back via `invoiceAddPayment` (`is-stripe/src/services/webhook/payment-intent-updated.ts:147`). Convert-first is pre-charge.
- **Size:** S–M — no new wiring, but two call sites + return-value plumbing (charge the new invoice id) + ordering/idempotency (a retry after a successful convert must not convert again — pairs with the `convertedTo` lock in item 14).

---

## §"Buyer Pre-conversion" → Error handling matrix (15b — convert/payment failure) — NEW ITEM

*(Convert-first splits the buyer action into two commits that fail independently. Failure semantics differ on each side of the conversion. Today only the two ENDS are handled — the middle "converted-but-unpaid" state is unowned.)*

The chain: **Pay Now → [convert] → [create PaymentIntent] → confirm/charge → [webhook] → success**

➕
- **Stage 1 — convert fails (pre-charge, synchronous):** nothing charged, no PaymentIntent. **Safe failure** → recoverable "couldn't start payment, try again", NOT "payment failed" (= item 15). Convert must be find-or-create (idempotent) so a retry after a *successful* convert returns the existing invoice, never mints a second (enforced by the `convertedTo` lock, item 14).
- **Stage 2 — convert succeeds, then PaymentIntent creation throws** (`payment-intent-create.ts:75`): invoice already minted + locked, no charge started → **converted-but-unpaid invoice exists.**
- **Stage 3 — charge fails/declines at the webhook:** the failure path only sends a notification + records status (`payment-intent-updated.ts:297-322`) — **it does NOT touch the invoice or conversion.** Invoice stays minted, locked, unpaid; estimate can't be re-paid as an estimate either.
- ✅ **Success-side dup already guarded:** webhook returns early if the intent already succeeded (`payment-intent-updated.ts:51`) → duplicate deliveries won't double-add a payment. (Protects stage-3 success, not convert rollback.)
- 🔴 **UNHANDLED STATE — "converted but not paid" (stages 2 & 3 both land here).** Real invoice minted from an estimate that never got paid + a locked source estimate. **Product decision needed (not in the original 24 items):**
  - **(a) Leave the invoice, buyer retries payment on it** — retry re-pays the SAME invoice (item 18). Least work, safest vs double-charge (lock stays). Cost: "phantom" converted-unpaid invoices appear in the merchant's list even though the merchant didn't convert.
  - **(b) Roll back conversion on definitive payment failure** — void/delete the minted invoice + clear `convertedTo`, returning the estimate to payable. Clean, but net-new rollback work and races the async webhook.
  - **(c) Soft-lock until first successful payment** — invoice exists but the estimate isn't hard-locked until money lands. Most complex.
  - **Recommendation: (a)** — least work, no double-charge risk; the only downside is merchant-visible phantom invoices, which is a copy/UX problem not a correctness one.
- **Size:** decision-first. (a) ≈ verify + retry wiring (small); (b)/(c) = net-new rollback/soft-lock (Medium+).

---

## §"Buyer Post-conversion" — fill in (empty)

➕ *(all captured in [`ticket-buyer-post-conversion.md`](./ticket-buyer-post-conversion.md))*
- **Success / failed labels:** "DEPOSIT PAID" / "DEPOSIT DUE" — verify only. Due label is driven by **payment type, not docType** → already docType-agnostic, renders correctly on a paid estimate. `is-packages/packages/invoice-viewer/src/lib/useDueLabel.ts:13` (`nearestUpcomingPayment.paymentType === DEPOSIT ? depositDue : …`); label strings `useLabel.ts:195` (`depositDue`), `:188` (`paid`).
- **Payment-failure retry:** verify retry re-pays the *same* invoice, never a second invoice / re-convert.
- **Old-link redirect:** stale post-conversion checkout link → public estimate `/v/` page (converted state, affordances hidden), never re-enters checkout. Guards against a second invoice from one estimate.
- **Pay-Now loading state:** convert-first adds a round-trip — show loading so the button doesn't look frozen.
- **Post-payment receipts** — verify only. Template is invoice-centric with **no docType branch** → correct automatically once convert-first mints a real invoice. `is-services/packages/services/is-stripe/src/utils/email/render-email-template.ts:62` (`invoice_no`), `:134`/`:195` (`invoice_url`), `:63` (`is_deposit` from `isPartialPayment`).

---

## §"Merchant Post-conversion" — fill in (one empty bullet)

➕ Once converted (manually or by buyer pay), the estimate is **locked** — no re-conversion, no re-payment. Three layers:

| Layer | Fires | Role |
| --- | --- | --- |
| **Server payment gate** | Pay time, server-side | **Real lock** — rejects if `convertedTo` set. 📁 `invoice-payments-status.ts` |
| **Atomic write-guard** | at conversion | Exactly one invoice per estimate |
| **UI hiding** (merchant + buyer) | page load | **Cosmetic only** — hides Convert / Pay CTA |

- **"Make Invoice" hidden** after first conversion (all convert entry points). **No converted-state guard exists today** — every entry point gates only on `docType === ESTIMATE`. Work = add one new "already converted" condition to each mount, based on the immutable `convertedTo` (NOT `fullyPaid`, which conversion sets but nothing reads to hide). **Blocked on `convertedTo` landing + syncing to device** (item 1).
  - Mobile mounts (5): `invoice.screen.tsx:400` (`MakeInvoiceButton`), `invoice.screen.tsx:267` (`EstimateSignedSection` pill), `invoice.navigator.tsx:331` (overflow menu, `isEstimate` at `:258`), `invoice-list.component.tsx:295` (list long-press), `convert-estimate-to-invoice-modal.tsx:84` (FTUE prompt).
  - Web mounts: `invoice-actions-migrated.tsx:338` (`ButtonMakeInvoice`), `settings-sidebar-migrated.tsx:206` (`SignedEstimateMakeInvoice`), `dropdown-menu-convert-document-item.tsx:59` (kebab); legacy `client/`: `InvoiceActions.tsx:83`, `InvoiceControls.tsx:21`, `SignedEstimateMakeInvoice.tsx:35`.
- **Layered against sync-lag (Juan sync, 2026-07-16):**
  - **Layer A — hide on synced `convertedTo`.** `convertedTo` lives on the Invoice → Realm syncs it down on network retry / back-online. Covers the common case (reconnect → sync → then merchant acts). This is item 22 as scoped above.
  - **Layer B — offline short-circuit (v1 mitigation for the residual window).** Between the buyer's online convert and the sync landing on the phone, there's a window where the phone still shows an unconverted estimate. Rule: **hide "Make Invoice" when `hasDeposit` (net-new feature) AND FF on AND device offline.** Mobile already has the signal (`services/network/hooks/use-is-device-offline.ts`; offline-gating precedent in recurring-invoice banners + `offline-section-cover.tsx`). Narrow + cheap.
  - ⚠️ **Layer B is a UX mitigation, not the correctness guarantee** — hiding a button while offline shrinks the duplicate window but doesn't close it (cached screen, or estimate converts online a second after tap). The true backstop is a **server-side ingestion guard** (reject/no-op an invoice create whose `estimateId` points to an estimate whose `convertedTo` is already set to a *different* invoice id; idempotent for same-device re-sync).
  - **v1 DECISION (Lenmor, 2026-07-16): lean on Layer B (offline-hide) for v1; DEFER the server ingestion guard** — it's complex + wide blast radius (touches every invoice-write/sync from every device). Consequence: offline duplicate remains *possible but rare*; financial risk zero (server payment gate blocks double-charge regardless); residual = stray duplicate invoice, manually fixable.
  - ⚠️ **Behavior-change to name for Liz:** payable-deposit estimates lose **offline Make-Invoice** (an offline merchant can't convert a legit deposit-estimate until back online). Acceptable since deposit estimates are online-only, but it's a real regression on that path.
- ⚠️ **DESIGN SUGGESTION — design still PENDING, not confirmed for v1.** Base decision is only "reuse = duplicate" (the estimate is locked after conversion). The *combined* "duplicate & convert in one action" is a design refinement Seth raised (template-reuse friction) + Liz responded to on Slack 2026-07-14 ([p1784067659924269](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784067659924269?thread_ts=1784052946.973869)) — she kept the lock and suggested softening the re-convert prompt into one action instead of two manual steps. **Never confirmed for v1** (open item #6). Fallback if deferred = plain "duplicate first, then convert" in two steps. Treat everything below as *scoping IF design lands it in v1*, not committed work.
- Retry offers **"duplicate & convert" in one action** (softens template-reuse friction). Duplicating yields a fresh, unlocked estimate. Duplicate already exists standalone both platforms — mobile `invoice.navigator.tsx:320`→`handleDuplicate:943`→`exportDuplicateInvoice` (`export-helpers.ts:110`)→`duplicateInvoice` (`repository.ts:433`); web `dropdown-menu-duplicate-item.tsx:49`→`duplicateDocumentAction` (`duplicate-document.action.tsx:35`). Convert handlers to chain into: mobile `makeInvoiceFromEstimate` (`use-cases.ts:22`) / `convertEstimateToInvoice` (`invoice.navigator.tsx:1050`); web `convertDocumentAction` (`convert-document.action.tsx:45`). New action chains duplicate→convert on the fresh estimate.
  - **Discoverability UX (design PENDING — design owns):** since "Make Invoice" is now hidden on a locked estimate (item 22), there likely needs to be a signal explaining why + pointing to duplicate-and-convert — e.g. an alert/explainer on tap, an inline banner, or a repurposed CTA. Whether this exists at all, and its exact treatment, is an open design call — not yet specced.

---

## §(new) — "Open items for the meeting"

➕
1. ✅ **Double-gate under document-level — DECIDED: relax gate 2 (honor per-doc `feesType`) behind a FF** (fallback to AND). See item 12 / checkout section. *(was open; resolved this session)*
1b. 🔴 **Converted-but-unpaid state** — when convert succeeds but payment then fails (create-intent throws or webhook declines), a real invoice is minted from an unpaid estimate + the estimate is locked. Pick: **(a)** leave it, buyer retries same invoice (phantom invoice in merchant list); **(b)** roll back conversion on failure; **(c)** soft-lock until first payment. *(new — see item 15b; recommend (a))*
2. **`feesType` backfill vs null→account fallback** for old estimates — pick one *(now in scope because document-level)*.
3. **Abandoned-cart follow-ups on estimates — intended?** (needs growth/marketing owner)
4. **Email send-path scope — RESOLVED (code-verified):** web sends via v2, mobile sends via v3, and **both funnel through the same shared `send()`/`getEmailTemplateData()`/`getSubject()` in `email.ts`** → the estimate template + subject + `deposit_due` changes are single-site edits covering both. v1 (multipart, no invoice object) is dead in-repo — no caller in mobile/web/is-services/admin. Only residual: old app binaries in the field could still hit v1 (not source-visible; check server logs if certainty needed — low priority).
5. **Email subject copy** — default "…sent you an estimate", confirm via prototype (low stakes)
6. **"Duplicate & convert" combined action** — v1 scope or deferred?
7. ✅ **Sync-lag on merchant "Make Invoice" hiding — DECIDED (Juan sync 2026-07-16): Layer A hide on synced `convertedTo` + Layer B offline short-circuit (`hasDeposit && FF && offline`); DEFER the server ingestion guard for v1** (complex, wide blast radius). Offline duplicate stays rare-but-possible; financial risk zero. See item 24 / §Merchant Post-conversion. *(resolved this session)*
8. 🔴 **Server-side ingestion guard — deferred past v1.** Needed to actually *close* the offline duplicate window (reject an invoice create whose `estimateId` → estimate with a different `convertedTo`). Revisit if stray duplicates become a support burden.

---

## ⚠️ Downstream docs now stale from the flip (need reconciling)

- `block-b-global-vs-document-level.md` — records RESOLVED = GLOBAL. Needs a reversal note: surcharge UI now IN scope; transform must preserve `feesType`; backfill/fallback now required; `/v/` plumbing no longer needed; double-gate now a live question.
- `confluence-scratch-doc-v1.md` + `confluence-page-draft.md` — argue the global path (open item "is global final", surcharge-preview bullets).
- Memory `project_dm_architecture_pivot.md` — records the global/vaulting direction.

**Recommend:** update these once you confirm the flip is final (esp. the double-gate decision), so we don't thrash them twice.
