# References — Deposit Vaulting Flow

Collected links for inline citations in our docs.

---

## GLBA Safeguards Rule

| Source | URL |
|--------|-----|
| Federal Register Final Rule (86 FR 70272) | https://www.govinfo.gov/content/pkg/FR-2021-12-09/pdf/2021-25736.pdf |
| 16 CFR 314.4 (regulatory text) | https://www.law.cornell.edu/cfr/text/16/314.4 |
| 16 CFR 314.2 (definitions) | https://www.law.cornell.edu/cfr/text/16/314.2 |
| 16 CFR 314.1 (purpose and scope) | https://www.law.cornell.edu/cfr/text/16/314.1 |
| 15 U.S.C. 6809 (GLBA statutory definitions) | https://www.law.cornell.edu/uscode/text/15/6809 |
| 12 CFR 225.28 (permissible nonbanking activities) | https://www.law.cornell.edu/cfr/text/12/225.28 |

---

## OTP / Payment Authorization Industry Standards

| Source | URL |
|--------|-----|
| OWASP Transaction Authorization Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html |
| NIST SP 800-63B Digital Identity Guidelines | https://pages.nist.gov/800-63-4/sp800-63b.html |
| Stripe Strong Customer Authentication (PSD2/SCA) | https://docs.stripe.com/strong-customer-authentication |
| Stripe Setup Intents (stored cards) | https://docs.stripe.com/payments/setup-intents |
| Stripe Link (OTP for saved PMs) | https://docs.stripe.com/payments/link |
| Adyen 3D Secure docs | https://docs.adyen.com/online-payments/3d-secure/ |

---

## PSD2 / SCA / Global Regulations

| Source | URL |
|--------|-----|
| PSD2 SCA requirement (Stripe) | https://docs.stripe.com/strong-customer-authentication |
| FCA (UK) SCA enforcement | https://www.fca.org.uk/firms/strong-customer-authentication |
| RBI AFA (India) | https://www.rbi.org.in/Scripts/NotificationUser.aspx?Id=11822 |

---

## US Payment Regulations

| Source | URL |
|--------|-----|
| Reg E (Electronic Fund Transfer Act) | https://www.law.cornell.edu/cfr/text/12/part-1005 |
| PCI DSS | https://www.pcisecuritystandards.org/ |
| Visa dispute rules | https://usa.visa.com/support/small-business/dispute-resolution.html |

---

## GLBA Clarifications

### What is it
- **GLBA** = Gramm-Leach-Bliley Act (the law, passed by Congress 1999)
- **16 CFR Part 314** = the "Safeguards Rule" (regulation implementing GLBA, issued by FTC)
- "GLBA Safeguards Rule" is the common name for 16 CFR 314
- Enforced by the FTC

### MFA requirement — 16 CFR 314.4(c)(5)
> "Implement multi-factor authentication for any individual accessing any information system"

### "Information system" — 16 CFR 314.2(j)
> "a discrete set of electronic information resources organized for the collection, processing, maintenance, use, sharing, dissemination or disposition of electronic information containing customer information or connected to a system containing customer information"

### "Financial institution" — 16 CFR 314.2(h)
- **(h)(1)** — definition: any institution engaging in financial activities per Bank Holding Company Act section 4(k)
- **(h)(2)** — 13 examples of what IS covered: mortgage brokers, check cashers, tax prep, money transfer, debt collectors, investment advisors, finders, etc.
- **(h)(3)** — explicit exclusions: CFTC-regulated entities, Federal Agricultural Mortgage Corp, entities not "significantly engaged"
- **(h)(4)** — NOT significantly engaged examples: retailers with lay-away plans, merchants allowing tabs, grocery stores cashing checks

### Where does Invoice Simple fall?
- **Not listed** in covered examples (h)(2)
- **Not listed** in excluded examples (h)(3)/(h)(4) either
- SaaS platforms are never mentioned — didn't exist when GLBA was written (1999)
- Closest excluded analogy: "retailer accepting third-party credit cards" (from Federal Register commentary [86 FR 70276](https://www.govinfo.gov/content/pkg/FR-2021-12-09/pdf/2021-25736.pdf), not from the regulation text itself)
- We don't lend, issue credit, collect debt, transfer funds, or broker — all we do is facilitate invoicing and connect to Stripe/PayPal

### Does MFA requirement apply to our checkout OTP flow?
- **Ambiguous.** The reg says MFA for "accessing any information system"
- It does NOT explicitly distinguish between "browsing stored data" vs "authorizing a transaction"
- Arguments it doesn't apply: buyer isn't browsing/managing data, just initiating a payment
- Arguments it might: buyer triggers a read of stored card info (last 4 digits displayed)
- **This is interpretation, not stated in the regulation**

### Bottom line
- Whether we're even a "financial institution" under GLBA = gray area (probably not, but not explicit)
- Whether checkout = "accessing an information system" = also gray area
- Legal should weigh in if this is a concern

---

## Usage

When writing docs, cite inline like:
- "per the [GLBA Safeguards Rule](https://www.law.cornell.edu/cfr/text/16/314.4)..."
- "OWASP's [Transaction Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html) recommends..."
- "Stripe Link uses [SMS OTP for saved payment methods](https://docs.stripe.com/payments/link)..."
