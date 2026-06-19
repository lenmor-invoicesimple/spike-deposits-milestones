# Legal Questions - Deposit Optimization

Date: 2026-06-18
Thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1781807357587409

---

## Liz's Questions — Payment Method Vaulting

### BUYER

**Vault with Purchase**

1. Does _requiring_ a buyer to vault their payment method (card or ACH) at the time of purchase require a legal agreement to authorize charges for:
   - The remaining balance on the current invoice?
   - Future invoices?
2. Does offering a buyer the _option_ to vault their payment method at the time of purchase require a legal agreement to authorize charges for:
   - The remaining balance on the current invoice?
   - Future invoices?
   - Can the _option_ to vault be defaulted to enabled?
3. What consent language should be used?
4. Any considerations between card vaulting and ACH vaulting?

**Vault without Purchase**

5. Does vaulting a buyer's payment method independently of a purchase require a legal agreement to authorize charges for future invoices?

**Post-Vault**

6. If our checkout is publicly accessible, what is the minimum requirement for a buyer to view, update, or remove their vaulted payment method?

### MERCHANT

7. Does using a vaulted token to charge the remaining balance on a specific invoice require a legal agreement between the buyer and the merchant?
8. Does using a vaulted token to charge a client for future invoices require a legal agreement between the buyer and the merchant?
9. What consent language should be used?
10. Any considerations between card vaulting and ACH vaulting?
11. Any considerations for revocation rights? What obligations does the merchant have when a buyer wants to delete their vaulted payment method?
12. Any considerations for failed payment handling?
13. If the token is stored by the payment processor (Stripe/PayPal) via the merchant's account, what happens to the token when a merchant offboards?

---

## Fred's Questions

14. Does _requiring_ or _offering the option_ to vault their payment method at the time of purchase require an additional legal agreement to authorize additional charges not present on the original invoice? (e.g. if during the job, the buyer asks for an extra something)
15. When the buyer vaults their payment method at the time of purchase (whether it's required or offered), is the merchant bound by the schedule initially written on the invoice? (e.g. if the job is finished sooner than originally anticipated, can the merchant initiate a charge for the remaining balance early?)

---

## Seth's Questions

16. Should a legal agreement be required for authorization, what is the bare minimum measure required from our product to present these terms to the buyer? (e.g. presenting upfront vs. collapsing under a "more details" interaction)
17. What is the bare minimum measure required for our product to fulfill FTC's Simple Cancellation rule?
    > Simple Cancellation: The cancellation mechanism must be at least as easy to use as the enrollment process and must be *available through the same medium used to consent*

---

## Lenmor's Questions

### Estimate to Invoice Buyer Link Flow

1. Is there legal risk in letting an unauthenticated person trigger a conversion (estimate → invoice), given no money moves until they enter payment info?
   - Context: We're thinking to remove the token/magic-link approach to simplify tech. The existing estimate preview page is already public (no auth). We just add an "Approve" button to it.

2. If the approval link is forwarded to other people and someone else approves, is there legal risk?
   - Context: Same forwarding risk that already exists with our current public document links. The difference is now the forwardee can trigger state change (approval), not just view.

3. Is there legal risk if the client the merchant sent the estimate to is a different person from the one who approves, and also different from the one who pays?
   - Context: We don't verify identity at approval or payment. The merchant creates an estimate for a specific client, but anyone with the link can approve, and anyone with the checkout link can pay (current behavior for invoice). This matters especially for merchant-initiated charges later — the vaulted card belongs to whoever paid, not necessarily the named client in the estimate.

### Merchant-Initiated Charge

4. If the merchant changes the invoice amount after the buyer vaulted (e.g., scope change mid-job), does the original vaulting consent still apply? Or is re-authorization needed?
5. (Fred's Q) Can the merchant charge earlier than the originally agreed schedule?
6. What chargeback liability does IS bear when a merchant initiates a charge? Does IS need to verify "work complete," or is that purely between buyer and merchant?
   - Context: Merchant-initiated charge removes the buyer's ability to withhold payment if unsatisfied. Buyer's recourse becomes chargeback. Is IS exposed, or does liability still sit with the merchant's Stripe/Paypal account?
7. Should there be a pre-charge notification to the buyer before a merchant-initiated charge fires?
   - e.g. a heads-up email for chargeback defense. Is it legally required?

### Buyer-Initiated Charge

8. Is email OTP (6-digit code with expiry, rate-limited) sufficient verification before charging a vaulted card?
   - Context: Buyer gets an invoice for remaining balance, opens checkout, we see a vaulted card → we send OTP to their email before revealing/charging it.
   - Does IS need to comply with GLBA Safeguards Rule - MFA requirement?
   - What is the minimum verification requirement for US/CA, and what if we expand to EU?
9. Is there a dollar threshold for buyer-initiated charge? Does it change by adding a factor to authentication?
