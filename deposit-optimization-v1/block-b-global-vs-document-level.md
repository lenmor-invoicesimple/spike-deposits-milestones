# Block B — Payment Settings & Surcharge: Global vs Document-Level (DECISION RECORD)

**Status:** ~~✅ RESOLVED — GLOBAL~~ → ⚠️ **REVERSED — DOCUMENT-LEVEL** (Liz Scott agreed, 2026-07-15)
**Owner:** Lenmor Dimanalata · **Deciders:** Liz Scott (product/design), Juan Angel (eng)
**Parent doc:** [`enable-payments-on-estimates.md`](./enable-payments-on-estimates.md) — see "Intended Design → block B". This file is the detailed record for that one fork.

> ⚠️ **REVERSAL NOTE (2026-07-15):** The decision flipped to **document-level**. Liz agreed to per-estimate payment + surcharge settings after all. The rest of this file records the global-path analysis (still useful for rationale), but the **chosen path is now document-level**. Key implications of the flip:
> - **Surcharge UI is now IN scope** — per-estimate surcharge slider on both mobile (uncommitted patch at `invoice-payments-passing-fees-section.tsx:88`) and web (mount gates `settings-sidebar-migrated.tsx:249` / `settings-sidebar.tsx:168`).
> - **Transform must PRESERVE `feesType`** — remove the account-default override at nextjs `document-transformation-utils.ts:106` and mobile `use-cases.ts:55`; copy verbatim instead.
> - **`feesType` backfill/fallback is now required** — old estimates have `feesType = null`; they'd convert fee-free without it. Either backfill the account value or add a null→account code fallback.
> - **`/v/` disclaimer plumbing is no longer needed** — the disclaimer already reads the stored per-doc value; that's now the correct behavior (was a cost under global, is a win under document-level).
> - **New open question:** the charge-time gate ANDs per-doc `feesType` with the account `feesType` (`stripe-fees.ts:38`+`:45`). Under document-level, if a merchant surcharges the estimate but the account is off, gate 2 vetoes the charge while the disclaimer showed a fee → mismatch. Decide: relax gate 2 to defer to per-doc, or keep the AND. Tracked as open item #1 in [`confluence-plan-2-proposed-edits.md`](./confluence-plan-2-proposed-edits.md).

> ~~**One-line decision (SUPERSEDED):** Estimates become payable from **global/account-level** payments + surcharge settings. **No per-estimate payment toggle, no per-estimate surcharge UI.** Merchants cannot tune the deposit's online-fee pass-through per estimate — the deposit always passes 100% of the country fee (the remaining balance is still tunable on the converted invoice). Liz explicitly accepted this trade-off.~~

---

## TL;DR — why global won

Global wins on ~7 dimensions (less UI work ×2, no conversion-transform work, cleaner merchant mental model, uniform retroactive surcharge, and the per-estimate toggle is nearly useless under the 1:1 lock). Document-level wins on exactly **1**: merchants could tune the deposit fee pass-through per estimate. Liz signed off on giving that up, so the whole decision reduced to that single question and it's now closed.

**Note we're not choosing global because document-level is too hard** — document-level is mostly already wired (mobile especially). It's a genuine product call: a single account-wide payment/surcharge policy is simpler for merchants to reason about than per-estimate knobs that only live for the short window between send and conversion.

---

## The decision, reduced to one question

Every other dimension was either a tie or a global win. The entire choice came down to:

> **Is a deposit online-fee locked at 100% (merchant can't discount it per-estimate) acceptable?**
> - **Yes → Global.** Less work *and* cleaner model.
> - **No → Document-level**, and we take on the surcharge UI (web) + a verbatim-copy conversion transform.

**Liz's answer (Slack, 2026-07-14):** *"That's a good point. I'm comfortable with that trade off."* → **Global.**

---

## Full comparison scorecard

Grouped so the deciding rows stand out from the wash rows.

### Rows that differentiate the choice

| Dimension | Document-level | Global | Winner |
|---|---|---|---|
| **Estimate surcharge UI — mobile** | Un-gate (~2 lines; already patched uncommitted — note PAYC-315 deliberately removed it before) | No UI to add | **Global** |
| **Estimate surcharge UI — web** | Medium: open 2 mount gates + 2 leaf gates + decide how much of `WidgetPayments` to expose | No UI to add | **Global** |
| **Conversion transform** | Just needs `feesType` copied through — a **one-liner each** (remove the override / omit). Transform is NOT the cost — the surcharge UI rows above are. | **Zero change** — both active clients already re-supply `feesType` from the account default | **≈ wash** (transform is trivial either way; Global marginally simpler as it's already the behavior) |
| **Preview ↔ charge consistency** | Achieved by copying fields verbatim (transform work) | Charged amount **already global** (checkout re-reads account `payment.feesType`). Only the `/v/` **surcharge disclaimer** reads stored `feesType` — fixing it = fetch account setting + plumb through the **shared invoice-viewer** (web+mobile). Small-to-medium, not 1-line — see §2 | **Global** (still smaller than doc-level's UI+backfill; and money is already correct) |
| **Two-toggle weirdness** | Estimate toggle + invoice toggle for one conceptual policy; estimate toggle only lives send→convert and can't affect the lock | One place (account settings) | **Global** (cleaner) |
| **Per-estimate off-switch value** | Thin — only useful send→convert, barred from controlling the post-lock state | None | **Global** (doc-level "benefit" barely applies under 1:1 lock) |
| **Retroactive surcharge on old estimates** | Old estimates have **`feesType = null`** (surcharge never existed on estimates). Under doc-level, `null` means "no fee" → a surcharging merchant's old estimate converts **fee-free** (wrong). Fix = **backfill the account value onto every old estimate**, OR a code fallback (`null` → account) — but that fallback *is* global behavior. | Uniform — all follow current account setting; old estimates' `null` is **never read**, so **no backfill and no fallback needed** | **Global** (doc-level carries a real backfill-or-fallback cost here) |
| **Deposit fee tunability** | Merchant *can* set <100% pass-through per estimate | **Deposit locked at 100%** (remaining balance still tunable on the invoice) | **Document-level** (the ONE thing global gives up — Liz accepted) |

### Rows that are the same either way (NOT decision factors)

| Dimension | Note |
|---|---|
| **docType gate** (`invoice-payments-status.ts:58`) | Relax one line — required either way |
| **Payability of existing estimates** | All carry `paymentSuppressed = null` → payable on ship, **identically** in both models |
| **Payment fails (post-conversion)** | Retry on invoice checkout in-session. Doc-level only "nicer" because it carried settings — moot under global since they already match |
| **Conversion fails (pre-payment)** | Pure atomicity requirement (block A): mint-invoice + set-`convertedTo` must commit together. Fork-independent |
| **Lock once converted** | Key off immutable `convertedTo`/`approvedAt` in `findNotPayableReason`, NOT `paymentSuppressed`. Same mechanism; doc-level only *reinforces* why it can't key off the toggle |
| **`convertedTo` is a new field** | Add to Realm + Parse validation + `CheckoutData`. Same work either way |
| **Old converted estimates stay locked (migration)** | `convertedTo` doesn't exist yet; old converts protected today only by `fullyPaid`→`balanceDueZero`. Confirm/backfill. Fork-independent |
| **Email estimate-vs-invoice rendering** | Estimates now payable → email flow re-checks. Fork-independent spike |

---

## Technical findings that drove the decision

### How payability resolves today (2 AND-ed gates + a separate surcharge gate)

1. **General/document-shape gate** — `findNotPayableReason` (`is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts:42-86`). Choke point: `documentType !== 0` (L58-60) rejects **every** estimate today, before any account/document setting is read. Also returns `balanceDueZero` when balance ≤ 0 (L70). **This one line must relax for estimates — regardless of the global-vs-doc choice.**
2. **Per-provider gate** — each SDK's `findNotPayableReason(accountStatus, documentStatus, paymentSuppressed)` (`.../utils/SDKs/sdks.ts`, Stripe 90-115 / PayPal 37-55). AND-ed with layer 1; either layer can independently veto. Per-doc flags fed via `providerPaymentSuppressedAdapter` (`user-payments-status.ts:75-83`).
3. **Surcharge DOUBLE-GATE** — surcharge applies only if **BOTH** `invoice.setting.feesType === SURCHARGE` **AND** account `payment.feesType === SURCHARGE` (`is-stripe/.../passing-fees/stripe-fees.ts:38+45`, `is-paypal/.../passing-fees/paypal-fees.ts:37+43`).

### The surcharge "amount" is not account-configurable — key finding

There is **no account-level surcharge rate/amount setting anywhere** (searched: `payment.feeRate`, `payment.surcharge`, etc. — none; the lone `SettingKeys.FeeRate` constant is dead code, zero usages). The fee amount is two pieces, neither editable at account level:

- **Base rate = hardcoded per-country platform constant**, keyed to the connected Stripe/PayPal account's country: US **3.49% + $0.49**, CA **2.9% + $0.30** (`stripe-fees.ts:6`, `paypal-fees-config.ts:8`). Unknown country → `null` → surcharge `0`.
- **`feeRate` (1–100) = per-document pass-through dial** — "what fraction of that fee to pass to the buyer." Read **only** at `getInvoiceSurcharge.ts:46`: `const feeRate = invoice.setting?.feeRate ?? 100`. **Undefined ⇒ 100 ⇒ full fee passed.** Defaults undefined; set only by the per-document surcharge slider (web `surcharge-fees.tsx`, mobile `configure-surcharge.tsx`). It does NOT decide *whether* surcharge applies (that's `feesType`).

### The two account-level toggles on mobile (from the Settings screenshot)

Neither is an amount:

| Toggle | Setting | Effect |
|---|---|---|
| **"Always add Online Payment Fee"** ("Applies to future invoices only") | `payment.alwaysAddSurcharge` (bool) — `payments-option-section.tsx:141-146` | Auto-apply fee to future docs |
| **"Add Fees" mode** (separate row) | `payment.feesType` (SURCHARGE / MARKUP) | Which fee mode |

Together they feed `getDefaultPassingFeesType()` (`payment-fees/utils.ts:6-24`), which seeds each new document's `feesType`. So "where the merchant configures the amount" = **they don't** — the amount is (country constant) × (feeRate, default 100%). This is *why* not showing surcharge UI on estimates costs almost nothing.

### Per-document fields already persist for estimates (no data-model gap)

- Realm (mobile) `invoice-setting/schema.ts` uses a **single unified** `RInvoiceSetting` for both docTypes: `paymentSuppressed` (39/78), `stripePaymentSuppressed` (40/79), `feesType` (45/84), `feeRate` (46/85).
- Parse validation `invoiceValidation.ts` `settingSchema(isEstimate)` validates all four for **both** types; the `isEstimate` branch only strips `termsDay`.
- `defaultInvoiceSetting` (`repository.ts:27-114`) writes for estimates too: `paymentSuppressed = null`/`stripePaymentSuppressed = null` (91-92), `feesType = getDefaultPassingFeesType()` unconditionally (97). `feeRate` left undefined.

### Existing-estimate edge cases (the crux)

- `paymentSuppressed`/`stripePaymentSuppressed` are **`null` on all pre-feature estimates** — no estimate UI ever wrote them. So **both models default existing estimates to payable identically**; the "silently honor a stale suppression flag" risk is near-zero.
- `feesType` **is populated** and **non-uniform** on old estimates (SURCHARGE/MARKUP/undefined per `deprecate_markup` flag + account state at creation). This is the only field that would behave inconsistently across the retroactive back catalog under document-level.

### Effort to expose per-document surcharge UI on estimates (if we had gone document-level)

- **Mobile = trivial / mostly done.** Shared editor (`invoice.screen.tsx`) already mounts the section; the only docType guard (`invoice-payments-passing-fees-section.tsx:88` + `use-track-surcharge-awareness.ts:23`) is **already patched in the working tree (uncommitted)** to allow estimates. Leaf components (`configure-surcharge.tsx`, `add-surcharge-base.tsx`) are docType-agnostic. ⚠️ Baseline commit `a7a0de18e` "PAYC-315 | Remove surcharge and markup from Estimates" deliberately removed this once — worth knowing why before re-adding.
- **Web = medium.** The whole `WidgetPayments` is gated off the estimate editor at the mount site (`settings-sidebar-migrated.tsx:249`, `settings-sidebar.tsx:168`), not just the leaf `surcharge-fees.tsx:205` / `payment-markup-fees.tsx:37`. Opening it also exposes the full pay-online widget, so you'd render only `<SurchargeFees>` for estimates — a small design decision, not a rebuild. Shared sidebar means no new scaffolding.

---

## Deep-dive: the three concepts that decided it

### 1. Conversion transform
The code turning an estimate into a new invoice. ✅ **Trace-verified 2026-07-15 (active paths) — both clients already behave the same, and it's the behavior GLOBAL wants:**

| Field | Mobile `makeInvoiceFromEstimate` (`use-cases.ts:45-62`) | Web — active **nextjs** path `transformDocumentSettings` (`document-transformation-utils.ts:64-114`, convert branch `:101-111`) |
|---|---|---|
| `feesType` | **dropped → re-supplied from account default** (`use-cases.ts:55` omit; default `repository.ts:97`) | **overridden with account default** (`utils.ts:106` `feesType: options.defaultFeesType`, fed by `convert-document.action.tsx:96` from `settings['payment.feesType']`) |
| `feeRate` | copied verbatim | copied verbatim (survives the `...transformedSettings` spread) |
| `paymentSuppressed` | copied verbatim | copied verbatim (only nulled on `duplicate`, not `convert`) |

> ⚠️ **Correction (2026-07-15):** an earlier revision of this doc claimed "web copies `feesType` verbatim (`InvoiceModel.ts:1305`), needs a fix." **That was the DEAD `client/` SPA** (`client/src/models/InvoiceModel.ts`), which is no longer used. The **active** web conversion is a **nextjs server action** (`convert-document.action.tsx:43` → `transformDocumentSettings` `utils.ts:64`) that **already overrides `feesType` with the account default** — same as mobile. No web transform fix is needed for global. `client/InvoiceModel` is not in the conversion chain (zero imports).

- **Document-level:** to preserve a per-estimate `feesType`, you'd **remove** the account-default override — one line each (nextjs `utils.ts:106`; mobile `use-cases.ts:55`). `feeRate`/`paymentSuppressed` already copy through. So doc-level's transform change is trivial, **not** a canonical-transform rewrite. Its real cost is the **surcharge UI** on the estimate editor (the rows above), not the transform.
- **Global:** the intended "invoice follows the account setting" behavior is **already what both active clients do** → **zero transform change**. (Global's small non-zero cost is elsewhere: the live-account **preview** read — item 2 below.)
- **Third writer, not yet built:** the buyer-path convert-first **server** cloud function (WI-0) doesn't exist yet. It must be built to re-supply `feesType` from the account (matching the two active clients) — trivial, and it's net-new work regardless of the global-vs-doc decision.

### 2. Preview ↔ charge consistency
✅ **Trace-verified 2026-07-15** (agent trace of the buyer-facing surfaces). There are **two distinct buyer surfaces**, and they behave differently — this is narrower and lower-stakes than earlier revisions implied:

| Surface | Reads `feesType` from | Under global today |
|---|---|---|
| **Actual checkout page** (is-unifiedxp, the `payUrl` target — where the buyer is **charged**) | **account** `payment.feesType`, re-read server-side (`is-stripe/.../passing-fees/stripe-fees.ts:38-45` `getSurchargeCents`) | ✅ **Already global** — the charged amount already follows the account. **No work.** |
| **`/v/` public document preview** (invoice-viewer, what the buyer sees **before** Pay Now) | **stored** `invoice.setting.feesType` (`invoice-viewer/.../PaymentInstructions.tsx:71`, `hideSurcharge` gate) | ❌ reads the estimate's frozen per-doc value |

**Key nuance:** the `/v/` preview does **not** compute a forward-looking fee *amount* — it only shows/hides a **surcharge disclaimer note** (one gate, `PaymentInstructions.tsx:71`). The dollar amount only appears on the checkout page, which is already global. So the "mismatch" is a **disclaimer-visibility** nuance, not a wrong charge.

- **Breaks for retroactively-payable old estimates** (Liz's requirement): the `/v/` disclaimer keys off the estimate's frozen `feesType`, which may differ from the current account setting → the *note* may show/hide inconsistently with what the checkout actually charges. The **money is still correct** (checkout re-reads the account).
- **Global fix — NOT a one-line swap (trace-corrected).** The account `payment.feesType` is a `Setting` row **not currently loaded** on the `/v/` page (`public-invoice.ts` fetches the `Account` object but not that Setting), and the gate lives in the **shared invoice-viewer package used by BOTH web and mobile**. The fix requires: (1) fetch the account Setting in `handlePublicInvoice`, (2) plumb it through `PublicInvoiceState` → `<Invoice>` → a new context/prop in invoice-viewer, (3) update `hideSurcharge()` to consult it, (4) ensure the **mobile** consumer supplies it (or a safe default). Mechanical, but **shared-package + cross-platform blast radius** — small-to-medium, not trivial. Arguably **deferrable**, since the actual charge is already correct.

### 3. Per-estimate off-switch value
The `paymentSuppressed` toggle document-level would expose on an estimate. Discounted because:
1. **1:1 lock kills most of its life** — only meaningful send→convert; post-conversion the estimate is locked via `convertedTo` regardless.
2. **It's barred from controlling the state that matters** — the lock must key off immutable `convertedTo`, not the toggle, or a merchant could flip payment back on after conversion.
3. **"Two toggles, one policy"** confusion (estimate toggle vs invoice toggle).

### 4. Old estimates have `feesType = null` — the backfill-vs-fallback point
Surcharge never existed on estimates, so **every old estimate has `feesType = null`** (or absent). `null` is not a neutral placeholder — in the surcharge gate it means **"no fee."** What that costs depends on the model:

- **Document-level:** the converted invoice inherits the estimate's stored `feesType`. `null` → the invoice charges **no surcharge** — even for a merchant who surcharges every invoice. So to make retroactively-payable old estimates behave correctly you must either:
  - **backfill** the account's `feesType` onto every old estimate (a real data migration), or
  - add a **code fallback**: at conversion/preview, if `feesType` is null, read the account setting.
  - ⚠️ **The fallback IS global behavior.** "When null, use the account" is exactly what global does — so choosing the fallback collapses document-level into global for these documents. This is another point *for* global, not a rescue of doc-level.
- **Global:** the estimate's `null` is **never read** — the invoice (and the fix in §2) take the account setting. So **no backfill and no fallback** — `null` is genuinely harmless.

**Note there is nothing to *write* to make old estimates `null`** — they already are. The cost (doc-level only) is writing the *account's real value* over that null, or the fallback code that stands in for it.

> Product caveat: doc-level's backfill is a real cost **only if** the rule is "old payable estimates should charge the merchant's usual fee" (almost certainly yes — a surcharging merchant won't want silent fee-free invoices). If product were fine with old estimates converting fee-free, doc-level's `null` would be acceptable and the backfill vanishes. It's a product call, but treat the backfill as real.

**How the three-plus concepts connect:** document-level's *cost* was the transform (trivial) **plus the surcharge UI (real) plus a `feesType` backfill/fallback on old estimates (real)**; its *benefit* was the off-switch (thin). Both models had to solve preview↔charge, but the charge is **already global**, so global's only remaining cost is the small/deferrable `/v/` disclaimer read. Net: global gives up the thin off-switch and pays one small deferrable disclaimer cost; document-level pays the surcharge UI + the backfill.

---

## Remaining work on the GLOBAL path

1. **Relax the docType gate** — `findNotPayableReason` (`invoice-payments-status.ts:58`) to allow docType=1 for estimate checkout-page eligibility (scope confirmed by "Convert-first gate scope" spike — payment-time gates see the invoice and pass).
2. **Live-account surcharge disclaimer on `/v/`** ⚠️ (trace-corrected) — the **charged amount is already global** (checkout re-reads the account). The remaining gap is the `/v/` preview **surcharge disclaimer** (`invoice-viewer/PaymentInstructions.tsx:71`), which reads the stored `feesType`. Fixing it = fetch account `payment.feesType` in `public-invoice.ts` + plumb through the **shared invoice-viewer** package (web+mobile) + update `hideSurcharge()`. Small-to-medium, cross-platform. **Deferrable** — the money is already correct; this only aligns the disclaimer note. See §2.
3. **`convertedTo` field + lock** — new indexed field (Realm + Parse validation + `CheckoutData`); non-payability check in `findNotPayableReason` keyed off it. Backfill/confirm old converted estimates stay locked (currently only `fullyPaid`→`balanceDueZero`).

> Note: the estimate→invoice **transform** needs **no change** for global — both active clients (mobile `use-cases.ts:55`, nextjs `document-transformation-utils.ts:106`) already re-supply `feesType` from the account default. The net-new convert-first **server** function (WI-0) must be built to do the same.
4. **Do NOT expose** per-estimate surcharge UI (web mount gates stay; revert/park the uncommitted mobile patch for estimates) or payment toggle.
5. **Email rendering** — confirm estimate-vs-invoice conditional render once estimates qualify as payable (separate assigned spike).

## Explicitly NOT doing (because global)

- No per-estimate surcharge slider / SURCHARGE-MARKUP toggle on estimates.
- No per-estimate payment on/off toggle.
- No verbatim field-copy conversion transform / §8 three-client reconciliation for surcharge fields (current re-supply-from-account behavior is correct).

---

## Chronological back-and-forth (how we got here)

1. **Spike run** comparing global vs document-level (agent, 2026-07-14). Initial recommendation leaned **document-level** — the per-doc fields already work end-to-end and both models default old estimates to payable identically, so the only divergence looked like surcharge + an off-switch.
2. **"Is it only surcharge that differs?"** — confirmed: yes, surcharge is the only *code* divergence; the per-estimate off-switch is a *capability* divergence; everything else is a wash.
3. **"If we don't add surcharge for estimates, does that help?"** — realized convert-first already routes the charge through the invoice, so surcharge is effectively invoice-governed already; the only requirement left is a read-only fee **preview** on the estimate.
4. **"How does the merchant configure the amount without estimate UI?"** — investigation showed there **is no account-level amount**; rate = country constant, `feeRate` = per-doc pass-through defaulting to 100. Merchant configures mode via the account toggle; amount is platform-fixed. So no estimate UI is needed to "configure" anything.
5. **Mobile Settings screenshot** ("Always add Online Payment Fee" toggle, no amount) — confirmed the two account toggles are mode-only (`payment.alwaysAddSurcharge`, `payment.feesType`); no amount field exists. Corrected the earlier assumption that `feesType` was the sole account knob.
6. **"How hard is document-level surcharge (the 1–100% slider)?"** — mobile trivial (already patched uncommitted; PAYC-315 removed it before), web medium (mount-gate, not just leaf-gate). Not hard, but non-zero, and the "why was it removed?" question is a flag.
7. **"How does document-level impact convert / fail paths / lock?"** — mapped it: cost concentrates in the conversion transform (verbatim copy + preview↔charge invariant); fail paths and lock are orthogonal; lock's correctness even *depends* on keying off `convertedTo` not the toggle.
8. **User began leaning global** — reasons: (a) we lock the tile after conversion anyway under 1:1, so the toggle "almost doesn't make sense"; (b) copying more fields + an estimate/invoice toggle disconnect is weird; (c) confirmed the transform/preview costs largely **don't apply** to global.
9. **Retroactive-payability check** — doesn't differentiate for payability (identical), slightly favors global for surcharge uniformity, and raises a shared migration concern (old converted estimates staying locked). Added one work item to global: the live-account preview read.
10. **Liz confirmation** — asked if deposit-locked-at-100% is acceptable; Liz: *"I'm comfortable with that trade off."* Decision finalized: **GLOBAL.**

### Key Slack references
- Liz's design-meeting summary + open items (thread parent): [p1784052946973869](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869)
- Lenmor's global-100%-deposit-fee question + Liz's sign-off (reply 4): same thread, ts 1784059173.884469

### Message drafted to Liz (for the thread)
> At this point I'm leaning into **global** payments/surcharge too (even though the document-level features are mostly already wired into Estimates). With global:
> - **No payment toggle on the Estimate** → simpler, and avoids a confusing case where the Estimate's toggle differs from the converted Invoice's. We still lock the Estimate after conversion so it can't be paid twice — there's just no per-Estimate toggle to manage.
> - **No surcharge settings on the Estimate** → nothing to copy/transform into the Invoice at conversion. One thing to handle either way: since we convert first, the fee shown on the Estimate checkout needs to read the current global setting so it matches what the converted Invoice actually charges.
> And thanks for confirming the 100% deposit-fee trade-off — that was the deciding factor.
