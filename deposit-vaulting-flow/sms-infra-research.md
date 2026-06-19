# SMS Infrastructure Research

> Researched 2026-06-11 — auditing existing Twilio/SMS code in `api/` before D&M vaulting OTP work.

## What Exists

### Twilio Provider
**`api/src/providers/twilio/index.ts`**
- `sendSMS(to, message)` — raw Twilio HTTP call (not Node SDK)
- Auth: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_MESSAGING_SERVICE_SID`
- URL shortening enabled by default

### SMS Service
**`api/src/services/sms/index.ts`**
- `sendMobileSMSCampaignMessage(number, token)` — sends a promotional discount message
- `isConsentGivenForPhoneAndAccount(accountId, phoneNumber)` — checks consent in DB
- `findOrCreateSMSToken(accountId)` / `createSMSToken` / `getSMSToken` — token lifecycle

### DB Models
- **`api/src/models/db/sms-token.ts`** — `sms_token` table; fields: `accountId`, `token` (PK), `createdAt`; `isExpired()` checks 30-day window
- **`api/src/models/db/public-account-consent.ts`** — `account_consent` table; tracks `consent_type`, `consent_value`, `consent_given_at`, `consent_revoked_at`

### Consent Types
Defined in `@invoice-simple/common`. Currently only `SMS_PROMOTIONAL` exists for SMS.
Validated in `api/src/services/account-consent/types.ts`.

## What Is NOT Wired Up

| Item | Status |
|------|--------|
| `api/src/routes/sms.ts` | Defined but **not mounted** in `server.ts` |
| `generateTokenAndSendSMS` controller | Exported, **no route pointing to it** |

The only mounted SMS-adjacent route is `POST /sms-invoice` (in `v1-root.ts`) — but it does **not** send an SMS; it returns a URL and fires a push notification.

## What the Existing Flow Actually Does

`generateTokenAndSendSMS` → consent check → `sendMobileSMSCampaignMessage` → sends:
```
"Invoice Simple: Get 17% discount when you subscribe to Invoice Simple via {URL}\n\nReply STOP to opt-out."
```
URL pattern: `{WEB_APP_HOST}/stripe/sms-checkout/{token}`

This is a **promotional campaign** flow, not transactional.

## Not Related: Mobile "Text" Button on Invoice Screen

The invoice screen has a "Text" share button — this is **not** using Twilio and is unrelated to this infra.

- Calls `POST /share` to get a shareable invoice URL
- Opens the **device's native SMS app** (iOS: `sms://` URL scheme, Android: native SMS chooser) with the message pre-filled
- The user hits send from their own phone — no Twilio, no server-side SMS
- Controlled by `doSupportsSms()` (device hardware check) and `email_send_variant` Optimizely flag (UI layout)
- Key file: `is-mobile/src/features/documents/components/invoice-actions/invoice-send-button.tsx`

## What D&M Vaulting OTP Would Need

The Twilio plumbing (`sendSMS`) is reusable as-is. New pieces required:

1. **New consent type** — `SMS_TRANSACTIONAL` or `SMS_OTP` in `@invoice-simple/common`
2. **New send function** — OTP template (amount, code, expiry) calling `sendSMS()` directly
3. **OTP token model** — separate from `sms_token` (short expiry ~10 min, single-use, tied to payment intent)
4. **Phone number collection** — not captured today; deferred to v2 per spec

**v1 path:** email OTP only (no phone required), SMS OTP as fast-follow when phone collection is added.

## Ownership & History

Built by **Mubin** (former employee) ~4 years ago ([PR #1433](https://github.com/invoice-simple/api/pull/1433)) for a one-off SMS-only promotional campaign run by the **IS Growth team**. It was only active for a few months and never used again.

Confirmed by **Gabs Simonetti** (EM, `gabs@invoicesimple.com`) in [#is-dev thread](https://ec-mobile-solutions.slack.com/archives/C0635LNNK27/p1781195826404679):
> "The code should still work, but we should do the due diligence of checking API costs"

**Next step before reusing:** Verify Twilio account is still active, check current API pricing/tier, and confirm the messaging service SID is still valid.

## Related Files

- `api/src/controllers/sms.ts` — controller for token generation and checkout data fetch
- `api/src/util/parse.ts` — `getSMSCheckoutData(accountId)` helper
- `api/src/config.ts` (lines 34–36) — Twilio env var exports
- `api/test/unit/controllers/sms.test.ts` — unit tests for all 3 controller functions
- `api/test/unit/services/sms.test.ts` — token lifecycle + consent tests
