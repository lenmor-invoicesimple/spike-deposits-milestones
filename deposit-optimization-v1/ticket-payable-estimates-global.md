> ✅ **CORRECTIONS APPLIED (2026-07-15)** — adversarial-review defects fixed in this revision. See [`self-review-findings.md`](./self-review-findings.md) for the full findings. Fixed here: convert-first is now an explicit work item + hard release gate (WI-0, H2); WI-6 rollout safety-floor claim removed and backfill stated as a hard precondition (H3); WI-5 retargeted to `InvoiceModel.convertEstimate()` + 4th call site added (M3); offline-Realm atomic-guard claim corrected — server is the authoritative lock (M8); client-side SDK gate family cross-referenced (M6). New **§0 Release sequencing** section captures the ordering + migration/infra work items.

# Ticket — Enable Online Payments on Estimates (Global Settings)

**Epic:** IS-11250 — Deposit Optimization v1
**Date:** 2026-07-14
**Owner:** Lenmor Dimanalata
**Status:** Draft — implementation-ready spec (pre-split)
**Source of truth:** Liz's design-meeting decision ([Slack p1784052946973869](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869)); decision record [`block-b-global-vs-document-level.md`](./block-b-global-vs-document-level.md); flow context [`enable-payments-on-estimates.md`](./enable-payments-on-estimates.md).

> **Convention:** M#/B#/P#/I# labels below map to the four headings in Liz's Slack post (Merchant Pre-Conversion, Buyer Pre-Conversion, Buyer Post-Conversion, To-Investigate). See the coverage matrix.

---

## 1. Scope

**This ticket = the payability engine for estimates + the conversion lock + retroactive migration.** It makes a deposit-estimate reach checkout using **global/account** payment settings (no per-estimate toggle, no per-estimate surcharge UI — Liz-confirmed GLOBAL), locks an estimate once converted, and handles the retroactive-payability rollout.

**In scope**
- **Convert-first cloud function** — synchronous estimate→invoice conversion at Pay Now, before the PaymentIntent (WI-0). *Our team's work; foundational; hard release gate for the flag.*
- Relax the single docType gate so estimates render on the checkout page (WI-1).
- Live-account surcharge on the estimate preview so it matches the post-conversion charge (WI-2). Charge amount is already global; the work is the checkout gate-1 substitution + an optional/deferrable `/v/` disclaimer read (shared invoice-viewer, web+mobile).
- New `convertedTo` field across all layers + the write-contract both conversion paths implement (WI-3).
- Payment lock keyed off `convertedTo` in `findNotPayableReason` (WI-4).
- Merchant-side conversion lock: write `convertedTo` back on manual convert, hide the Convert button, duplicate prompt (WI-5).
- Retroactive-payability migration + rollout plan (WI-6).

**Out of scope — routed elsewhere (nothing dropped)**
- **Post-conversion buyer UX** (P1–P2, P4, P6–P7): payment-fail retry, conversion-fail error UI, old-link→public-estimate-URL redirect, tab-label + success-screen changes → **Buyer Post-Conversion ticket.**
- **Buyer Pay Now UX shell** (P8): loading state during conversion latency → **Buyer Post-Conversion ticket.** (The convert-first *mechanic* itself is WI-0, in scope here; only its surrounding failure/loading UX is routed out.)
- **Email estimate-vs-invoice conditional render** (P5 / I2) → **separate spike doc** (`spike-email-estimate-vs-invoice.md`, to be created).

**Dependency:** WI-4's lock and WI-3's field are consumed by the convert-first cloud function (WI-0, this ticket / same team). The buyer flow **cannot ship** without WI-0 + WI-3 + WI-4.

> **Ownership correction (2026-07-15):** the convert-first cloud function is **our team's** work — most likely the epic owner (Lenmor) — **not** an external/Juan dependency. It is foundational to the feature and is captured here as **WI-0**, not routed out.

---

## 0. Release sequencing — READ FIRST ⚠️

The adversarial review surfaced a hard ordering constraint: **the WI-1 feature flag CANNOT be turned on the day WI-1 merges.** Two things must be live and complete *first*, or the feature breaks for 100% of buyers and re-exposes already-collected estimates.

**Mandatory order:**

```
1. WI-0  Convert-first cloud function deployed         ─┐  both must be
2. WI-6b convertedTo INDEX in place (+ audit run)       ─┘  done BEFORE step 3
3. WI-1  Flip the payability flag (gradual rollout)
```
*(WI-6a historical backfill is contingent on the audit — likely unnecessary; see WI-6.)*

**Why the order is non-negotiable:**

1. **Flag-on without convert-first (WI-0) → every deposit payment fails (H2).** WI-1 deliberately leaves the *payment-time* validators (is-stripe, is-checkout confirm path) rejecting `docType === 1`. The design assumes convert-first has already turned the estimate into an invoice *before* the PaymentIntent is created. If the flag is on but convert-first isn't deployed, the buyer reaches checkout, the payment-time gate rejects the estimate, and **payment fails for every buyer.** Convert-first is a hard release gate, not a follow-up.

2. **Flag-on without the conversion lock → an added-later deposit re-opens a converted estimate (revised H3).** ⚠️ **The historical population is empty** — deposits on estimates are net-new (deposit editor always `docType===INVOICE`-gated), so **no old estimate has a deposit** and there is nothing historical to re-open. The live risk is **forward-going**: after IS-11406, a merchant can add a deposit to an **already-converted** estimate → payable → double-collection on the old estimate link. The defense is the **live lock** (WI-4 reads `convertedTo` at Pay time) + writing `convertedTo` on all conversion paths, **not** a historical backfill. What gates the flag here is the **`convertedTo` index (WI-6b)** the lock queries — plus the audit confirming the historical count is 0. See WI-6.

**Migration / infra work items** (captured here; may be forked into a dedicated migration sub-ticket):
- **WI-6a — Mongo `convertedTo` backfill** — **CONTINGENT on the audit; likely unnecessary** (historical deposit-estimate population expected to be 0).
- **WI-6b — Mongo index on `convertedTo`** (the live lock queries this). Atlas/infra action, not a code diff. **REQUIRED, gates the flag.**
- **WI-6c — Realm `convertedTo` field addition** (v61, additive); on-device data backfill only if WI-6a is.

See WI-0 and WI-6 for detail.

---

## 2. Liz's-list coverage matrix

| Bullet | Requirement | This ticket | Where |
|---|---|---|---|
| **M1** | Payable if deposit set AND PayPal/Stripe enabled globally | ✅ | WI-1 (docType gate); provider gates already global — no work |
| **M2** | Inherits global settings same as invoices | ✅ | Whole premise; WI-2 confirms surcharge inheritance |
| **M3** | Surcharge inherits global, no doc-level override | ✅ | WI-2 |
| **M4** | All existing deposit-estimates payable retroactively | ✅ | WI-6 |
| **M5** | Eligibility = deposit set + payments global + **not yet converted** | ✅ | WI-1 + WI-4 (`convertedTo` unset) |
| **M6** | Once converted → locked (no further conversion, no further payment) | ✅ | WI-4 (payment) + WI-5 (conversion) |
| **M7** | Manual "Convert to Invoice" hidden after first conversion | ✅ | WI-5 |
| **M8** | Prompt to duplicate on 2nd convert attempt | ✅ | WI-5 |
| **B1–B3** | Convert at Pay Now, invoice-ID to processor, before payment | ✅ | **WI-0 (this ticket, our team)** — the convert-first cloud function; satisfies WI-3's write-contract |
| **P1–P2** | Payment-fail retry / conversion-fail error UI | ➡️ | Buyer Post-Conversion ticket |
| **P3** | Post-conversion not payable; QR + CTA hidden | ◐ | Lock = WI-4 (this ticket); hide QR/CTA = Post-Conversion ticket |
| **P4** | Old email link → public estimate URL | ➡️ | Buyer Post-Conversion ticket |
| **P5** | Email conditional render for estimate vs invoice | ➡️ | Email spike (new doc) |
| **P6–P7** | Tab label / success screen reflect invoice | ➡️ | Buyer Post-Conversion ticket |
| **P8** | Pay Now loading state (conversion latency) | ➡️ | Buyer Post-Conversion ticket (WI-0 provides the mechanic it wraps) |
| **I1** | Confirm technical approach: global payments + deposit → payable | ✅ | **This ticket is the deliverable** |
| **I2** | Confirm email renders estimate vs invoice | ➡️ | Email spike (new doc) |

Legend: ✅ owned here · ◐ split · ➡️ routed out.

---

## 3. Work items

All file:line references verified against the working tree on 2026-07-14.

### WI-0 — Convert-first cloud function (B1–B3) — HARD RELEASE GATE

**Current state.** No estimate→invoice cloud function exists; conversion is **client-only** today (mobile `makeInvoiceFromEstimate`, web `InvoiceModel.convertEstimate()`). Verified: repo-wide there is no server-side conversion path.

**Why it's in this ticket.** The buyer "Pay Deposit" flow converts the estimate to an invoice **synchronously, server-side, before creating the PaymentIntent**, so the payment-time validators (is-stripe, is-checkout confirm path) see a real invoice (`docType === 0`) and pass unchanged. Without this, WI-1's flag exposes a checkout that fails at the payment gate for every buyer (see §0). It is **our team's** foundational work — most likely the epic owner — not an external dependency.

**Change.**
1. New Parse cloud function: given an estimate remoteId, perform the estimate→invoice conversion server-side (mirroring the client `convertEstimate` semantics — new invoice with `setting.estimateId` back-link), returning the new invoice remoteId.
2. **Implement WI-3's write-contract atomically here** — this is the *only* writer that can honor the atomic guard (see WI-3/M8): a `findOneAndUpdate`-style guard on the estimate where `convertedTo` is empty ensures a single conversion wins under double-click / retry at Pay Now. Append the new invoice remoteId to the estimate's `convertedTo`.
3. Wire the buyer Pay Now path (is-unifiedxp / checkout) to call this before PaymentIntent creation, then hand the **invoice** remoteId to the processor for L2/L3.

**Acceptance criteria**
- A buyer paying a deposit-estimate triggers server-side conversion → PaymentIntent is created against the resulting **invoice** (docType 0), so payment-time validators pass with no relaxation.
- Double-click / retry at Pay Now results in exactly **one** conversion (atomic guard on `convertedTo`).
- The source estimate's `convertedTo` is populated by this function.

**Dependency note.** WI-0 consumes WI-3 (the `convertedTo` field + contract). The surrounding buyer UX — loading state, conversion-failure screen, retry — is routed to the Buyer Post-Conversion ticket; WI-0 is the mechanic only.

---

### WI-1 — Relax the docType gate (M1, M5)

**Current state.** A single line rejects every estimate before any account/document setting is read:

`is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58`
```ts
if (documentType !== 0) {
  return GeneralNotPayableReason.docTypeNotInvoice;
}
```

`findNotPayableReason` (L42) order: `noCheckoutData` (54) → **`docTypeNotInvoice` (58)** → `deleted` (62) → `totalCents<=0`→`totalZero` (66) → `balanceDueCents<=0`→`balanceDueZero` (70) → `balanceDueTooLarge` (74) → `pendingOrders` (81) → `null`.

**Provider gates are docType-agnostic — no work needed.** Stripe/PayPal `findNotPayableReason` (`utils/SDKs/sdks.ts` — PayPal L37-55, Stripe L90-115) check only global account accepting-status (`isPaypalAcceptingAndEnabled` / `isStripeSubsetOfAcceptingAndEnabled` + `extractAcceptingProductsV2`), currency mismatch, and per-doc `paymentSuppressed`. **No docType check exists in them** → they pass estimates once the account is accepting. This satisfies Liz's "PayPal/Stripe enabled globally" half automatically.

> ⚠️ **Scope of this "no work needed" claim (M6).** It applies ONLY to the **is-services** `sdks.ts` provider gates. There is a **second, distinct SDK gate family** in the published packages — `is-packages/packages/is-paypal-sdk/src/invoice-payable.ts` and `is-stripe-sdk/src/invoice-payable.ts` — which **do** hard-check `docType === DOCTYPE_INVOICE` and **must be relaxed** for the client-side (invoice-viewer QR) payability. That work lives in the **review-page ticket** ([`ticket-public-review-page-estimate-payable.md`](./ticket-public-review-page-estimate-payable.md) WI-C, §5). Do not read "no work needed" as covering all SDK payability — it doesn't.

**Change.** Allow `documentType === 1` (estimate) through L58. Gate the relaxation behind a Flagsmith flag at the caller level (see WI-6). Scope the estimate allowance to deposit-set estimates (an estimate without a deposit has no balance-to-pay semantics — confirm via `balanceDueCents` for estimates, below).

**Audit the 6 callers** (split eligibility vs payment-time):
| Caller | file:line | Type | Note |
|---|---|---|---|
| `is-checkout` `validateDocumentPayable` | `document/document-payable.ts:47` | payment-time (throws) | must accept estimate up to Pay Now |
| `is-checkout` `getReportRedirectUrl` | `checkout/service.ts:65` | checkout-page redirect | tolerant switch L71-81 |
| `is-unifiedxp` `validateDocumentPayable` | `utils/document-payable.ts:28` | checkout-page eligibility | `docTypeNotInvoice`→`DOCUMENT_NOT_AN_INVOICE` (L41-42) — remove/relax mapping |
| `is-abandoned-cart` `isInvoicePayable` | `utils/is-invoice-payable.ts:22` | eligibility (targeting) | ⚠️ estimates would start qualifying for abandoned-cart — confirm desired |
| `is-stripe` `validateDocumentPayableToConfirmPayment` | `services/payments-status/document-payable.ts:13` | payment-time | sees invoice (docType 0) post-convert-first → passes naturally |
| `is-stripe` `validateDocumentPayable` | `services/payments-status/document-payable.ts:41` | payment-time | same |

**Convert-first (WI-0) shrinks this.** Because the buyer flow converts before creating the PaymentIntent, the payment-time validators (is-stripe, is-checkout confirm path) see an **invoice** (docType 0) and pass unchanged. Only the **checkout-page eligibility** callers (is-unifiedxp, is-checkout redirect) must allow docType 1 so the deposit checkout can render. ⚠️ **This split is exactly why WI-0 is a hard release gate (§0):** the payment-time callers stay unrelaxed *on the assumption WI-0 already ran*. If the flag is enabled before WI-0 ships, those callers reject the estimate and every payment fails. Verify the caller split holds before finalizing which validators are touched.

**Acceptance criteria**
- A deposit-estimate with an accepting Stripe/PayPal account and `convertedTo` unset returns `null` (payable) from `findNotPayableReason`.
- An estimate with no deposit remains non-payable.
- is-unifiedxp renders the checkout page for a payable estimate (no `DOCUMENT_NOT_AN_INVOICE` error).
- Flag off → estimates behave exactly as today (non-payable).
- ⚠️ abandoned-cart behavior for estimates is an explicit decision, not an accident.

**Edge cases.** `totalCents<=0` / `balanceDueCents<=0` estimates rejected at L66/L70 (fine — nothing to pay). Verify `formatDocumentData` (`is-checkout/document/document-payable.ts:20-39`) computes sensible `totalCents`/`balanceDueCents` for a deposit-estimate (estimate has no payments → balanceDue == total; deposit amount handled by `getInvoiceBalance`).

---

### WI-2 — Live-account surcharge on the estimate preview (M3)

> **Scope at a glance (trace-verified 2026-07-15):** the surcharge **amount charged** is already global (checkout re-reads account `payment.feesType`). WI-2 = (a) substitute the account value into the checkout **on/off gate 1** for estimates (small), + (b) optionally align the `/v/` review-page **disclaimer note** which reads the stored value (small-medium, shared invoice-viewer package web+mobile, **deferrable** — see the Trace update below). **No `feesType` backfill needed** (global never reads the estimate's stored/`null` value).

**The concern, corrected & narrowed.** Preview (estimate checkout) and charge (converted invoice) run the **same** `getSurchargeCents`, but on **different document objects**. Two per-document reads can go stale on retroactively-payable old estimates:

1. **The on/off gate — the real risk.** `getSurchargeCents` double-gates: `universalInvoice.setting.feesType === SURCHARGE` (per-doc) **AND** account `payment.feesType === SURCHARGE` (live). Refs: `is-stripe/.../passing-fees/stripe-fees.ts:38` (gate 1) + `:39-45` (gate 2, `Setting.getOneByAccount(accountId,'payment.feesType')`); PayPal `paypal-fees.ts:37` + `:38-43`. If an old estimate froze `setting.feesType = MARKUP`/undefined but the account is now SURCHARGE → **preview shows no fee, converted-invoice charge does** → divergence.
2. **The magnitude — low risk.** `getInvoiceSurcharge.ts:46`: `const feeRate = invoice.setting?.feeRate ?? 100`. Per-doc, but since we are **not** exposing the estimate surcharge slider (global decision), `feeRate` is ~always undefined→100. Low risk; note it, don't over-engineer.

**Base rate is not the problem** — it's a live per-country account constant (`stripe-fees.ts:7-14` US 3.49%+$0.49 / CA 2.9%+$0.30; `paypal-fees-config.ts:9-24` adds GB/AU). Read from `accountStatus.stripe.stripeCountry` at compute time, already live.

**Preview compute path.** `is-unifiedxp` `PaymentsSection.tsx:234` `getSurchargeCents(documentStatus)` → `payments-status/surcharge.ts:4-16` → SDK reads pre-computed `documentStatus.surchargeCents` (`sdks.ts:135` Stripe / `:73` PayPal) → populated in `get-checkout-document-details.ts:32-33` → Stripe `backend/checkout-document-details.ts:30-35` calls the same `passing-fees/stripe-fees.ts getSurchargeCents`. Charge path: `is-stripe/.../payment-intent-create.ts:46-51` — identical function.

**Change.** On the **estimate checkout path**, resolve the surcharge **on/off gate** from the **live account setting** rather than the estimate's stored `setting.feesType`, so preview == post-conversion charge. Cleanest option: when building `documentStatus` for a docType=1 document, substitute the account `payment.feesType` for the estimate's stored value in gate 1 (mirrors how the converted invoice's `feesType` gets re-seeded from the account at conversion — `defaultInvoiceSetting`/`repository.ts:97`). Keep `feeRate` as-is (defaults to 100).

> ✅ **The conversion transform already re-reads the account `feesType` on both active clients (trace-verified 2026-07-15) — no transform fix needed.** The converted **invoice**'s `feesType` is re-supplied from the account default, so it follows the current account setting: mobile `makeInvoiceFromEstimate` omits `feesType` and re-supplies `getDefaultPassingFeesType()` (`use-cases.ts:55`, `repository.ts:97`); **active web** is a nextjs server action `transformDocumentSettings` that overrides `feesType: options.defaultFeesType` (`document-transformation-utils.ts:106`, fed from `settings['payment.feesType']` at `convert-document.action.tsx:96`). ⚠️ The legacy `client/InvoiceModel.convertEstimate` (`InvoiceModel.ts:1305`) copied `feesType` verbatim, but that SPA is **dead code** — not in the conversion chain. So WI-2 is **only** the preview-side change (make the estimate *checkout preview* read the live account setting); the invoice-side charge already matches. The net-new WI-0 convert-first **server** function must be built to re-supply `feesType` from the account too (matching the active clients).

**Acceptance criteria**
- For an estimate whose stored `feesType` differs from the current account setting, the preview fee equals what the converted invoice charges.
- For a fresh estimate (stored == account), no behavior change.
- Non-surcharge accounts still show no fee.

> 🔎 **Trace update (2026-07-15) — a SECOND preview surface, and a scope narrowing.** A dedicated trace of the buyer-facing surfaces found there are **two**, not one:
> 1. **is-unifiedxp checkout page** (the `payUrl` target, analyzed above) — its surcharge **amount** already re-reads the account `payment.feesType` (gate 2, `stripe-fees.ts:39-45`). The remaining exposure here is **gate 1** (`stripe-fees.ts:38`, the per-doc `feesType === SURCHARGE` check): because it's an **AND**, a stale per-doc `feesType` on an old estimate can still veto the fee even when the account says SURCHARGE. **This is exactly what the WI-2 "Change" above fixes** (substitute the account value into gate 1 for docType=1). ⚠️ **Reconcile:** the trace characterized this checkout as "already global"; that's true for the *amount* (gate 2) but **not** for the on/off decision (gate 1) — WI-2's gate-1 substitution is still needed. Verify gate 1's actual behavior on a docType=1 doc before sizing.
> 2. **`/v/` public review page** (invoice-viewer `PaymentInstructions.tsx:71`, `hideSurcharge`) — a surface WI-2 did **not** previously cover. It shows/hides only the **surcharge disclaimer note** (no computed amount) and reads the **stored** `invoice.setting.feesType`. The account `payment.feesType` is **not loaded** on the `/v/` page (`public-invoice.ts` fetches the `Account` object but not that Setting), and the gate lives in the **shared invoice-viewer package used by BOTH web and mobile**. Aligning it = fetch the account Setting in `handlePublicInvoice` + plumb through `PublicInvoiceState` → `<Invoice>` → a new invoice-viewer prop/context + update `hideSurcharge()` + supply it (or a safe default) on the **mobile** consumer. Small-to-medium, **cross-platform blast radius**, and **deferrable** (it's disclaimer-visibility, not a wrong charge — the money is governed by surface 1 + the converted invoice).
>
> **Net:** WI-2 is two small pieces — (a) the gate-1 account substitution on the checkout path (as written), and (b) optionally the `/v/` disclaimer read in invoice-viewer (deferrable). Neither is a rewrite; (b) carries the shared-package coordination.

**Old-estimate `feesType = null` (relates to WI-2 + migration).** Old estimates have `feesType = null` (surcharge never existed on estimates). Under **global** this is harmless — both the checkout gate-1 substitution and the converted-invoice re-seed read the **account** value, so `null` is never consulted. (Under document-level it would mean old estimates convert **fee-free** unless backfilled — a real cost that global avoids. See block-b §4.) **No `feesType` backfill is required on the global path.**

**Open question.** Confirm the gate-1 substitution point is the estimate checkout-detail builder (docType-scoped) and doesn't leak into invoice previews; and decide whether the `/v/` disclaimer alignment (surface 2) is in v1 scope or deferred.

---

### WI-3 — `convertedTo` field + write-contract (M5, M6; dependency for B1–B3)

**Current state.** `convertedTo`/`convertedFrom` exist **nowhere** (repo-wide grep = 0 hits). The only estimate↔invoice link is `estimateId` on the invoice's **setting** (backward link: invoice→source estimate), never forward. No re-conversion lock exists today at all.

**Add the field at every layer:**

1. **Parse validation** — `is-parse-server/cloud/collections/invoice/invoiceValidation.ts`, top-level document schema `invoiceConstraints.schema` (L680+, alongside `recurringInvoiceSeriesId` L885). Add `convertedTo` as an array-of-string (remoteIds) constraint. Set/read in `beforeSaveInvoice` (`invoiceHooks.ts:36+`).
2. **Mongo index** — `convertedTo` must be indexed (searchable estimate-by-invoiceId). ⚠️ **Infra action, not a code diff** — no index-definition code exists in `is-parse-server/cloud` (indexes managed in Mongo/Atlas directly). **Remind the user to create the index** (per the DB-index rule).
3. **Realm (mobile)** — `is-mobile/src/services/realm/entities/invoice/schema.ts` (interface L14-53, `ObjectSchema` L55-88). Add `convertedTo: { type: 'list', objectType: 'string' }`. Bump `schemaVersion` **60 → 61** in `db.ts:195` + add the history-comment entry (L104-166). **Additive field → no `onMigration` branch needed** (matches how `estimateId`/`recurringInvoiceSeriesId` were added); add a migration branch only for the backfill in WI-6.
4. **UniversalInvoice** — `is-packages/packages/common/src/types.ts:309` — add `convertedTo?: string[]`.
5. **CheckoutData** — `is-services/packages/payments/payments-status/src/types/types.ts:86` — add `convertedTo?: string[]`; populate in transformer `utils/transformers/document.ts:15` (`universalInvoiceToCheckoutData`, alongside `recurringInvoiceSeriesId` L57).

**Write-contract (all conversion paths append to `convertedTo`):**
> On a successful estimate→invoice conversion, append the new invoice's `remoteId` to the source estimate's `convertedTo` array and save the estimate. `approvedAt` does **not** exist and is not used.

> ⚠️ **Atomicity is server-side only (M8).** The `findOneAndUpdate`-style atomic guard (only one conversion wins under double-click / retry) can be honored **only** by the server-side convert-first function (WI-0), which runs in Parse cloud. The **client conversion paths are offline-first Realm** (WI-5) — there is no server `findOneAndUpdate` in the client path, so two offline devices (or an offline merchant + the buyer cloud function) can each see `convertedTo` empty and both write. **The authoritative lock is therefore the server-side checkout gate (WI-4)**, which reads `convertedTo` at Pay time and blocks a second payment/conversion. Client writes are best-effort/optimistic; do not rely on them for uniqueness. Client convergence is handled by Parse/Realm sync (last-write-wins on the array is acceptable because WI-4 is the real gate).

- **Convert-first cloud function** (server-side, WI-0) → the **atomic** writer; the one that honors the guard.
- **Merchant manual path** (client-side, offline-first) → implemented in WI-5; best-effort append, NOT atomic.

**Acceptance criteria**
- Field validates/persists on Parse, Realm, and flows through to `CheckoutData` at checkout.
- Realm 60→61 migration ships without data loss for existing invoices.
- WI-0 (server) honors the atomic guard; WI-5 (client) appends best-effort; WI-4 is the authoritative lock regardless of writer.

---

### WI-4 — Payment lock keyed off `convertedTo` (M5, M6, P3-lock)

**Current state.** Payability keys off financial state (`balanceDueCents`), not conversion state. `fullyPaid` is a setting flag not consulted by `findNotPayableReason`. No conversion-based lock.

**Change.** In `findNotPayableReason` (`invoice-payments-status.ts:42`), after the docType allowance (WI-1), return a **new** `GeneralNotPayableReason` (e.g. `estimateAlreadyConverted`) when `convertedTo` is non-empty. Add the enum value at `payments-status/src/types/types.ts:126`. **Key off `convertedTo`, NOT the toggleable `paymentSuppressed`** — so a merchant can't re-expose payment on a converted estimate by flipping suppression. Requires `convertedTo` on `CheckoutData` (WI-3).

**Acceptance criteria**
- Estimate with non-empty `convertedTo` → non-payable, regardless of `paymentSuppressed`.
- Fresh estimate (`convertedTo` empty) → payable (subject to WI-1).
- The converted **invoice** itself remains payable (lock is on the estimate only).

**Edge case.** Atomic guard at conversion (WI-3) is the primary double-conversion defense; this lock is the checkout-side source of truth for double-**payment**.

---

### WI-5 — Merchant conversion lock UI (M6, M7, M8)

> 🚨 **STALE WEB TARGETS — needs re-tracing before implementation (flagged 2026-07-15).** All the **web** file references in this WI-5 (`InvoiceModel.convertEstimate()`, `InvoiceModel.ts:1016/1022/1305/1312`, `_convertEstimateData:1296`, `InvoiceRow.tsx:109`, `estimateToInvoice.ts`) point at the **legacy `client/` SPA, which is NO LONGER USED.** The **active** web conversion is a **nextjs server action**: `nextjs/app/(authenticated)/(core)/(documents)/actions/convert-document.action.tsx:43` (`convertDocumentAction`) → `nextjs/app/services/documents/document-transformation-utils.ts:64` (`transformDocumentSettings`, convert branch `:101-111`) → `document-transformation.ts:52` (`createDocumentInParse`). The estimate is marked closed via `markEstimateAsClosed()` (`convert-document.action.tsx:178-190`, sets `setting.fullyPaid:true`). Web convert entry points are 3 client components (dropdown-menu-convert-document-item, button-make-invoice, signed-estimate-make-invoice) all calling `convertDocumentAction`. **Before building WI-5's web slice, re-trace where `convertedTo` should be written on the active nextjs path (likely in `transformDocumentSettings` / the action) and which nextjs components need the convert-action guarded.** The `fullyPaid`, promo-modal, and mobile findings below remain valid; only the **web** file:line targets are wrong. *(The active nextjs path already re-supplies `feesType` from the account default at `document-transformation-utils.ts:106` — this is also the correct home for the `convertedTo` write.)*

**Current state (corrected 2026-07-15 — verified by trace + live DB inspection).** Conversion **does** mutate the source estimate: it sets **`setting.fullyPaid = true`** on the estimate and saves it (an earlier draft wrongly said "mutates nothing on the source estimate"). Confirmed on 3 of 4 conversion paths — web `InvoiceModel.convertEstimate()` (`InvoiceModel.ts:1022`), mobile doc-list (`invoice-list.component.tsx:384`), mobile navigator (`invoice.navigator.tsx:1060`). The new **invoice** carries the only estimate→invoice link, `setting.estimateId` (mobile `use-cases.ts:58`, web `InvoiceModel.ts:1312`) — a *backward* link. There is **no** dedicated forward "was-converted" marker on the estimate today (no `convertedTo`/`converted`/status field; repo-wide grep = 0). No call site checks prior conversion.

> ⚠️ **Why `fullyPaid` is NOT reused as the conversion lock (decided 2026-07-15).** `fullyPaid = true` is written *only* by conversion — mark-as-paid, payment recording, deposit-fully-paid, and the Parse `beforeSave` hook all leave it untouched or write `false` (exhaustive trace: 3 write sites, all conversion). So it *looks* like a clean "was-converted" signal. **Rejected for two reasons:** (1) **False negative** — the live, Optimizely-gated promo modal (see 4th call site below) converts **without** setting `fullyPaid`, so a promo-converted estimate would stay unlocked/re-payable; (2) **semantic overload** — `fullyPaid` means "paid," not "converted"; keying a payment lock off it is fragile if either meaning shifts. Also, `fullyPaid` is a boolean and can't carry the **invoice identity** Liz's constraint needs ("which invoice does checkout point to"). Purpose-built **`convertedTo`** (array of invoice remoteIds, WI-3) is the robust answer and is retained.

> ⚠️ **Web target corrected (M3).** `is-web-app/client/src/util/estimateToInvoice.ts` is a thin delegator — it does **not** do the conversion or save the estimate. The real logic lives in **`InvoiceModel.convertEstimate()` (`InvoiceModel.ts:1016`)** / `_convertEstimateData` (`:1296`). Writing `convertedTo` to `estimateToInvoice.ts` would silently miss the model and never persist. Target `InvoiceModel.convertEstimate()`. Also there is a **4th call site** the earlier draft missed: **`InvoiceRow.tsx:109`** calls `convertEstimate()` directly, bypassing `handleMakeInvoice` — it must be guarded too.

> ⚠️ **5th conversion path — the mobile promo modal (flag-gated, easy to miss).** `is-mobile/src/features/documents/modals/convert-estimate-to-invoice-modal.tsx` (`handleConvert` → `makeInvoiceFromEstimate`) is a **live** conversion path gated behind the Optimizely experiment **`trial_all_set_experiment`** (mounted globally in `modal-manager.tsx:100`; triggered from `invoice-email.screen.tsx:551` after sending an estimate). It is the **only** conversion path that sets **neither `fullyPaid` nor** (currently) any estimate-side marker — it saves only the new invoice. **It must also write `convertedTo` and be guarded.** Because it is flag-gated and shows at most once per install, it will not surface in casual manual testing — call it out explicitly for QA.

**Change.**
1. In mobile `makeInvoiceFromEstimate` **and** web **`InvoiceModel.convertEstimate()`** (not `estimateToInvoice.ts`), **write the new invoice remoteId back onto the source estimate's `convertedTo`** and save. This is a **best-effort** client write (NOT atomic — see WI-3/M8); the authoritative lock is the server checkout gate (WI-4).
2. Guard **all 5 call sites** (3 standard mobile + web `InvoiceRow.tsx:109` + the flag-gated mobile promo modal): if `convertedTo` is non-empty, **hide/block the "Convert to Invoice" action** (M7) and show a **"duplicate & convert" prompt** (M8) — see design note below. The promo modal (`trial_all_set_experiment`) must also write `convertedTo` on its convert.

> **Design refinement (Slack, Liz 2026-07-14, [p1784070...](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784067659924269?thread_ts=1784052946.973869)).** Seth flagged that a hard lock adds friction for merchants who use estimates as **templates**. Liz confirmed the lock stays (hard technical constraint — checkout can't disambiguate which invoice a multiply-converted estimate points to), but the re-convert prompt should let the merchant **duplicate AND convert in one action**, not force two separate steps (duplicate, then convert). Implement the prompt as a single combined action. ⚠️ **Confirm this is in v1 scope** at the design meeting before building the combined action — a plain "duplicate first" prompt is the fallback if deferred.

**Acceptance criteria**
- First manual convert appends to `convertedTo` and saves the estimate (via `InvoiceModel.convertEstimate()` on web, `makeInvoiceFromEstimate` on mobile).
- After conversion, the Convert action is hidden across **all 5** entry points (incl. the flag-gated promo modal); the "duplicate & convert" prompt is shown on attempt.
- Duplicating produces a fresh estimate with empty `convertedTo` (payable/convertible again).
- A converted estimate cannot be re-paid even if the client write is lost/raced — WI-4 (server) blocks it.

**Note.** This is the merchant UI slice — a natural fork target once the ticket splits. Web parity via `InvoiceModel.convertEstimate()` tracked here too. Client atomicity is explicitly NOT relied upon (M8).

---

### WI-6 — Retroactive payability migration + rollout (M4) — HARD PRECONDITION for flag-on

> 📖 **What "retroactive" actually means (clarified 2026-07-15).** The requirement "all existing estimates with deposits become payable retroactively" reads as: an **existing (old) estimate** on which the merchant **now adds a deposit** (via the net-new estimate-deposit capability, IS-11406) becomes payable — **not** "estimates that already carried a deposit." Deposits on estimates are **net-new**: the deposit editor has **always** been hard-gated to `docType === DOCTYPE_INVOICE` (web `PaymentDeposit.tsx:96`; mobile `invoice.screen.tsx:305-309`), longstanding across git history. **So no historical estimate can carry a deposit** — the deposit *field* is on the shared `setting` object, but no UI/write path ever set it on a docType=1 document. "Retroactive" therefore applies to old estimates going *forward* (add a deposit now), not to a pre-existing deposit population.

> ⚠️ **H3 population is EMPTY, but the risk is FORWARD-GOING (revised 2026-07-15).** The earlier H3 framing ("historical converted **deposit**-estimates become payable on flag-flip") assumed such documents exist. They don't — per the deposit-gating finding above, **zero historical estimates have a deposit**, so there is no historical deposit-estimate (converted or not) to re-open. The `fullyPaid`-safety-floor point still stands as *true* (`getInvoiceBalance.ts:55-58` returns `deposit.amount` early, never reads `fullyPaid`) but is **moot** for historical data because no old estimate has a deposit to trigger it. **The real, live hole is forward-going:** once IS-11406 opens the deposit editor on estimates, a merchant can add a deposit to an **already-converted** estimate (the estimate document persists post-conversion, `deleted:false`, `docType:1`). Deposit set + payments enabled + (pre-guard) empty conversion marker → the live payability check returns **payable** → a buyer on the old estimate link could pay a deposit on a doc already converted & collected. **This is why the conversion lock (`convertedTo` + WI-4) is load-bearing and forward-going — not merely a historical backfill.** The lock must key off *was-converted* (`convertedTo`), independent of *when* the deposit was added.

**Consequence for rollout.** Because no historical estimate has a deposit, a **historical `convertedTo` backfill is likely UNNECESSARY as a data-safety gate** (the population it protects — old converted deposit-estimates — is empty). ⚠️ **Confirm with a cheap audit query** (count `docType==1 AND (depositType!=0 OR depositRate>0 OR depositAmount>0)` → expect 0) before dropping WI-6a. The still-required protection is the **live lock** (WI-4 reads `convertedTo` at Pay time) + writing `convertedTo` on every conversion path **going forward** (WI-3/WI-5). The index (WI-6b) is still needed for the lock's query, regardless of backfill.

**Plan.**
1. **Feature-flag the gate** — Flagsmith flag at the caller level (WI-1) so exposure is controlled/gradual after the preconditions are met.
2. **Data audit** — confirm no existing deposit-estimate carries an unexpected `paymentSuppressed`/`stripePaymentSuppressed` that would surprise on enable; quantify how many old estimates have a frozen `feesType` ≠ current account (drives WI-2 urgency).
3. **Audit FIRST (decides whether WI-6a is needed).** Run `count(docType==1 AND (depositType!=0 OR depositRate>0 OR depositAmount>0))`. Per the deposit-gating finding, expect **0**. If 0 → **WI-6a backfill is unnecessary** (no historical deposit-estimate to protect). If non-zero → investigate (API/non-UI writes) and treat WI-6a as required.
4. **Migration / infra work items:**
   - **WI-6a — Mongo `convertedTo` backfill (CONTINGENT — likely unnecessary).** Only required if the audit finds historical deposit-estimates. Would backfill by matching invoices' `setting.estimateId` back to their source estimate. Server-side migration script.
   - **WI-6b — Mongo index on `convertedTo` (STILL REQUIRED).** The live lock (WI-4) queries this field regardless of backfill. Atlas/infra action — no index-definition code exists in `is-parse-server/cloud`; indexes are managed in Atlas directly. **Requires a Mongo migration to be generated and run.**
   - **WI-6c — Realm v61 backfill branch.** Additive `convertedTo` field (WI-3); a data-backfill branch is only needed if WI-6a is (i.e. probably not) — the field addition itself is unconditional.

**Acceptance criteria**
- Flag off → zero behavior change.
- **Audit query run and count documented before flag-on**; WI-6a scoped in/out based on the result.
- **Index (WI-6b) complete BEFORE the flag is enabled in production** (§0 ordering).
- The forward lock holds: adding a deposit to an already-converted estimate does **not** make it payable (WI-4 reads `convertedTo`).

**Reminder to user:** WI-6b index requires a Mongo migration to be generated and run, and gates the flag. WI-6a backfill is contingent on the audit (expected: not needed).

---

## 4. Open questions & cross-section dependencies

1. **Convert-first cloud function is WI-0 (this ticket, our team)** — no server-side conversion exists today; conversion is client-only. WI-0 builds the net-new Parse cloud function implementing WI-3's write-contract atomically. It is a **hard release gate** for the WI-1 flag (§0), not an external/Juan dependency.
2. **Convert-first gate scope** — verify only the checkout-page eligibility callers (WI-1) need docType relaxation and the payment-time validators pass on the invoice (post-WI-0). Confirms the §0 assumption that leaves payment-time validators unrelaxed.
3. **3-writer conversion drift** — mobile `makeInvoiceFromEstimate`, active web nextjs `transformDocumentSettings` (`document-transformation-utils.ts`), and the net-new WI-0 server cloud function must all append `convertedTo`. Only WI-0 is atomic (M8); WI-4 is the authoritative lock. ✅ **On `feesType` the two active clients already AGREE** (both re-supply the account default — mobile `use-cases.ts:55`, nextjs `utils.ts:106`); no fix needed. WI-0 must be built to match them (re-supply from account). The legacy `client/InvoiceModel` copies verbatim but is dead code. Consider consolidating to one canonical server transform (tracked, not required for v1).
4. **abandoned-cart** (WI-1 caller #4) — estimates will start qualifying; confirm intended.
5. **Email conditional render** (P5/I2) — separate spike doc.
