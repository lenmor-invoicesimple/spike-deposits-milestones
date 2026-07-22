# Deposit Optimization v1 — Session Handoff Log

Tracks what each session is doing and where things stand for cross-session coordination.

---

## Current Session Split (as of 2026-07-14)

| Session (working dir) | Scope | Status |
|-----------------------|-------|--------|
| `/Users/lenmor/is5` | Planning, architecture decisions, docs, is-parse-server cloud functions | Active — this file |
| `/Users/lenmor/is/is-mobile` | Mobile app implementation (IS-11406) | Active — handed off |
| Future: `/Users/lenmor/is5` | is-parse-server `estimateSetDeposit` cloud function body | Pending |
| Future: TBD | Enable Payments on Estimates backend (IS-??) | Pending investigation |

---

## Ticket Split

| Ticket | Title | Session | Status |
|--------|-------|---------|--------|
| IS-11406 | Estimate Deposit CRUD (Mobile) | is-mobile | In progress |
| IS-11407 (TBD) | Enable Online Payments on Estimates (Backend) | is5 | Investigation complete, impl pending |
| Future | Estimate Deposit — is-parse-server cloud function | is5 | Placeholder only for now |
| Future | Web Estimate Deposit | TBD | Blocked on IS-11406 |
| Future | Invoice Deposit Redesign (Mobile) | TBD | Blocked on designs |

---

## IS-11406 — Estimate Deposit CRUD (Mobile)

**Handoff doc:** `/Users/lenmor/is5/SCRATCH/estimate-deposit-mobile-impl-progress.md`

### What the mobile session is implementing

1. New `estimateSetDeposit()` wrapper in `src/services/parse/models/payment.ts`
2. Rewire `onToggleDeposit`, `onChangeDepositType`, `onChangeDepositAmount` in `invoice.screen.tsx` to call cloud function instead of Realm→Parse sync
3. Confirm `!isEstimate` guard in `invoice.navigator.tsx:updatePaymentsAndDeposit()`
4. `RequestDepositSection` component — already exists, no changes needed

### What the mobile session is NOT doing

- Cloud function body in is-parse-server (placeholder/throw only)
- Any changes to is-parse-server

### Key architecture decision

New `estimateSetDeposit` Parse Cloud Function (idempotent find-or-create). Client sends desired state, server handles create/update/delete. Uses masterKey → `handleDepositChange` hook bypassed. Avoids zero-value bug (PR #4252).

---

## IS-?? — Enable Online Payments on Estimates (Backend)

**Investigation doc:** `/Users/lenmor/is5/docs/superpowers/specs/deposit-optimization-v1/enable-payments-on-estimates.md`

### What's been investigated

- Single blocking gate: `findNotPayableReason()` in `payments-status` package, line 58
- All 6 callers documented (is-checkout, is-unifiedxp, is-stripe x2, is-abandoned-cart)
- 3 fix options documented; recommendation: Option B (allowlist) + feature flag at caller level
- Mobile payments tile already works for Estimates — no mobile changes needed

### Open questions still needing investigation

1. Does `CheckoutData.balanceDueCents` compute correctly for estimates in is-checkout?
2. Does payment confirmation write to the inline `payments` array on Parse Invoice? (length=0 validation would block it for estimates)
3. Feature flag — which service (Flagsmith? Optimizely?) and which layer?
4. Should abandoned cart exclude estimates even after we enable payments?

### Next step for this ticket

Resolve open questions 1 & 2 by reading `is-checkout` format/checkout data logic and the payment confirmation flow in `is-stripe`. Then write the ticket and hand off implementation.

---

## `estimateSetDeposit` Parse Cloud Function (is-parse-server)

**Will be implemented in this `/is5` session** once mobile session confirms the interface is stable.

**Interface (agreed):**
```typescript
Parse.Cloud.define('estimateSetDeposit', async (request) => {
  // Input: { remoteInvoiceId, depositType, depositRate?, depositAmount? }
  // Output: { objectId } | null
  // Logic: find-or-create Payment record, delete if depositType === NONE
  // Uses masterKey — bypasses handleDepositChange hook
});
```

**File to create:** `is-parse-server/cloud/collections/invoice/functions/estimateSetDeposit.ts`  
**Register in:** `is-parse-server/cloud/collections/invoice/index.ts` (or wherever functions are registered)

---

## Ticket Writing Rules (carry forward from D&M sprint)

- Verify line numbers against remote branches (`git show master:...`), never local working dir
- Branch conventions: `is-web-app` → `master`, `is-mobile` → `develop`, `is-parse-server` → `master`
- No feature flags on is-parse-server (backend guards unconditional)
- Iterate locally → push to Jira once when final (preserves user's inline images)
- No local spec file paths in Jira tickets
- No complexity estimates in tickets
