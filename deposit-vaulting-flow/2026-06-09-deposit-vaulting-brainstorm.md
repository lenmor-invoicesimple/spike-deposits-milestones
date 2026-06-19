# Deposit Vaulting — Brainstorm & Design Thinking

> Date: 2026-06-09
> Status: Brainstorm — not a spec yet
> Context: Liz & Seth reworking pivot to "Deposit Vaulting"

---

## Why the Original Plan Had Issues

1. **Only improved D&M UX, no direct TPV impact** — milestone sync and APR make the experience better but don't put new payments on rails
2. **Deposits were ignored** — treated as "already working," missed the vaulting opportunity
3. **Milestone work was expensive** — complex data structures in Parse, mobile offline sync, not our domain
4. **No TPV lens in planning** — optimized for feature completeness, not revenue impact

### Why we didn't think of this earlier

- **Bottom-up framing**: Started from "D&M exists, how to improve?" instead of "shortest path to TPV?"
- **Deposits felt done**: They work today, no obvious gap — missed that the card is charged and forgotten
- **Milestone complexity consumed bandwidth**: Edge cases took all planning energy
- **Conflated "improving D&M" with "improving milestones"**: Deposits are part of D&M too (higher adoption, simpler)

---

## TPV — Why It Matters

**TPV = Total Payment Volume** — total dollars processed through our payment rails.

- It's the direct revenue metric (we take a cut)
- It compounds: one vaulted card can auto-pay every future milestone/invoice
- RP proved this model: stored payment method → auto-charge → TPV growth
- Milestone sync and APR don't directly drive TPV — they improve experience but still require client to manually return and pay
- Deposit Vaulting = RP for project-based merchants

---

## Core Insight

The deposit payment is the **one moment where the client is already entering their card.** Every other touchpoint (milestone due, reminder email, manual follow-up) requires the client to come back. That return trip is where TPV leaks.

**Question: How do we turn the deposit moment into a vaulting moment with near-zero additional friction?**

---

## Learning from Uber/Amazon/Netflix

These services vault cards with near-zero perceived friction, but their model is different from ours:

| Model | What actually happens | Why you don't notice |
|-------|----------------------|---------------------|
| **Uber** | Card required to use service at all | Vaulting = onboarding. No service without it. |
| **Amazon** | Card saved during first buy, default on | Framed as "convenience" not "permission" |
| **Netflix** | Card required for subscription | You expect recurring charges — that's the product |

**Key principle they share:** Saving the card is inseparable from the primary action the user already wants to do.

### Why we can't do exactly this

- Client is paying a one-time deposit on a contractor's invoice
- They have **no expectation** of future auto-charges
- They don't have an "account" with us — they're a guest payer
- The relationship is with the *merchant*, not with Invoice Simple
- Silently vaulting would rightfully surprise them

### What we CAN borrow

**Make the vaulting feel like part of the thing they're already doing, not a separate decision.**

Don't ask "can we save your card?" → Say "set up auto-pay for this project."

The deposit isn't the vaulting moment — it's the **trust-building moment.** Client just paid, it worked, they trust the flow. Present auto-pay as the natural way to handle what's next.

---

## Reframed Design Direction

Instead of "consent to card storage" → **"how do you want to handle the remaining payments?"**

```
Client opens invoice link → sees:

  Project: Kitchen Renovation
  ─────────────────────────
  Deposit (due now):     $1,000
  Milestone 2 (Jul 15):  $1,000  
  Milestone 3 (Aug 15):  $1,000
  ─────────────────────────
  Total:                 $3,000

  [ Pay deposit — $1,000 ]

  ○ Pay each milestone manually (we'll email you)
  ● Auto-pay milestones with same card (we'll notify you 3 days before)
```

### Why this works

- **No scary "save card" language** — it's about auto-pay, not storage
- **Client sees the full picture** — amounts, dates, total. No surprises
- **It's a choice between two valid options** — not a permission request
- **The default (pre-selected auto-pay) feels helpful**, not predatory
- **Deposit payment and auto-pay decision happen on same screen** — one flow, not two
- Vaulting is invisible — it's an implementation detail of choosing auto-pay

---

## TPV Amplification (Once Card is Vaulted)

Vaulting alone isn't TPV. The charge is. What triggers auto-charges?

**A. Auto-charge remaining milestones on due date**
Client vaults at deposit → milestones 2, 3, 4 auto-charge when due. No client action needed. Direct TPV.

**B. Auto-charge the full invoice if no milestones**
For simple invoices (no D&M, just a total) — vault → charge balance on due date. Extends vaulting beyond D&M to all invoices.

**C. "Card on file" for repeat clients**
Once vaulted for one invoice, offer merchant: "This client has a card on file. Enable auto-pay for future invoices?" One vault → TPV across multiple invoices over time. This is where it compounds.

**D. Reduce payment failure rate**
Vaulted cards can retry on soft decline. Today if manual payment fails, client has to return. With vault, retry automatically → recovered TPV.

---

## Client Psychology — Why They'd Want to Vault

- **"I don't want to think about this again"** — convenience
- **"I trust this contractor, just charge me"** — established relationship
- **"I get a discount / avoid late fees"** — incentive (merchant-configured)
- **"The amount and timing are clear, no surprises"** — transparency

**Reducing friction ≠ fewer clicks. It's reducing anxiety.**

Client needs to feel:
1. They know exactly what they'll be charged and when
2. They can cancel anytime
3. They'll get notified before each charge

Showing a clear schedule ("$500 on Jul 15, $500 on Aug 15, email 3 days before, cancel anytime") isn't friction — it's **confidence**. Makes them more likely to vault.

---

## What the Merchant Needs to Do (Minimal)

**Ideally: nothing extra.** If D&M is set up (or even just a due date) and client pays with card, vaulting should be offered automatically.

Merchant might want (later):
- See which clients have vaulted cards
- Manually trigger a charge (scope changes)
- Disable auto-charge for specific clients
- Dashboard of upcoming auto-charges

**Minimal v1:** Merchant sees a badge/indicator on invoices where client has vaulted. That's it.

---

## Open Questions to Pressure-Test

- **Pre-checked vs unchecked "auto-pay" option?** Pre-checked = higher vault rate, could feel aggressive. A/B testable.
- **PayPal support?** PayPal billing agreements differ from Stripe card vaulting. Both day 1 or Stripe-only first?
- **Amount changes after vaulting?** Client agreed to $1,000 but merchant changed to $1,200. Need consent flow.
- **Chargebacks?** Client forgets, disputes. Mitigation: pre-charge email + clear consent record.
- **Bank transfer clients?** Probably not v1 — focus on card + PayPal.
- **No milestones case?** Just invoice with due date — does auto-pay still make sense?
- **Repeat client compound effect** — how does vault persist across invoices?

---

## Principles for the New Plan (Avoiding Old Mistakes)

1. **Start from TPV path, work backwards** — "shortest path to card on rails" not "what D&M features to build"
2. **Scope to what we own** — is-payments, Stripe/PayPal APIs, consent UX. Less Parse/mobile offline dependency.
3. **Define "done" as TPV number** — not feature list. X% vault rate, Y auto-charges fired.
4. **Validate hardest assumptions first** — consent UX, Stripe `setup_future_usage` feasibility, PayPal billing agreements.
5. **Keep debounced pipeline as backend** — same SQS architecture, simpler payload for deposits than milestones.

---

## Relationship to Other Docs

- [Debounced SQS Pipeline](./2026-06-09-debounced-sqs-pipeline-proposal.md) — backend architecture that supports this
- [APR implementation](./automatic-payment-requests/) — may be replaced or simplified by auto-pay
- [Milestone sync](./invoice-duedate-milestone-sync/) — less urgent if deposits drive TPV directly
