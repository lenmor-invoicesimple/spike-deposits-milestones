# Deposit Optimization v1 — Investigation Summary (Scratch v1)

**Epic:** IS-11250 — Deposit Optimization v1
**Author:** Lenmor Dimanalata
**Date:** 2026-07-14
**Source of truth:** Liz's design-meeting decision ([Slack thread](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869))

> Plain-language summary of the two items I was asked to investigate, plus the caveats worth knowing before we build. Implementation detail lives in the linked ticket docs.
>
> **Updated 2026-07-15** after an adversarial self-review + verifying behavior against the live staging deposit checkout. Key changes from v1: (1) there's a **hard release order** — a convert-first step + a data backfill must ship *before* the payability flag is turned on, or payments break / old estimates re-open; (2) **I2 (email) is NOT fully safe** — the email *subject line* mislabels a payable estimate as an "invoice" and needs a fix; (3) the convert-first work is **our team's**, not an external dependency.

---

## The two things to investigate

Liz flagged two items for me to confirm before we commit to building:

1. **I1 — Can we make an estimate payable using our existing (global) payment settings + a deposit?**
2. **I2 — Once estimates become payable, does the buyer email still render the right thing for an estimate vs. an invoice?**

Both came back **yes, feasible** — but **I2 turned up a real bug** (the email subject line), and I1 turned up a **release-ordering constraint** that's the most important thing in this doc. Details below.

---

## What we landed on: three pieces of work

The investigation produced **three tickets**. Two directly answer Liz's items; the third is the buyer-facing page where payment actually happens (it naturally falls out of making estimates payable).

| # | Ticket | What it covers | Answers | Size |
|---|--------|----------------|---------|------|
| 1 | Enable Online Payments on Estimates (Global) | The "engine" that makes a deposit-estimate payable + locks it once converted + **the convert-first step + the data backfill** | **I1** | Large |
| 2 | Public Review Page: Pay-Deposit for Estimates | The buyer's document page — "Pay Deposit" button, the on-page payment block. **Blocked on #1.** | — | Medium–Large |
| 3 | Estimate vs. Invoice Email Context | Fix the email **subject-line** mislabel + add a deposit-amount line to the estimate email | **I2** | Small |

---

## I1 — Making estimates payable (Ticket #1)

**Finding: feasible, and cleaner than expected.** An estimate can reuse the same account-wide payment and surcharge settings an invoice uses — no per-estimate toggles, no per-estimate surcharge UI. An estimate becomes payable when three things are true:

- The account has online payments enabled (PayPal/Stripe), **and**
- The estimate has a deposit set, **and**
- The estimate hasn't already been converted to an invoice.

**Why it's mostly unlocking, not building:** our payment provider checks already ignore document type — they only care whether the account accepts payments. Today a *single line* rejects every estimate up front. Relaxing that one gate (behind a feature flag) is the core of the work.

**⚠️ The most important caveat — release order is fixed:**

> **We CANNOT just "turn on the flag" the day the code merges.** Two things must be live and done *first*:
> 1. **Convert-first** — the step that turns the estimate into a real invoice at the moment the buyer pays. If the flag is on without this, **every buyer's payment fails** (the payment system still checks "is this an invoice?" at charge time).
> 2. **Data backfill** — mark all *already-converted* historical estimates as converted. We double-checked and the safety net we assumed protects them **does not exist** — without the backfill, old already-paid estimates become payable again on flip-day.
>
> **Order: convert-first + backfill → THEN turn on the flag (gradually).**

**Ownership note:** the convert-first step is **our team's** work (likely the epic owner), not an external/Juan dependency. It's foundational, so it's folded into Ticket #1 as its first work item.

**Other caveats / notes:**
- **A new "conversion lock" is needed.** Once an estimate converts to an invoice, it must stop being payable and stop being convertible again. There's no such lock today, so we're adding one (a new field that records what an estimate converted into). The *authoritative* lock is server-side at checkout; the phone-app copy of it is best-effort (offline devices can't guarantee uniqueness).
- **Surcharge preview accuracy (trace-corrected).** The fee the buyer is *charged* already follows the account setting — the checkout page re-reads `payment.feesType` server-side, so old estimates with a stale frozen fee still charge the current account rate. The only gap is the `/v/` review page's **surcharge disclaimer note**, which reads the estimate's stored fee mode; aligning it means fetching the account setting into the shared invoice-viewer component (used by web + mobile). Small, cross-platform, and **deferrable** — it's a disclaimer-visibility nuance, not a wrong charge.
- **Old estimates have `feesType = null`.** Surcharge never existed on estimates, so old ones store `null` (= "no fee"). Under **global** this is harmless (the account setting is read instead). Under **document-level** it would mean old estimates convert fee-free unless we **backfill** the account value onto them (or add a null→account code fallback — which is just global behavior). Another point for global: no backfill needed.
- **Abandoned-cart side effect.** Estimates would start qualifying for abandoned-cart follow-ups. That's a product decision, not an accident — flagging it.
- **Migration/infra is real work, not a footnote.** The backfill + a new database index are captured as explicit work items (WI-6a/b/c) and gate the flag. May become its own migration ticket, but must not be dropped.

---

## I2 — Email renders correctly for estimate vs. invoice (Ticket #3)

**Finding: the email BODY is safe — but the SUBJECT LINE is not.** "Estimate vs. invoice" refers to the **two document types**. The question: once estimates are payable, does any email accidentally label a payable estimate as an "invoice"?

**Answer: yes — the subject line does (this is a real bug).** The email *body* is fine — it always decides its wording by document type, so a payable estimate body keeps saying "Estimate." **But the email *subject* is chosen by a different rule: whether the document is *payable*.** Today only invoices are payable, so the subject "…sent you an invoice" is always right. **The moment estimates become payable, that subject fires for estimates too** — so the buyer gets "…sent you an **invoice**" over an email that says "Estimate #123" inside. That's exactly the mislabel Liz asked about. **So I2 is not "safe/nil" — it needs a subject-line fix.**

**Second gap:** **no email today shows a deposit dollar amount** — estimate or invoice. The mock wants a "DEPOSIT DUE $500" line, so we add a data field + one conditional line to the estimate email.

**Caveats / notes:**
- **Product decision needed:** what should the subject say for a payable estimate? (e.g. "…sent you an estimate".)
- **Two different email pipelines.** The modern (v2) send path can show the deposit amount easily. The older mobile-originated (v1) path **can't** without extra work — it doesn't load the invoice data. Decision needed: is it OK for mobile-originated sends to omit the deposit amount, or do we do the extra work?
- **The email does NOT get a "Pay" button.** Its call-to-action stays "Review Estimate" → the review page (Ticket #2). Payment happens there.
- **Template edits happen in SendWithUS**, not in our repo — needs SWU access + a staging test-send. There are **three** template records to check, not two.
- Effort is small-ish (subject fix + one template line), *not* trivial as v1 originally suggested.

---

## Ticket #2 — The Pay-Deposit review page (not one of Liz's items, but required)

This is the buyer-facing web page the email links to. It's where the buyer actually pays the deposit, so it falls out of making estimates payable.

**What it adds:**
- A **"Pay Deposit"** button (label changes based on document type).
- The on-page payment block for estimates: the **DEPOSIT DUE** line, the **QR code** to pay online, and the **fee disclaimer** — all invoice-only today.
- **Post-payment cleanup:** once paid/converted, the payment affordances disappear. *Most* of this is automatic (they key off the server "can this be paid?" signal) — but the **DEPOSIT DUE line is the exception** and needs an explicit rule tied to payability (it was previously assumed to be free; it isn't).
- **Seth's new layout pattern:** the action buttons render at both the **top and bottom** of the page. *(Seth's design reviewed clean — no issues.)*

**Caveats / notes:**
- **⚠️ Blocked on Ticket #1.** The server "can this be paid?" check the page needs is *the same code* Ticket #1 relaxes — this page can't produce a working payment link until Ticket #1 lands. Not parallel work.
- **Two systems must agree on "is this payable?"** — the on-page display (client) and the server checkout check use *different* code and must share one rule. If they disagree, you get a QR code with no working payment link (or vice versa).
- **Shared-component blast radius.** The on-page document is a shared package used by **both web and the phone app.** Changing it means a package release + re-testing mobile — a coordination step the original estimate missed. This is why the size moved from Medium to Medium–Large.
- **Good news from staging:** the actual checkout + "payment successful" experience (including the correct "Deposit Paid" / recalculated balance) is **inherited for free** from the existing invoice deposit flow. Only the *pre-payment* review page needs our changes.
- Seth's top+bottom pattern needs a small refactor so the buttons (and the signature pop-up) don't render twice.
- **Figma mock nit:** the "Payment Successful" mock still shows "DEPOSIT DUE" — real behavior already shows **"Deposit Paid."** Mock should be relabeled (no code impact).

---

## Not yet written (rest of the epic)

For completeness — the full epic is bigger than these three. Still to be specced:

- **Buyer "Convert at Pay Now" ticket** — the synchronous estimate→invoice conversion that fires when the buyer clicks Pay. Ticket #1 defines the contract it must satisfy but doesn't build it.
- ~~**Buyer Post-Conversion ticket**~~ — ✅ **now written** ([`ticket-buyer-post-conversion.md`](./ticket-buyer-post-conversion.md)): payment-failure retry, conversion-failure error screen, old-link redirects, success/failed deposit labels (code confirmed correct), Pay-Now loading state.
- **Estimate Deposit CRUD (mobile)** — IS-11406, already handed off to the mobile team (the deposit-setting groundwork these three build on).

---

## Bottom line

- **I1 (payable estimates): feasible, but has a hard release order.** Mostly unlocking existing behavior + a conversion lock — but **do NOT enable the flag until (1) convert-first is deployed and (2) the data backfill is complete.** Skipping either breaks payments for all buyers or re-opens already-paid estimates. Convert-first is our team's foundational work, not an external dependency.
- **I2 (email): NOT fully safe — one real bug.** The email *body* is fine, but the *subject line* will mislabel a payable estimate as an "invoice." That needs a fix (+ a product decision on the wording), plus the deposit-amount line. Small, but not zero.
- **Ticket #2 (review page) is the real build** — and it's **blocked on Ticket #1**. Key watch-items: client/server payability must agree, and it touches a shared component used by the phone app (package release + mobile retest).
- **Encouraging:** the checkout + success experience is inherited for free from the existing invoice deposit flow (verified on staging), so the risk concentrates in the *pre-payment* surfaces, not the payment itself.

### Decisions to bring to Product/Design

**✅ Nothing from Liz is blocking implementation (as of 2026-07-15).** Every item that once needed her has resolved to: confirmed, defaulted-with-prototype-later, or answered by Slack's own written requirements. The only remaining Liz touchpoint is a **prototype walkthrough down the line** — a validation checkpoint, not a decision gate.

**Later prototype walkthrough with Liz (validation, not a blocker):**
Show her the working change and catch any copy/UX nits:
- Payable-estimate **email subject** ("…sent you an estimate" default — confirm wording).
- The `/v/` **converted state** (payment affordances hidden after conversion).
- The **"Pay Deposit"** flow end-to-end.

**Already resolved — no ask needed:**
- ✅ **"Pay Deposit" charges the deposit only** — confirmed; the converted invoice carries the remaining balance.
- ✅ **Post-conversion return flow** — answered by Slack's own requirement: buyer sees checkout-success only right after paying; returning via the old email link lands on the **public estimate `/v/` page** (converted state, affordances hidden), **not** checkout. This is the wrong-customer/second-invoice guard. No need to re-ask Liz.
- ✅ **Estimate email subject copy** — defaulted to "…sent you an estimate" (low-complexity docType branch, code not SWU); ship now, confirm via the prototype above.
- ⏭️ **Figma "Payment Successful" mock relabel** — dropped per epic owner (code already correct; mock fix optional).

**Product decision, but NOT Liz's (needs an owner):**
- Should estimates start qualifying for abandoned-cart follow-ups? (side effect of I1 — growth/marketing, not design.)
- v1 (mobile-send) email: OK to omit the deposit amount, or do the extra work to load it? (eng scope call.)

> Note: **none of these are Seth's.** Seth's only contribution (top+bottom buttons, WI-E) reviewed clean and needs no product decision.

### Dependencies to line up (our team + others)
- **Convert-first cloud function** (our team / epic owner) — the release gate for everything.
- **Data backfill + new DB index** (our team + infra/Atlas) — must finish before flag-on.
- **Shared-package release** (invoice-viewer + payment SDKs) — coordinate web + mobile version bump and mobile QA.
- **SendWithUS template edits** — needs SWU access; three template records to verify.
