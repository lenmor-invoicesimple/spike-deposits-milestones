# Deposit Optimizations PRD — Technical Evaluation

> Date: 2026-06-09
> Source: [Deposits Optimizations PRD](https://everpro-tech.atlassian.net/wiki/spaces/PM/pages/1213300774/Deposits+Optimizations+PRD)
> Status: Evaluation in progress

---

## Item 1: One-Step Deposit Creation

**PRD requirement:** Remove the extra percent/flat step. Return to the old "just enter amount and send" flow.

**Technical assessment:** Pure UX/UI change. Backend already supports both deposit types — the `Payment` object in Parse stores `amount` and `percentage`. No backend changes needed.

**Options:**
- Default to flat amount, show percent as secondary toggle
- Inline both options on same screen instead of separate step
- Auto-detect: if number < total matches a round percentage, suggest as chip

**Feasibility:** ✅ Very High — quickest win in the PRD
**Complexity:** Low (mobile + web UI only)
**Backend work:** None

---

## Item 2: Partial Payment Receipt (Email + SMS)

**PRD requirement:** Auto-send receipt when a deposit/partial payment is made on an invoice. Merchants currently create handwritten receipts.

### Email Receipt

**Technical assessment:** Piggybacks on existing `is-messages` infrastructure. No new system (like APR scheduling) needed.

**How it works:**
1. Deposit payment succeeds → Stripe/PayPal webhook → is-payments
2. is-payments calls `is-messages` with "deposit receipt" template
3. Immediate, fire-and-forget — no scheduling needed

**Why NOT like APR:**
- APR = future-dated, needs scheduling, cancellation logic, debounced pipeline
- Receipt = synchronous, event-driven, fire once after payment confirmation

**Work needed:**
1. New email template in is-messages (amount paid, date, remaining balance, invoice link)
2. New trigger in payment confirmation flow (Stripe/PayPal success webhook handler for deposits)
3. Template data assembly (pull invoice total, amount paid, remaining balance)

**Feasibility:** ✅ High
**Complexity:** Low-Medium (template + trigger wiring)

### SMS/Text Receipt

**Technical assessment:** We ALREADY have Twilio SMS infrastructure in the legacy `api/` codebase.

**What exists today:**
- Twilio integration (`TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_MESSAGING_SERVICE_SID`)
- SMS sending function (`sendMobileSMSCampaignMessage`) — currently used for promotional invoice links
- Consent tracking model (`ConsentType.SMS_PROMOTIONAL`) with `consent_given_at`/`consent_revoked_at`
- Phone number validation (E.164 format, US numbers)
- Opt-out mechanism ("Reply STOP to opt-out")
- Located in: `/is5/api/src/providers/twilio/`, `/is5/api/src/services/sms/`

**What's NOT there yet:**
- Transactional receipt SMS template (only promotional messages today)
- Some SMS routes defined but not mounted (partially built, never shipped)
- Twilio lives in legacy `api/`, not in `is-services/is-messages`

**What SMS receipt requires:**
1. New transactional SMS template (amount paid, remaining balance, invoice link)
2. Trigger in payment confirmation flow (same trigger point as email receipt)
3. Buyer phone number at checkout — **open question: do we collect this today?**
4. TCPA: transactional messages have lower consent bar than promotional, but still need disclosure

**Key considerations:**
- Twilio infra exists → no new provider setup needed
- Consent model exists → extend with `ConsentType.SMS_TRANSACTIONAL` or similar
- May want to move Twilio integration from legacy `api/` into `is-services` long-term
- Receipt is transactional (lower TCPA bar) vs existing promotional messages (higher bar)

**Feasibility:** ✅ High (infra exists, just needs new template + trigger)
**Complexity:** Low-Medium (if buyer phone number already collected) / Medium (if phone collection UI needed)
**Recommendation:** Can ship SMS receipt in v1 alongside email if buyer phone is available; otherwise email-only v1, SMS fast-follow

---

## Item 3: Mandatory Card-on-File (Vaulting)

**PRD requirement:** "Request Mandatory Payment Method on File with Deposit payment on an Invoice." When buyer pays the deposit, their payment method is saved for future charges on the remaining balance.

**Scope note:** Estimates are OUT of v1 scope. The mockup shows Estimates but Liz confirmed v1 is Invoice → Deposit only. Estimate vaulting is a separate future workstream.

**How it connects to the next items:**
- Item 3 = the vaulting moment (card gets saved during deposit payment)
- Item 4 = Path A: merchant uses the vaulted card (authenticated, safe)
- Item 5 = Path B: buyer uses the vaulted card (public URL, security concern)

These aren't three separate features — they're one flow with a fork at the end.

### How Stripe Vaulting Works

Stripe's built-in mechanism: `setup_future_usage: 'off_session'` on a PaymentIntent.

```
PaymentIntent.create({
  amount: 1000,              // deposit amount
  customer: 'cus_xxx',      // Stripe Customer (must exist)
  setup_future_usage: 'off_session',  // saves the card
})
```

When buyer pays, Stripe automatically:
1. Charges the deposit amount
2. Saves the payment method to the Stripe Customer
3. Returns a `payment_method` ID we can reuse later

### What We Need

| Piece | Exists today? | Work |
|-------|--------------|------|
| Stripe Customer per buyer | ❓ Need to check | May need to create Customers for guest payers |
| `setup_future_usage` on PaymentIntent | ❌ | Add flag to checkout PaymentIntent creation |
| Store vaulted PM reference | ❌ | New PG table in is-payments |
| Consent UI at checkout | ❌ | Clear disclosure ("card will be saved for future payments") |
| GDPR: right to delete | Low effort | Call Stripe API to detach PM on request |

### Proposed PG Table

```sql
vaulted_payment_methods (
  id, account_id, buyer_email, invoice_id,
  stripe_customer_id, stripe_payment_method_id,
  card_last4, card_brand,
  consent_given_at, consent_scope, status,
  created_at, updated_at
)
```

### PayPal Vaulting

PayPal uses Billing Agreements — buyer approves future charges from this merchant by logging into PayPal. Different mechanism but same end result: we hold a reference we can charge later. PayPal login IS the buyer auth.

### Security Model

Card details never touch our servers — Stripe holds them. We only store a `pm_xxx` reference. The security question is: who can TRIGGER a charge using that reference?

**Feasibility:** ✅ High for Stripe (proven pattern from RP)
**Complexity:** Medium (Stripe Customer management + new table + consent UI)
**PayPal:** Separate workstream, billing agreements are different mechanism

---

## Item 4: Merchant-Initiated Charge (Path A)

**PRD requirement:** "Merchant-initiated payment to card on file for remaining balance."

**What the mockup shows:** Merchant opens invoice that has a remaining balance + vaulted card → sees button "Charge AMEX ***123 & Send Receipt" → taps → card charged + receipt sent.

**How it works:**

1. Merchant taps "Charge" → mobile/web calls is-payments endpoint (authenticated)
2. is-payments looks up vaulted `payment_method_id` for this buyer/invoice
3. Calls Stripe:
   ```
   PaymentIntent.create({
     amount: remaining_balance,
     customer: 'cus_xxx',
     payment_method: 'pm_xxx',
     off_session: true,
     confirm: true
   })
   ```
4. Stripe charges the card → webhook confirms → receipt sends to buyer

**The button appears contextually when:**
- Invoice has remaining balance
- Buyer has a vaulted card on file
- (Per Liz's description)

**What we need:**

| Piece | Exists today? | Work |
|-------|--------------|------|
| Authenticated merchant API | ✅ (is-payments has auth) | New endpoint |
| Stripe off-session charge | ✅ (RP already does this) | Same pattern |
| Look up vaulted PM by buyer/invoice | ❌ | Query new PG table from Item 3 |
| Receipt trigger after charge | ❌ | Item 2 receipt work |
| Mobile UI ("Charge & Send Receipt" button) | ❌ | New conditional button |

**Why this is safe:** Merchant is authenticated (logged in). Only they can trigger the charge. Clear authorization chain. Same security model as Recurring Payments today.

**Edge cases:**
- Card declined (expired, insufficient funds) → show error, merchant asks buyer for new card
- Double-charge prevention → track what's been charged, disable button after full payment
- Partial charge? → PRD shows full remaining balance only (no partial)

**Feasibility:** ✅ High — proven pattern from RP, just a new endpoint + UI button
**Complexity:** Low-Medium

---

## Item 5: Buyer-Initiated Charge (Path B)

**PRD requirement:** "Buyer-initiated payment to card on file for remaining balance."

**What the mockup shows:** Buyer opens checkout (public URL) → checkout recognizes vaulted card → shows "Card on File: AMEX ***123" + "Pay Now" → one click charges the card.

**How buyer gets there:** Same as today — clicks checkout link from any invoice email (original send, reminder, etc.). No new email needed. Checkout just behaves differently when a vaulted card exists.

### ⚠️ CRITICAL SECURITY CHALLENGE

**The fundamental problem:**

Today's public checkout is safe because the buyer enters THEIR OWN card details. Entering your card IS the authentication — you prove you're the cardholder by having the card.

With vaulting, the public checkout shows SOMEONE ELSE's previously-saved card + a "Pay Now" button. Anyone with the URL can trigger a charge on that card without proving they're the cardholder.

**This applies equally to Stripe AND PayPal:**
- Stripe: vaulted card charged without CVV or 3D Secure
- PayPal: vaulted billing agreement charged without PayPal re-login

**PayPal nuance:** PayPal buyers can fund their account with cards, bank, balance, Pay Later, etc. From our side, we only hold a billing agreement reference — we don't know or interact with the underlying funding source. But that doesn't matter: the risk is the same. If the public checkout can fire "charge billing agreement BA-xxx," anyone with the URL can do that, regardless of what's behind the billing agreement.

If the checkout just calls the backend saying "charge the vaulted method," there's zero verification that the clicker is the person who originally vaulted.

### Why This Is Not Acceptable

| Risk | Impact |
|------|--------|
| Unauthorized charge | Anyone with URL triggers charge on buyer's card |
| Chargeback | Buyer disputes unrecognized charge → merchant loses money + fees |
| PCI/compliance | No cardholder verification = negligent from compliance standpoint |
| EU SCA violation | Strong Customer Authentication requires 2FA for customer-initiated online payments |
| Reputational | Buyer blames Invoice Simple, not the merchant |

### Recommended Approach: Fresh OTP at Checkout Time (Payment-Method Agnostic)

**Why agnostic:** We support (or will support) Stripe cards, PayPal, Apple Pay, Google Pay, Stripe Link, ACH. Each has its own native verification (CVV, PayPal login, biometrics, etc.) — we can't build a different auth flow for each. We need ONE verification layer that works regardless of what's vaulted behind it.

**The solution: Fresh email OTP sent when buyer opens checkout with a vaulted PM.**

```
Buyer clicks checkout URL →
  we detect vaulted PM exists →
  send fresh OTP code to buyer's email →
  buyer enters code on checkout →
  verified → show vaulted PM → "Pay Now" →
  charge whatever PM type is on file
```

**What this proves:** Person has access to this email *right now* — same email used when vaulting. Chain: "same email = same person who authorized vaulting."

**Why this works across all payment methods:**
- Stripe card → OTP → charge `pm_xxx`
- PayPal BA → OTP → charge `BA-xxx`
- Apple Pay → OTP → charge vaulted token
- Google Pay → OTP → charge vaulted token
- Stripe Link → OTP → charge Link reference
- ACH → OTP → charge `ba_xxx`

**Architecture:** Verification layer is entirely ours, decoupled from payment provider. Add a new PM? Zero changes to auth. Upgrade email→SMS later? Zero changes to payment logic.

**What it requires:**
1. Fresh code generation per checkout session (`crypto.randomUUID()` or 6-digit numeric)
2. PG table: `{ code, invoice_id, buyer_email, created_at, expires_at, verified_at }`
3. Send via is-messages (same infra as receipt email)
4. Verify endpoint: check code + expiry → unlock vaulted PM display
5. Expiry: short-lived (10-15 minutes), single-use

**Comparison to Stripe Link:** Same model — Stripe Link sends a fresh code to your phone/email every time you pay. We're replicating that pattern on our side so it's provider-agnostic.

**Strength assessment:**
- Fresh SMS OTP is the strongest option (SCA-compliant, tied to SIM, industry standard via Stripe Link/Klarna)
- Fresh email OTP can also be used — moderate-to-good for US/CA market, acceptable when buyer phone number isn't available
- Email and SMS can coexist — offer SMS if phone is available, fall back to email

### Alternative Approaches (Discussed, Not Recommended for v1)

**Alternative A: CVV re-entry via Stripe Elements (cards only):**
- Proves physical card possession — strongest signal for chargebacks
- CVV goes browser → Stripe directly, we never see/store it. PCI-compliant.
- UX: "Pay $1,500 with AMEX ***123" → [CVV: ___] → [Pay Now]
- ❌ Problem: Only works for cards. No equivalent for PayPal, Apple Pay, Google Pay, ACH.
- ❌ Problem: Requires different UX per payment method — not scalable.

**Alternative B: PayPal re-login (PayPal only):**
- Force buyer to re-authenticate with PayPal before charging billing agreement
- PayPal login IS their 2FA (email + password, sometimes SMS)
- ✅ Strong security — PayPal handles it natively
- ❌ Problem: Only works for PayPal. Different UX from card flow.

**Alternative C: Stale OTP from vault time (Juan's initial idea):**
- Code sent at vault time → buyer re-enters it weeks later when paying
- Better than raw public URL, but code is stale — if email was compromised anytime between vaulting and charging, code is compromised too
- ❌ Not industry standard — Stripe Link sends fresh codes per-transaction precisely because stale codes aren't sufficient
- ❌ Won't hold up in chargeback dispute ("buyer entered a code we emailed 3 weeks ago")

**Alternative D: Magic link alone (weakest):**
- Proves email access only, not card possession — those are different things
- Vulnerable to: forwarded emails ("can you pay this for me?"), shared inboxes, compromised accounts
- A magic link click is not evidence the cardholder authorized the charge — won't hold up in a chargeback dispute
- Does not meet EU SCA requirements (legally requires two factors)
- Same risk profile as a password reset link — acceptable for low-value identity actions, NOT for charging a saved card
- Note: for US/Canada with low-value transactions and a known buyer-merchant relationship, magic link alone may be *practically* fine, but it's not best practice and leaves us exposed if chargeback rates climb or we serve EU buyers

### Stripe Link Reference Model

Stripe Link is the closest industry analog to what we're building. How it works:

1. **Save**: Buyer enters email + payment info on any Link-enabled checkout → opts to save
2. **Recognize**: On next purchase, buyer enters email → Link detects saved payment info
3. **Verify**: Fresh SMS OTP sent to buyer's phone → buyer enters code
4. **Pay**: Saved payment details auto-filled → one-click pay

**Key design choices in Link:**
- Email is the **identifier** (finds your account), phone is the **verifier** (proves it's you) — two separate channels
- OTP is **per-session**, not per-transaction — verify once, then pay without re-verifying within that session
- Device trust via **cookies** — known devices skip OTP on return visits
- No login, no password, no portal — OTP alone is sufficient for payment authorization
- PCI Level 1 certified, data encrypted at rest
- Result: +14% returning user conversion, 3x faster checkout

**Ideas we can borrow:**

1. **Session-based verification** — verify once per checkout session, not per page load. If buyer refreshes or navigates back, they stay verified. Store state in a short-lived server-side session (not a cookie on public checkout).

2. **Separate identifier from verifier** — the checkout URL identifies the invoice (who/what). The OTP proves the person (auth). These are independent layers. Even with email OTP in v1, the principle holds.

3. **Graceful fallback** — if SMS unavailable (no phone number), fall back to email OTP. Link doesn't do this (SMS-only), but we can offer both channels with SMS preferred when available.

4. **Device trust (future)** — on known/verified devices, skip OTP for repeat payments on the same invoice. Not for v1 (our checkout is stateless/public), but a future UX improvement for returning buyers.

5. **No portal needed** — Link proves that OTP-based verification is sufficient for payment authorization without requiring accounts, passwords, or portals. Validates our decision to skip Buyer Portal in v1.

### Is Email + SMS Considered 2FA?

**Strictly: no.** Two-factor authentication requires two DIFFERENT categories:

| Category | Examples |
|----------|----------|
| Something you **know** | Password, PIN, CVV |
| Something you **have** | Phone (SIM), hardware token, physical card |
| Something you **are** | Fingerprint, Face ID |

Email access + SMS OTP are both "something you have" (possession × 2). Same category = not true 2FA under SCA/PSD2 rules.

**In practice:** Two independent channels (email inbox + phone SIM) are very difficult to compromise simultaneously. The industry calls this "2-step verification" and it's widely accepted for payments in low-risk contexts (US/CA, known buyer-merchant relationship, moderate transaction amounts).

**What Stripe Link does:** Just SMS OTP alone (single factor). They compensate with backend risk signals — device fingerprinting, behavioral analytics, fraud ML. Stripe absorbs the fraud liability.

**For our purposes:**
- Email link + SMS OTP = "2-step verification" — strong enough for US/CA invoice payments
- Not formal 2FA, but matches what Stripe Link and Klarna ship
- If we ever need strict SCA compliance (EU expansion): add CVV re-entry (knowledge factor) on top of OTP (possession factor) — that's true 2FA

**True 2FA combinations (for reference):**
- SMS OTP (have) + CVV (know) ✅
- Email link (have) + PIN (know) ✅
- SMS OTP (have) + biometric (are) ✅

### Upgrade Path (v1 → v2)

| Phase | Channel | Strength | Trigger |
|-------|---------|----------|---------|
| v1 | Fresh email OTP | Moderate-Good | Ship now, no new data needed |
| v2 | Fresh SMS OTP | Strong (Stripe Link level) | When we collect buyer phone numbers |
| v2+ | Email + SMS combo | Very strong | SCA compliance if we expand to EU |

### Industry Comparison

| Service | How they verify buyer for saved-card charges | Security level |
|---------|----------------------------------------------|----------------|
| Amazon | Full login (email + password) | High |
| Uber | Full login + biometrics | High |
| Apple Pay | Biometric (Face ID / fingerprint) | Very high |
| PayPal | Full PayPal login | High |
| Stripe Link | Fresh OTP to phone/email per transaction | Medium-High |
| Klarna | Fresh OTP to phone per transaction | Medium-High |
| **Our proposal: fresh email OTP** | Fresh code to email per transaction | **Medium-High** |
| **Our proposal + SMS (v2)** | Fresh code to phone per transaction | **High** |

### Feasibility Assessment

| Approach | Feasibility | Acceptable? | Payment-method agnostic? |
|----------|-------------|-------------|--------------------------|
| Public checkout + vaulted PM + no auth | ✅ Easy to build | ❌ NOT acceptable | N/A |
| Magic link only (click = verified) | ✅ Easy to build | ⚠️ Weak | ✅ Yes |
| Stale OTP from vault time | ✅ Easy to build | ⚠️ Weak (code is stale) | ✅ Yes |
| **Fresh email OTP (recommended v1)** | ✅ Feasible | ✅ Acceptable | ✅ Yes |
| Fresh SMS OTP (v2) | ✅ Feasible (Twilio exists) | ✅ Strong | ✅ Yes |
| CVV re-entry (Stripe Elements) | ✅ Feasible | ✅ Strong | ❌ Cards only |
| PayPal re-login | ✅ Feasible | ✅ Strong | ❌ PayPal only |
| Buyer Portal with full auth | Medium effort (new infra) | ✅ Best security | ✅ Yes |

**Recommendation for v1:**
1. Ship Path A (merchant-initiated) — no auth needed, merchant is authenticated
2. Ship Path B with **fresh email OTP** — agnostic, minimal infra, proven pattern (Stripe Link model)
3. Upgrade to SMS OTP when buyer phone numbers are available

Do NOT ship Path B with just a public checkout URL showing vaulted cards.

**Feasibility:** ✅ High — fresh OTP is simpler than CVV + PayPal re-auth combined
**Complexity:** Medium (OTP generation + email send + verify endpoint + short-lived PG rows)
**Key advantage:** One verification flow for all payment methods, now and future

---

## Item 6: Buyer Portal

_Evaluation pending_

---

## Item 7: Estimate → Invoice Auto-Conversion

_Evaluation pending_

---
