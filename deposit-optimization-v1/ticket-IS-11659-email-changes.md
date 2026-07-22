# IS-11659 — Payable Estimate: Email changes

**Ticket:** https://everpro-tech.atlassian.net/browse/IS-11659
**Figma:** https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=6125-46057
**Status:** Code edits made (uncommitted, on the verified-LIVE path). SWU template copy + `deposit_due` still open.
**Depends on:** IS-11663 (once estimates are payable, `paymentSettings.isEligible` flips true for them → they get the invoice/payments email).

Juan's framing: don't restyle the old estimate email — instead let payable estimates flow into the **invoice/payments email** (the deposit email), because after IS-11663 they go through the "payable" case. Adjust copy to say "Estimate" instead of "Invoice."

---

## RESOLVED: which email path is live (agent-verified 2026-07-17)

There are two email builders in `api/`. **The LIVE path is `email.ts` `send()` → `getEmailTemplateData()`.**
- Reached by **`v2EmailInvoiceHandler`** (`/api/v2/email-invoice`, route `routes/v2-root.ts:145`; calls `send()` at `invoice-sending.ts:480`) — web sends here.
- Also reached by **`internal-notifications.ts:133`** (automated/recurring sends).
- `invoice-sending.ts:820-985` is the **LEGACY v1** path (`v1EmailInvoiceHandler`, declared `:504`). Fully self-contained, does NOT call `email.ts`, has NO payments template. **Dead for modern buyer email.**

**Plan-doc corrections:**
- "v2 handler `invoice-sending.ts:480` routes through `email.ts` `send()`" — **CONFIRMED.**
- "v3 handler `app.ts:296`" — **REFUTED.** `app.ts` is a 31-line bootstrap file; no line 296, no v3 `/email-invoice` route exists. The "v3" path is actually `internal-notifications.ts:133` (mislabeled in plan doc).

**Consequence:** My `email.ts` edits (below) ARE on the live path. The second `getSubject` call at `invoice-sending.ts:907` is in the **v1 legacy handler** → NOT live → does NOT need the docType fix. (Earlier worry that my edits were misplaced: resolved — they're correct.)

## RESOLVED: the deposit email is NOT a separate template

The screenshot ("Deposit request from {company}", USD $120.00, REVIEW AND PAY, Visa/MC/Amex/PayPal logos) is the **same** `paymentsSwuInvoiceTemple` template rendering its **deposit variant** when `is_deposit_eligible` is present.
- Template id: `email.ts:52` `paymentsSwuInvoiceTemple = env.require('SWU_PAYMENTS_INVOICE_TEMPLATE')`, selected by `getPaymentsSWUTemplateId()` (`email.ts:550`) for vendor paypal/stripe.
- Deposit flag: `email.ts:767` `if (isEligibleForDeposit) emailData.is_deposit_eligible = ...`.
- The "Deposit request" subject, "REVIEW AND PAY", body copy, and the $120 figure are **rendered SWU-side** — NOT in this repo. Grep of all /is5 for "Deposit request"/"REVIEW AND PAY" returns nothing.
- Pay-button + logos template = `SWU_PAYMENTS_INVOICE_TEMPLATE`; plain no-button fallback = `swuInvoiceTemplate` (`SWU_INVOICE_TEMPLATE`, `email.ts:679`).
- Deposit AMOUNT: API passes only the boolean `is_deposit_eligible`. The `$120` is computed SWU-side / on the checkout page reached via `pay_url` — NOT injected as a `deposit_due` variable today. (`getDeposit().amount` exists at `is-packages/.../calculator/src/invoice/getDeposit.ts:163` but is not called by the email builder.)

## Code edits made (UNCOMMITTED, in working tree, on the LIVE path — verified correct)

**1. Routing — `api/src/services/email.ts` (~line 656→660):** payable estimate now falls through to the payments-template branch instead of short-circuiting to `swuEstimateTemplateV2`:
```ts
const isPayableEstimate = docType === DocTypes.DOCTYPE_ESTIMATE && !!paymentSettings?.isEligible;
if (docType === DocTypes.DOCTYPE_ESTIMATE && !isPayableEstimate) {
  template = swuEstimateTemplateV2;      // non-payable estimate → unchanged
} else {
  const paymentsTemplateId = paymentSettings?.isEligible && getPaymentsSWUTemplateId(paymentSettings);
  // ... payable estimate now reaches paymentsSwuInvoiceTemple + is_deposit_eligible (email.ts:663/683)
```
Effect: payable estimate → `paymentsSwuInvoiceTemple`, AND now reaches the `isEligibleForDeposit` computation (which it short-circuited past before — this resolves plan item 10's flagged catch "estimate short-circuits before isEligibleForDeposit is set").

**2. Subject fix — `api/src/util/email-experiment-service.ts`:** `getSubject` made docType-aware (was hardcoded "…sent you an invoice", fires on `isEligible` not docType → payable estimate would get "invoice" subject over "Estimate" body). Added `DocType`/`DocTypes` import. Call site `email.ts:289` now passes `invoice.docType`.
NOTE: this is the **IR A/B experiment subject** ("Action Needed: {name} sent you an {estimate|invoice}"), gated on `ir_subject_email` flag — distinct from the SWU-rendered "Deposit request" subject. The SWU subject is a template-side concern.

**3. `doc` variable** already passes `'estimate'`/`'invoice'` (`email.ts:646` `strDocType`, sent as `doc:` at `:748`) — this is what makes the shared payments template render "Estimate" copy. Available; template must USE it.

## SCOPE (confirmed 2026-07-20)

IS-11659 owns **ALL email-related side-effects of IS-11663** — both (a) copy changes ("Invoice"→"Estimate") AND (b) suppress/guard decisions (whether an email should fire for estimates at all). If an email flow is affected by estimates becoming payable, resolving it — copy OR guard — is in this ticket. Non-email side-effects of 11663 (redirect routers, pay-link/QR) stay in 11663.

## CAUSAL MODEL — how to be sure 11659 covers ALL of 11663's fallout

The IS-11663 gate change (relax `findNotPayableReason` in is-services + `validateDocumentPayable` in is-paypal, both FF+deposit-conditional) pulls exactly **two levers**. Every affected email traces to one:

- **Effect A — estimates become ELIGIBLE.** The is-checkout `checkout/eligibility` endpoint runs `findNotPayableReason`; `api` reads it via `getPayDetailsIfEligible` → `paymentSettings.isEligible`. Emails that key off this flip.
- **Effect B — estimates can now be PAID.** A real Stripe charge / PayPal capture can now occur (impossible before). Emails triggered *by a charge/capture existing* fire for the first time — even though they never call the gate themselves.

Complete affected set = (every email reading the eligibility endpoint) ∪ (every email triggered by a charge/capture). That's the guarantee the sweep below satisfies.

- **Effect A emails:** #1 buyer send email (routing), #6 abandoned-cart (now fires; gated on `isInvoicePayable`→`findNotPayableReason`).
- **Effect B emails:** #4 Stripe receipt+merchant, #5 PayPal receipt+merchant (all four share SWU `tem_x7Vc…`).
- **Edge (in cone only if estimate checkout reachable):** #8 light-intent payment-intent merchant email.
- **Probably NOT gate-caused:** #7 bank-deposit/Layer — fires on manually-recorded/offline payment, not `findNotPayableReason`; may already be possible for estimates independent of 11663. Renders "invoice" copy but arguably a separate ticket.

## FULL EMAIL SWEEP (2026-07-20) — "cover ALL affected templates"

The ticket says "Please cover all email templates affected by Estimates being payable." Our original analysis only covered the buyer send email (#1). A full sweep across api/ + is-services + is-paypal + is-stripe found **9 touchpoints**. Every email that keys off payability/eligibility (not its own `docType===INVOICE` check) now fires for payable estimates. Grouped by risk:

### 🔴 HIGH RISK — fires for estimates, NO docType branch, copy is SWU-side (needs SWU template work, not just repo edits)

| # | Email | Trigger | Template id | File |
|---|---|---|---|---|
| 4 | Stripe **payment receipt** (buyer) + **merchant "you got paid"** | Stripe charge webhook `charge-event-updated.ts:14` — no docType check, loads invoice by paymentIntent.invoiceId. Paid estimate → fires. | `tem_x7VcHp8VmPWd4GRtKDHtThP8` (`is-stripe/.../email/utils.ts:14`) | `is-stripe/src/utils/email/render-email-template.ts:72` (buyer), `:156` (merchant) |
| 5 | PayPal **payment receipt** (buyer) + **merchant "you got paid"** | PayPal order-capture success `payment-event-email.ts:39` — no docType filter. | **SAME** `tem_x7VcHp8VmPWd4GRtKDHtThP8` (`is-paypal/.../email-templates/utils.ts:16`) | `is-paypal/packages/server/src/utils/send-with-us/email-templates/payment-email.ts:160` (buyer), `:235` (merchant) |
| 6 | **Abandoned-cart / payment reminder** (buyer) | `scheduled-handler.ts:42` gates on `isInvoicePayable()` → `findNotPayableReason` (the EXACT gate IS-11663 removes). Scheduling has no docType filter. Estimate with a checkout view → reminder. | `tem_Hcqm4wpPTbpMwgJ4cCtkjBbD` (`is-abandoned-cart/.../email-config.ts:3`) | `is-abandoned-cart/src/utils/send-reminder-email.ts:12` |

**#4 and #5 share the SAME SWU template `tem_x7VcHp8VmPWd4GRtKDHtThP8`** — fixing that one SWU template covers Stripe + PayPal receipts AND both merchant notifications.

### 🟡 FLAG FOR GROOMING — uncertain whether estimates reach them

| # | Email | Uncertainty | File |
|---|---|---|---|
| 7 | Bank-deposit / bookkeeping merchant "payment received" | Fires on Layer `InvoicePayment.created`. Depends on whether estimates can get bank-deposit payments recorded. | `is-bookkeeping/src/utils/merchant-email.ts:69`, tmpl `tem_63WBtHddcpgTYx46V9p9vbKd` |
| 8 | Checkout "payment intent requested" merchant email | Fires when buyer clicks pay on a doc without payments enabled. Depends on whether estimate checkout surfaces that CTA. | `is-checkout/src/intent/service.ts:51`, tmpl `tem_vqdyKHfJ9WgGjQMqdBSRGPrQ` |
| 3 | v1 mobile send handler | Has its OWN `docType != INVOICE` guard (`getPayUrlForEmailInvoiceV1` `invoice-sending.ts:1011`) → estimates never get pay button here. Confirm payable estimates route through main `send()` (#1), not v1. | `api/src/controllers/invoice-sending.ts:869` |

### ✅ DONE / already docType-aware

- **#1 Main buyer send email** (`email.ts` — our edit, done).
- **#2 `getSubject` A/B subject** (`email-experiment-service.ts` — our edit, done). ⚠️ CAUTION: the v1 path `invoice-sending.ts:907` calls `getSubject(accountId, fromName)` with NO docType → defaults to "invoice". Safe ONLY because v1 is invoice-gated (#3). If #3's gating changes, this breaks.
- **CC email subject** `getCCEmail()` `email.ts:544` — already `'Invoice' : 'Estimate'`.
- **#9 Acorn approval email** `is-acorn-webhook/.../send-document-email.ts:10` — already passes `document_type: docType`; trigger is Acorn approval, not payability. Low priority.

### Checked & NOT affected (estimates can't reach)
- Recurring-payments automatic emails (`is-payments/.../build-templates.ts`) — recurring invoices only (series query is `docType===0`). All 4 lifecycle emails (enabled/success/failure/cancelled × buyer+merchant) are unreachable.
- Recurring invoice sent notification (`internal-notifications.ts:184`) — recurring invoices only.
- Report-invoice-to-support — internal fraud-report flow, not per-document buyer copy.

### Implication for ticket scope
IS-11659 is **bigger than copy on the send email.** The real deliverable is making these SWU templates estimate-aware (esp. the shared `tem_x7Vc...` payment-event template + the abandoned-cart template). Repo edits are mostly done; the bulk of remaining work is SWU-side + deciding whether abandoned-cart/payment-intent emails SHOULD fire for estimates at all (may want a docType guard rather than just re-copy).

## COMPLETE EMAIL SWEEP (2026-07-20, agent-verified across api + is-services + is-paypal)

Full inventory of every invoice-related email send across all three repos, with IS-11663 impact verdict.

### AFFECTED — in scope for IS-11659

| # | Email | Repo / template | Lever | Why affected | Action |
|---|---|---|---|---|---|
| 1 | Buyer send email (v2) | api `email.ts` / `SWU_PAYMENTS_INVOICE_TEMPLATE` | A | Core routing site — estimate→payments template on `isEligible` | Copy: routing + `doc` variable |
| 1b | A/B subject (`getSubject`) | api `email-experiment-service.ts` | A | Part of send; hardcodes "invoice" | Copy: docType-aware subject |
| 1c | CC copy to sender | api, inherits send template | A | Inherits #1 template; subject already estimate-aware in `getCCEmail()` | Inherits automatically — verify |
| 2 | Abandoned-cart reminder | is-abandoned-cart / prod `tem_Hcqm4wpPTbpMwgJ4cCtkjBbD`, staging `tem_R4Jym6MFBBRwx6xYRYkkfptT` | A | `isInvoicePayable→findNotPayableReason` gate flips; no docType variable in renderData today | **Product decision: copy or suppress.** Copy → add `doc` to renderData + SWU edit. Suppress → docType guard in `scheduled-handler.ts` |
| 3 | Stripe receipt — buyer + merchant | is-stripe / `tem_x7VcHp8VmPWd4GRtKDHtThP8` | B | Fires on charge; estimate can now be charged; no docType in renderData | Copy: add `doc` to `getSharedEmailTemplateData` in `render-email-template.ts:59` |
| 4 | PayPal receipt — buyer + merchant | is-paypal / `tem_x7VcHp8VmPWd4GRtKDHtThP8` (same template) | B | Fires on capture; estimate can now be captured; no docType in renderData | Copy: add `doc` to `getSharedEmailTemplateData` in `payment-email.ts:144`. **Note:** #3+#4 share `tem_x7Vc…` → one SWU edit covers all 4 emails. Both also require updating `SharedEmailPaymentData` type in `@invoice-simple/is-msg` (new package version). |
| 5 | Light-intent "set up payments" (merchant) | is-checkout / `tem_vqdyKHfJ9WgGjQMqdBSRGPrQ` | A | Reachable once estimate passes eligibility gate (`validateDocumentPayable:63` → `findNotPayableReason`); no docType in renderData | **Product decision: likely suppress.** Guard in `requests.ts:33` (keeps `trackIntentEmailEvent` firing). Copy → add `doc` to renderData in `service.ts:109`. |

### NEEDS VERIFICATION — conditionally in scope

| # | Email | Repo | Question |
|---|---|---|---|
| v1 | Buyer send email (v1 legacy) | api `invoice-sending.ts:868` | Confirm payable estimates always route through v2 `send()`, not v1. v1 hard-returns no pay-URL for non-invoices (`:1010`) — if estimates ever hit v1 they silently get no pay button. Verified dead for web, but confirm mobile routes. |
| v1-sms | v1 SMS pay-URL | api `invoice-sending.ts:228` | Same v1 path — do estimates need pay URLs over SMS? |
| 6 | Bank-deposit / Layer (merchant) | is-bookkeeping / `tem_63WBtHddcpgTYx46V9p9vbKd` | **Not gate-caused.** `InvoicePayment.created` webhook has no `findNotPayableReason` in path. One question: can Layer record a payment against a `docType===1` estimate today? If **no** → nothing to do. If **yes** → pre-existing issue, separate ticket. |

### NOT AFFECTED — confirmed out of scope

| Email | Repo | Why safe |
|---|---|---|
| Invoice/Estimate opened (merchant) | api / `tem_Px3BRSjBK8Bd3YWgdKR3DDVJ` | Cosmetic `doc` only, single template, already renders estimates correctly. No payability gate. |
| Estimate signed (merchant) | api / `tem_yRvJJGjvphJP8ThHMqPVXFJ9` | Always an estimate; no gate, no branch. |
| Acorn financing FUNDED (merchant) | is-services / `tem_RvpkBMQ3VQw4kj9qg3KCSVKf` | Acorn webhook trigger, independent of payability. Already passes `document_type: docType` to renderData — already estimate-aware. |
| Recurring auto-payment: enabled / success / failure / cancelled (buyer + merchant) | is-payments / `tem_GXvTQhVf…` + `tem_btmMv…` | **Unreachable** — estimates cannot be recurring series (source series query filters `docType===0`). All 4 lifecycle × 2 recipients = 8 emails are safe. |
| Recurring invoice notification (client + merchant) | is-recurring-invoices → api template | **Unreachable** — recurring invoices only, same reason. |
| Report invoice to support | api / `SWU_REPORT_INVOICE_TEMPLATE` | Internal fraud-report flow, no docType/payability logic. |
| Password reset / OTP / account deletion / multiple-accounts / premium welcome | api (various `SWU_*` + `tem_BdFb…` `tem_h4wg6…`) | Account/auth emails — not document-scoped. |
| PayPal merchant signup thank-you | is-paypal / `tem_7PTrgC9Pwtj6PrVxX6Kptvk7` | Onboarding webhook, no document. |
| Expenses image export | is-services / `tem_jcCmgvGdmVVFvfyjhXC39FVc` | Expenses feature, unrelated to payability. |

### Key reviewer question to pre-empt
**"What about recurring-payment emails?"** — The lifecycle emails (enabled / successful / failed / cancelled) are triggered by recurring invoice series, which are created from `docType===0` invoices only (`invoice-helpers.ts:374` filters `equalTo('docType', 0)`). An estimate can never be a recurring series member, so those 8 email paths (4 events × buyer + merchant) are structurally unreachable regardless of IS-11663.

## Still open (NOT code — flagged)

- **Naming cleanup (Sonya, 2026-07-21) — `invoice` → `document` in `email.ts`.** Once estimates flow through the payments/invoice branch, the `invoice` naming in `getEmailTemplateData()` is misleading (the doc can be an estimate). Flagged for whoever implements — NOT a blocker for the functional change. Scope options: (a) trivial — rename the module-local `swuInvoiceTemplate` const (`email.ts:50`, used only `:670`); (b) medium — rename the local `invoice` var/param → `document` (~100 refs, contained to `email.ts`, pure rename, keep the `parse.Invoice` *type* as-is); (c) out of scope — renaming the shared `parse.Invoice` type ripples across all of `api/`. Recommend (a)+(b), leave the type. Note: a payable estimate never actually hits `:670` (that's the no-payments fallback branch C); the point is the general "invoice = document" naming across the function, not that line specifically.

- **SWU template copy.** `paymentsSwuInvoiceTemple` HTML says "Invoice." Must be edited in **SendWithUs** to use `{{ doc }}` for the payable-estimate copy. This IS the ticket's "modify existing + add conditionals vs. new template" decision. I went with **reuse the payments template + conditional on `doc`** (no new code path / env var). If a dedicated payable-estimate template is preferred instead → one-line addition to `getPaymentsSWUTemplateId` + new `env.require`. **Decision not finalized — check Figma.**
- **`deposit_due` amount field (plan item 10).** Separate line item, NOT done. If wanted: add `deposit_due` to `SwuEmailTemplateData` (`email.ts:106`), compute via `getDeposit(invoice, payments).amount`, format with `formatAmount`, reference in SWU template. My routing change already resolved item 10's eligibility-wiring catch, so remaining work is just field + amount.

## To verify

- **IR A/B invoice template** (`email.ts:691`, `ir_invoice_send_email` flag) — **CONFIRMED (2026-07-20, code-verified).** Guard `if (templateId && docType === DocTypes.DOCTYPE_INVOICE)` is intentional — payable estimates correctly skip the IR experiment and fall through to the standard payments template (`SWU_PAYMENTS_INVOICE_TEMPLATE`). No change needed.
- **Tests** — fixtures likely assert estimates get `swuEstimateTemplateV2`. Need updating for the payable-estimate case. (Not run — per CLAUDE.md.)
