# Deposit Optimization v1 — Making Deposit-Estimates Payable Online

> **How to use this page:** the top section is the 1-minute skim. Detail lives in the collapsed sections below (click to expand). **Highlight any sentence to leave an inline comment** — that's how we want technical feedback.
>
> **Meeting goal:** close the open items below, align on the two risk items, and assign owners to the open questions.

---

## 🗣️ Open items to discuss (meeting agenda)

**The things we actually need the room for. Everything else is captured for reference.**

1. **Is the "global settings" decision final?** *(needs Seth + Liz)*
   The whole engine is built on using **global** payment + surcharge settings, not per-estimate ones. In the design thread Seth pushed back — he'd keep **document-level** settings on the estimate, since estimate changes don't propagate to the converted invoice. Liz asked him to restate and **the thread trailed off unresolved.** We need to close this explicitly, because it's the foundation everything else sits on.

2. **The conversion lock adds friction for "estimate as template" merchants — is the mitigation right?** *(Seth's concern, Liz's proposal)*
   Once an estimate is converted, it's locked (can't re-convert, can't re-pay). Seth flagged this hurts merchants who reuse estimates as templates. Liz's answer: keep the lock (it's a hard technical constraint — checkout can't know which invoice to point to if one estimate spawned many), but soften it with a **single "duplicate & convert" action** instead of making the merchant duplicate, then convert separately. **Confirm this combined action is in v1 scope** or explicitly deferred.

3. **"Retroactive" = old estimates + a deposit added *now* — confirm the reading, and that the lock covers it.** *(confirm intent)*
   Deposits on estimates are **net-new** — no historical estimate has ever had one. So "existing estimates with deposits become payable retroactively" means: take an old estimate, **add a deposit now**, it becomes payable. **The catch:** a merchant could add a deposit to an **already-converted** estimate → it would read as payable → a buyer on the old link could pay a deposit on a doc already converted & collected. Our conversion lock (keyed off *was-converted*) must block this. Confirm the reading and that the forward case is in scope. *(Upside: no historical data backfill needed — the old-deposit-estimate population is empty; we'll confirm with an audit query.)*

4. **Abandoned-cart follow-ups on estimates — intended?** *(needs an owner — growth/marketing)*
   Payable estimates would start qualifying for abandoned-cart emails. Side effect of making them payable. Someone needs to own the yes/no.

5. **v1 (mobile-originated) email: show the deposit amount or not?** *(eng scope call)*
   The modern send path can show "DEPOSIT DUE $X" easily; the older mobile-send path can't without extra invoice-fetch work. OK to omit it on mobile-originated sends, or do the work?

6. **Email subject copy sign-off** *(low stakes — defaulted, validate later)*
   We're shipping "…sent you an estimate" as the default. Confirm wording via prototype later; not a blocker.

---

## What & why

We're making **deposit-estimates payable online** (epic IS-11250). A buyer receives an estimate that has a deposit set, and can pay that deposit directly — today only invoices are payable.

Two things were investigated before committing:

- **I1 — Can an estimate be payable using our existing global payment settings + a deposit?** → ✅ **Yes**, mostly unlocking existing behavior.
- **I2 — Does the buyer email still render "estimate" vs "invoice" correctly once estimates are payable?** → ⚠️ **Body is safe; the subject line is a bug** (see risk #2).

---

## The work, in four pieces

| Piece | One-line scope |
|---|---|
| **1. Merchant — Pre-Conversion** | Merchant turns on payability, sets a deposit, sends the estimate email. *(This is the foundation — the others are inert without it.)* |
| **2. Buyer — Pre-Conversion** | Buyer opens the email → public review page → sees a **"Pay Deposit"** button. Payment hasn't happened yet. |
| **3. Buyer — Post-Conversion** | Buyer pays → estimate converts to a real invoice → checkout success / failed / error pages, and old links behave sensibly. |
| **4. Merchant — Post-Conversion** | After conversion, the merchant's estimate is locked: **Convert-to-Invoice hidden**, a "duplicate & convert" prompt if they try again. |

The hinge between the Pre- and Post- pieces is **convert-first**: the estimate becomes a real invoice at the moment the buyer clicks Pay (or when the merchant converts manually).

---

## ⚠️ Two things to align on (the reason for this meeting)

**1. Release order is fixed — we cannot just "flip the flag."**
Two things must be live **first**, or we break buyers / re-open old estimates:
- **Convert-first** must be deployed — otherwise every buyer's payment fails at charge time.
- A **data backfill** must mark already-converted historical estimates — otherwise old, already-paid estimates become payable again on flip-day.

  **Order: convert-first + backfill → then enable the flag (gradually).**

**2. The email subject line mislabels a payable estimate as an "invoice" (real bug).**
The email *body* is safe (it's chosen by document type). But the *subject* is chosen by whether the doc is *payable* — so a payable estimate would get "…sent you an **invoice**" over a body that says "Estimate #123." We have a low-complexity fix and a default wording; we want a copy sign-off later via prototype (not a blocker).

---

<!-- ==================================================================== -->
<!-- ACCORDIONS BELOW — in Confluence, wrap each section in an `expand` macro. -->
<!-- Type /expand, set the title to the ## heading, paste the bullets inside.  -->
<!-- Ticket links to be added once tickets are filed.                          -->
<!-- ==================================================================== -->

## ▸ Piece 1 — Merchant, Pre-Conversion  *(expand)*

**The payability engine (I1):** an estimate becomes payable when three things are true — the account has online payments enabled, the estimate has a deposit set, and it hasn't already been converted. Our payment checks already ignore document type; today a single line rejects every estimate up front. Relaxing that one gate (behind a flag) is the core of the work.

- A new **conversion lock** is added — once an estimate converts, it stops being payable and can't convert again (a new field records what it converted into). Authoritative server-side; the phone-app copy is best-effort.
- **Surcharge disclaimer on the review page** — the amount the buyer is *charged* already follows the account setting (checkout re-reads it), so the money is correct. The one gap is the `/v/` review page's **surcharge disclaimer note**, which reads the estimate's stored fee mode; aligning it means plumbing the account setting into the shared invoice-viewer (web+mobile). Small, cross-platform, and **deferrable** since the charge is already right.
- **Migration is real work:** the backfill + a new DB index are explicit work items and gate the flag.

**The email send (I2):** subject-line fix (above) + a new "DEPOSIT DUE $X" line on the estimate email. Template edits happen in SendWithUS, not our repo. The email keeps its "Review Estimate" CTA — no Pay button in the email.

*Mobile deposit CRUD (setting the deposit) is IS-11406, already handed to the mobile team.*

---

## ▸ Piece 2 — Buyer, Pre-Conversion (review page)  *(expand)*

The buyer-facing web page the email links to — where the deposit actually gets paid.

- Adds a **"Pay Deposit"** button + the on-page payment block for estimates: the **DEPOSIT DUE** line, the **QR code**, and the **fee disclaimer** (all invoice-only today).
- **Blocked on Piece 1** — the "can this be paid?" server check is the same code Piece 1 relaxes. Not parallel work.
- The on-page document is a **shared component used by both web and the phone app** — changing it means a package release + mobile retest.
- **Good news:** the checkout + "payment successful" experience is inherited for free from the existing invoice deposit flow. Only the *pre-payment* review page needs changes.

---

## ▸ Piece 3 — Buyer, Post-Conversion (checkout & after)  *(expand)*

Everything the buyer sees at and after Pay Now. Because convert-first mints a real invoice first, most of this is **inherited from the invoice deposit flow** — this piece is largely verification + a few gaps.

- **Success / failed labels** — "DEPOSIT PAID" on success, "DEPOSIT DUE" on failed. Confirmed correct in code; verification only.
- **Payment-failure retry** — verify a retry re-pays the *same* invoice and never creates a second invoice / re-converts.
- **Conversion-failure error screen** — if convert-first fails *before* the charge, show a recoverable error, not a "payment failed" screen (buyer wasn't charged).
- **Old-link redirect** — a stale post-conversion checkout link redirects to the public estimate page (which now hides all payment affordances), never re-entering checkout. This is the guard against a second invoice from the same estimate.
- **Pay-Now loading state** — convert-first adds a server round-trip; show a loading state so the button doesn't look frozen.

---

## ▸ Piece 4 — Merchant, Post-Conversion (the lock)  *(expand)*

Once an estimate is converted (by the merchant manually, or automatically when the buyer pays), it's **locked** — no re-conversion, no re-payment.

**Why the lock exists:** if one estimate could spawn many invoices, checkout wouldn't know which invoice to point a returning buyer at (the "wrong-customer-gets-wrong-invoice" risk). So conversion is one-way.

**How the lock is actually enforced — three layers, not one page-load check:**

| Layer | Fires | Role |
|---|---|---|
| **Server payment gate** | at **Pay time**, server-side | **The real lock.** Rejects payment if the estimate is already converted — independent of any page load, keyed off the conversion field (not a toggle a merchant could flip back). |
| **Atomic write-guard** | at the **moment of conversion** | Ensures a double-click / retry produces exactly **one** invoice. |
| **UI hiding** (merchant device + buyer review page) | on **page load** | **Cosmetic only** — hides the Convert button / Pay CTA. Makes the UI correct; does *not* enforce the lock. |

**What the merchant sees:**
- The **"Convert to Invoice" button is hidden** after the first conversion (all convert entry points).
- If they try to reuse the estimate, a prompt offers to **duplicate & convert in one action** *(per Liz — softens the friction for merchants who use estimates as templates; ⚠️ confirm this combined action is in v1 — see open item #2)*.
- Duplicating yields a fresh, unlocked estimate.

**Caveat — sync lag:** when the *buyer* triggers conversion by paying, the merchant's button-hiding depends on Parse/Realm **sync** reaching their device. Until it syncs, the merchant's UI is briefly stale — harmless, because a re-convert attempt still hits the server guard.

---

## ▸ Still to be specced  *(expand)*

- **Convert-at-Pay-Now build** — the synchronous estimate→invoice conversion itself. Its contract is defined in Piece 1, but the build isn't written yet. This is the hinge between Piece 2 and Piece 3.
