# Exploration: Automatic Payment Reminders

> How should the system send payment reminder emails for Deposits & Milestones, and log them in invoice history?

**Figma references:**
- Configure Payment Requests (settings row): https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5349-23055
- Payment Reminder in Invoice Log: https://www.figma.com/design/Xq8u2VsUw8uPoZKjHPBc02/Deposits---Payments-Scheduling?node-id=5285-33486

---

## Scope Clarification (from Seth)

This feature — "Automatic Payment Requests" — is **Option 3 only** and applies exclusively to **non-vaulted invoices** (i.e. invoices with milestones but no auto-pay vault set up).

- If the merchant has **Automatic Payment Requests** enabled in Settings, the client automatically gets a reminder email on the day each milestone is due
- `Payment Requested` is logged in the Invoice History (Msg class) when the reminder is sent
- This is **not** the same as the auto-charge reminder for vaulted invoices — it's a nudge to a client who will pay manually

The only other automatically-sent non-RP email today is the **abandoned cart** reminder (two reminders at 30min and 2 days after checkout open, for buyers who don't complete payment). This feature would be the second new category of automated email.

---

## Existing Pattern: Abandoned Cart Reminders

The abandoned cart flow (`is-abandoned-cart`) is the most relevant existing pattern for this feature — it's event-driven, per-invoice, and uses EventBridge Scheduler:

```
Checkout viewed event
→ schedule-reminder Lambda — creates at() EventBridge schedule per checkout
→ [on due time] send-reminder Lambda — sends email, updates DynamoDB record
→ remove-schedulers Lambda — cleans up schedules when payment is attempted
```

Key files:
- `is-abandoned-cart/src/lambda-handlers/send-reminder/scheduled-handler.ts`
- `cdk/src/lib/constructs/abandoned-cart.ts`
- DynamoDB table `CheckoutReminderSchedule` tracks reminder state

---

## Scheduling Options

For milestones, each payment has an arbitrary due date set by the merchant — and that date can change at any time before vaulting.

**Option A: Per-milestone EventBridge schedule**
- Create an `at(YYYY-MM-DDTHH:MM:SS)` one-time schedule per milestone when it's created
- Delete and recreate the schedule whenever the milestone due date changes
- Problem: Every merchant edit of milestone dates requires a reschedule. Juan flagged this as overkill.

**Option B: Daily cron job (Juan's suggestion)**
- A Lambda runs once per day, queries all invoices with "Automatic Payment Requests" enabled and milestones due today
- Sends reminder emails for any matching milestones
- No per-invoice schedule management; no reschedule-on-edit
- Tradeoff: Reminder fires at a fixed daily time (e.g. 9am UTC), not at a merchant-specified time

**No existing daily cron found** — a search of the CDK constructs confirms there is no existing Lambda that scans invoices by due date daily. This would be new infrastructure (but straightforward — a scheduled EventBridge rule + Lambda).

**Recommendation: Option B (daily cron).** Simpler operationally, no rescheduling logic on milestone edits, and reminders don't need minute-level precision. The abandoned cart pattern (Option A) is per-checkout, but those schedules are created once and never updated — milestones are editable, making Option B more robust.

---

## Proposed Flow

```
Daily cron Lambda (EventBridge rate(1 day) rule)
→ Query is-payments DB: upcoming_payments with due_date = today, auto_pay_enabled = true, status != paid
→ Filter: account has "Automatic Payment Requests" setting enabled
→ For each: send reminder email via sendTemplatedEmail() / is-msg
→ Parse.Cloud.run('msgAddAutoPayEvent') — creates PAYMENT_REQUEST_SENT Msg entry
→ Invoice History updated on web + mobile automatically
```

### Where the cron lives

- **is-payments** — owns the milestone/upcoming_payment data, knows what's due when; natural home for this Lambda
- Calls is-msg/is-messaging for the actual email send (same pattern as is-payments today for other notification emails)
- See `is-payments/src/lambda-handlers/util/email/send-templates.ts` for existing email-send patterns within is-payments

---

## Parse Msg Creation

Use `Parse.Cloud.run('msgAddAutoPayEvent')` from the Lambda to write the `PAYMENT_REQUEST_SENT` Msg entry after the email is sent. See `docs/exploration-invoice-history-auto-pay-statuses.md` for full rationale (Juan: backend writes to Parse should go via Cloud Functions, not direct REST API).

---

## Email Content

The reminder email would:
- Reference the specific milestone amount and due date
- Include a link to the client checkout/payment page (for manual payment)
- Be a new SendWithUs template (similar to existing RP reminder template)

---

## Account Setting: "Automatic Payment Requests"

Seth's design has this as a merchant-level setting (visible in the "Configure Payment Requests" settings row in Figma). The cron Lambda needs to check this setting per account before sending. Where this setting lives (Parse `Account` object, is-payments config, or elsewhere) needs to be confirmed.

---

## Open Questions

1. **Same-day or advance notice?** Does the reminder fire on the due date, or N days before? Seth's description says "on the day the invoice is due" — so same-day is the current intent.

2. **Deposit milestone excluded?** The deposit is paid at checkout. Presumably no reminder needed for it — only future milestones.

3. **Multiple milestones due the same day?** Batched into one email or separate per milestone?

4. **Where is the "Automatic Payment Requests" account setting stored?** Parse Account object, or a new is-payments config record?

5. **Merchant copy?** Does the merchant also get a notification when the reminder is sent, or is it client-only?

6. **What if the client vaults after a reminder is sent?** Should future reminders stop once vaulting occurs (switching to auto-pay mode)?

---

## Relationship to Other Decisions

- **Decision 12 (reminder emails):** This doc expands on that decision — daily cron is the recommended implementation approach.
- **Decision 15 (successive scheduling):** For vaulted invoices, the auto-charge EventBridge schedule handles execution — this reminder cron is only for non-vaulted (Option 3) invoices.
- **Invoice history doc:** `PAYMENT_REQUEST_SENT` Msg entry is created here in the cron Lambda after the email is sent. See `docs/exploration-invoice-history-auto-pay-statuses.md`.
