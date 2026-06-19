# Feature Flag Feasibility — D&M Phases 1, 2 & 3

> Context: Liz confirmed feature flags are required for Phase 1 (Milestone dueDate Sync), Phase 2 (Automatic Payment Requests), and Phase 3 (Autopay). This doc captures the findings on existing flag infra per system and the agreed approach for each phase.

---

## SDK Infra by System

| System | Optimizely | Flagsmith | Notes |
|--------|-----------|-----------|-------|
| **is-services** (APR + Autopay home) | `@is/optimizely` workspace package | `@is/flagsmith` workspace package | Used by is-checkout, is-stripe, is-unifiedxp |
| **is-parse-server** (Milestone sync) | None | None | Zero feature flag infra today |
| **is-web-app** | Yes | Yes | In server, nextjs, and client packages |
| **is-mobile** | `@optimizely/react-sdk` | `react-native-flagsmith` | Both installed |

---

## Phase 1 — Milestone dueDate Sync (is-parse-server)

**Problem:** is-parse-server has no feature flag SDK. The changes live in 3 hook points (`invoiceAddPayment`, `updatePayment`, `addPaymentToInvoice`) + the `beforeSaveInvoice` server guard.

Three approaches explored, ordered by complexity:

---

### Option A — Env var boolean toggle (simplest)

```
ENABLE_MILESTONE_DUEDATE_SYNC=true
```

- Simplest option — deploy to toggle, no per-user targeting
- Sufficient because the `beforeSaveInvoice` server guard already protects against old clients regardless of flag state
- No new SDK dependency in is-parse-server
- **Trade-off:** Deploy required to enable/disable; no gradual rollout

---

### Option B — Env var percentage rollout

```
MILESTONE_DUEDATE_SYNC_ROLLOUT_PCT=10   # 0–100
```

```typescript
// In recalculateInvoiceDueDate.ts — called from all 3 hook points
const pct = parseInt(process.env.MILESTONE_DUEDATE_SYNC_ROLLOUT_PCT ?? '0', 10);
if (pct === 0) return;
if (pct < 100) {
  // Stable per-account bucketing — same account always gets same bucket
  const bucket = parseInt(accountId.slice(-4), 16) % 100;
  if (bucket >= pct) return;
}
// proceed with recalculation
```

- No new SDK — just an env var + deterministic bucketing on `accountId`
- Gradual rollout without a feature flag service: set to `10`, observe, increase to `50`, then `100`
- Stable: same account always falls in or out of the rollout cohort (last 4 hex chars of accountId → 0–65535 → mod 100)
- **Trade-off:** Still requires a deploy to change percentage; no per-account override or kill switch from a dashboard

---

### Option C — Flag on clients + pass recalculated value to is-parse-server (no parse-server changes needed)

Rather than adding any flag logic to is-parse-server, put the flag on **every client that calls the milestone Cloud Functions** and have those clients compute and pass the new `dueDate` directly. is-parse-server just saves whatever it receives.

**Entry points that call milestone Cloud Functions (all need the flag):**

| Client | Cloud Function called | Trigger |
|--------|----------------------|---------|
| **is-mobile** | `invoiceAddPayment` | Merchant adds a milestone |
| **is-mobile / is-web-app** | `invoiceUpdatePayment` | Merchant edits/deletes a milestone |
| **is-mobile** | `invoiceUpdatePayment` (markPaid) | Merchant marks milestone paid |
| **is-stripe webhook** | `invoiceUpdatePayment` (markPaid) | Autopay charges a milestone |
| **is-paypal webhook** | `invoiceUpdatePayment` (markPaid) | PayPal charges a milestone (future) |

**How it works:**

```
Flag ON (new behaviour):
  Client computes recalculated dueDate from local milestone state
  → passes dueDate + termsDay=CUSTOM in the Cloud Function payload
  → is-parse-server saves it (no server-side recalculation needed)

Flag OFF (old behaviour):
  Client sends payload as today — no dueDate override
  → is-parse-server saves as-is
```

- **Pros:** Zero changes to is-parse-server; flag lives where the SDK already exists (web, mobile, is-stripe, is-paypal all have Optimizely/Flagsmith)
- **Cons:**
  - Logic is distributed across 4–5 clients instead of centralised in one server helper
  - Stripe/PayPal webhooks (is-services) need to compute the new dueDate from milestone data — requires a Mongo read that is-parse-server's `recalculateInvoiceDueDate` already does internally
  - More surface area to keep in sync; if one client misses the flag, dueDate diverges
  - The `beforeSaveInvoice` server guard (old-client protection) still needs to be shipped to is-parse-server regardless — old app versions don't know about the flag

**Verdict:** Higher coordination cost than Options A/B. Best considered if is-parse-server deploys are unusually slow or risky, otherwise the server-side approach (A or B) is cleaner.

---

### Comparison

| | Option A (bool env var) | Option B (% env var) | Option C (client-side flag) |
|---|---|---|---|
| **SDK changes to is-parse-server** | None | None | None |
| **Gradual rollout** | No | Yes (by accountId %) | Yes (Optimizely/Flagsmith per user) |
| **Per-account override** | No | No | Yes (flag targeting rules) |
| **Clients to change** | None | None | Mobile + Web + is-stripe + is-paypal |
| **Logic centralisation** | Server (1 helper) | Server (1 helper) | Distributed (4–5 clients) |
| **Effort** | ~30 min | ~1 hr | ~4–6 hrs |

---

## Phase 2 — Automatic Payment Requests / APR (is-services)

**APR Lambda:** New package `is-payment-requests` in is-services. `@is/flagsmith` is already a workspace package — no new infra needed.

**Agreed approach: gate on the settings UI toggle (web + mobile)**

- The `AutomaticPaymentRequests` Parse Setting is already **opt-in / disabled by default**
- Gate the UI toggle visibility with a Flagsmith flag on web and mobile — merchants can't see or enable it until rollout is ready
- The Lambda handler needs no flag logic: if the setting is off (UI hidden by flag → merchant never enabled it), the sweep query's `valBool: true` filter returns zero accounts → Lambda exits early naturally
- No flag logic inside the Lambda itself — consistent with how the whole codebase works today

**Pattern for the UI flag (web + mobile already have SDK):**
```typescript
// web / mobile — hide the toggle until ready
const flagEnabled = await isFeatureEnabled({ featureName: 'automatic_payment_requests' });
// only render the AutomaticPaymentRequests toggle if flagEnabled
```

**Note:** `is-abandoned-cart` (APR's structural pattern source) has **zero** feature flag usage — not a pattern source for flags. The correct patterns are `is-stripe/src/utils/flagsmith.ts` and `is-checkout/src/is-flagsmith/is-feature-enabled.ts`.

---

## Phase 3 — Autopay / Milestone Auto-charge (is-stripe)

**Agreed approach: flag in is-stripe's execute route**

Same pattern already exists for recurring payments (`isRecurringPaymentsEnabled` via Optimizely at `is-stripe/src/routes/backend/recurring-payment-execute.ts:118`). Add a parallel check for the milestone autopay flag at the same location.

```typescript
// is-stripe/src/routes/backend/recurring-payment-execute.ts
import { isRecurringPaymentsEnabled } from '../../utils/optimizely';
// add alongside existing RP flag check:
import { isMilestoneAutopayEnabled } from '../../utils/optimizely';

const isFeatureEnabled = await isMilestoneAutopayEnabled(user.id);
if (!isFeatureEnabled) {
  return res.status(403).json({ error: 'Milestone autopay flag is not enabled' });
}
```

- Lambda handlers in is-payments stay **flag-free** — consistent with all RP Lambdas today (`payment-execution`, `payment-events`, `payment-retry-scheduler`, all topic handlers)
- Flag lives at the HTTP route level (is-stripe), not inside the Lambda — same as RP today

---

## Summary

| Phase | Feature | Where flag lives | SDK | Lambda flag-free? | Effort |
|-------|---------|-----------------|-----|-------------------|--------|
| 1 | Milestone dueDate Sync | is-parse-server env var | None (env var) | N/A | ~30 min |
| 2 | APR email | Settings UI toggle (web + mobile) | Flagsmith | ✅ Yes | ~1 hr |
| 3 | Autopay execution | is-stripe execute route | Optimizely | ✅ Yes | ~1 hr |

All Lambda handlers stay flag-free across all phases — consistent with the existing RP and abandoned-cart patterns.
