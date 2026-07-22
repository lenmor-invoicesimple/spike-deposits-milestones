# Ticket: Add Request Deposit + Online Payment Fee to Estimate Editor (Mobile)

**Epic:** IS-11250 — Deposit Optimization v1
**Type:** Story
**Priority:** High — first slice of the epic, no backend dependencies

---

## Summary

Add the ability for merchants to configure a deposit on estimates in the mobile editor and enable the Online Payment Fee (surcharge) section. This includes:
1. A new "Request Deposit" toggle + type/amount picker on the estimate editor
2. "Deposit Due" line in totals section when deposit is ON
3. Online Payment Fee section (surcharge slider) enabled for estimates

The IS Payments section (Stripe/PayPal tiles, Configure Payment Methods, Manage Payment Settings) **already exists on the estimate editor today** — no work needed there.

This is merchant-side setup only — no buyer checkout flow, no conversion logic.

---

## Acceptance Criteria

### Request Deposit Section (NEW)
- [ ] "Request Deposit" toggle appears on the estimate editor between Balance Due and IS Payments section
- [ ] Toggle only appears if merchant has IS Payments configured (Stripe or PayPal connected)
- [ ] When toggled ON:
  - Shows "Type" selector (Flat Amount / Percentage)
  - Shows "Amount" field ($0 default for flat, percentage value for %)
- [ ] Percentage type: max 50% cap enforced (validation + UI constraint)
- [ ] Deposit data persists on save (`depositType`, `depositRate`, `depositAmount` fields)
- [ ] Deposit amount reflects in the estimate Preview/PDF when sent

### Totals Section Update
- [ ] Add "Payments" row to estimate totals (between Total and Balance Due) — currently only on invoices
- [ ] When deposit is ON: totals row changes from "Balance Due: $2,000" to "Deposit Due: $500" (showing the deposit amount due, not the full balance)
- [ ] "Payments" row shows $0.00 initially (will reflect paid amount after buyer pays deposit in future flow)

### Online Payment Fee Section (EXISTING component, enable for estimates)
- [ ] "Online Payment Fee" section renders on estimate editor (below IS Payments section)
- [ ] Shows toggle + description text ("When your client pays online, add a fee to cover payment processing costs")
- [ ] "How much of the processing cost should your client cover?" slider (0-100%)
- [ ] Fee preview: "A $69.80 fee is added to the estimate if paid online"
- [ ] Same behavior as on invoices today

### Conditions / Gating
- [ ] "Request Deposit" toggle only appears if merchant has IS Payments configured
- [ ] No subscription tier gate — any merchant with IS Payments can use deposits on estimates (same as invoices on mobile)
- [ ] "Request Deposit" toggle is independent of "Request Client Signature" (Premium-only separate feature)

---

## Design Reference

- Figma: [Estimate Deposit Setup](https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=6098-34819)
- Proposed design shows 5 states:
  1. Toggle OFF — "Request Deposit" toggle (off), no type/amount fields
  2. Flat Amount, $0 — toggle ON, Type: Flat Amount, Amount: $0 (default ON state)
  3. Flat Amount, $500 — toggle ON, Type: Flat Amount, Amount: $500, Deposit Due: $500
  4. Percentage, 25% — toggle ON, Type: Percentage, Amount: 25%
  5. Percentage, 50% — toggle ON, Type: Percentage, Amount: 50%
- All states show: IS Payments section (Stripe/PayPal) + Online Payment Fee section below

### What already exists (confirmed in-app, "before" screenshots)
- IS Payments header + Stripe/PayPal tiles + Configure Payment Methods + Manage Payment Settings
- Request Client Signature toggle
- Show Financing Options toggle
- Signature field

### What's new vs current estimate editor
- "Request Deposit" toggle + Type + Amount (between Balance Due and IS Payments)
- "Deposit Due" line replacing "Balance Due" in totals when deposit configured
- Online Payment Fee section with slider (below IS Payments)

---

## Technical Notes

### Data Model (already exists, no migration needed)
The estimate shares the same MongoDB document (`Invoice` Parse class) as invoices. These fields already exist on the schema:
- `depositType`: enum (NONE / PERCENT / FLAT)
- `depositRate`: number (percentage value when type=PERCENT)
- `depositAmount`: number (calculated or flat amount)

The `@invoice-simple/calculator` package's `getDeposit()` function already works for any document regardless of docType.

### Mobile Architecture
Both invoices and estimates use the same screen component (`InvoiceScreen`) differentiated by `docType` prop.

**Request Deposit section (NEW component):**
- This is NOT the existing "Payment Scheduling" section — it's a new, simpler inline UI
- Existing Payment Scheduling has a hard `docType === DocTypes.DOCTYPE_INVOICE` guard — unrelated to this work
- New component renders between Balance Due totals and the IS Payments section
- Can reference the existing invoice deposit picker for data logic, but UI is different (inline toggle+type+amount vs navigating to a scheduling page)

**Online Payment Fee (`InvoicePaymentsPassingFeesSection`):**
- Component already exists and works on invoices
- Blocked on estimates by a single guard: `invoice-payments-passing-fees-section.tsx:85` → `docType !== DocTypes.DOCTYPE_INVOICE` returns null
- Fix: remove/relax this guard to include estimates
- Also update tracking hook: `use-track-surcharge-awareness.ts:23` has the same early return

### Guards to Remove / Relax (Mobile only)

| Location | Guard | Action |
|----------|-------|--------|
| `is-mobile/.../invoice-payments-passing-fees-section.tsx:85` | `docType !== DocTypes.DOCTYPE_INVOICE` returns null | Allow estimates |
| `is-mobile/.../use-track-surcharge-awareness.ts:23` | `docType !== DocTypes.DOCTYPE_INVOICE` early return | Allow estimates |

Note: Mobile has its own Parse mapping layer that does a simple `...setting` spread — it bypasses `is-packages/parse-domain` entirely, so no package changes needed for mobile.

### Guards NOT needed for mobile (web parity — future ticket)

| Location | Guard | Notes |
|----------|-------|-------|
| `is-packages/packages/parse-domain/src/mapping-functions/parseInvoiceToDocument.ts` | `getEstimateSettings()` doesn't extract deposit fields | Only affects web app read path |
| `is-packages/packages/domain-invoicing/src/document/estimate.ts` | `EstimateSettings` type excludes deposit fields | Only affects web app type system |
| `is-services/packages/payments/payments-status/src/payments-status/invoice-payments-status.ts` | `if (documentType !== 0)` rejects estimates | Only needed for buyer checkout flow |

---

## Future Considerations (Invoice reuse)

The `RequestDepositSection` component should be built **docType-agnostic** — it reads/writes `depositType`, `depositRate`, `depositAmount` on whatever document it's given.

In `invoice.screen.tsx`, gate rendering with a simple condition:
```tsx
{docType === DocTypes.DOCTYPE_ESTIMATE && <RequestDepositSection ... />}
```

When Invoice adopts the same UI later (Seth's plan: replace the deposit portion of Payment Scheduling with this simpler toggle):
1. Change the condition to include invoices (or use a feature flag)
2. Remove deposit setup from `PaymentSchedulingSection` (keep milestones there)
3. No data model or calculator changes needed — fields are shared

This means: **don't embed estimate-specific logic in the component**. Keep it purely about deposit fields.

---

## Out of Scope
- IS Payments section (Stripe/PayPal tiles) — already works on estimates
- Buyer approval / checkout flow (separate ticket)
- Estimate → Invoice conversion logic (separate ticket)
- Card vaulting / save payment method (deferred — legal review pending)
- Payment Scheduling (milestones) on estimates
- QR code on estimate
- Email template changes

---

## Open Questions
1. ~~**Percentage options**~~ — ANSWERED: Free-entry with max 50% cap (Seth, 2026-07-07). Not preset values.
2. **Validation** — Can deposit amount be $0 with toggle ON and the estimate be sent? Can it exceed the total?
3. ~~**Web parity**~~ — ANSWERED: Mobile first. Web estimate editor will be addressed later.
4. ~~**Surcharge fee text**~~ — ANSWERED: Same logic as Invoice. On invoices it already calculates from the deposit amount when one is configured. Just reuse the same calculation — no special handling needed.

---

## Ticket Split

| # | Ticket | Scope | Dependencies |
|---|--------|-------|--------------|
| 1 | **Mobile — Enable Online Payment Fee on estimates** | Remove docType guard in `invoice-payments-passing-fees-section.tsx:85`; update tracking hook in `use-track-surcharge-awareness.ts:23` | None — independent, can ship anytime |
| 2 | **Mobile — Request Deposit UI component** | New `RequestDepositSection` component (toggle + type selector + amount input); 50% max cap on percentage; build docType-agnostic for future Invoice reuse | None — mobile bypasses is-packages |
| 3 | **Mobile — Estimate totals update (Payments row + Deposit Due)** | Add "Payments" row to estimate totals (currently invoice-only); show "Deposit Due" instead of "Balance Due" when deposit is configured | Depends on #2 |
| 4 | **Feature flag — Gate estimate deposit behind Flagsmith flag** | New Flagsmith flag (e.g. `estimate_deposit_enabled`) to control visibility of Request Deposit toggle + Deposit Due totals; allows gradual rollout and kill switch | None — can be created early, wired into #2 and #3 |
| 5 | **Web — Enable deposit fields for estimates in is-packages** | Extend `EstimateSettings` type in `domain-invoicing`; add deposit fields to `getEstimateSettings()` mapping in `parse-domain` | None — only needed when web estimate editor adds deposit support |

**Dependency chain:**
```
#2 (deposit UI) ─→ #3 (totals)
#1 (surcharge)   ← independent
#4 (flag)        ← wired into #2 and #3
#5 (web packages) ← deferred, not needed for mobile
```

### Why is-packages is NOT a blocker for mobile

Mobile has its own Parse mapping layer that bypasses `is-packages/parse-domain` entirely:
- **Read path:** `createInvoiceFromParseInvoice()` in `services/parse/models/invoice.ts` uses `...setting` spread — captures ALL Parse fields including `depositType/Rate/Amount` without needing explicit extraction
- **Write path:** `createParseInvoiceFromInvoice()` explicitly handles deposit fields with omit/include logic based on `depositType !== NONE`
- **Realm schema:** Already has `depositType`, `depositRate`, `depositAmount` as optional fields

The `getEstimateSettings()` function in `is-packages/parse-domain` (which omits deposit fields) and the `EstimateSettings` type in `is-packages/domain-invoicing` (which excludes deposit fields) are only used by the **web app** (`is-web-app`). Mobile never imports or calls these functions.

Ticket #5 is only needed when we build deposit support into the web estimate editor.
