# Checkout Flow — CURRENT full-stack ("Pay Now")

**Purpose:** Factual, code-verified trace of what happens end-to-end today when a buyer on the public checkout page enters payment details and clicks "Pay Now". Written to ground the **convert-first vs convert-last** decision (Juan/Sonya reopened it 2026-07-20) and IS-11663's gate work.

**Verified:** 2026-07-21, four parallel code-trace agents across is-unifiedxp, is-stripe, is-paypal, is-checkout, api, is-web-app, is-mobile.

**TL;DR for the decision:**
- The **only** thing blocking a paid estimate today is the `docType !== 0` gate — and it fires at **3 points per processor** (6 total), not one.
- **L2/L3 interchange data is fully docType-agnostic** — built from line items + tax, US-country-gated, never docType-gated. No invoice ID or invoice number is required by either processor. → the sole justification for convert-first (mint invoice ID before PaymentIntent for L2/L3) **does not hold up in code**.
- **No automated conversion exists today** — convert-first / convert-at-pay / webhook-convert are all unbuilt. Conversion is only a manual merchant button.

---

## Where the checkout page actually lives

The public buyer checkout is **NOT** in `is-web-app` or `is-mobile`. It is the **`is-unifiedxp`** Next.js app in is-services:
- Route: `is-services/packages/services/is-unifiedxp/src/app/checkout/[documentId]/page.tsx`
- is-web-app `payments/` + `invoice-editor/` and all is-mobile Stripe/PayPal code are **seller-side** (settings, onboarding, Tap-to-Pay, checkout preview). No buyer "Pay Now" page exists there.

The frontend hands off to two separate backend services:
- **is-stripe** (`NEXT_PUBLIC_IS_STRIPE_SERVICE_URL`)
- **is-paypal** (`NEXT_PUBLIC_IS_PAYPAL_SERVICE_URL`)

---

## Stripe path

```
1. PAGE LOAD (before Pay Now — intent created on mount)
   Frontend → POST {is-stripe}/payment-intent
              body: { documentId, paymentMethodTypes, paymentId?, isRecurringPayment?, ... }
              headers: { idempotencyKey }
   is-stripe:
     • getUniversalInvoiceById(documentId)          → Parse cloud fn `publicInvoice`
     • validateAccountAndDocument({ docType, ... })  ──► findNotPayableReason  ⛔ GATE #1 (docType!==0)
     • createPaymentIntent → buildStripePaymentIntentBody
         - L2/L3 params built from universalInvoice.items + tax  (NO invoice id/number to Stripe)
         - PI created against NO document id; only amount/currency/methods/metadata
     • stores paymentIntentId → invoiceId in IS Postgres `PaymentIntent` table
     • returns { client_secret }

2. PAY NOW CLICK
   Frontend → stripe.confirmPayment({ elements, confirmParams: { return_url } })
              ── goes DIRECTLY to Stripe's API, not our backend
   Stripe runs validation + 3DS + charge, then redirects the browser to return_url:

3. REDIRECT LANDING
   GET {is-stripe}/payment-intent/confirm/{documentId}?is_deposit=<bool>[&recurring...]
   is-stripe confirmStripePaymentIntent:
     • re-fetches PI from Stripe; if status==='succeeded' & not already recorded:
         addPaymentToDocument → tryAddingPaymentToInvoice
           ──► validateDocumentPayableToConfirmPayment → findNotPayableReason ⛔ GATE #2 (docType!==0)
         writes Payment record to Parse
     • redirects → /checkout/{documentId}/success

4. WEBHOOKS (async, parallel — signature + IP validated)
   payment_intent.succeeded → handleStripeWebhookPaymentIntentEvent
       addPaymentToDocument (idempotent) ──► findNotPayableReason ⛔ GATE #3 (docType!==0)
       + triggerBookkeepingSync + ACH recurring setup
   charge.succeeded → handleChargeEvent
       sends buyer + merchant SWU emails, merchant push notification
       (does NOT write the Payment record — that's payment_intent's job)
```

**Confirmation is client-side** — server returns only `client_secret`; the server never calls `stripe.paymentIntents.confirm`. The `/payment-intent/confirm/{documentId}` endpoint is a post-redirect **reconciliation + redirect** step, not a confirm.

**Key files:**
- `is-stripe/src/routes/public/payment-intent-create.ts` (create)
- `is-stripe/src/services/create-or-update-payment-intent.ts` (`buildStripePaymentIntentBody`)
- `is-stripe/src/services/l3.ts`, `l2.ts` (interchange data)
- `is-stripe/src/routes/public/payment-intent-confirm.ts` (redirect landing)
- `is-stripe/src/services/webhook/payment-intent-updated.ts` (writes Payment)
- `is-stripe/src/services/webhook/charge-event-updated.ts` (emails/push)

---

## PayPal path (is-paypal — separate service)

```
1. POST {is-paypal}/orders   body: { invoiceId, requestId, fundingSource?, recurringPaymentConsent? }
   is-paypal createPaypalOrder:
     • getUniversalInvoiceById(invoiceId)
     • validateDocumentPayable({ documentType: docType }) ⛔ GATE #1 (docType!==0 → INVOICE_NOT_AN_INVOICE)
     • builds L2/L3 from universalInvoice.items + tax
     • creates PayPal order: custom_id=uuid(), purchase_unit.invoice_id=`{invoiceId}-{timestamp}`
     • persists local PaypalOrder row (orderId, customId, invoiceId, accountId)
     • returns { orderId, status, payer_action_url? }

2. onApprove → POST {is-paypal}/orders/capture[?paymentId=<upcomingId>]  body: { orderId, fundingSource, ... }
   is-paypal capturePaypalOrder:
     • validateDocumentPayableById(invoiceId) ⛔ GATE #2 (docType!==0)
     • captures order (idempotency key md5(CO-{orderId}-{invoiceId}))
     • updateOrderCaptureStatusApi → writes Payment (see below)
     • push notification + tryToSendPaymentCaptureEmails

3. WEBHOOK PAYMENT.CAPTURE.COMPLETED → handlePaymentCaptureCompleted → updateOrderCaptureStatusWebhook
     (API path and webhook path converge on updateOrderCaptureStatusApi/Webhook)
     • getInvoicePayment → tryAddingPaymentToInvoice (5 retries)
         ──► validateDocumentPayableById ⛔ GATE #3 (docType!==0)
         → addPaymentToInvoice → Parse cloud fn invoiceAddPayment (writes Payment to Parse)
         + SQS event + triggerBookkeepingSync
     • updates local PaypalOrder row
```

**Key files:**
- `is-paypal/packages/server/src/routes/public/orders.ts` (routes)
- `is-paypal/.../services/paypal-create-order.ts`, `paypal-capture-order.ts`
- `is-paypal/.../providers/paypal/index.ts` (L2/L3 supplementary_data)
- `is-paypal/.../services/paypal-l3/l3.ts`, `l3-eligibility-check.ts`
- `is-paypal/.../services/paypal-order-capture-status.ts` (writes Payment, both API + webhook)
- `is-paypal/.../services/invoice-payable.ts` (the gate)

---

## The docType gate — 6 enforcement points

Everything funnels through **one shared rule**:

`is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:58-60`
```ts
if (documentType !== 0) {
  return GeneralNotPayableReason.docTypeNotInvoice;
}
```
(PayPal has its own parallel copy in `is-paypal/.../services/invoice-payable.ts:21-31` throwing `INVOICE_NOT_AN_INVOICE`.)

| # | Processor | Stage | Location |
|---|-----------|-------|----------|
| 1 | Stripe | intent create | `payment-intent-create.ts:66-73` → `validateAccountAndDocument` → `findNotPayableReason` |
| 2 | Stripe | confirm landing (payment record) | `payment-intent-confirm.ts` → `tryAddingPaymentToInvoice` → `validateDocumentPayableToConfirmPayment` |
| 3 | Stripe | webhook (payment record) | `payment-intent-updated.ts` → `addPaymentToDocument` → same |
| 4 | PayPal | order create | `paypal-create-order.ts:132-139` → `validateDocumentPayable` |
| 5 | PayPal | capture | `paypal-capture-order.ts:113` → `validateDocumentPayableById` |
| 6 | PayPal | payment record (API + webhook) | `invoice-payments.ts:29` → `validateDocumentPayableById` |

**Implication:** IS-11663 relaxing only the **checkout-page eligibility** gate (so the estimate checkout renders) is NOT sufficient for a paid estimate to complete. Under **convert-last**, all 6 must accept docType=1. Under **convert-first**, the document is already an invoice (docType=0) by the time gates 1–6 run, so they pass unchanged — only the checkout-page eligibility needs relaxing.

---

## L2/L3 — docType-agnostic (this reopens the decision)

The load-bearing justification for **convert-first** was: *"Invoice ID (not estimate ID) must be passed to Stripe/PayPal — required to supply L2/L3 interchange fields."* Two independent agents confirmed this **does not hold up in code**:

**Stripe** (`l3.ts`, `l2.ts`):
- L3 eligibility is **US-country-only**: `isL3Eligible = countryCode?.toUpperCase() === 'US'`. Never checks docType.
- L2/L3 params built from `universalInvoice.items`, `setting.taxType/taxInclusive`, calculator amounts — none inspect docType.
- `customer_reference` / `order_reference` sent to Stripe are **hardcoded string literals** — the invoice number is **not passed at all**.
- The PaymentIntent is created against **no document id**; linkage lives only in IS Postgres.

**PayPal** (`providers/paypal/index.ts`, `paypal-l3/l3.ts`):
- L3 eligibility also **US-only**.
- `supplementary_data.card.level_2/level_3` built from `items`; `invoice_id` sent to PayPal is a generated `{id}-{timestamp}` string; `custom_id` is a fresh `uuid()`.

**Conclusion:** An estimate's line items would produce a byte-identical L2/L3 payload. **Neither processor requires an invoice ID or invoice number for L2/L3 qualification.** The interchange-fee argument no longer forces convert-first — the choice is now free on that axis.

---

## Doc-ID passing — broader than L2/L3 (directly code-verified 2026-07-21; every line below opened and confirmed, not agent-reported)

Juan's claim ("we don't need to pass the estimate/invoice id to the Stripe PaymentIntent, L2/L3, or the PayPal order — which makes convert-last work") is **confirmed in code and against processor docs.** The doc id is nowhere functionally required. Package roots: is-stripe = `is-services/packages/services/is-stripe`, is-paypal = `is-paypal/packages/server`.

### Stripe — the doc id / invoice number never reaches Stripe at all

Every field that goes into a Stripe PaymentIntent/charge body, exhaustively (4 build sites):

| Build site | File:line | id fields in the Stripe body? |
|---|---|---|
| Main checkout PI (`buildStripePaymentIntentBody`) | `create-or-update-payment-intent.ts:175-193` | **None.** Body = `amount`, `currency`, `payment_method_types`, `payment_method_options`, L2/L3 spread, `customer`, `setup_future_usage`, `metadata`. `metadata` = `{ requestId?: <idempotency header>, isRecurringPayment?, recurringPaymentStatus? }` — no `invoiceId`, no `invoiceNo`, no `description`, no `statement_descriptor`, no `receipt_email`. |
| L2 params (`getL2StripeParams`) | `l2.ts:21-25` | `payment_details.customer_reference` / `order_reference` = **hardcoded literal strings** `'customer_reference'` / `'order_reference'`. Not the invoice id/number. |
| L3 params (`getL3StripeParams`) | `l3.ts:272-276` | Same two **hardcoded literals**. The invoice `objectId` is in scope but used only for `trackCheckoutEvent({ documentId: objectId })` analytics (`l3.ts:263-270`), never the returned payload. |
| Recurring auto-payment PI (create+confirm off-session) | `recurring-payment-execute.ts:317-320` | `metadata = { requestId?: <idempotency> }` only; L2/L3 spread again carries only the hardcoded literals. |
| Confirm-landing ACH metadata update | `payment-intent-confirm.ts:387-393` | `metadata` = `recurring_invoice_series_id` (the recurring **series** id, not the doc objectId/number), `consent_granted_at`, `customer_email?`. ACH-only, not docType-gated. |

- The invoice `objectId` appears only in **local Postgres writes** — `PaymentIntent.create({ invoiceId: universalInvoice.objectId })` (`create-or-update-payment-intent.ts:100`, `recurring-payment-execute.ts:330`) — never in a Stripe body.
- `invoiceNo` (human number) is used only for push notifications and email rendering; never sent to Stripe.
- Docs (api/payment_intents/create): only `amount` + `currency` are required; `metadata`/`description`/`statement_descriptor` are informational and don't affect processing or fees. `order_reference` is "required for L2/L3" but we already satisfy it with a **constant** → an estimate yields a **byte-identical** body.

### PayPal — an id-shaped value IS sent, but it's cosmetic, not load-bearing

All order-body construction is in `providers/paypal/index.ts` `createOrder()`:

| Field sent to PayPal | File:line | Value / source | docType-gated? |
|---|---|---|---|
| `purchase_unit.custom_id` | `index.ts:201` ← `paypal-create-order.ts:227` | `uuid()` (fresh v1 uuid, always set) | No |
| `purchase_unit.invoice_id` | `index.ts:246-249` ← `helper.ts:77-86` | `{objectId}-{YYYY-MM-DD-HH-MM-SS}` (Parse objectId + local timestamp) | No — only `if (invoiceId)` |
| `supplementary_data.card.level_2.invoice_id` | `index.ts:250-253` | **same** `{objectId}-{timestamp}` string | No (same block) |
| `items[0].name` (non-L3 path only) | `index.ts:222-237` | `` `INVOICE ${invoiceNo}` `` — human number as a **display name** | No — `if (invoiceNo)`; L3 path builds items from line items instead |
| `reference_id`, `description` (unit-level), `soft_descriptor` | — | **Never set.** (`description` appears only on individual L3 line items, sourced from item text, never the invoice id/number.) | — |

- The **authoritative** order→document linkage is the local `PaypalOrder` DB row: written at create (`paypal-create-order.ts:328` `PaypalOrder.create({ invoiceId, orderId, customId, ... })`) and resolved at capture via `PaypalOrder.findByOrderId(orderId)` → `paypalOrder.invoiceId` (`paypal-capture-order.ts:94-98`). **Not** the value sent to PayPal — which is timestamp-salted and so isn't even a clean reverse key.
- Capture sends **no id fields** in the body — `invoiceId`/`customId` appear only as **hashed idempotency headers** (`PayPal-Request-Id = md5('CO-{orderId}-{invoiceId}')`).

**⚠️ One caveat to carry forward:** PayPal's `invoice_id` field can, in some PayPal products, trigger **duplicate-payment blocking**. The Orders v2 docs did not confirm it's active on this integration, and our value is timestamp-salted (unique per attempt) so it is **not** functioning as a dedup key today regardless. Not a blocker for convert-last — but don't let a future change repurpose `invoice_id` as a dedup key without re-checking.

**Net:** this removes the *last* processor-level argument for convert-first (the L2/L3 point above was the first half; this is the general "no doc id is required anywhere" half). An estimate id — or the timestamp-salted string derived from it — is functionally equivalent to an invoice id at both processors. Convert-last is unblocked on the payment-rail axis; the remaining trade-offs are purely the ones in the table below (relax all 6 docType gates, payment carryover, ACH timing, paid-but-not-converted reconciliation).

---

## Conversion today — nothing automated

**No convert-first, no convert-at-pay, no webhook-triggered conversion exists anywhere.** (Consistent with the IS-11663 note "No convert-first in this ticket.") Conversion is exclusively a **manual merchant button**, implemented client-side and independently in two codebases:

| | Web (`is-web-app`) | Mobile (`is-mobile`) |
|---|---|---|
| Fn | `InvoiceModel.convertEstimate()` (`InvoiceModel.ts:1016`) + `_convertEstimateData()` (`:1296`) | `makeInvoiceFromEstimate()` (`realm/entities/invoice/use-cases.ts:22`) |
| Transform | new InvoiceModel, `docType=INVOICE`, fresh remoteIds, `setting.estimateId=<estimate id>` | `defaultInvoice(INVOICE)`, deep-copy w/ fresh remoteIds, `setting.estimateId` |
| Payments carried? | **No** — marks estimate `fullyPaid=true` | **Yes** — shallow-copies payment sub-docs with new remoteIds |
| Deposit fields | stripped (invoice-setting defaults) | stripped (`depositType/Rate/Amount` omitted) |

**Not present anywhere:**
- No atomic guard (no `findOneAndUpdate` compare-and-set). Only signal is estimate `fullyPaid=true` — does not block re-conversion.
- No `convertedTo` / `approvedAt` fields exist in any codebase (planned in the spec, unbuilt).
- No conversion caller in is-services, api/, is-parse-server/, or is-web-app/nextjs. (`is-acorn-webhook` only *reads* docType for analytics/email; does not convert.)
- Only linkage is a one-directional back-pointer `setting.estimateId` on the new invoice; the estimate gets no forward pointer.

---

## What this means for convert-first vs convert-last

| Axis | convert-first (current spec) | convert-last (Juan/Sonya, 2026-07-20) |
|------|------------------------------|----------------------------------------|
| Ordering | Pay Now → convert (sync) → PaymentIntent → charge | Pay Now → PaymentIntent → charge → confirm → convert (webhook) |
| L2/L3 | works (invoice) | **works (estimate)** — code-confirmed docType-agnostic |
| docType gates | pass unchanged (already invoice); only relax checkout eligibility | must relax **all 6** payment-path gates to accept docType=1 |
| Payment carryover | N/A (payment vs invoice from start) | **returns** — payment made vs estimate; must re-key/carry to new invoice |
| ACH settlement timing | moot (invoice exists pre-pay) | **returns** — webhook conversion gated on settlement |
| "converted-but-unpaid" middle state | risk (open a/b/c product decision) | **eliminated** (original pay-first rationale — no orphaned unpaid invoices) |
| "paid-but-not-converted" state | N/A | **new risk** — charge succeeds but convert webhook fails → paid estimate, no invoice; needs reconciliation |
| Pay Now UX | needs sync loading state on click | no sync step |

**Neither exists today** — both require net-new conversion wiring. The decision is now genuinely open on architecture merits (failure-state shape, gate surface area, payment carryover), no longer forced by interchange fees.
