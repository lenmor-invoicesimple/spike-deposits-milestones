> ✅ **CORRECTIONS APPLIED (2026-07-15)** — adversarial-review defects fixed in this revision. See [`self-review-findings.md`](./self-review-findings.md) for the full findings. Fixed here: **I2 is NOT closed** — the A/B subject override at `email-experiment-service.ts:79` hardcodes "…sent you an invoice" and fires on *payability*, not docType, so a payable estimate gets an "invoice" subject (H1 — new work item added); v1 (mobile-send) path can't call `getDeposit()` as written — no invoice object/`currency_code` loaded (M5); third SWU template ID `swuPDFEstimateTemplateV2` added (LOW). Body labeling verdict stands (docType-driven, safe); the **subject line** is the uncovered gap.

# Spike / Ticket — Estimate vs. Invoice Email Context (payable estimates)

**Epic:** IS-11250 — Deposit Optimization v1
**Date:** 2026-07-14
**Owner:** Lenmor Dimanalata
**Status:** Draft — implementation-ready spec
**Source of truth:** Liz's design-meeting decision ([Slack p1784052946973869](https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1784052946973869)); item **P5 / I2** ("Confirm the email correctly renders estimate vs. invoice context once estimates become payable").
**Sibling ticket:** [`ticket-public-review-page-estimate-payable.md`](./ticket-public-review-page-estimate-payable.md) (the Pay-Deposit review page). This doc is the **email** surface only.

---

## 1. What the item actually means (de-risked)

**"Estimate vs. invoice" = the two document-type entities, not a choice between contexts.** The item is: *once estimates become payable, confirm the email renders correctly on **both** doc types.* Restated with the eligibility definition:

> Confirm the buyer email renders correctly for both an **invoice** and a newly-payable **estimate**, where a payable estimate = payments globally enabled + deposit set + not yet converted.

- **Invoice doc type** — unchanged; already the payable path. This half is a **no-regression confirmation**, not new work.
- **Estimate doc type (newly payable)** — where the check + the one real gap live (§2).

The underlying worry is a **coupling that is about to break**: historically `payable ≡ invoice`, so code has used "has payment stuff" as a proxy for "is an invoice." The concern: when estimates become payable, does any email branch mislabel a payable **estimate** as an **invoice** ("Invoice #", "Amount Due")?

**Code verdict: the email BODY is safe; the SUBJECT LINE is NOT (H1).**

The email **body** keys its wording on `docType` **explicitly**, not on payability — this half is safe:
- `api/src/controllers/invoice-sending.ts:838` — `strDocType = docType == DOCTYPE_ESTIMATE ? 'estimate' : 'invoice'` → passed to the template as `doc` (`:946`).
- Template chosen by docType: estimate → `swuEstimateTemplate` (`:869`); v2/PDF path `email.ts:654-657`.
- Post-payment **receipt** emails (`is-services/is-stripe/.../render-email-template.ts`) are already invoice-centric (`invoice_no`, `invoice_url`) and, since convert-first mints a real invoice, render correct invoice context automatically — no change.

> 🔴 **H1 — the subject line breaks I2, and this is the real answer to Liz's concern.** The A/B **subject override** at `api/src/util/email-experiment-service.ts:79` hardcodes `"Action Needed: {name} sent you an invoice"` and fires on **`isEligible` (payability), NOT docType.** On the v2 path (`email.ts:287-291`) it activates **exactly when an estimate becomes payable.** Result: a buyer receives a subject saying *"…sent you an **invoice**"* over a body that says *"Estimate #123"* — the precise mislabeling I2 was asking about. **I2 cannot be closed as "risk nil."** The body is safe; the subject is an uncovered gap that becomes live the moment estimates are payable.

So the item is **two** things, not zero: (1) fix the payability-triggered subject override so payable estimates don't get an "invoice" subject (§2a, new), and (2) the additive deposit-amount gap (§2b, below).

---

## 2a. Gap #1 (H1) — payability-triggered subject override mislabels payable estimates

**The defect.** `api/src/util/email-experiment-service.ts:79` sets the subject to a hardcoded `"Action Needed: {name} sent you an invoice"`. It is applied based on **`isEligible`** (the payability experiment), invoked on the v2 send path at `email.ts:287-291` (and mirrored on the v1 mobile path at `invoice-sending.ts:903-912`). Today this only ever fires for invoices (only invoices are payable), so the "invoice" wording is always correct. **The moment estimates become payable, it fires for payable estimates too** — and the subject says "invoice" while the body says "Estimate."

**Mechanic (verified 2026-07-15) — the fix MUST be in code, not the SWU template.** The SWU send payload has **no top-level subject field** (`is-messages/.../sendwithus/client.ts` `ISwuEmailData` — subject is absent). The SWU template's own subject line always renders, and it *is* variable-capable (Handlebars; `SWURenderedTemplate.subject`). Code passes `subject` as **just a merge variable** inside `template_data`; the template resolves roughly `{{#if subject}}{{subject}}{{else}}<own default subject>{{/if}}`. So when the `ir_subject_email` flag is on, `getSubject()`'s hardcoded string is injected as `{{subject}}` and the template renders *that*, bypassing its own default. **Consequence: editing the SWU template subject alone does NOT fix the bug** — the injected code string wins as long as the flag is on. The hardcoded `getSubject()` string is the sole cause.

**Change.** Make `getSubject()` **docType-aware**: for a payable **estimate**, inject estimate-appropriate copy (or inject a `doc_type` variable and let the template subject compose the noun). Do not let the payability branch emit "invoice" for a docType=1 document. Apply on both send paths (v2 `email.ts:287-291`, v1 `invoice-sending.ts:903-912`). Confirm no other path sets the `subject` merge variable.

> **Provisional copy (implement this now; confirm with Liz via prototype later):**
> - Estimate (docType=1): `"Action Needed: {name} sent you an estimate"`
> - Invoice (docType=0): `"Action Needed: {name} sent you an invoice"` (unchanged)
>
> This is a low-complexity swap (branch the string on docType). Ship the default; we'll validate final wording against a prototype showing the change rather than block on sign-off.

**Acceptance criteria**
- A payable estimate's email subject does NOT contain "invoice"; it uses the approved estimate wording.
- A payable invoice's subject is unchanged.
- Verified on the v2 path (`email.ts:287-291`) where the override activates.

---

## 2b. Gap #2 — deposit-due amount is not in the estimate email

The mock ([estimate email](#)) shows a **"DEPOSIT DUE  USD $500.00"** line. **No email carries a deposit dollar amount today** — estimate *or* invoice:

- `invoice-sending.ts` (v1 send): grep "deposit" = **0 hits**. Render-data literal (`:932-963`) has `balance_due` but no deposit field.
- `email.ts` `getEmailData` (v2/PDF/payments path): the only deposit field is **`is_deposit_eligible`** (`:761`) — a **boolean, not an amount** — and it is **invoice-only**: `isEligibleForDeposit` is set exclusively inside the `else` (invoice) branch (`:659-687`); the estimate branch (`:656-657`) only picks the template.
- The helper that returns the deposit **amount** — `getDeposit().amount` in `@invoice-simple/calculator` (`is-packages/.../getDeposit.ts:163-209`) — is **never called** by either email builder. It is docType-agnostic, so it works for an estimate document.

### Change
1. **v2 path (`email.ts` `getEmailData`):** compute the deposit-due amount via `getDeposit()` (already docType-agnostic) and add a new render-data field — e.g. `deposit_due` (formatted string, following the `balance_due` pattern at `email.ts:758`). Populate it on the estimate branch. Add the field to the `SwuEmailTemplateData` type (`email.ts:138`). **This path has the invoice object + currency loaded, so `getDeposit()` is callable here.**
2. **⚠️ v1 path (`invoice-sending.ts`, primary mobile-send route) — NOT a one-line add (M5).** This builder assembles render-data from **multipart form fields only** — it never loads an invoice object, deposit settings/items, or `currency_code`. `getDeposit()` is **uncallable** without fetching the invoice, and the `{% if currency_code is defined %}` snippet renders no currency prefix on v1. Two options, **product-scoped** (see §4):
   - **(a) Scope the deposit line to v2 only** — accept that mobile-originated sends don't show the deposit amount (simplest; confirm acceptable).
   - **(b) Add invoice-fetch to v1** — load the invoice + currency in `invoice-sending.ts` so `getDeposit()` works. Real added work; not XS–S.
3. **SendWithUS template:** add a conditional DEPOSIT DUE line to the estimate template. **SWU-hosted — edited in SendWithUS, not the repo.**

### Confirmed against the actual template (both `SWU_ESTIMATE_TEMPLATE` and `_V2` are byte-identical)
- **Syntax = Jinja/Django** (`{% if %}`, `{% trans %}`, `|default()`), so the conditional is `{% if deposit_due %}...{% endif %}`.
- **Insertion point:** the amount block currently renders a single big number — `balance_due` if defined, else `total` (the `<span style="font-size: 40px...">` block). Add the DEPOSIT DUE line directly below it:
  ```jinja
  {% if deposit_due %}
    <p style="margin:0;font-size:14px;font-weight:bold;color:#0B7A3B;text-align:center;text-transform:uppercase">
      {% trans %}Deposit Due{% endtrans %} {% if currency_code is defined %}{{ currency_code }} {% endif %}{{ deposit_due }}
    </p>
  {% endif %}
  ```
- **No `pay_url` change:** the template CTA already handles `pay_url` (`{% if pay_url or invoice_url %}`, `href="{% if pay_url %}{{pay_url}}{% else %}{{invoice_url}}{% endif %}"`), but `api` never passes `pay_url` for estimates (`getPayUrlForEmailInvoiceV1` returns early for non-invoice docType) → it always falls to `invoice_url` → the review page. **Leave this alone.** The CTA label is `{% if is_pending_approval %}REVIEW AND SIGN{% else %}VIEW ESTIMATE{% endif %}` — the Pay button lives on the review page (sibling ticket), not the email.
- **Body labeling safety confirmed (BODY ONLY):** the type noun is `{% trans %}Estimate{% endtrans %} {{ invoice_no }}` (template lines 166, 199) — a localized string keyed on nothing payability-related. The email **body** cannot render as an invoice. ⚠️ This does **not** cover the **subject line**, which IS payability-driven and mislabels payable estimates — see §2a (H1). The complete answer to Liz's I2: body safe, subject broken.

### Explicitly NOT changing
- No `pay_url` / Pay button in the email (the email links to the review page; payment happens there).
- No template-type flip, no wording change — labeling stays docType-driven.
- Post-payment receipts — already correct.

---

## 3. Acceptance criteria

- **(H1) A payable estimate's email SUBJECT does not say "invoice"** — the payability override (`email-experiment-service.ts:79`) emits estimate-appropriate copy for docType=1. Invoice subject unchanged.
- A deposit-estimate email renders "DEPOSIT DUE {amount}" matching `getDeposit().amount` (v2 path; v1 per the §4 scope decision).
- A non-deposit estimate email renders no deposit line (field absent → `{% if %}` false) — identical to today.
- The estimate email CTA remains "Review Estimate" → `/v/{id}` public review page; no pay button appears in the email.
- Estimate email **body** continues to render "Estimate #" / estimate wording after the payable change (docType-driven).
- Reminders (`is-messages/.../email-reminder`) reuse the snapshotted estimate render-data and render consistently (see reminder staleness note in §4).

## 4. Notes / open

- **🎯 Product/design decisions needed:**
  1. **Subject copy for payable estimates (H1) — DEFAULTED, confirm later via prototype.** Implementing `"Action Needed: {name} sent you an estimate"` now (low-complexity docType branch). Final wording to be validated with Liz against a prototype showing the change — not a blocker.
  2. **v1 deposit-amount scope (M5)** — is it acceptable that mobile-originated (v1) sends omit the deposit-due amount, or must we add invoice-fetch to v1? This is the difference between XS and a real chunk of work.
- **SWU access:** the template edit requires SendWithUS access + a staging test send. Flag to whoever owns SWU templates.
- **⚠️ Three SWU template IDs, not two (LOW).** `SWU_ESTIMATE_TEMPLATE`, `SWU_ESTIMATE_TEMPLATE_V2`, **and `swuPDFEstimateTemplateV2`** (send-to-self / PDF, `email.ts:654`). Local byte-identical comparison of the first two does **not** prove the SWU-hosted records are in sync. Apply the DEPOSIT DUE line to **whichever the estimate path actually sends** (v2 via `email.ts:657`; v1 via `invoice-sending.ts:869`; send-to-self via `:654`) and verify each hosted template individually in SWU.
- **`{% trans %}` for the label:** "Deposit Due" must go through `{% trans %}` like every other string in the template so SWU localizes it. It does not conflict with the invoice-viewer `RenameEstimateTitle` (that governs the document title, a different surface).
- **Reminder `deposit_due` staleness:** reminders reuse snapshotted render-data, so a `deposit_due` field is frozen at send time and re-shown even if the estimate changes — same class as `balance_due` today. AC should read "consistent with the snapshot," not "always current."
- **Effort revised:** **S for the subject fix + v2 deposit line.** If v1 must carry the deposit amount (§4 decision b), add invoice-fetch work — no longer XS. The subject-override fix (H1) is net-new and was not in the original scope. The heavy lifting is still the sibling review-page ticket.

### Template treatment — restyle vs new template (⏳ PENDING TEAM INPUT, 2026-07-15)

Compared the **current live estimate email** against the **new design mock** (both screenshots reviewed). Verdict: the mock is a **full visual restyle of the SAME structure**, not a new layout.

- **Structurally identical:** header (business / doc-no.), logo, business name, a content card, greeting + body, a single amount block, one CTA, "Powered by Invoice Simple", footer address. Same components in the same order.
- **The one real content delta:** the amount block. Current = single big number (`total`/`balance_due`). Mock = **Date + Total rows + a `DEPOSIT DUE $X` line**. The deposit line is exactly §2b (compute + pass `deposit_due`); Date/Total are existing fields rearranged.
- **Everything else that "looks different"** is CSS/copy: green vs blue button, bordered white card, tighter type, personalized greeting ("Hey John! You have an estimate to review"), footer address. Pure SWU-side HTML/CSS.

**Decision in flight (user taking to team):** since this would be **gated behind a flag**, two options —
1. **Edit the existing template** with internal `{% if new_layout %}…{% else %}…{% endif %}` — one record, but both full designs tangled together (messy for a near-total restyle).
2. **New SWU template selected by the flag in code** — cleaner for a full restyle, easy to A/B and delete the loser post-rollout. **Leaning this way.**

**Open questions for the team:**
- Does the new look apply to **all 3 estimate template records** (`SWU_ESTIMATE_TEMPLATE` v1-send, `SWU_ESTIMATE_TEMPLATE_V2` v2-send, `swuPDFEstimateTemplateV2` send-to-self/PDF), or just the modern v2 path? (Partial rebuild = mixed look across send paths.)
- Confirm the deposit-amount line is wanted (drives the `deposit_due` field work regardless of template choice).

**Key point — the template choice does NOT change the code scope.** Edit-vs-new only affects the SWU/HTML side + (if new) a small flag-based template-selection branch in the send code (`email.ts` / `invoice-sending.ts` where `swuEstimateTemplate` is chosen). The `deposit_due` field work (§2b) is required either way; the restyle itself is entirely SWU-authored.
