# IS-11663 — Payable Estimate: allow Estimate checkout

**Ticket:** https://everpro-tech.atlassian.net/browse/IS-11663
**Status:** Analysis complete + verified in code (session 2026-07-17). No code changes made yet for this ticket.
**Scope:** ONLY "let an estimate-with-deposit load and be chargeable on checkout." The `convertedTo` / "already converted → locked" eligibility is **explicitly deferred** to a later ticket (ticket body says so). Do NOT add the conversion lock here.

---

## The symptom

Loading an estimate at `checkout.staging.invoicesimple.com/<id>` shows a generic "Something went wrong! Try again" error boundary. Sentry trace showed the `docTypeNotInvoice → throw new InvalidStateError(DOCUMENT_NOT_AN_INVOICE)` branch. The page **THROWS during SSR load** — it is NOT just a "not payable" message being displayed.

## Root cause = the docType gate throws

The estimate **loads fine** (no DB/loader filter drops it — `getDocumentByRemoteId` filters only `accountId` + `remoteId`, no `WHERE documentType === 0`). The page throws *after* load during payability validation:

1. `findNotPayableReason` returns `docTypeNotInvoice` because `documentType !== 0`
   — is-services `packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58`
2. is-unifiedxp `utils/document-payable.ts:41` → `throw new InvalidStateError(DOCUMENT_NOT_AN_INVOICE)`
3. `PaymentsSection.tsx` catch block has **no case** for `DOCUMENT_NOT_AN_INVOICE` → falls to `default: reportServerError(error); throw error;` (PaymentsSection.tsx:206-208)
4. Next.js segment error boundary (`app/checkout/[documentId]/error.tsx` → `PageErrorBoundary`) renders "Something went wrong!"

## THE CORE CHANGE — two independent `documentType !== 0` gates in TWO repos

| # | Gate | Location | Behavior |
|---|---|---|---|
| 1 | `findNotPayableReason` | **is-services** `packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58` | `if (documentType !== 0) return docTypeNotInvoice`. **Every** payable check in is-services/is-stripe/is-unifiedxp/is-checkout/is-abandoned-cart funnels through this one function. |
| 2 | `validateDocumentPayable` (PayPal) | **is-paypal** (SEPARATE REPO `/is5/is-paypal`) `packages/server/src/services/invoice-payable.ts:29` | `if (documentType !== 0) throw new ISError(INVOICE_NOT_AN_INVOICE)`. Fully separate gate, own throw. Fires at PayPal *order-create* time, not on page load. |

**IMPORTANT process note:** the first code-sweep agent only searched is-services and wrongly reported "is-paypal has no separate gate." is-paypal is a separate repo outside the monorepo. The ticket was right. **Always check both repos when reasoning about payability.**

> ⚠️ **CORRECTION (2026-07-20, from re-reading the Jira ticket):** this is NOT a blanket "remove the docType line." The ticket's Eligibility + payment-status sections say to **replace the gate with a conditional**: an estimate is payable only if **(FF on) AND (deposit set) AND (not yet converted)**. So both gates must **accept new params** (feature flag + payment-collection/deposit state) and evaluate them — not just drop `documentType !== 0`. This is a signature change to `findNotPayableReason` / `validateDocumentPayable` that ripples to every caller. (`convertedTo` / conversion-lock is explicitly deferred to a later ticket — see below — so for THIS ticket the condition is FF + has-deposit.) The "blanket relax" language elsewhere in this doc predates the ticket being fleshed out; read it as "relax for FF+deposit estimates."
>
> **Ticket link bug to fix:** the ticket's inline "Pass flag and Payment collection via `findNotPayableReason`" sentence links to the **is-paypal** `invoice-payable.ts#L29`, but `findNotPayableReason` lives in **is-services** `payments-status/invoice-payments-status.ts#L58`. The two-gate table below it links correctly; only the inline link points at the wrong repo.

Relaxing gate #1 fixes TWO things at once: (a) stops the SSR throw, AND (b) makes the is-checkout eligibility endpoint start returning a `payUrl` for estimates (see below).

## Hidden prerequisite: the pay LINK is gated too

An estimate gets **no `payUrl` today**. The checkout URL is produced by the is-checkout eligibility endpoint (`is-checkout/src/eligibility/router.ts:60-73`), which runs `findNotPayableReason` via `getPayButtonEligibilityVendor` (`eligibility/service.ts:60-63`) → `validateDocumentPayable` (`is-checkout/src/document/document-payable.ts:41-62`). For an estimate this returns/throws not-eligible, so `{ isEligible: false }` with no URL.
- Merchant web/mobile + public `/v/` page render no pay button/QR because they only render when `payUrl` exists.
- The URL builder itself (`is-checkout/src/url/url.ts:15`) has NO docType gate — it's purely upstream eligibility.
- You reached the error boundary by hand-constructing `/checkout/<id>` (nothing blocks direct nav).

So relaxing gate #1 also un-gates link generation. Same function, easy to miss.

## Full prerequisite list for an estimate to load AND be payable

**Document-level (all already true for a normal deposit estimate), from `findNotPayableReason`:**
- docType gate relaxed ← *the change*
- not deleted
- `totalCents > 0`
- `balanceDueCents > 0` (fully-paid → `balanceDueZero`)
- `balanceDue + surcharge < 1,000,000`
- no pending orders
- (PayPal path adds: currency supported)

**Account-level (INDEPENDENT of the estimate — the easy-to-trip-over part when testing):**
- `accountStatus` must be populated — if BOTH provider status calls fail → page **THROWS** (`account-payable.ts:20-22`, `INVALID_ACCOUNT_STATUS` re-thrown → same error boundary).
- at least one provider **connected, accepting + enabled, currency-matched, not suppressed** — else `ACCOUNT_INELIGIBLE` → **redirects away** (`PaymentsSection.tsx:197-205`), OR the account qualifies for the payment-intent "lead" path (`isUserEligibleForPaymentIntent`).
- `payments_enabled` FF on; valid subscription for the link path (`hasEverSubscribed`).

**Toggle/suppression:** `paymentSuppressed` (PayPal) / `stripePaymentSuppressed` (Stripe) on `invoice.setting`. If on → provider drops out of eligibility → if no eligible provider → `ACCOUNT_INELIGIBLE` redirect (does NOT hard-throw). Orthogonal to docType. Checked in `payments-status/.../sdks.ts:50` (paypal) / `:106` (stripe).

**DEPOSIT IS NOT A PREREQUISITE to load/pay.** Never read in `findNotPayableReason`, `validateAccountStatus`, or `getProvidersEligible`. Deposit fields (`depositType/depositRate`) exist on checkout types (`types.ts:107-108`, transformed `transformers/document.ts:47-48`) but are used ONLY for payment amount + display (`PaymentsSection.tsx:358 isDepositPayment`, `payment-intent-create.ts:102`).

## Answering "is having a deposit enough?" — NO, neither necessary nor sufficient

- **Not sufficient:** a deposit estimate on an account with no connected/accepting provider still fails (redirect or throw) — account prereqs are independent of the document.
- **Not necessary (as of the fleshed-out ticket, THIS IS NOW ENFORCED IN THE GATE):** the ticket wants payability gated on **FF + has-deposit** inside `findNotPayableReason` / `validateDocumentPayable` themselves (pass flag + payment-collection state as params). So the gate WILL have a "has deposit" condition — an estimate with no deposit stays non-payable. (Earlier framing in this doc said "the gate has no has-deposit condition / enforced only by FF+UI" — that was the pre-groom reading; the ticket now puts the deposit check in the gate. A payable estimate therefore always has a deposit.)

## Fallout to verify once gate #1 relaxes (branches that go dead-for-estimates)

- **is-unifiedxp SSR** `utils/document-payable.ts:41` + `PaymentsSection.tsx` default-throw — the Sentry crash. Goes away. ✅ verify.
- **is-checkout redirect router** `checkout/router.ts:206-215` (`isDocumentNotInvoiceOrDeleted`) — sends estimates to "Go to Invoice Simple" invalid page. Confirm stops firing.
- **`getReportRedirectUrl` reason switch** `checkout/service.ts:71-81` — estimates now flow through instead of returning null. Behavior change to confirm.
- **is-abandoned-cart** `is-invoice-payable.ts:22` → `send-reminder/scheduled-handler.ts:43` — 🔴 **estimates now become eligible for abandoned-cart reminder emails.** Real side-effect (plan open item #3 "abandoned-cart on estimates, intended?"). May need a separate guard if unwanted.
- **Value guard** `is-checkout/document/document-payable.ts:22-29` — throws `'Document is not an invoice'` on falsy total/balanceDue (value check, not docType). Verify estimate amounts populate.
- **Pay-button eligibility endpoint** `is-checkout/eligibility/service.ts:63` ALSO throws for estimates (not only the SSR page). Same gate fixes both.

## Defense-in-depth (optional, not required for this ticket)

`PaymentsSection.tsx:206 default: throw` surfaces ANY unmapped `InvalidStateError` as the generic boundary. If you ever want a non-payable estimate to degrade gracefully (redirect to `/v/`) instead of erroring, add `case ErrorCodes.DOCUMENT_NOT_AN_INVOICE: redirectToPublicInvoice(...)`. Relaxing the gate is enough for this ticket.

## No convert-first in this ticket

Since the lock is deferred, an estimate stays `docType===1` all the way through charge. Sweep found NO downstream numeric `docType===0` assumption in the charge/webhook paths (is-stripe confirm + webhook just pass `docType` through). Reassuring, but re-verify if scope grows.

## Tests

Fixtures assert estimates are non-payable (e.g. `invoice-payments-status.test.ts:65` feeds `documentType: 1`, expects `docTypeNotInvoice`). Will need updating alongside the gate. (Not run — per CLAUDE.md.)

## Staging test checklist

Use an estimate that: (1) is on an account with Stripe/PayPal **connected and accepting**, (2) payments **not suppressed**, (3) **balance due > 0**, (4) account has `payments_enabled`. Then relax gate #1 → eligibility endpoint yields `payUrl` → link/QR appear → checkout page loads + renders pay options.
