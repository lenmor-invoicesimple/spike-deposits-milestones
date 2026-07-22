# Self-Review Findings — Deposit Optimization v1 Tickets

**Date:** 2026-07-15
**Scope:** Adversarial review of all three ticket docs + confluence scratch summary
**Method:** Four independent reviewers — one per ticket + one cross-doc/consistency pass. Verified claims against actual codebase; did not trust the docs' own line references.

---

## 🔴 HIGH — conclusions that are wrong or plans that ship broken

### H1 — Email spike (spike-email-estimate-vs-invoice.md): headline answer to I2 is wrong

**Claim:** "Mislabeling risk is nil — email wording is docType-driven, not payability-driven."
**Reality:** The A/B subject override (`api/src/util/email-experiment-service.ts:79`) hardcodes `"Action Needed: {name} sent you an invoice"` and fires on `isEligible` (payability), NOT docType. On the v2 path (`email.ts:287-291`) it activates exactly when an estimate becomes payable. A buyer receives "…sent you an invoice" subject over a body saying "Estimate #123" — the precise I2 failure.

**Impact:** I2 cannot be closed as "risk nil." The subject line is an uncovered gap.

---

### H2 — Global ticket (ticket-payable-estimates-global.md) + scratch doc: flag enables a broken payment path

**Claim:** Scratch "Bottom line" — the only risk is retroactive rollout, handled with a flag + backfill.
**Reality:** WI-1 deliberately leaves the is-stripe *payment-time* validators unrelaxed, assuming convert-first will have run first. But convert-first is the unwritten Buyer Pre-Conversion ticket. Ship #1+#2 + flip the flag without it → buyer clicks "Pay Deposit," reaches checkout, payment-time gate rejects the docType=1 estimate → **payment fails for every buyer.** The convert-first cloud function is a hard release gate, not a future nice-to-have.

**Impact:** Stakeholder reading the scratch summary would greenlight flag rollout believing it's safe. The summary must say "do not enable flag until convert-first is deployed."

---

### H3 — Global ticket (WI-6): the rollout safety floor does not exist

**Claim:** Old converted estimates protected by `fullyPaid`→`balanceDueZero` until backfill completes.
**Reality:** `is-packages/packages/calculator/src/invoice/getInvoiceBalance.ts:55-58` returns `deposit.amount` early and never reads `fullyPaid`. grep `fullyPaid` in the calculator = **0 hits**. On flag-flip, every already-converted deposit-estimate is immediately payable (balanceDue = deposit.amount > 0, `convertedTo` still empty pre-backfill). A buyer can pay a document that was already converted and collected.

**Impact:** Backfill is a hard prerequisite, not a gradual follow-up. The safety net cited doesn't exist.

---

### H4 — Review page ticket (ticket-public-review-page-estimate-payable.md §5/WI-A): "two distinct gate families" claim is partly false

**Claim:** "Neither ticket covers the other's gates." WI-A owns relaxing is-checkout eligibility.
**Reality:** The is-checkout server eligibility path is `eligibility/service.ts:63` → `document-payable.ts:42` → `findNotPayableReason` → `invoice-payments-status.ts:58` — the **exact gate ticket #1 owns**. WI-A's "relax is-checkout eligibility" has no separate gate to relax; it's blocked on ticket #1's WI-1. The two tickets collide on the same line and the dependency isn't shown.

**Impact:** WI-A can't produce a `payUrl` for estimates until ticket #1 lands. The review-page ticket has an implicit hard blocker.

---

## 🟠 MED — plans that won't do what they claim

### M1 — Review page ticket (WI-C): contradicts its own acceptance criteria

**Claim:** AC — "non-payable / plain estimate shows no DEPOSIT DUE."
**Reality:** `hideInvoiceBalance` (`InvoiceBalance.tsx:11-17`) is purely docType-based — no payUrl, no payability input. Relaxing it by docType shows DEPOSIT DUE for every deposit-estimate regardless of payability. You'd need to add a payUrl/payability condition to satisfy the AC, which the ticket never specifies.

---

### M2 — Review page ticket (WI-D): "post-payment hiding comes for free" is false for the DEPOSIT DUE row

**Claim:** Pay button, QR, and fee disclaimer all key off `payUrl` and drop out together post-conversion.
**Reality:** DEPOSIT DUE row is `hideInvoiceBalance` (docType only), never `payUrl`. Since conversion mints a new invoice and leaves the estimate's docType=1, the converted estimate **shows DEPOSIT DUE forever**. Also: the "single payUrl" framing is wrong — there are ≥3 independent gate families (server `payUrl`, client SDK `isInvoicePayable`, docType row visibility).

---

### M3 — Global ticket (WI-5): targets the wrong web file, misses a call site

**Claim:** "Web equivalent: `is-web-app/client/src/util/estimateToInvoice.ts`."
**Reality:** `estimateToInvoice.ts` delegates to `estimate.convertEstimate()`. The real conversion logic (and the actual estimate save) lives in `InvoiceModel.ts:1016` (`convertEstimate`) / `:1296` (`_convertEstimateData`). Writing `convertedTo` to `estimateToInvoice.ts` misses the model method entirely. Also a 4th call site the doc never lists: `InvoiceRow.tsx:109` calls `convertEstimate()` directly, bypassing `handleMakeInvoice`.

---

### M4 — Review page ticket: shared-package mobile blast radius unmentioned

`invoice-viewer` and the payment SDKs are published packages pinned by both web (`@10.0.25`) and mobile (`@10.0.29`). Relaxing `hideInvoiceBalance`/`isInvoicePayable` in the package changes **mobile document rendering** too and requires a package version cut + pin bump in both apps — a release-coordination step the "M" estimate ignores entirely.

---

### M5 — Email spike: v1 (mobile) path can't call `getDeposit()` as written

`invoice-sending.ts` (v1, primary mobile-send route) builds render-data from multipart form fields only — no invoice object loaded, no deposit settings/items, and no `currency_code` in the render-data literal. `getDeposit()` is uncallable without fetching the invoice. The `{% if currency_code is defined %}` template snippet renders no currency prefix on v1, inconsistent with the mock. "XS–S, one render-data field" hides real v1-path work.

---

### M6 — Gate-family framing creates reader confusion across ticket #1 and #2

Ticket #1 says "provider gates already global — no work" and debunks the is-paypal gate. Ticket #2 requires relaxing `is-paypal-sdk/invoice-payable.ts` and `is-stripe-sdk/invoice-payable.ts`. These are genuinely different files (is-services `sdks.ts` vs is-packages `*-sdk/invoice-payable.ts`) but Ticket #1's absolute "verified, no work needed" language — with no cross-reference — would lead anyone reading it alone to conclude all SDK payability is handled. The scratch doc inherits this framing. Ticket #1 needs a pointer to the client-side gate family.

---

### M7 — P3 (hide post-conversion QR/CTA) is double-owned

Ticket #1 coverage matrix routes P3's "hide QR/CTA" to the unwritten Buyer Post-Conversion ticket. Ticket #2 WI-D explicitly owns the same hiding in-scope. Someone will either duplicate the work or skip it assuming the other ticket covers it.

---

### M8 — WI-5 mobile atomic-guard claim is unsound for offline Realm clients

WI-3's write-contract requires a `findOneAndUpdate`-style atomic guard where `convertedTo` is empty. Mobile conversion is offline-first Realm; there's no server `findOneAndUpdate` in the client conversion path. Two offline devices (or buyer cloud function + offline merchant) can each see `convertedTo` empty and both win. Atomicity holds only for the server cloud function — the 2 client writers the doc also asks to satisfy this contract cannot deliver it.

---

## 🟡 LOW

- **"Pay Deposit" may charge full balance** — nothing in the traced checkout path restricts the charge to the deposit amount; the label/behavior mismatch is unacknowledged.
- **Three SWU template IDs**, not two — `swuPDFEstimateTemplateV2` (send-to-self, `email.ts:654`) never mentioned; local "byte-identical" comparison doesn't prove SWU-hosted records are in sync.
- **Scratch oversells I1 scope** — "small amount of net-new work" while the ticket table says "Large."
- **Reminder `deposit_due` staleness** — reminders reuse snapshotted render-data; a `deposit_due` field will be frozen at send time and re-shown even if the estimate changes. Same class as `balance_due` today, but the AC should say "consistent with the snapshot" not just "consistent."
- **`block-b` stale on lock key** — references `approvedAt` as the lock; ticket #1 WI-3 says it doesn't exist. Low because block-b is a decision record, not a build ticket.
- **i18n understated** — "Pay Deposit" touches 12 locale files; fee-disclaimer estimate variant is a separate package dictionary (`invoice-viewer/src/is-intl/dictionary.ts`), different pipeline.

---

## ✅ Claims that checked out (verified correct)

- `invoice-payments-status.ts:58` is the single docType gate in `findNotPayableReason` ✅
- Provider gates in `sdks.ts` are docType-agnostic (PayPal L37-55, Stripe L90-115) ✅
- `convertedTo`/`convertedFrom` grep = 0 repo-wide ✅
- No estimate→invoice cloud function exists; conversion is client-only ✅
- Exactly 6 callers of `findNotPayableReason` as listed ✅
- Page at `nextjs/app/(public)/v/[documentId]/page.tsx` → `PublicDocumentFeature` ✅
- Both `is-paypal-sdk` and `is-stripe-sdk` `invoice-payable.ts` hard-check `docType === DOCTYPE_INVOICE` ✅
- `hideInvoiceBalance` hides estimate row; `useDueLabel` deposits label — both verified ✅
- `convertedTo` backfill via `invoice.setting.estimateId` is reliable (both conversion paths set it) ✅
- WI-E refactor is genuinely small; signature modal is a global zustand singleton (no analytics double-fire) ✅
- §2 email gap accurate: no email carries a deposit amount today; `is_deposit_eligible` is boolean+invoice-only ✅
- Template selection: v1 → `swuEstimateTemplate`, v2 → `swuEstimateTemplateV2` ✅
- `{% if deposit_due %}` / `{% trans %}` placement in template is syntactically sound ✅
- `convertedTo` field ownership is clean — exactly one owner per layer ✅

---

## Recommended next actions

1. **Reopen I2** — add subject-override gap to the email spike; remove "mislabeling risk nil" verdict.
2. **Add convert-first release-gate** to ticket #1 WI-1 and the scratch bottom line.
3. **Fix WI-6** — remove the `fullyPaid` safety-floor claim; state backfill as a hard precondition.
4. **Fix §5/WI-A** — acknowledge is-checkout eligibility runs through the same gate as ticket #1; add the dependency.
5. **Fix WI-C/WI-D** — `hideInvoiceBalance` needs a payability condition to satisfy its own AC; DEPOSIT DUE row won't auto-hide post-conversion.
6. **Fix WI-5** — target `InvoiceModel.convertEstimate()` not `estimateToInvoice.ts`; list `InvoiceRow.tsx:109` as 4th call site.
