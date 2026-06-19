# IS-10984 — Enable Name/Label field on Deposits, Milestones, and Record Payment

## Summary

The "Name" field on deposit, milestone, and record payment forms is currently visible but locked — always auto-assigned as "Deposit" or "Payment". This ticket enables users to type a custom label (e.g. "Foundation Work", "Final Delivery") when creating or editing a payment.

The `label` field already exists on Payment documents in MongoDB and is already rendered across all surfaces (public invoice, PDF, mobile, web). The change is enabling the input and ensuring custom values render properly.

---

## Feature Flag

This feature must be gated behind a feature flag (client-side, Optimizely). The FF controls whether the label input is editable or remains locked.

- **FF creation tracked in a separate ticket**
- Web + mobile wrap the enabled state with FF check
- **Do NOT add any feature flag on is-parse-server** — the backend accepts custom labels unconditionally. The guards and validation work regardless of FF state. Old clients that don't send label won't break, and if we need to disable the feature, we just turn off the client-side FF (clients go back to sending the default).

---

## Changes Required

### is-parse-server (backend)

1. **Fix [updatePayment.ts](https://github.com/invoice-simple/is-parse-server/blob/master/cloud/collections/payment/updatePayment.ts#L129)** — guard against overwriting label with `undefined` when clients omit it:

```typescript
// Current — wipes label if client doesn't send it
paymentObj.set('label', payment?.label);

// Fix — only update if explicitly provided
if (payment?.label !== undefined) {
  paymentObj.set('label', payment.label);
}
```

2. **Fix [invoiceHooks.ts](https://github.com/invoice-simple/is-parse-server/blob/master/cloud/collections/invoice/invoiceHooks.ts#L319)** (lines 319, 354, 406) — add fallback so legacy sync path doesn't clobber custom labels:

```typescript
// Current
label: PaymentLabel.PAYMENT,

// Fix
label: payment.label || PaymentLabel.PAYMENT,
```

3. **Add validation in [invoiceValidation.ts](https://github.com/invoice-simple/is-parse-server/blob/master/cloud/collections/invoice/invoiceValidation.ts)** — max character limit TBD (work with design, see Open Questions), no special/control characters, whitespace-only falls back to default.

4. **No change needed on [invoiceAddPayment.ts](https://github.com/invoice-simple/is-parse-server/blob/master/cloud/collections/invoice/functions/invoiceAddPayment.ts#L84)** — already has correct fallback:
```typescript
payment?.label || (isDeposit ? PaymentLabel.DEPOSIT : PaymentLabel.PAYMENT)
```

### is-web-app (web)

1. Remove `disabled={true}` on [upcoming-payment-form-modal.tsx](https://github.com/invoice-simple/is-web-app/blob/master/nextjs/components/invoice-editor/invoice-payment/payment-scheduling/upcoming-payment-form-modal.tsx#L193) (gated by FF)
2. Remove `disabled={true}` on [completed-payment-form-modal.tsx](https://github.com/invoice-simple/is-web-app/blob/master/nextjs/components/invoice-editor/invoice-payment/payment-scheduling/completed-payment-form-modal.tsx#L247) (gated by FF)
3. Add `maxLength` attribute on input
4. Keep auto-generated default ("Deposit"/"Payment") as initial value so blank labels aren't possible

### is-mobile (mobile)

1. **Upcoming/Deposit screen** — [manage-payment.screen.tsx](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/documents/screens/manage-payment.screen.tsx#L401): Remove `editable={false}` (gated by FF)
2. **Upcoming/Deposit screen** — [manage-payment.screen.tsx](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/documents/screens/manage-payment.screen.tsx#L352): `updatePayment` must destructure `name` and pass as `label` to `updateDeposit`/`updateUpcomingPayment` (otherwise label is lost on save)
3. **Service layer** — [payment.ts](https://github.com/invoice-simple/is-mobile/blob/develop/src/services/parse/models/payment.ts#L319): Add `label` param to `updateDeposit` and `updateUpcomingPayment` function signatures and pass through to `updateUpcomingOrDepositPayment`
4. **Historical payment screen** — [manage-historical-payment.screen.tsx](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/documents/screens/manage-historical-payment.screen.tsx): **Add a Name/Label field** — the field does not exist on this screen today. It needs to be added to the form and included in the submission payload. The backend already stores it via `invoiceAddPayment` (which `addHistoricalPayments` delegates to), so no backend change needed — just pass `label` from the client.
5. **Form type** — [type.ts](https://github.com/invoice-simple/is-mobile/blob/develop/src/features/upcoming-payments/components/manage-payment/type.ts): `HistoricalPaymentFormItem` must include `name` (remove from `Omit`)
6. Add character limit on both inputs
7. Keep auto-generated default as initial value

---

## Rendering — No Code Changes Expected (Verify Only)

All invoice rendering (public invoice, preview, PDF) uses the shared `@invoice-simple/invoice-viewer` package in `is-packages/packages/invoice-viewer/`.

The label is already rendered dynamically with a fallback:

- [UpcomingPayments.tsx](https://github.com/invoice-simple/is-packages/blob/master/packages/invoice-viewer/src/components/common/summary/UpcomingPayments.tsx#L101): `{payment.label || getLabel(labels.invoice.payment)}`
- [InvoicePayment.tsx](https://github.com/invoice-simple/is-packages/blob/master/packages/invoice-viewer/src/components/common/summary/InvoicePayment.tsx#L107): `{payment.label || getLabel(labels.invoice.payment)}`

PDF is generated from the same rendered HTML (via `getPdfFromHTML()` in `is-web-app/server/utils/pdf.ts`), so it inherits the same rendering.

Custom labels will render automatically — no template changes needed. The only concern is **visual overflow with long labels**.

**Key files in `@invoice-simple/invoice-viewer`:**

| File | Role |
|---|---|
| [UpcomingPayments.tsx](https://github.com/invoice-simple/is-packages/blob/master/packages/invoice-viewer/src/components/common/summary/UpcomingPayments.tsx) | Renders upcoming payment rows with label |
| [InvoicePayment.tsx](https://github.com/invoice-simple/is-packages/blob/master/packages/invoice-viewer/src/components/common/summary/InvoicePayment.tsx) | Renders completed payment rows with label |
| [InvoiceSummary.tsx](https://github.com/invoice-simple/is-packages/blob/master/packages/invoice-viewer/src/components/invoice/templates/style1/summary/InvoiceSummary.tsx) | Template-specific summary layout (style1–7) |
| [dictionary.ts](https://github.com/invoice-simple/is-packages/blob/master/packages/invoice-viewer/src/is-intl/dictionary.ts#L65) | Default label string ("Upcoming Payments" header) |

**Surfaces to verify with long labels:**

| Surface | Location | Concern |
|---|---|---|
| Public invoice (`/v/` route) | "UPCOMING PAYMENTS" section | Long labels could overlap or wrap |
| PDF export | Same section in generated PDF | Fixed-width layout — may overflow |
| Invoice preview (in-app) | Same `@invoice-simple/invoice-viewer` components | Same |
| Mobile Payment Scheduling | Payment row label | Should handle custom values (renders plain text) |
| Web Payment Scheduling | Payment row in lists | Same |
| Email templates | Check if payment breakdown appears in transactional emails | TBD |

---

## Acceptance Criteria

- [ ] User can type a custom name when creating a deposit, milestone, or recording a payment (web + mobile)
- [ ] User can type a custom name when recording a historical payment (mobile — new field)
- [ ] Custom label persists after save and displays correctly on Payment Scheduling screen
- [ ] Custom label renders correctly on public invoice (`/v/` route) without layout overflow
- [ ] Custom label renders correctly on PDF export without overflow or truncation
- [ ] Custom label renders correctly on invoice preview (`@invoice-simple/invoice-viewer`)
- [ ] Editing a payment without changing label does NOT reset it to default
- [ ] Old clients that don't send `label` field don't clobber existing custom labels (backend guard)
- [ ] Empty/whitespace-only input falls back to auto-generated default ("Deposit"/"Payment")
- [ ] Character limit enforced (client + server)
- [ ] Feature gated behind feature flag — field remains locked when FF is off

---

## Open Questions — Engineering

- [ ] Max character limit — work with design to determine best cap based on rendering/overflow concerns. Current pattern for name fields: server silently truncates to `maxReasonableStringLength` (100 chars) via `removeAdditionalAndBrokenFieldsAndTrimStringth` in prod — no client-side limit (e.g. business name is unlimited on client). If we pick a reasonable limit, same pattern works here (no client enforcement needed).
- [ ] Verify label rendering on public invoice with long strings — does it wrap, truncate, or overflow?
- [ ] Verify PDF rendering with long labels
- [ ] Verify `@invoice-simple/invoice-viewer` handles long labels across all 7 template styles
- [ ] Check if any email templates include the payment label
- [ ] Confirm `invoiceValidation.ts` is the right place for server-side validation

## Open Questions — Product/Design (discuss during grooming)
- [ ] Should label remain editable after a payment is marked as paid, or lock once completed?
- [ ] Should the default "Deposit"/"Payment" be a placeholder hint or a pre-filled value?
- [ ] Does the unlocked field need design review for styling?
- [ ] Any analytics event needed when a user customizes the label?

---

## Dependencies

- **Feature flag creation** — separate ticket (must be created in Optimizely before client work can be tested)

## Sub-tasks (split later)

1. **is-parse-server** — backend guards + validation (no FF needed server-side)
2. **is-web-app** — enable field behind FF + maxLength
3. **is-mobile** — enable field behind FF + character limit + **add new field to historical payment screen**
4. **Rendering QA** — verify `@invoice-simple/invoice-viewer` on public invoice, PDF, preview with long labels

---

## References

- [Confluence: D&M Phase 1/2 Plan — Item #3](https://everpro-tech.atlassian.net/wiki/spaces/dev/pages/1169883137)
- [Figma: Payment Scheduling designs](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5795-11544)
- Local spec: `docs/superpowers/specs/payment-label-reactivation.md`
