# Deposit Vaulting Flow — Summary

> Live plan document. For alternatives, decisions, and full analysis see the PRD evaluation doc.

---

## #1 — One-Step Deposit Creation

- Remove extra percent/flat step from deposit creation UI
- Return to "just enter amount and send" flow
- Pure UI change (mobile + web), no backend work
- Backend already supports both deposit types

---

## #2 — Partial Payment Receipt (Email + SMS)

- Auto-send receipt when deposit/partial payment is made
- **Email:** Deposit payment succeeds → webhook → is-payments → is-messages (fire-and-forget, not scheduled like APR)
- **SMS:** Twilio infra already exists in legacy API — new transactional template needed
- Triggered by: payment success webhook (same trigger point for both email and SMS)
- Template data: amount paid, date, remaining balance, invoice link

====
-> Mark paid
send an email

can be edited, etc

offline

I wouldn't add that
very open
===

phone number


---
invoices 
blocking Estimate types for everything


## #3 — Vaulting at Deposit Payment

-> hire someone to do plumbing
pay a deposit
remaining balance based on Merchant

-> client decides that they pay

if client not happy, then they wont want to pay

1. Merchant creates Invoice with a deposit amount
2. Buyer opens checkout link
3. Buyer enters email (already required, same as RP)
   - Email is the verification channel for OTP later (Path B)
   - Phone collection deferred — SMS OTP is a future upgrade when we start collecting phone numbers
4. Buyer pays deposit — payment method is vaulted (same as RP, reuse existing infrastructure)
5. Store vaulted PM reference in PG (`vaulted_payment_methods` table) — include buyer_email
6. Send deposit receipt to buyer (Item #2)
7. If remaining balance exists → two paths to collect it

---

## #4 — Merchant-Initiated Charge (Path A)

1. Merchant opens Invoice in app
2. System checks: vaulted PM exists AND balance remaining?
   - No → normal invoice view, no special button
   - Yes → show "Charge AMEX ***123 & Send Receipt" button
3. Merchant taps button
4. Authenticated API call to is-payments
5. Charge vaulted PM (off_session / billing agreement / etc.)
6. Outcome:
   - **Success** → update invoice balance to $0 → send receipt to buyer
   - **Failure** → show error (card declined/expired) → merchant contacts buyer for new PM

**Why this is safe:** Merchant is authenticated (logged in). Same security model as RP today.

---

## #5 — Buyer-Initiated Charge (Path B)

1. Buyer clicks checkout link (from invoice email, reminder, etc.)
2. System checks: vaulted PM exists for this buyer + invoice?
   - No → normal checkout, buyer enters payment details as usual
   - Yes → checkout detects vaulted PM
3. ⚠️ **Verification required** — cannot show vaulted PM without auth
4. Send fresh OTP to buyer (SMS if phone available, else email)
5. Buyer enters OTP code
6. System validates:
   - Invalid/expired → error, retry or resend
   - Valid → session verified ✓
7. Show vaulted PM: "Pay $X with AMEX ***123"
8. Buyer taps "Pay Now"
9. Charge vaulted PM (same backend call as Path A)
10. Outcome:
    - **Success** → update invoice balance to $0 → send receipt to buyer
    - **Failure** → show error → offer to enter new payment method or contact merchant

rate limit - only one code

---

## Verification Layer (Stripe Link Model)

**Principle:** Separate the identifier from the verifier.

- **Checkout URL** = identifier (tells us WHICH invoice)
- **OTP** = verifier (proves WHO is paying)

**Channel selection:**
- Phone on file → SMS OTP (strongest) — 6-digit code, 10-min expiry
- Email only → Email OTP (acceptable) — 6-digit code, 15-min expiry

**Session behavior:**
- Verify once per checkout session
- Buyer stays verified if they refresh or navigate back
- New session = re-verify

**Not 2FA but acceptable:**
- Email + SMS = two-step verification (both "possession" category)
- Matches what Stripe Link and Klarna ship
- Sufficient for US/CA invoice payments
- For strict SCA/EU compliance: add CVV (knowledge factor) on top

---

## Implementation Complexity

### #3 — Vaulting at Deposit Payment → Low

| Component | Services/Tech | Why this complexity |
|-----------|---------------|---------------------|
| `setup_future_usage: 'off_session'` on PaymentIntent | is-payments (Stripe SDK) | One flag — RP already does this exact call |
| PayPal billing agreement at deposit | is-payments (PayPal SDK) | Already built for RP, same flow |
| New PG table `vaulted_payment_methods` | is-payments (Knex migration) | Standard schema + migration, no novel design |
| Consent disclosure | Checkout UI | Copy + checkbox, no backend logic |

**Why Low:** Vault mechanism already exists in RP — we're reusing `setup_future_usage`, billing agreements, and the Stripe/PayPal SDK calls. Only new piece is the PG table (standard schema + migration). Email already collected at checkout today, no new UI fields needed for v1 (phone collection deferred to SMS upgrade).

### #4 — Merchant-Initiated Charge (Path A) → Low

| Component | Services/Tech | Why this complexity |
|-----------|---------------|---------------------|
| New endpoint `POST /charge-vaulted` | is-payments (Express router) | Standard authenticated endpoint, same auth middleware as RP |
| Look up vaulted PM | is-payments → PG query | Simple `SELECT` from new table by invoice_id |
| Stripe off_session charge | is-payments (Stripe SDK) | Exact same `PaymentIntent.create({ off_session: true, confirm: true })` as RP scheduling |
| PayPal BA charge | is-payments (PayPal SDK) | Same `executeAgreementPayment` as RP |
| Conditional UI button | is-mobile, is-web-app | Check: PM exists + balance > 0 → show button. Standard conditional render |
| Receipt after charge | is-payments → is-messages | Reuse Item #2 trigger |
| Double-charge prevention | is-payments | Track charge status on invoice, disable button after full payment |

**Why Low:** Every backend piece is a proven pattern from RP. The off_session charge, billing agreement execution, authenticated endpoints — all exist today. We're writing a new endpoint that calls the same underlying functions with a different trigger (button tap instead of schedule). The UI is one conditional button.

### #5 — Buyer-Initiated Charge (Path B) → Low-Medium

| Component | Services/Tech | Why this complexity |
|-----------|---------------|---------------------|
| Detect vaulted PM on checkout load | is-web-app checkout / is-payments API | New query on checkout init — needs new endpoint or extend existing |
| OTP generation | is-payments | Simple code generation (`Math.floor(100000 + random * 900000)`) |
| OTP PG table (`checkout_verifications`) | is-payments (Knex migration) | New table: code, invoice_id, buyer_email, expires_at, verified_at |
| Send OTP via email | is-payments → is-messages | New template, same send mechanism as receipts — no new infra |
| Verify endpoint | is-payments (Express) | New endpoint: validate code + expiry + mark used |
| Session state after verification | is-web-app checkout | **New pattern** — checkout is currently stateless. Need server-side session or short-lived JWT to track "this buyer verified for this invoice" |
| OTP entry UI screen | is-web-app checkout, is-mobile checkout | **New screen/component** — doesn't exist anywhere today. Input field + "Resend" + countdown timer |
| Show vaulted PM conditionally | Checkout UI | Only render saved PM after session is verified |
| Rate limiting / brute-force protection | is-payments | Max attempts per code, max codes per hour — new logic |
| Resend logic | is-payments + UI | Cooldown timer, generate new code, invalidate old one |

**Why Low-Medium:** OTP sends through is-messages (same pipe as receipt emails) — no Twilio bridging or new infra needed. But still bumped above Low because:
1. **Checkout is currently stateless** — there's no concept of a "verified session." Adding this is a new architectural pattern for checkout.
2. **New UI screen** — the OTP entry component doesn't exist. Needs design, implementation on both web and mobile.
3. **Security surface area** — rate limiting, brute force, code expiry, session invalidation. Each is simple alone but they add up.

If we had an existing OTP/verification system, this would be Low. The complexity comes from building the pattern for the first time.

---

## Shipping Strategy

| Phase | What | Effort | Value |
|-------|------|--------|-------|
| **Phase 1** | #3 + #4 (vault + merchant charge) | ~1 sprint | Immediate — merchants can charge saved cards |
| **Phase 2** | #5 (buyer-initiated with email OTP) | ~1-1.5 sprints | Full feature — buyers can self-pay with saved PM |
| **Total** | All | ~2-2.5 sprints | Complete deposit vaulting flow |

**Why this order:**
- #3 + #4 can ship together because #4 is just a button + endpoint on top of #3's table
- Merchants get value immediately (no waiting for OTP system)
- #5 ships separately because the OTP system is new infra — needs design review, security review, and testing
- #4 and #5 share the same backend charge call — once #4 works, #5's payment part is free

**Parallel work opportunities:**
- #3 backend (PG table + vault flag) ‖ #4 UI (button component) — different teams
- #5 backend (OTP service) ‖ #5 UI (OTP entry screen) — can develop in parallel with mocked endpoints

---

## Known Limitations

### Deposits & Milestones CRUD — No Offline Support

All deposit/milestone writes go directly to Parse Cloud Functions with no offline queue:

```typescript
// manage-payment.screen.tsx:225-244
try {
  await savePayment(formValues);     // → Parse.Cloud.run() — requires network
  loadPayments(remoteInvoiceId);     // → refresh Realm cache AFTER success only
  await ensureInvoiceInSync(...);
  navigation.goBack();
} catch (error) {
  alertMessage({ message: errorMessages.errorHasOccurred, onPress: navigation.goBack });
  // ↑ generic error + navigate back — operation is silently lost
}
```

Three things that confirm no offline support:
1. `savePayment()` calls `Parse.Cloud.run()` directly — no local Realm write first
2. `loadPayments()` only runs after a successful save — Realm is a read cache, not a write queue
3. `catch` block just shows a generic error and navigates back — no retry, no offline queue

**Contrast with invoices:** Invoices are offline-first (written to Realm locally, bidirectionally synced via `SyncStore`). `RPayment` is explicitly NOT in the sync table list — it's populated only by explicit `loadPayments()` pulls after successful API calls.

**Impact on D&M features:** Any new deposit/milestone work (vaulting, merchant-initiated charge, PIN setup) inherits this same limitation. Adding offline support would require:
- Adding `RPayment` to the sync engine's table list
- Writing optimistically to Realm before the Parse call
- Wiring in the `offlineOkAction` branch (exists in `sync/utils.ts` but not connected here)

This is a non-trivial change, separate from the D&M feature work itself. Treat as a known limitation for v1.

---

## Stakeholder Sync Notes (Jun 11–15, 2026)

### #1 — One-Step Deposit Creation
Seth's approach:
- Entry point moves to invoice editor
- Default to Flat Amount (straight to number input)
- Percentage becomes a numerical input constrained to 1–100
- Aligned, no concerns raised

### #3 — Vaulting at Deposit Payment
Liz's response:
- Taking the "is vaulting worth it for one remaining payment?" value question to Legal
- Thinking about extending vaulting to **milestone payments** (not just deposit → balance)
- Acknowledged this might require a **Client Portal** for buyers to manage saved payment methods
- Client Portal deferred — heavy lift for this epic, potential to bundle with RP (both need one)

### #4 — Merchant-Initiated Charge
Liz agreed on the consent language / trust dynamic concern. No further questions.

### #5 — Buyer-Initiated (OTP)
Liz raised **GLBA Safeguards Rule** ([16 CFR 314.4(c)(5)](https://www.law.cornell.edu/cfr/text/16/314.4)) — FTC MFA requirement.
Proposed: Magic Link (possession) + PIN (knowledge) = 2FA.

Our response:
- GLBA MFA triggers when someone is "accessing an information system" — more applicable to a portal than a one-time checkout payment
- Whether we're even a "financial institution" under GLBA is ambiguous — SaaS platforms not mentioned in the regulation (written 1999)
- We're not pursuing Magic Link — it's passive, no transaction context, weak chargeback defense (see `otp-verification-design.md`)
- PIN idea is good though — see PIN + Email OTP section in `otp-verification-design.md`

---

## #2 — Partial Payment Receipt — Extended Discussion

### What we currently have
- `transactionId` stored on Parse `Payment` object (MongoDB)
  - Stripe: Payment Intent ID (`pi_3Abc...`)
  - PayPal: Capture ID
  - Manual: `null`
- Not user-facing anywhere (UI or emails)
- Existing receipt emails only cover recurring/automatic payments — not one-time Stripe/PayPal payments
- Success page already shows full payment ledger (Deposit + Milestone entries) + "Download as PDF" button
- History tab (mobile) shows document events (sent, opened, signed) — not payment events

### Confirmation code options

**Option A — Expose existing `transactionId`:**
- Zero backend work — surface `pi_3Abc...` or PayPal capture ID
- Downside: ugly, provider-specific, meaningless to users

**Option B — Generate human-friendly code:**
- Add `confirmationCode` field to Payment object (e.g. `RCP-A3F9X2`)
- Generate at payment confirmation time (nanoid or similar — ~1 line)
- Save alongside `transactionId` on the Parse Payment document (MongoDB — no migration needed)
- Surface in emails + success page
- Effort: a few hours end-to-end

### Receipt access approaches

| Approach | Effort | Notes |
|----------|--------|-------|
| Email IS the receipt | None | Add confirmation code, amount, method, invoice ref to existing email |
| Persistent receipt URL | Low | `checkout.invoicesimple.com/receipt/abc123` — always accessible, buyer can bookmark |
| Download PDF on success page | Low-medium | PDF service already exists for invoices — new receipt template |
| Receipt attached to email | Medium | Generate PDF, attach to confirmation email |
| History tab payment events | Low-medium | Add new `Msg` event type to Parse — one new row per payment, mobile picks up via existing sync |
| Receipts page under Settings | High | New screen, cross-invoice query, pagination, search — separate feature |

### Key insight
The checkout success page already shows the full payment ledger (Deposit -$300, Payment -$200, Balance Due) with a Download PDF button. The main gaps are:
1. Raw `pi_xxx` displayed instead of a human-friendly confirmation code
2. Success page may not be persistently accessible (one-time URL)
3. No payment events in the History tab

### History tab — adding payment events
History tab reads from `Msg` collection in Parse (MongoDB), synced to Realm on mobile.
Adding payment events = write a new `Msg` document at payment success time with a new event type (e.g. `PAYMENT_RECEIVED`, `DEPOSIT_PAID`).
- Backend: one new write in the payment webhook handler
- Mobile: one new `case` in the history screen switch statement
- Low lift overall
