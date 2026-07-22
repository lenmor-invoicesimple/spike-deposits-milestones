# Ticket — Buyer Post-Conversion Experience (checkout success / failed / conversion errors)

**Epic:** IS-11250 — Deposit Optimization v1
**Date:** 2026-07-15
**Owner:** Lenmor Dimanalata
**Status:** Draft — implementation-ready spec
**Source of truth:** Liz's design-meeting decision ([Slack p1784052946973869](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869)); Buyer Post-Conversion (P#) items.
**Related tickets:**
- [`ticket-payable-estimates-global.md`](./ticket-payable-estimates-global.md) — payability engine + convert-first (WI-0) + `convertedTo` lock. **This ticket depends on it.**
- [`ticket-public-review-page-estimate-payable.md`](./ticket-public-review-page-estimate-payable.md) — the pre-payment `/v/` review page. Payment happens *after* the buyer leaves that page for checkout.
- [`spike-email-estimate-vs-invoice.md`](./spike-email-estimate-vs-invoice.md) — email surface.

> **Convention:** P# labels map to the "Buyer Post-Conversion" heading in Liz's Slack post. This ticket collects the P-items routed out of the global + review-page tickets.

---

## 1. Scope

**This ticket = everything the buyer sees on the checkout app (`is-unifiedxp`) at and after Pay Now** — the success page, the failed/declined page, the conversion-failure error screen, the Pay-Now loading state during conversion latency, and old-link redirects. It is the buyer-facing surface **downstream of convert-first** (WI-0 in the global ticket).

Because convert-first mints a real **invoice** (docType=0) before the PaymentIntent, most of this surface **already works** for the estimate-deposit case — it inherits the existing invoice deposit flow. This ticket's job is largely **verification + a few gaps**, not net-new UX.

**In scope**
- **WI-P1 — Payment-failure retry** (P1): the failed page lets the buyer retry. Verify it works for a converted-from-estimate invoice.
- **WI-P2 — Conversion-failure error screen** (P2): if convert-first fails *before* the charge, show a recoverable error (not a payment error).
- **WI-P3 — Success / failed page deposit labels** (P4, P6): confirm "DEPOSIT PAID" on success and "DEPOSIT DUE" on failed render correctly for the estimate-deposit case. **Code already correct — verification + Figma mock relabel only.**
- **WI-P4 — Old-link redirect** (P7): a stale estimate `/v/` or checkout link (post-conversion) redirects sensibly rather than erroring.
- **WI-P5 — Pay-Now loading state** (P8): show a loading/interstitial state during the convert-first latency (synchronous conversion adds a round-trip before the PaymentIntent).

**Out of scope**
- The convert-first mechanic itself → global ticket **WI-0**.
- Pre-payment review page (`/v/`) affordances → review-page ticket.
- Email surfaces → email spike.

**Hard dependency:** convert-first (WI-0) + `convertedTo` lock (WI-3/WI-4) must exist. This ticket is inert without them.

---

## 2. How the checkout app works today (verified 2026-07-15)

App: **`is-services/packages/services/is-unifiedxp`** (Next.js checkout app).

- **Success page** — `src/app/checkout/[documentId]/success/page.tsx` → `<PaymentSuccess>` (`src/components/PaymentSuccess/PaymentSuccess.tsx`).
- **Failed page** — `src/app/checkout/[documentId]/error/page.tsx` → `<PaymentError>` (`src/components/PaymentsError/PaymentError.tsx`).
- **Page-level error boundary** — `src/app/checkout/[documentId]/error.tsx` → `<PageErrorBoundary>` (distinct from the *payment*-error route above; this catches thrown render/server errors).
- **Deposit labels** — defined once in `src/components/PaymentDetails/messages.ts:3-4`:
  ```ts
  depositDue: 'DEPOSIT DUE',
  depositPaid: 'DEPOSIT PAID',
  ```
  rendered via `PaymentDetails/PaymentDepositDetails.tsx` (`PaymentDepositDetails` = due, `PaymentDepositPaidDetails` = paid).
- **Payment-events / conversion** handlers live in `is-payments`: `src/lambda-handlers/topics/handle-payment-success.ts`, `handle-payment-failure.ts`.

**Key inheritance fact:** post-convert-first the checkout operates on an **invoice** (docType=0). Every deposit affordance here already keys off invoice state, so the estimate-deposit buyer inherits the invoice experience for free. This ticket verifies that inheritance rather than rebuilding it.

---

## 3. Work items

### WI-P3 — Success / failed page deposit labels (P4, P6) — CODE CONFIRMED CORRECT

**Verified in code (2026-07-15) — no code change needed:**

| Page | File:line | Renders | Label | Correct? |
|---|---|---|---|---|
| **Success** | `PaymentSuccess/PaymentSuccess.tsx:86` → `PaymentDepositPaidDetails` | deposit paid row | **DEPOSIT PAID** | ✓ |
| **Failed** | `PaymentsError/PaymentError.tsx` → `PaymentDetails` → `PaymentDepositDetails` | deposit due row | **DEPOSIT DUE** | ✓ (still due after a failed payment) |

Labels centralized in `PaymentDetails/messages.ts:3-4`. Because convert-first mints an invoice, the estimate-deposit case inherits both labels correctly.

**Only actions:**
1. **Figma mock relabel (no code):** the "Payment Successful" mock shows `DEPOSIT DUE $500` → should read **`DEPOSIT PAID`**. Flag to design.
2. **Verification (QA/test):** run a real deposit-estimate through checkout end-to-end and confirm success shows DEPOSIT PAID + recalculated BALANCE DUE, and a forced failure shows DEPOSIT DUE.

**AC:** deposit-estimate → paid deposit shows "DEPOSIT PAID" + remaining "BALANCE DUE"; failed payment shows "DEPOSIT DUE"; Figma success mock relabeled.

### WI-P1 — Payment-failure retry (P1)

The failed page (`PaymentError.tsx`) already renders a `RefreshButton` that routes back to `/checkout/{documentId}`. **Verify** the converted-from-estimate invoice retries cleanly (the retry hits an invoice, since conversion already happened at first Pay-Now).

**Open question — retry after conversion:** convert-first converts the estimate *before* the charge. If the charge then fails, the estimate is **already converted** (an invoice exists). Confirm the retry path re-attempts payment on that invoice and does **not** attempt a second conversion (the `convertedTo` lock / WI-4 should make a re-convert a no-op). Verify no double-charge / double-convert.

**AC:** after a failed deposit payment on a converted estimate, the buyer can retry and succeed; no duplicate invoice is created; the retry charges the same deposit amount.

### WI-P2 — Conversion-failure error screen (P2)

Distinct from a *payment* failure: convert-first can fail **before** any charge (conversion error). This must land on a **recoverable error** state (`PageErrorBoundary` / `RecoverableError`), not the payment-error page — the buyer hasn't been charged and no invoice may exist.

**Change:** ensure a convert-first failure surfaces a clear "something went wrong, please try again" state (existing `RecoverableError` component is the likely home) and does **not** show a misleading "payment failed" / "deposit due" screen implying a charge was attempted.

**AC:** a simulated convert-first failure shows a recoverable error, no charge is made, retry re-attempts conversion; the buyer is never shown a payment-failed state for a conversion-only failure.

### WI-P4 — Old-link redirect (P7)

**Slack requirement (verbatim):** *"If buyer returns via old email link, they are redirected to the **public estimate URL** [`/v/`], **not** the checkout."*

So the buyer sees the checkout-success page **only the first time, immediately after paying.** Any later return via the old email link or a reloaded checkout URL must land on the **public estimate `/v/` page** — where the now-converted estimate renders with QR + Pay Deposit CTA + DEPOSIT DUE all hidden (owned by the review-page ticket's M1/WI-C). This redirect is what **prevents a second checkout / second invoice** from the same estimate (the wrong-customer-gets-wrong-invoice risk in Slack).

**⚠️ Net-new work — no redirect logic exists today.** Grep of `is-web-app` `/v/` + `public-invoice` and the `is-unifiedxp` checkout routes found **no** converted-state redirect. Must be built:
- A stale **checkout** URL (`/checkout/{id}`) for a converted estimate → redirect to the public estimate `/v/{id}` page (not re-enter checkout).
- The emailed `/v/{id}` link already lands on `/v/` — it just needs the converted state to render correctly (review-page ticket), no redirect needed for that one.

**AC:** a stale post-conversion **checkout** link redirects to the public estimate `/v/` page (never re-enters checkout, never creates a second invoice); the `/v/` page itself shows the converted estimate with all payment affordances hidden. Coordinate with the review-page ticket's payability lock (`convertedTo` → payUrl null) and the global ticket's WI-4 lock.

### WI-P5 — Pay-Now loading state (P8)

Convert-first is a synchronous server round-trip inserted before the PaymentIntent, adding latency at Pay-Now. Show a loading/interstitial state so the buyer isn't staring at a frozen button.

**AC:** clicking Pay Now on a deposit-estimate shows a loading state during conversion; on success it proceeds to the payment step; on conversion failure it routes to WI-P2's recoverable error.

---

## 4. Acceptance criteria (rollup)

- Deposit-estimate paid via checkout: success page shows **DEPOSIT PAID** + recalculated BALANCE DUE (inherited from invoice flow, verified).
- Failed deposit payment: failed page shows **DEPOSIT DUE**, retry works, no duplicate invoice/convert.
- Convert-first failure (pre-charge): recoverable error, no charge, retry re-converts.
- Stale post-conversion links redirect sensibly.
- Pay-Now shows a loading state during conversion latency.
- Figma "Payment Successful" mock relabeled "Deposit Due" → "Deposit Paid" (design task, no code).

## 5. Notes / open

- **Mostly verification, not new UX.** The strong inheritance from the invoice deposit flow (post-convert-first) means WI-P3 is confirmed-done-in-code and WI-P1 is largely a retry-path check. The genuinely new work is **WI-P2 (conversion-failure state)** and **WI-P5 (loading state)** — both stem from convert-first being synchronous.
- **Depends entirely on the global ticket** (WI-0 convert-first, WI-3/WI-4 `convertedTo`). Sequence after it.
- **Retry-after-conversion semantics (WI-P1)** is the sharpest open question — confirm the failed-charge retry re-pays the existing invoice and never re-converts.
- **Effort:** S–M. Most items are verification; WI-P2 + WI-P5 are the real build.
