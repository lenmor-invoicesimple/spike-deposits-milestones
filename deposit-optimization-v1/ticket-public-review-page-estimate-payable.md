> ✅ **CORRECTIONS APPLIED (2026-07-15)** — adversarial-review defects fixed in this revision. See [`self-review-findings.md`](./self-review-findings.md) for the full findings. Fixed here: WI-A no longer claims to relax is-checkout eligibility — that runs through the *same* gate as ticket #1 WI-1, so WI-A is now marked **blocked on ticket #1** (H4); WI-C now specifies a payability condition on `hideInvoiceBalance` so it can meet its own AC (M1); WI-D "comes for free" corrected — the DEPOSIT DUE row is docType-gated and won't auto-hide post-conversion (M2); shared-package **mobile blast radius** + version-cut/pin-bump added (M4). Seth's top+bottom pattern (WI-E) reviewed **clean** — no design defect.

# Ticket — Public Review Page: Pay-Deposit for Estimates

**Epic:** IS-11250 — Deposit Optimization v1
**Date:** 2026-07-14
**Owner:** Lenmor Dimanalata
**Status:** Draft — implementation-ready spec
**Source of truth:** Liz's design-meeting decision ([Slack p1784052946973869](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869)); relates to items **B (buyer pre-conversion)**, **P3 (post-conversion QR + CTA hidden)**, and Seth's top+bottom button pattern.
**Sibling ticket:** [`spike-email-estimate-vs-invoice.md`](./spike-email-estimate-vs-invoice.md) (email surface). **Related:** [`ticket-payable-estimates-global.md`](./ticket-payable-estimates-global.md) (payability engine — see gate overlap in §5).

---

## 1. Scope

The public buyer-facing document review page — **`is-web-app` route `/v/[documentId]`** — must let a buyer **pay the deposit on a payable estimate**, and must strip all payment affordances once the deposit is paid / estimate converts. Plus Seth's new pattern: render the action buttons at **both top and bottom** of the page.

**The page = two layers:**
- **Action buttons** (local): `is-web-app/nextjs/app/(public)/v/[documentId]/components/invoice-actions/` (`PdfButton`, `PayButton`, `PrintButton`, `SignButton`).
- **Document body** (package): `@invoice-simple/invoice-viewer` (`is-packages/packages/invoice-viewer`) — renders totals, the DEPOSIT DUE / Balance Due row, QR "Scan to pay online" block, and the surcharge disclaimer.

**In scope**
- Unlock estimate payability so the review page produces a `payUrl` for a deposit-estimate (WI-A).
- "Pay Deposit" button + label on the review page (WI-B).
- In-document payment block for estimates: DEPOSIT DUE line, QR block, fee disclaimer (WI-C).
- Post-payment / converted state: all payment affordances hidden (WI-D).
- Seth's top+bottom repeated-button pattern (WI-E).

**Out of scope — routed elsewhere**
- The estimate **email** (deposit-due line, labeling) → [`spike-email-estimate-vs-invoice.md`](./spike-email-estimate-vs-invoice.md).
- The **convert-first cloud function** at Pay Now (server-side conversion, invoice-ID→processor) → [`ticket-payable-estimates-global.md`](./ticket-payable-estimates-global.md) **WI-0 (our team)**. This page routes to checkout; the conversion happens downstream.
- The core `findNotPayableReason` docType relaxation in `invoice-payments-status.ts` → [`ticket-payable-estimates-global.md`](./ticket-payable-estimates-global.md) WI-1. **This ticket's server `payUrl` (WI-A) is BLOCKED on that gate** (see §5, H4) — it owns only the *client-side* SDK gates + `hideInvoiceBalance`, which are genuinely additional.

---

## 2. How the page works today (verified 2026-07-14)

- Page: `nextjs/app/(public)/v/[documentId]/page.tsx` → `handlePublicInvoice()` → `<PublicDocumentFeature>`.
- Emailed "Review" link (`api` `mkDownloadUrl`, `invoice-sending.ts:85`) → `/v/{hashId}/...` → served by is-web-app. is-services links here too (`getPublicDocumentViewUrl` → `${WEB_APP_URL}/v/${id}`). This is the canonical review page.
- **Everything payment-related keys off a single server-computed `payUrl`** from is-services `/checkout/eligibility/{accountId}/{remoteId}`:
  - `public-invoice.ts:43-58` → `getEligibleInvoicePaymentUrlAndVendor` → `get-payments-eligibility.ts:37-59` → is-checkout `eligibility/router.ts:67-72` → `getUnifiedCheckoutUrl` (`is-checkout/src/url/url.ts:16-21`) = is-unifiedxp `/checkout/{accountId}/{remoteId}`.
  - Eligibility logic: `is-checkout/src/eligibility/service.ts` — `isValidSubscription` → `validateDocumentPayable` → `getCurrentPaymentVendorIfPayable` (balance/surcharge/paymentSuppressed, vendor priority). Not eligible → `payUrl = null`.
- **Estimates already render** on this page (body, PDF/Print/Sign) — `getUniversalInvoiceTitle` handles `DOCTYPE_ESTIMATE` (`public-invoice.ts:99-106`); estimate sign path exists. **No Pay button for estimates** — payability is hard-locked to invoices (§5).

---

## 3. Work items

### WI-A — Unlock estimate payability → produce a `payUrl` (blocks B, C) — ⚠️ BLOCKED ON TICKET #1

> ⚠️ **Corrected (H4): the server `payUrl` gate is NOT this ticket's to relax.** The is-checkout server eligibility path is `eligibility/service.ts:63` → `document-payable.ts:42` → `findNotPayableReason` → `invoice-payments-status.ts:58` — **the exact single docType gate that `ticket-payable-estimates-global.md` WI-1 owns.** There is no separate is-checkout gate to relax here. **WI-A cannot produce a `payUrl` for estimates until ticket #1 WI-1 lands.** This is a hard cross-ticket blocker, not parallel work.

**What actually splits between the two tickets:**

| Layer | Gate | file:line | Owned by |
|---|---|---|---|
| Server `payUrl` (is-checkout eligibility) | `findNotPayableReason` docType gate | `invoice-payments-status.ts:58` | **Ticket #1 WI-1** (this ticket depends on it) |
| Client SDK payability (invoice-viewer QR) | PayPal SDK payable `docType === DOCTYPE_INVOICE` | `is-packages/packages/is-paypal-sdk/src/invoice-payable.ts:4-13` | **This ticket, WI-C** |
| Client SDK payability (invoice-viewer QR) | Stripe SDK payable | `is-packages/packages/is-stripe-sdk/src/invoice-payable.ts:4-13` | **This ticket, WI-C** |

**So WI-A reduces to: consume the `payUrl` that ticket #1 makes possible**, gated on the same eligibility definition (deposit set + payments globally enabled + `convertedTo` empty), behind the same flag. The only *new* payability relaxation this ticket owns is the **client-side SDK gates** (moved to WI-C, where the QR/viewer work lives).

**AC:** once ticket #1 WI-1 is enabled, a deposit-estimate with an accepting account and no conversion returns a `checkoutUrl` from `/checkout/eligibility`; a plain estimate (no deposit) returns none. **Verify against ticket #1's flag being on** — with it off, `payUrl` is null and this whole ticket is inert (acceptable/expected).

### WI-B — "Pay Deposit" button (top; bottom via WI-E)

`invoice-actions.tsx` renders `PayButton` iff `props.payUrl` truthy (`:15,38`), label `public.invoice.actions.payOnline` ("Pay Online"). No docType check here today.

**Change:** when the document is a deposit-estimate, label the CTA **"Pay Deposit"** (docType/payment-schedule-aware label; add i18n key). Button routes to the existing `payUrl` (→ is-unifiedxp checkout, where convert-first fires — Buyer Pre-Conversion ticket). No new routing logic here — it rides the `payUrl` from WI-A.

**AC:** payable deposit-estimate shows "Pay Deposit"; payable invoice still shows its normal pay CTA; non-payable → no button.

### WI-C — In-document payment block for estimates (invoice-viewer)

All three sub-elements live in the viewer package and are currently invoice-only:

- **QR "Scan this code to pay online"** — `invoice-viewer/.../common/payments/PaymentInstructions.tsx` (`PaymentsQRCodeContent` :131-144), gated by `useHidePaymentsQRCode()` (`lib/useHidePaymentsQRCode.ts:6-17`): `isPayable = isInvoicePayable(invoice) || isStripeInvoicePayable(invoice)` (both require `DOCTYPE_INVOICE`) AND `qrCode.payUrl` AND `qrCodeAllowed`. → won't show for estimates today.
- **DEPOSIT DUE line** — `useDueLabel()` (`lib/useDueLabel.ts:5-16`) returns `depositDue` label when nearest payment `paymentType === DEPOSIT`, else `balanceDue`; dict `pdfTotalDepositDue`/`pdfTotalBalanceDue` (`dictionary.ts:47-48`); rendered as `invoice-balance` row in each `templates/styleN/summary/InvoiceSummary.tsx`. **`hideInvoiceBalance` (`common/summary/InvoiceBalance.tsx:11-17`) hides the whole row purely on `docType === DOCTYPE_ESTIMATE`** — **it has NO payUrl/payability input** (M1).

> ⚠️ **M1 — a naive docType relaxation breaks the AC.** If we relax `hideInvoiceBalance` on docType alone, DEPOSIT DUE shows for **every** deposit-estimate — including non-payable ones and (post-conversion) converted ones — which contradicts "non-payable shows none" AND the post-payment mock. `hideInvoiceBalance` **must gain a payability condition** (payUrl present / payable), not just a docType allowance.

- **Fee disclaimer** — `pdfSurchargeNotes` ("An Online Payment Fee will be charged if this invoice is paid online.", `dictionary.ts:33-34`), rendered by `SurchargeContent` (`PaymentInstructions.tsx:196-212`), gated by `hideSurcharge()` (needs `qrCode.payUrl` + `feesType === SURCHARGE`). String is hardcoded "invoice" wording.

**Change:**
1. Extend the **client SDK payability gates** — `is-paypal-sdk/invoice-payable.ts` + `is-stripe-sdk/invoice-payable.ts` (both hard-check `docType === DOCTYPE_INVOICE`) — to accept deposit-estimates. **This is the client-side gate family (see §5); coordinate with ticket #1 so client and server share one deposit-estimate rule.**
2. **Relax `hideInvoiceBalance` with a payability condition, NOT docType alone (M1):** show the DEPOSIT DUE row only when the document is a deposit-estimate **and** payable (payUrl present / passes the viewer payability check). This is what lets the AC's "non-payable shows none" and WI-D's post-conversion hide both hold.
3. Add an **estimate variant** of the fee disclaimer ("...if this **estimate** is paid online" — matches the mock) as a new dict label; select by docType.

**AC:** payable deposit-estimate shows DEPOSIT DUE + QR + fee note (estimate wording); **non-payable / plain estimate / converted estimate shows none** (all three depend on the payability condition, not docType); invoices unchanged.

> ⚠️ **M4 — shared-package mobile blast radius.** `invoice-viewer`, `is-paypal-sdk`, and `is-stripe-sdk` are published packages pinned by **both** web (`invoice-viewer@10.0.25`) and mobile (`@10.0.29`). Relaxing `hideInvoiceBalance` / `isInvoicePayable` in these packages changes **mobile document rendering too**. This ticket must include: a **package version cut**, a **pin bump in both apps**, and **mobile rendering QA** (a deposit-estimate should not suddenly show a DEPOSIT DUE row on mobile unless intended). The "M" size estimate previously ignored this coordination + retest.

### WI-D — Post-payment / converted state (P3)

**Comes free for the Pay button / QR / disclaimer — NOT for the DEPOSIT DUE row (M2).** Pay button, QR, and fee disclaimer all key off `payUrl`, so once the deposit is paid / estimate converts and eligibility returns `payUrl = null`, **those three drop out automatically.** Document identity ("ESTIMATE EST0001") and totals render independently of `payUrl` and stay.

> ⚠️ **M2 — the DEPOSIT DUE row is the exception.** It is gated by `hideInvoiceBalance` (docType only, no `payUrl`). Conversion mints a new invoice but leaves the **source estimate at `docType=1`**, so a naive docType relaxation would keep DEPOSIT DUE showing **forever** after payment/conversion. The "everything keys off one `payUrl`" framing is wrong — there are **≥3 independent gate families**: (a) server `payUrl` (Pay button), (b) client SDK `isInvoicePayable` (QR/disclaimer), (c) docType row visibility (DEPOSIT DUE). Only (a) and (b) auto-hide.
>
> **Fix:** the M1 change (add a payability condition to `hideInvoiceBalance`) is what makes the DEPOSIT DUE row hide post-conversion — because a converted estimate is no longer payable (WI-4 lock → payUrl null). So WI-D depends on WI-C/M1 being done correctly; it is NOT free.

> 🎨 **Resolved against real behavior + Figma (2026-07-15).** Verified against the current **invoice** deposit checkout (staging screenshots):
> - **Checkout surface is fine, inherited for free.** The `checkout.staging` page already recalculates correctly: right-panel flips `DEPOSIT DUE` → **`DEPOSIT PAID`** after payment, and the document body flips `DEPOSIT DUE $100` → **`BALANCE DUE $20`** (with a `Deposit -$100` line) via `useDueLabel()` + the calculator. Post-convert-first the estimate checkout operates on an **invoice** (docType=0), so it inherits all of this. **No "DEPOSIT DUE forever" risk on checkout.**
> - **Figma mock error to flag to design:** the "Payment Successful" screen in the mock still shows `DEPOSIT DUE $500` — it should read **`DEPOSIT PAID`** (matching real invoice behavior). This is a *mock* fix; the implementation already does the right thing inherited from invoices. No code work, just tell design.
> - **M2 narrowed:** the only surface needing our change is **this `/v/` review page** (the estimate view). It needs `DEPOSIT DUE` shown when payable and hidden when not — the **M1 payability condition** covers both.
> - **⚠️ CORRECTED (2026-07-15) — the `/v/` page IS the post-conversion landing surface (per Slack).** Earlier I wrote "buyer redirected away, no post-payment state needed" — **that was backwards.** Slack requirement: *"If buyer returns via old email link, they are redirected to the **public estimate URL** [`/v/`], **not** the checkout."* So the buyer only sees checkout-success **the first time, right after paying**; any later return to the old link lands them **here on `/v/`**, showing the (now-converted) estimate with **QR + Pay Deposit CTA + DEPOSIT DUE all hidden** (`convertedTo` non-empty → not payable). This converted state is **required and is exactly what M1/WI-C delivers** — not "nothing to design." The redirect *into* `/v/` (from stale checkout/email links) is owned by the [Buyer Post-Conversion ticket, WI-P4](./ticket-buyer-post-conversion.md).

**Requirement:** ensure the WI-A/ticket-#1 eligibility gate returns `null` once `convertedTo` is non-empty (the conversion lock), AND `hideInvoiceBalance` keys off that same payability (M1).

**AC:** after conversion, the review page shows only "Download as PDF" + the document body (identity + totals); **no Pay Deposit button, no QR, no fee disclaimer, and no DEPOSIT DUE row** (assuming design confirms it should hide). Verify the is-web-app button gate, the viewer's QR gate, AND the DEPOSIT DUE row all null-out together.

### WI-E — Seth's top+bottom button pattern

Today buttons render once: `renderActions()` before `renderInvoiceContent()` (`public-document.feature.tsx:81-103`, placed before content in all 3 return branches). **No existing duplication pattern.** Naive re-call double-renders the `<header>` and the bundled `EstimateSignatureModal`.

**Change:** small refactor — extract the button **row** from the `<header>` wrapper into a reusable component; hoist/dedupe the signature modal so it mounts once; add a `position: 'top' | 'bottom'` prop; render the row before and after the document body.

**AC:** the same action buttons (incl. Pay Deposit) render at top and bottom; the signature modal mounts exactly once; no duplicate semantic `<header>`; applies to all doc types (generic presentation change).

---

## 4. Acceptance criteria (rollup)

- Deposit-estimate on `/v/{id}` (with ticket #1 flag on): renders body + DEPOSIT DUE + QR + fee note + "Pay Deposit" (top & bottom); Pay Deposit → is-unifiedxp checkout.
- Post-conversion: **all** payment affordances gone — Pay button, QR, fee note, **and DEPOSIT DUE row** (via M1 payability condition); identity + totals remain.
- Plain estimate / non-payable estimate / invoice: no regression (mobile included — M4).
- All behind the epic feature flag; server `payUrl` (WI-A) depends on ticket #1 WI-1.

## 5. Gate relationship with the global-payments ticket ⚠️ (corrected H4)

**Three gate families — and one of them is ticket #1's, not this ticket's.** The earlier "two distinct families, neither covers the other's" framing was **partly false**: the server `payUrl` gate this ticket needs is *the same gate* ticket #1 owns.

| # | Gate family | file:line | Owner | Note |
|---|---|---|---|---|
| (a) | **Server `payUrl`** (is-checkout eligibility) | `eligibility/service.ts:63` → `document-payable.ts:42` → `findNotPayableReason` → `invoice-payments-status.ts:58` | **Ticket #1 WI-1** | This ticket has **no separate gate** here — WI-A is **blocked on ticket #1** (H4). |
| (b) | **Client SDK payability** (invoice-viewer QR/disclaimer) | `is-paypal-sdk/invoice-payable.ts:4-13` + `is-stripe-sdk/invoice-payable.ts:4-13` (`docType === DOCTYPE_INVOICE`) | **This ticket, WI-C** | Genuinely additional — ticket #1 does NOT touch these. |
| (c) | **DEPOSIT DUE row visibility** | `hideInvoiceBalance` (`InvoiceBalance.tsx:11-17`) (docType only) | **This ticket, WI-C (M1)** | Needs a payability condition added, not just docType. |

**The `sdks.ts` provider gates ticket #1 calls "docType-agnostic, no work" are a FOURTH thing** — they're the is-services provider-accepting checks, unrelated to (b). Do not conflate "ticket #1 says SDK gates need no work" (that's `sdks.ts`) with the client SDK `invoice-payable.ts` gates (b), which **do** need work here.

**Coordinate so client-side (b) and server-side (a) payability agree on "deposit-estimate is payable"** — ideally one shared rule (see §6 Q1). Divergence = QR shows but no `payUrl`, or vice versa.

## 6. Open questions

1. **Client vs server payability parity** — the viewer's client SDK payability (b) and is-services server eligibility (a) should share **one** source of truth for the deposit-estimate rule (deposit set + payments enabled + `convertedTo` empty). (Recommend: yes — otherwise QR/`payUrl` diverge.)
2. **`payUrl` for estimates end-to-end** — confirm is-unifiedxp checkout accepts a docType=1 document at the URL, or whether **convert-first (ticket #1 WI-0)** mints the invoice before the checkout page renders. This ties to ticket #1's §0 release sequencing.
3. **QR `payUrl`** — the QR uses `qrCode.payUrl`, nulled for `LIGHT_INTENT` vendor. Confirm the deposit-estimate vendor path populates it.
4. **"Pay Deposit" charge amount (RESOLVED 2026-07-15)** — confirmed with product: **"Pay Deposit" charges the deposit only**; the converted invoice carries the remaining balance. Implementation must ensure the checkout charge = deposit amount, not full balance. (Was: nothing in the traced checkout path restricts the charge — so this must be enforced/verified in the convert-first + checkout path.)
5. **DEPOSIT DUE post-payment (RESOLVED 2026-07-15)** — verified against real invoice checkout: checkout/success already flips to **`DEPOSIT PAID`** + `BALANCE DUE` (inherited for free post-convert-first). The Figma "Payment Successful" mock incorrectly shows `DEPOSIT DUE $500` → **flag to design to relabel "Deposit Paid"** (mock fix only, no code). On the `/v/` review page, M1's payability condition handles show-when-payable / hide-when-converted. **The `/v/` page IS the post-conversion landing surface** (Slack: old email link → public estimate URL, not checkout) — so the converted state (all payment affordances hidden) is required, and M1/WI-C delivers exactly that.

> **Checkout success/failed page labels — code confirmed correct (2026-07-15).** These pages live in **`is-unifiedxp`** (the checkout app), scope of the **Buyer Post-Conversion ticket** (not yet written), NOT this `/v/` review-page ticket. Verified in code — no code change needed, only the Figma mock:
> - **Success page** — `PaymentSuccess/PaymentSuccess.tsx:86` renders `PaymentDepositPaidDetails` → **"DEPOSIT PAID"** ✓
> - **Failed page** — `PaymentsError/PaymentError.tsx` → `PaymentDetails` → `PaymentDepositDetails` → **"DEPOSIT DUE"** ✓ (correct — deposit is still due after a failed payment)
> - Labels defined once in `PaymentDetails/messages.ts:3-4` (`depositDue: 'DEPOSIT DUE'`, `depositPaid: 'DEPOSIT PAID'`). Post-convert-first these run on an **invoice**, so the estimate flow inherits both for free.
> - **Only action:** relabel the Figma "Payment Successful" mock (success) "Deposit Due" → "Deposit Paid." The failed-page mock, if it shows "Deposit Due," is already correct. **Now captured in [`ticket-buyer-post-conversion.md`](./ticket-buyer-post-conversion.md) WI-P3.**
6. **Effort — revised up from M.** Spans is-web-app (buttons + WI-E refactor), **is-packages (2 client SDK gates + invoice-viewer `hideInvoiceBalance` + dictionary)**, plus a **shared-package version cut + pin bump in both web and mobile + mobile QA** (M4), and is **hard-blocked on ticket #1 WI-1** (H4). Realistically **M–L**, not M.
