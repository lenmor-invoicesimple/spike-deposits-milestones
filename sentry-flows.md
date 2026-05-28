# Sentry Error Flows — `/checkout/...`

All Sentry-capturable flows in `is-unifiedxp` for checkout routes.

---

## How errors reach Sentry

There are two paths:

1. **Explicit** — code calls `reportServerError()` or `Sentry.captureMessage()` directly
2. **Implicit** — server component `throw`s → Next.js catches → renders `error.tsx` → `PageErrorBoundary` calls `reportClientError(error)` on the client (only if `error.reportedOnServer !== true`)

---

## Server-side

### Explicit captures

| # | File | Error / Message | Level | Trigger |
|---|------|----------------|-------|---------|
| 1 | `app/checkout/[documentId]/page.tsx:121` | `"Invalid search params in checkout: {msg}"` | warning | `GET /checkout/[id]` with malformed query string |
| 2 | `app/checkout/[documentId]/page.tsx:193` | `"PayPal ACH bank verification failed for document {id}"` | warning | Returning from PayPal bank verification with `confirm_error` param |
| 3 | `components/PaymentsSection/PaymentsSection.tsx:166` | Any non-`InvalidStateError` from `validateDocumentPayable()` / `validateAccountStatus()` | error | Checkout render with bad account/doc state |
| 4 | `components/PaymentsSection/PaymentsSection.tsx:203` | Unhandled `InvalidStateError` (default switch branch) | error | Unknown validation failure on checkout |
| 5 | `components/PaymentsSection/PaymentsSection.tsx:265` | Error from `getPaymentMethods()` | error | Payment methods fetch fails; user gets redirected to public invoice |
| 6 | `utils/is-payments/get-payment-methods.ts:40` | `"Failed to fetch payment methods for checkout"` | warning | `/backend/checkout/payment-methods` errors during page render |
| 7 | `utils/is-payments/get-payment-methods.ts:61` | `"Failed to fetch account payment methods"` | warning | `/backend/payment-methods/{accountId}` errors during page render |
| 8 | `utils/parse/get-upcoming-payment.ts:24` | `"Failed to fetch upcoming payment"` | warning | `Parse.Cloud.run('invoiceGetPayments')` fails on checkout/success/error pages |
| 9 | `app/intent/[accountId]/[documentRemoteId]/route.ts:22` | Validation error from `paramsSchema.safeParse()` | error | `POST /intent/[accountId]/[documentRemoteId]` with invalid params |
| 10 | `app/intent/[accountId]/[documentRemoteId]/route.ts:29` | `"Invalid accountId or documentRemoteId"` | error | Payment intent triggered with missing params |
| 11 | `actions/putPaymentAttemptedEvent.ts:32` | Any error from schema parse or `putEventsCommand()` | error | Server action during payment attempt *(silent — no rethrow)* |
| 12 | `actions/putCheckoutViewedEvent.ts:48` | Any error from schema validation or `putEventsCommand()` | error | Server action on checkout view *(silent — no rethrow)* |

### Throws → bubbles to `PageErrorBoundary` → `reportClientError`

| # | File | Error message | Trigger |
|---|------|--------------|---------|
| 13 | `app/checkout/[documentId]/page.tsx:112` | `"Invalid document ID."` | Invalid doc ID in URL on main checkout page |
| 14 | `app/checkout/[documentId]/page.tsx:134` | `"Invalid search params."` | Invalid search params after warning already logged (#1) |
| 15 | `app/checkout/[documentId]/layout.tsx:20` | Zod parse error (`too_small`, min 10 chars) | Doc ID too short — caught by `paramsSchema.parse()` |
| 16 | `app/checkout/[documentId]/error/page.tsx:46` | `"Invalid document ID."` | Invalid doc ID on `/checkout/[id]/error` page |
| 17 | `app/checkout/[documentId]/error/page.tsx:56` | `"Invalid search params."` | Bad params on error page |
| 18 | `app/checkout/[documentId]/success/page.tsx:40` | `"Invalid document ID."` | Invalid doc ID on `/checkout/[id]/success` page |
| 19 | `app/checkout/[documentId]/success/page.tsx:50` | `"Invalid search params."` | Bad params on success page |

---

## Client-side

| # | File | Error / Message | Trigger |
|---|------|----------------|---------|
| 20 | `components/StripeCheckout/useStripeConfirmPayment.ts:85` | `"Stripe payment error: {stripe error message}"` | User submits Stripe payment and gets a non-`validation_error` error back |
| 21 | `components/PaypalACH/PaypalACH.tsx:67` | `"PayPal ACH error: {error}"` | ACH hook detects error state on component mount |
| 22 | `components/PaypalACH/PaypalACH.tsx:103` | Error from ACH CTA action | User clicks ACH verify/capture button and it throws |
| 23 | `components/PaypalCheckout/PaypalCheckout.tsx:56` | PayPal `onError` callback error | PayPal checkout SDK fires `onError` |
| 24 | `components/PaypalUnbrandedCheckoutWrapper/PaypalHostedFieldsNotEligibleFallback.tsx:28` | `"User not eligible for hosted fields"` | Component mounts when user isn't eligible for PayPal hosted fields |
| 25 | `components/PaymentIntentBottomSection/useSendPaymentIntent.ts:45` | Error from `fetch(/intent/...)` | Payment intent auto-fetch on component mount fails *(silent — no rethrow)* |
| 26 | `components/PaypalApplePay/useIsPaypalApplePaySdkEligible.ts:18` | Apple Pay config error | Effect runs to check Apple Pay eligibility and throws |
| 27 | `components/PaypalGooglePay/useIsPaypalGooglePaySdkEligible.ts:33` | Google Pay eligibility error | Effect runs to check Google Pay eligibility and throws |
| 28 | `components/ErrorBoundary/ErrorBoundary.tsx:30` | Any unhandled render error + component stack | React `componentDidCatch` — catches any uncaught render error in wrapped children |
| 29 | `app/global-error.tsx:14` | Any error reaching the global boundary | Last-resort catch-all for unhandled client errors |

---

## Notes

- **Silent errors** (#11, #12, #25): captured in Sentry but don't affect UX — no throw, no redirect.
- **`logToSentry: false`**: any error with this flag set is silently skipped by both `reportClientError` and `reportServerError`.
- **`reportedOnServer`**: server errors that call `reportServerError` set this flag so `PageErrorBoundary` doesn't double-report on the client.
- **Sample rates**: server `tracesSampleRate: 0.1`, client `tracesSampleRate: 0.5` — low-frequency errors may not appear on every occurrence.
