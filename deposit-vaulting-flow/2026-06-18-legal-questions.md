# Legal Questions - Deposit Optimization

Date: 2026-06-18
Thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1781807357587409

---

## Final List (Sent to Legal — Liz's consolidated version)

### BUYER

**Vault with Purchase**

1. Does _requiring_ a buyer to vault their payment method (card or ACH) at the time of purchase require a legal agreement to authorize charges for:
   - The remaining balance on the current invoice?
   - Charges for scope additions not reflected on the original invoice (e.g., buyer requests extra work mid-job)?
   - Future invoices?
2. Does offering a buyer the _option_ to vault their payment method at the time of purchase require a legal agreement to authorize charges for:
   - The remaining balance on the current invoice?
   - Charges for scope additions not reflected on the original invoice (e.g., buyer requests extra work mid-job)?
   - Future invoices?
   - Can the _option_ to vault be defaulted to enabled?
3. What consent language should be used?
   - If a legal agreement is required for any of the above, what is the minimum product requirement for presenting those terms to the buyer? (e.g., displayed upfront vs. available under a "more details" interaction)
4. Any considerations between card vaulting and ACH vaulting?

**Vault without Purchase**

5. Does vaulting a buyer's payment method independently of a purchase require a legal agreement to authorize charges for future invoices?

**Post-Vault**

6. If our checkout is publicly accessible, what is the minimum requirement for a buyer to view, update, or remove their vaulted payment method?
7. Is email OTP (6-digit code with expiry, rate-limited) sufficient verification before charging a vaulted card?
8. What is the minimum product requirement to comply with the FTC's Simple Cancellation rule? _(The cancellation mechanism must be at least as easy to use as enrollment and must be available through the same medium used to consent.)_

### MERCHANT

9. Does using a vaulted token to charge the remaining balance on a specific invoice require a legal agreement between the buyer and the merchant?
   - If the merchant changes the invoice amount or payment due dates after the buyer has vaulted (e.g., mid-job scope change), does the original consent still apply? Or is re-authorization required?
10. Does using a vaulted token to charge a client for future invoices require a legal agreement between the buyer and the merchant?
11. What consent language should be used?
12. Any considerations between card vaulting and ACH vaulting?
13. Any considerations for revocation rights? What obligations does the merchant have when a buyer wants to delete their vaulted payment method?
14. Any considerations for failed payment handling?
15. If the token is stored by the payment processor (Stripe/PayPal) via the merchant's account, what happens to the token when a merchant offboards?

### OTHER

**Estimate → Invoice Flow**

16. Is there legal risk if the approver, the payer, and the client named on the estimate are three different people?
    - _(We don't verify identity at approval or payment. This is especially relevant for merchant-initiated charges later — the vaulted card belongs to whoever paid, not necessarily the named client.)_

---

## Extended Research List (Full team questions, pre-consolidation)

Questions from the full thread before Liz consolidated. Some were dropped as product decisions, some were absorbed into the final list above.

### Estimate to Invoice Buyer Link Flow (Lenmor)

1. Is there legal risk in letting an unauthenticated person trigger a conversion (estimate → invoice), given no money moves until they enter payment info?
   - Context: We're thinking to remove the token/magic-link approach to simplify tech. The existing estimate preview page is already public (no auth). We just add an "Approve" button to it.

2. If the approval link is forwarded to other people and someone else approves, is there legal risk?
   - Context: Same forwarding risk that already exists with our current public document links. The difference is now the forwardee can trigger state change (approval), not just view.

**Sharper framing (2026-07-06), reusing precedent from the existing share-link model:**
- Does clicking "Approve" on an existing public document page (same URL merchants already share with clients) constitute legally binding acceptance?
- The approve action reuses the same security model as current share links (public invoice, checkout) — if Legal is already fine with that sharing model, is "Approve" legally distinct from "viewing," or does it inherit the same accepted risk?
- Forwarding risk (link shared with someone else, who approves) already applies to existing share links today — is this already an accepted risk, or does attaching a state-changing action to it change the analysis?
- Approve link always shows the latest version of the estimate if merchant edits after sending (same as today's share links) — does this change anything, or is it the same behavior with an approve action now attached?

3. Is there legal risk if the client the merchant sent the estimate to is a different person from the one who approves, and also different from the one who pays?
   - Context: We don't verify identity at approval or payment. The merchant creates an estimate for a specific client, but anyone with the link can approve, and anyone with the checkout link can pay (current behavior for invoice). This matters especially for merchant-initiated charges later — the vaulted card belongs to whoever paid, not necessarily the named client in the estimate.
   - _→ Absorbed into final list as Q16_

### Merchant-Initiated Charge (Lenmor — dropped as product decisions)

4. What chargeback liability does IS bear when a merchant initiates a charge? Does IS need to verify "work complete," or is that purely between buyer and merchant?
   - Context: Merchant-initiated charge removes the buyer's ability to withhold payment if unsatisfied. Buyer's recourse becomes chargeback. Is IS exposed, or does liability still sit with the merchant's Stripe/Paypal account?

5. Should there be a pre-charge notification to the buyer before a merchant-initiated charge fires?
   - e.g. a heads-up email for chargeback defense. Is it legally required?

### Buyer-Initiated Charge (Lenmor)

6. Is email OTP (6-digit code with expiry, rate-limited) sufficient verification before charging a vaulted card?
   - Context: Buyer gets an invoice for remaining balance, opens checkout, we see a vaulted card → we send OTP to their email before revealing/charging it.
   - Does IS need to comply with GLBA Safeguards Rule - MFA requirement?
   - What is the minimum verification requirement for US/CA, and what if we expand to EU?
   - _→ Absorbed into final list as Q7_

7. Is there a dollar threshold for buyer-initiated charge? Does it change by adding a factor to authentication?
   - _→ Dropped as product decision_

### Fred's Questions (absorbed into final list)

- Does requiring or offering the option to vault require an additional legal agreement to authorize charges not present on the original invoice? → _absorbed into Q1/Q2_
- Is the merchant bound by the schedule initially written on the invoice? → _absorbed into Q9 sub-bullet_

### Seth's Questions (absorbed into final list)

- Bare minimum product requirement to present legal agreement terms to buyer → _absorbed into Q3 sub-bullet_
- FTC Simple Cancellation rule minimum product requirement → _absorbed into Q8_
