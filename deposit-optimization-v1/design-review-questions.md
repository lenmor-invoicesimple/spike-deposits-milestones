# Design Review — Product/Design Questions

**Epic:** IS-11250 — Deposit Optimization v1
**Date:** 2026-07-07
**Status:** In review — questions for Liz/Seth

---

## Screen 1: Estimate Deposit Setup (Merchant Editor)

**Design reference:** Figma "Estimate Deposit Setup" — mobile editor with "Request Deposit" toggle, Flat Amount / Percentage modes, payments tile below.

### Technical Summary

- Deposit fields (`depositType`, `depositRate`, `depositAmount`) already exist in Mongo — shared collection with invoices
- 7 guards currently prevent estimates from using deposits — all need relaxing
- Deposit calculator already works for any doc with deposit fields
- Effort: S (guard removal + type extension)

### Questions

1. **Payments tile prerequisite** — Can a merchant enable "Request Deposit" on an estimate if they haven't set up Stripe/PayPal yet? Or does the toggle only appear once payments are configured? (Today on invoices, the payments section only shows if the merchant has onboarded to IS Payments.)

2. **Deposit without payments enabled** — If deposit is toggled ON but no payment method (Stripe/PayPal) is active, what happens? Is the deposit just informational on the PDF/email, or must online payment be enabled for deposit to work?

3. **Deposit validation** — Can the deposit amount exceed the total? Can it be $0 with the toggle ON? The design shows $0 as the default when toggled on — is that a valid state to send?

4. **Percentage options** — First design showed specific percentage presets (25%, 50%). The Direction variations show free-entry. Is percentage free-entry (any %) or preset values only? (Existing Invoice deposit uses a preset enum: 10/20/30/40/50%.)

5. **Web parity** — Is this mobile-only first, or does the web Estimate editor also need the deposit section simultaneously?

---

## Screen 2: Buyer Approval → Checkout

**Design reference:** Figma "Buyer Approval → Checkout" — email with "Review Estimate" CTA, public review page with QR code, signature modal flow.

### Technical Summary

- Prior spec had plain "Approve Estimate" button — design now uses **signature approval**
- Flow: Email → Review page → "Sign" button → Signature modal (name → cursive → "Continue To Deposit") → Checkout
- Two variants shown: "No Signature" (just review + Sign button) and "Signature Approval" (full modal)
- QR code on review page is new (not in prior spec)
- Conversion still happens at approval (Option A from prior spec), buyer pays the resulting invoice

### Questions

6. **~~Signature required vs optional~~** — ANSWERED: Signature is controlled by the existing "Request Client Signature" toggle on the estimate editor. The "No Signature" and "Signature Approval" buyer flow variants correspond to this toggle being OFF vs ON.

7. **Signature data storage** — "Request Client Signature" already exists today. Where does the signature currently get stored? Is the existing storage sufficient for the deposit flow, or do we need additional fields (e.g., tying signature to approval event, timestamp, IP)?

8. **QR code destination + signature bypass** — The QR code on the review page says "Scan this code to pay online." Three sub-questions:
   - Is this for the PDF/print scenario only (merchant hands printed estimate to client, client scans)?
   - Does scanning the QR skip the signature step and go directly to checkout? If so, signature isn't actually required to pay.
   - If signature IS required, should the QR land on the same review page (with Sign button), making the QR just an entry point equivalent to the email link?

9. **~~"Continue To Deposit" after signing~~** — ANSWERED (Seth, 2026-07-07): Conversion happens AFTER payment, not at signing. If buyer signs but abandons before paying, estimate stays unconverted. No orphaned invoices. Juan confirmed: "buyer pays deposit first, then convert estimate to payable invoice once transaction's complete."

10. **~~No-signature flow CTA~~** — ANSWERED (Seth, 2026-07-07): CTA is "Approve & Pay" — implies buyer is approving the estimate by making the deposit payment. No separate approve step needed. Premium users get Sign → Continue To Deposit. Non-Premium users get "Approve & Pay" directly.

---

## Screen 3: Post-Conversion Estimate (Merchant View)

**Design reference:** Figma "Post-Conversion Estimate" — editor with banner, invoices modal, history tab.

### Technical Summary

- Banner: "This estimate has been converted into an invoice(s)" with chevron → opens modal listing all linked invoices
- Modal shows multiple invoices (INV0022, INV0023, INV####) — implies one estimate can spawn multiple invoices
- History tab shows full timeline: Saved → Delivered → Opened → Approved → Converted events
- Prior spec had `approvedAt` as a single-use guard — design allows multiple conversions

### Questions

11. **Multiple conversions** — The modal shows 3 invoices from one estimate. Is this combining buyer-initiated conversions AND merchant manual "Make Invoice" into one list? Or can the buyer approve + convert the same estimate multiple times (e.g., milestone payments)?

12. **Estimate state after conversion** — Does the estimate stay "Open" for re-use, or move to a "Converted"/"Closed" state? Can merchant still edit and re-send it after a conversion?

13. **History: "Opened" event** — The history shows "Opened - hello@johnsmith.com". Is this a new tracking event (buyer opened the review page), or does this already exist for estimates today?

14. **Banner tap behavior** — If only one invoice was created, does tapping the banner navigate directly to that invoice (skip the modal)? Or always show the modal?

---

## Screen 4: Post-Conversion Invoice (Merchant View)

**Design reference:** Figma "Invoice" — editor with source banner, push notifications, history tab.

### Technical Summary

- Banner: "This invoice was created from estimate EST0001" with chevron → navigates to source estimate
- Push notifications: "Your estimate EST0001 has been approved by {name}" + "New Invoice Created"
- Invoice shows: Paid $500 (deposit), Balance Due $1,500, "Manage Payment Schedule" section
- History shows "Created from estimate EST0001" or "Created from a deleted estimate" (graceful degradation)

### Questions

15. **Deposit carries over** — The invoice shows "Paid (Jan 1, 2026) $500.00". Is this automatically recorded as a payment on the invoice at creation time, or does it only appear after the buyer completes checkout?

16. **"Manage Payment Schedule" on converted invoice** — Is this the existing payment scheduling feature (milestones, future payments), or something new specific to deposit invoices? Can the merchant add additional scheduled payments for the remaining $1,500?

17. **Push notification timing** — Are both notifications sent simultaneously (at conversion), or is "approved" sent at signature and "invoice created" sent when conversion completes?

18. **Deleted estimate edge case** — If the source estimate is deleted, the history shows "Created from a deleted estimate." Should we prevent merchants from deleting estimates that have linked invoices? Or just degrade gracefully?

19. **Banner on invoice when estimate deleted** — Does the "This invoice was created from estimate EST0001" banner still show (with a dead link), hide entirely, or show "created from a deleted estimate"?

---

## Cross-Cutting Questions

20. **Vaulting / "Save Payment Method"** — The prior spec included card vaulting (`setup_future_usage`) for the remaining balance. Is this in scope for v1, or deferred? The checkout designs show a "Save Payment Method" checkbox but no expanded details.

21. **ACH handling** — Epic lists ACH as an outstanding question. If buyer pays deposit via ACH (delayed settlement), does conversion still happen immediately at payment initiation? Or wait for settlement confirmation?

22. **Feature flag strategy** — Should this be behind a flag (gradual rollout), or ship to all merchants at once? Affects whether we need flag checks at each guard point.

23. **Subscription tier gating** — Today, deposits on invoices (mobile + new web) are NOT tier-gated — they only need IS Payments. But "Request Client Signature" is PREMIUM-only. For the new estimate deposit flow:
    - Should "Request Deposit" on estimates follow the same ungated pattern (any merchant with IS Payments)?
    - Or should it require PREMIUM (like the legacy web deposit gate)?
    - Since signature is separately gated, a free merchant could offer deposits without the signature step. Is that intentional or a problem?
