# Deposit vaulting — Initial Thoughts

> For sync with Liz and Seth (after quick sync with Juan)

---

## #1 — One Step Deposit Creation

- No concerns here
- Seems it's just a UI/UX change, just removing the extra %/flat step or resurfacing the Deposit

---

## #2 — Partial Payment Receipt

- I tested this and we currently already send a "Deposit Complete" email when a deposit gets paid and a "Payment Complete" email when a milestone gets paid

- Do we mean something else here?

- Emails we already have infra

- SMS could be a heavier lift, I'm still exploring if we have a usable infra or we have to integrate from scratch

---

## #3 — Vaulting at Deposit Payment

If I understand correctly, this is the flow in my head:

1. Merchant creates Invoice with a deposit amount
2. Merchant sends invoice to client via email
3. Buyer opens checkout link
    - Current system already detects the deposit amount to be paid on checkout
4. (NEW) Vaulting flow to save client's payment method
    - Buyer enters email (same as RP)
    - Email will be used as the verification channel for OTP on #5 Buyer-initiated charge
    - SMS can be an option too but I'm leaning to defer SMS as a V2 (see note above)
5. Buyer pays deposit — payment method is vaulted (same as RP, reuse existing infrastructure)
6. Send deposit receipt to buyer (#2)
7. Once we have the vaulted payment method saved, if they open checkout again at a later time, and there is a remaining balance, it can then be paid through either path: #4 Merchant-initiated and #5 Buyer-initiated

Notes/concerns:

- There might not be enough motivation/value for client to vault their card if it will only be used once for the remaining balance — is that worth the trouble compared to just re-entering card details one more time?

---

## #4 — Merchant-Initiated Charge

- Tech-wise, I couldn't see any big issues here, just charging the vaulted payment method like we do in RP
- Merchant is authenticated so not a security issue

Notes/concerns:

- At least from personal experience with contractors, client pays remaining balance when job is actually complete and they're satisfied with the service. So there was that verification step from the client on regular checkout.

- With merchant-initiated, some charges can be potentially contentious with client. If we want to go ahead with this, we'll need to think about UX a bit more. Some ideas:
    - During vaulting, the consent language should be explicit like `you are trusting the merchant...`
    - During merchant-initiated charge, show confirmation modal that `by doing this, you are charging client who trusts that the work is done...`
    (just examples, need to polish)

- Strong consent language at vault time also serves as chargeback defense — if a buyer disputes the charge later, our evidence is the explicit consent they agreed to when saving their card.

- **This needs product/design input** — merchant-initiated charging changes the trust dynamic between merchant and buyer. It's not just a UX polish question, it's a product-market tension that needs deliberate design.

---

## #5 — Buyer-Initiated

This is how I understand it:

1. Buyer clicks checkout link from invoice email
2. System checks: vaulted PM exists for this buyer + invoice?
   - No → normal checkout, buyer enters payment details as usual
   - Yes → checkout detects vaulted PM
3. **Verification required** — cannot show vaulted PM without auth
4. Send fresh OTP to buyer
5. Buyer enters OTP code
6. System validates:
   - Invalid/expired → error, retry or resend
   - Valid → session verified
7. Show vaulted PM: "Pay $X with AMEX ***123"
8. Buyer taps "Pay Now"
9. Charge vaulted PM (same backend call as #4 merchant-initiated)
10. Outcome:
    - **Success** → update invoice balance → send receipt to buyer
    - **Failure** → show error → offer to enter new payment method or contact merchant

Notes/concerns:

- Magic link is not good enough for payments because it's "passive" (click from inbox)
    - Magic links are common for accessing a site, password reset, and viewing/updating user info, but not for authorizing a charge
    - Industry standard for authorizing a charge is OTP / biometrics / 3DS
- OTP is "active" (present at checkout + inbox at same time) — fresh, rate-limited
    - We can draw some inspiration from Stripe Link
    - Email is acceptable for now for US/CA
    - SMS can be v2; it also adds another factor that is more acceptable for EU (PSD2/SCA)
- **No US/CA legal requirement for two-factor on payments** (unlike EU/UK/India/Japan/Australia which mandate SCA). Email OTP for us is a risk/liability trade-off, not a compliance gap. The main concern is chargeback defensibility, not regulatory compliance.

---

## #6 — Buyer Portal

- Seems a heavy lift for this epic, but in the near future, we can potentially "bundle" the value with RP since IIRC we also need a client portal there

---

## #7 — Converting Estimate to Invoice

- I'll reach out to PCore team to get more info
