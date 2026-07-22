# Ticket: Enable Deposit Fields for Estimates on Web

**Epic:** IS-11250 — Deposit Optimization v1
**Type:** Story
**Priority:** Low — deferred until mobile is complete
**Status:** Not started — gathering requirements

---

## Summary

Enable deposit support for estimates on the web app. This requires changes across three codebases:
1. **is-packages** — type definitions and Parse↔Domain mapping
2. **is-web-app** — UI guards and REST document transformation
3. **is-services** — backend payment status guard (only if online payments needed)

Mobile does NOT need these changes — it uses an inclusive `...setting` spread that bypasses is-packages entirely (see comments in `is-mobile/src/services/parse/models/invoice.ts`).

---

## Why Web Needs Changes (But Mobile Doesn't)

**Web architecture:** selective field extraction
```
Parse → getEstimateSettings() picks NAMED fields → EstimateSettings type → UI
```
If a field isn't explicitly extracted, it's invisible to the web app.

**Mobile architecture:** inclusive spread
```
Parse → ...setting spread captures ALL fields → Realm schema → UI
```
Any field on Parse flows through automatically.

This means web requires explicit additions to types, mapping functions, and UI guards.

---

## Changes Required

### Layer 1: is-packages (Types + Parse Mapping)

| # | File | Change | Details |
|---|------|--------|---------|
| 1 | `packages/domain-invoicing/src/document/estimate.ts` | Add deposit fields to `EstimateSettings` type | Add `depositType: DepositType \| undefined`, `depositRate: DepositRate \| undefined`, `depositAmount: DepositAmount \| undefined` (same as `InvoiceSettings` lines 38-44) |
| 2 | `packages/domain-invoicing/src/document/createDefaultDocument.ts` | Add deposit defaults to `createDefaultEstimateSettings()` | Add `depositType: undefined`, `depositRate: undefined`, `depositAmount: undefined` (same as `createDefaultInvoiceSettings()`) |
| 3 | `packages/parse-domain/src/mapping-functions/parseInvoiceToDocument.ts` | Add deposit fields to `getEstimateSettings()` (line 171-191) | Add `depositType`, `depositRate`, `depositAmount` to destructuring and return object. Compare with `getInvoiceSettings()` (line 135-169) which already extracts them. |

**Write path is fine:** `documentToParseJson.ts` uses `{ ...setting, comment: notes }` spread for estimates (line 46-48) — once the type includes deposit fields, they'll flow through automatically.

### Layer 2: is-web-app (UI + REST Transformation)

| # | File | Change | Details |
|---|------|--------|---------|
| 4 | `nextjs/app/services/documents/transform-rest-document.ts` | Add deposit fields to `getEstimateSettings()` (line 165-183) | Same pattern as is-packages — this is the NextJS app's own mapping. Compare with `getInvoiceSettings()` at line 132-163. |
| 5 | `client/src/payments/components/PaymentDeposit/PaymentDeposit.tsx` (line 96) | Relax docType guard | Currently: `if (docType !== DocTypes.DOCTYPE_INVOICE \|\| paymentsCount > 0) return null;` — needs to allow `DOCTYPE_ESTIMATE` |
| 6 | `nextjs/components/invoice-editor/invoice-actions/invoice-actions-migrated.tsx` (line 268-277) | Allow `isDepositEligible` for estimates | Currently: `const hasDeposit = isInvoice(document) ? isDepositEligible({...}) : false;` — unconditionally returns false for estimates |
| 7 | `nextjs/components/invoice-editor/invoice-payment/hooks/use-record-payment.ts` (line 15) | Allow estimates in payment scheduling | Currently: `const shouldShowRecordPayment = isInvoice(document) && !recurringInvoiceSeriesId;` — gates the entire deposit/payment scheduling form to invoices |
| 8 | `nextjs/app/(authenticated)/(core)/(documents)/components/email-document/modal-email-document-form.tsx` (line 115) | Handle estimate deposit check | Currently casts to `Invoice` type: `const hasDeposit = isDepositEligible(props.document as Invoice);` — needs estimate support |

### Layer 3: is-services (Backend — only if online payments needed)

| # | File | Change | Details |
|---|------|--------|---------|
| 9 | `packages/payments/payments-status/src/payments-status/invoice-payments-status.ts` (line 58) | Relax documentType guard | Currently: `if (documentType !== 0) return GeneralNotPayableReason.docTypeNotInvoice;` — blocks all non-invoice docs from being payable online. Only needed if estimates with deposits should be payable via Stripe/PayPal. |

### Layer 4: is-packages/calculator (Deposit Eligibility)

| # | File | Change | Details |
|---|------|--------|---------|
| 10 | `packages/calculator/src/invoice/getDeposit.ts` (line 65-77) | Verify `isDepositEligible` works for estimates | Currently typed against `UniversalInvoice` which has `payments` array. Estimates may not have this — either add `payments: []` when calling, or create parallel `isEstimateDepositEligible`. |

---

## Open Questions

### Product / Design (needs design input)

1. **How will Request Deposit coexist with the Payment Scheduling page on Invoice?**
   - Invoice currently has a full Payment Scheduling flow: Editor → PaymentSchedulingScreen → ManagePaymentScreen (deposit + milestones + payment mode)
   - If we add an inline "Request Deposit" toggle (like Estimate's mobile UI), do we:
     - **Replace** the deposit portion of Payment Scheduling with the inline toggle? (milestones stay on the separate screen)
     - **Duplicate** — keep both, inline toggle is a shortcut and Payment Scheduling is the advanced view?
     - **Remove Payment Scheduling entirely** and move everything inline?
   - This affects both mobile and web — mobile also has this question for the future Invoice deposit ticket (see `/docs/superpowers/specs/invoice-deposit-crud/planning.md`)

2. **Should Estimate and Invoice deposit UX converge?**
   - Estimate (mobile, done): inline toggle + action sheet type selector + amount input
   - Invoice (mobile, existing): navigate to ManagePaymentScreen → form fields
   - Ideally both should feel the same — but we don't have designs for the Invoice refactor yet
   - Risk: if we build web Estimate deposit now with one pattern, then Invoice gets redesigned differently, we refactor twice
   - **Recommendation:** wait for Invoice deposit redesign designs before starting web, so both can align

3. **Same UI as Invoice, or new?** Invoice web uses `PaymentDeposit` component (in `client/src/payments/components/PaymentDeposit/`). Should estimates reuse that component, or get a different inline UI (like mobile's `RequestDepositSection`)?

### Technical

4. **Do estimates need online payment for deposits?** If yes, the `is-services` guard (#9) must change. If deposits on estimates are just for record-keeping / invoicing after acceptance, it may not be needed yet.

5. **Estimate → Invoice conversion on web:** When an accepted estimate becomes an invoice, should deposit config carry over? The Parse data is on the same document, but the web domain model separates them.

6. **Calculator `isDepositEligible`:** Does it work if you pass an estimate with `payments: []`? Or does the function assume invoice-specific shape?

7. **Feature flag:** Should web deposit be gated behind the same Flagsmith flag as mobile (`estimate_deposit_enabled`)?

---

## Dependency Chain

```
#1-3 (is-packages types + mapping) ─→ #4-8 (is-web-app UI)
                                    ─→ #9 (is-services, only if online payments)
                                    ─→ #10 (calculator, verify)
```

is-packages changes are the prerequisite for everything else on web.

---

## Reference: Comparison with Invoice Deposit (existing, working)

### getInvoiceSettings() — what we're replicating for estimates

```typescript
// is-packages/packages/parse-domain/src/mapping-functions/parseInvoiceToDocument.ts
function getInvoiceSettings(parseInvoice: ParseInvoice): InvoiceSettings {
  const sharedSettings = getBaseSettings(parseInvoice);
  const parseSetting = parseObject.get('setting');
  const {
    depositType,      // ← these three are what's missing from getEstimateSettings()
    depositRate,
    depositAmount,
    // ... other invoice-specific fields
  } = parseSetting;
  return { ...sharedSettings, depositType, depositRate, depositAmount, ... };
}
```

### getEstimateSettings() — what needs the deposit fields added

```typescript
function getEstimateSettings(parseInvoice: ParseInvoice): EstimateSettings {
  const sharedSettings = getBaseSettings(parseInvoice);
  const parseSetting = parseObject.get('setting');
  const {
    estimateSignatureRequired,
    estimateSignedText,
    estimateSignedAt,
    acornFinancing,
    comment,
    feeRate,
    // MISSING: depositType, depositRate, depositAmount
  } = parseSetting;
  return { ...sharedSettings, estimateSignatureRequired, ... };
}
```

---

## Notes

- Mobile work is complete and ships independently — web is a follow-up
- Parse Server already accepts deposit fields for both doc types (`settingSchema(isEstimate)` in `invoiceValidation.ts`)
- Parse Server `handleDepositChange` hook creates Payment records regardless of docType
- No database migration needed — fields already exist on the shared `Invoice` Parse class
