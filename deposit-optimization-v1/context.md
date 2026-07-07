# Deposit Optimization v1: Estimate → Payable Invoice Buyer Conversion

**Epic:** [IS-11250](https://everpro-tech.atlassian.net/browse/IS-11250)
**Cloned from:** [IS-10436](https://everpro-tech.atlassian.net/browse/IS-10436) (Deposits & Milestones Automatic Payments — solutioning)
**Status:** Parking Lot
**Quarter:** Q3 2026
**Assignee:** Liz Scott
**Board:** Invoice Simple - Payments Growth
**Label:** payments
**PRD:** https://everpro-tech.atlassian.net/wiki/x/JoBRS

---

## Problem Statement

Customers want to collect deposits upfront on Estimates. Juan Maldonado (auto repair shop owner) requested partial payment tracking that deducted deposits from the total and then converted it to an invoice.

## Strategic Rationale

- We lose TPV because the current flow requires an invoice for a buyer to pay online
- Invoices often arrive after work has begun — they function as receipts, not payment requests
- The estimate is the missing high-intent payment moment (buyer approves with deposit)
- Contractors collect deposits offline (cash, check, manual transfer) or skip entirely
- **Joist validation:** 14% of their TPV originates from Estimate → Payable Invoice conversion

**Success metric:** 10% of TPV originates from Estimate with Deposit → Payable Invoice Buyer Conversion

---

## Scope (from Epic)

### MERCHANT Side

**Deposit Creation**
- Simpler Deposit CRUD on Invoice and Estimate
- Add Payments Tile/Toggle to Estimate

**Payable Deposit Estimate — Pre-Invoice Conversion**
- Add QR Code to Estimate
- Merchant Email Send Button, Buyer Email CTA, SMS Estimate Link, and Get Link → resolves to Estimate URL with Checkout

**Payable Deposit Estimate — Post-Invoice Conversion**
- Banner that links Estimate to Invoice, and Invoice to Estimate

### BUYER Side

**Pre-Invoice Conversion**
- Estimate URL with Checkout
- Payment Initiated/Success converts Estimate to Invoice
- Payment Failure Handling

**Post-Invoice Conversion**
- Estimate converts to Invoice
- Payment confirmation page (with invoice?)
- Invoice emailed with paid amount

---

## Outstanding Questions (from Epic)

1. Can a Deposit amount be entered in-line on the document, with the additional option to set other parameters?
2. How will ACH be handled since it's a delayed payment method?
3. Once the Estimate is converted to an Invoice:
   - Does the Payment Confirmation show the Invoice?
   - Do Buyer Email CTA, SMS Estimate Link, and Get Link resolve to regular Estimate URL (without the checkout)?
4. **Sync/double-click:** The approve action calls a cloud function that creates the invoice server-side, and the buyer's client won't see confirmation instantly. The atomic `approvedAt` guard (see `deposit-vaulting-flow/estimate-auto-accept-link-spec.md`) already prevents a duplicate *conversion*, but does the UI need its own loading/disabled state on the button so a buyer isn't confused by an unresponsive tap or a jarring "already approved" error on a second click?
5. **Offline:** The approve link is a public buyer-facing page, not the offline-first mobile app. Should "Approve" even render without connectivity — should we block/hide it and show a "check your connection" message, similar to how mobile blocks vault edits offline?

---

## Prior Work (from IS-10436 solutioning)

Our tech spec in `specs/deposit-vaulting-flow/` covers the **buyer approval + vaulting plumbing** — one slice of this full epic. Key artifacts:

- `estimate-auto-accept-link-spec.md` — public approval URL, guards table, token analysis
- `feedback-tracker.md` — design decisions log (token vs no-token, doc limit bypass)
- `session-recap-2026-06-18.md` — Dip/Jackson/Sonya feedback, 8 Confluence inline comments

### Decisions already made (carry forward):
- **No token for approval URL** — defense-in-depth doesn't solve identity; checkout is the security boundary (pending Legal confirmation)
- **Doc limit guard** — cloud function MUST check merchant's subscription doc limit before creating invoice
- **`setting.fullyPaid: true`** — marks estimate as closed/converted
- **`setting.approvedAt`** — single-use guard for approve flow

### Open items from prior spec (still pending):
- [ ] 8 Confluence inline comments not yet responded to (Dip x3, Sonya x5)
- [ ] Legal response from Alex Higgins (was expected week of June 23 — overdue)
- [ ] Loop in Jackson's team on estimate→invoice conversion design

---

## Liz's Slack Input (June 23)

In #core-team, Liz tagged Seth:
> "the report merchant flow pattern could work for the approval estimate"

Attached screenshot: "Buyer email w report invoice..png" — showing the existing buyer-email UI for "report" flow on invoices. She's suggesting reusing this pattern for estimate approval UX.

No replies yet. **Action:** Investigate the "report merchant" flow pattern and assess fit.

---

## What's New vs Our Prior Spec

| Item | Status |
|------|--------|
| Simpler Deposit CRUD (merchant) | New — not in our spec |
| Payments Tile/Toggle on Estimate | New |
| QR Code delivery channel | New |
| SMS delivery channel | New |
| Banner Estimate↔Invoice post-conversion | New |
| Payment Failure Handling (buyer) | New — our spec assumed happy path |
| ACH / delayed payment methods | New — we only considered card/Stripe |
| Post-conversion link behavior | New open question |
| "Report merchant" flow as approval UX | New suggestion from Liz |
| 10% TPV target | New explicit metric |
| Public approval URL + guards | Already spec'd |
| Token vs no-token | Already decided |
| Doc limit bypass guard | Already added to spec |
| `fullyPaid` / `approvedAt` mechanics | Already spec'd |

---

## Next Steps

- [ ] Check if child tickets exist under IS-11250
- [ ] Review the "report merchant" buyer-email pattern Liz referenced
- [ ] Respond to 8 Confluence inline comments (carried from IS-10436)
- [ ] Follow up on Legal response (overdue)
- [ ] Spec out new items (deposit CRUD, QR, SMS, failure handling, ACH, post-conversion UX)
