# Auto-Pay Options Comparison — Session Notes (2026-05-14)

> One-page catch-up doc. Decision made: **Option 3 confirmed** by Liz, Seth, and Lenmor.

---

## Decision (from Slack thread)

- **Liz:** "Ok, let's move forward with Option 3."
- **Seth:** Leaned Option 2 + 3. Agreed lock-all-after-vault makes sense ("that schedule is what the client approved").
- **Lenmor:** Leaned Option 3. Posted detailed trade-offs for all 3 options.

Thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1778699694576969

---

## The Core Problem All Options Address

**Buyer vaults after a milestone date has already passed.** If the buyer is slow to pay the first milestone (deposit), subsequent milestone dates may have already passed by the time they vault. What do we do?

---

## Option 1 — Paradigm shift from dates to intervals

**pros:** Solves date-passed issue since intervalled payments start on first payment when they vault.

**cons:** Replacing due date with intervals — big change on merchant UI and on Parse (which we don't want to touch, it's more PCore).

**note:** Once intervals are in place, checkout can easily compute dynamic due dates on vault. Scheduling flows from vaulting so not a big deal. Failure scenario same as RI/RP (retries finish before next interval, minimum 1 week).

**upgrade paths:** From Option 1 → not really anywhere. From Option 3 → Option 1: can add intervals later as alternative mode, builds on top of scheduling.

---

## Option 2 — Consent shift: system charges on known dates → merchant charges saved card on unknown date

**pros:** Kinda solves date-passed issue by ensuring merchant receives notification and triggers charge. Simplest infra of all options — cron checks `vaulted invoices dueDate <= today`, sends email/push. We just need to save vaulted payment. Push notifications already exist (is-messages SDK, one call).

**cons:**
- Merchant action becomes a blocker. If they don't click? Re-notify daily? Accumulate? No terminal state.
- Security/Auth — in-app fine (already logged in). Email must link to authenticated session. Cannot put one-click charge link in email (bots, interception, double-clicks, phishing filters).
- Won't scale for merchants with many invoices (lots of taps).
- Failure scenarios — no automated retries, merchant contacts client manually.
- PCI/consent — language is more vague ("card saved for future payments" vs specific dates). Businesses do this (hotels), but needs tight consent.

**note:** Not really "automatic" — more just vaulting. Buyer loses predictability, "automatic" becomes murky.

**upgrade paths:** Option 2 → Option 3 is cleanest upgrade. Just add scheduling on top. Can offer both modes (notify-me vs auto-charge).

---

## Option 3 — Auto-charge on fixed calendar dates, all-or-nothing ✅ CHOSEN

**pros:** Truly automatic. Keeps dates mental model for merchants AND clear consent dates for clients. Enforce proper future dates on milestones (UI restriction only, no Parse changes). Infra similar to RI/RP.

**cons:**
- All-or-nothing limits adoption for short-interval milestones and/or slow-paying buyers.
- May need merchant UI nudge: "auto-pay works best when milestones are spaced further apart."
- Once vaulted, milestones + invoice are locked — consent only valid if dates/amounts stay fixed.
- Limits TPV generation (Liz's point) — fewer invoices qualify.

**notes:**
- Failure scenario: cancel auto-pay for all upcoming, revert to manual. Send emails (breaks consent, but same as RI/RP).
- Re-vault after failure: gate must exempt milestones already attempted (have an `upcoming_payments` row). Otherwise buyer can never re-vault since that date is now past.

**upgrade paths:** Option 3 → Option 1 (add intervals as alternative). Option 3 → Option 2 (fallback when gate blocks).

---

## Open Decisions (for after break)

### 1. Schedule all at once vs. schedule in succession

| | Schedule all | Succession |
|---|---|---|
| Merchant can edit future milestones after vault? | No (all locked) | Yes |
| Failure isolation | Independent (one failure ≠ all stop) | Chain stops on failure |
| Re-vault recovery | Simpler (skip past, reschedule future) | More complex |
| Consent alignment | Cleaner (fires on dates buyer saw) | Hidden conditional |
| Cancel cleanup | Delete N schedule pairs | Delete 1 |
| Infra complexity | Simpler (no chain logic) | More complex |

**Seth said:** Lock everything after vault makes sense ("that schedule is what the client approved"). This points toward schedule-all.

**Counter:** If merchants frequently edit future milestones (project scope changes, timeline shifts), succession is better UX.

**Question posted to Slack:** Do merchants have a strong need to edit future milestones after invoice is sent?

### Schedule-all vs succession: deeper implications

**Consent & UX:**
- At checkout, buyer sees ALL dates+amounts ("$500 on Jun 1, $1500 on Jul 1, $3000 on Aug 1").
- With **schedule-all**: system fires on each date independently. Matches consent exactly.
- With **succession**: system only fires next if previous succeeded. Checkout display is the same, but there's a hidden conditional — chain can stop mid-way. What you show ≠ what you do. Hard to communicate without making the feature sound unreliable.

**Failure recovery:**
- **Schedule-all (independent)**: Milestone 2 fails → milestone 3 still fires on its date. Respects buyer's consent per-date. But if card is dead, all milestones fail in sequence (cascading failures, noisy — multiple failure emails).
- **Schedule-all (dependent — cancel rest on one failure)**: Same terminal behavior as succession. Cancel remaining on first max-retry. Same email.
- **Succession**: Chain stops naturally on failure. No future row/schedule ever created. Clean, simple. Same as RP today.

**In both cases:**
- We send a cancellation/failure email (same as RP)
- Buyer can re-vault (same flow either way)
- Re-vault must exempt already-attempted milestones from the date gate

**Editing future milestones:**
- **Schedule-all**: All milestones locked at vault. Merchant wants to change milestone 3's date → must cancel auto-pay, edit, buyer re-vaults. Painful for long-timeline invoices (6 months, 4+ milestones — likely things will change).
- **Succession**: Only active (next) milestone locked. Future milestones editable in Parse. Chain reads current values when it advances. Merchant adjusts freely. But adds chain logic complexity.

**AWS resources:**
- Schedule-all: N schedule pairs exist simultaneously (e.g., 4 milestones = 8 EventBridge schedules).
- Succession: 1-2 schedules at any time. Lighter footprint, cheaper.

**Bottom line:** If we lock everything (Seth's lean), schedule-all is simpler to build and has cleaner consent. If merchants need to edit future milestones, succession earns its complexity. The answer depends on real merchant behavior.

---

### 2. Re-vault after failure

Do we allow re-vaulting for milestones that failed and are now past due? Or is (re)vaulting always from future (today + 1) only?

Likely answer: exempt already-attempted milestones from the gate. Simple check.

### 3. Typical milestone spacing (data question)

How often are milestones < 7 days apart? This determines how often the all-or-nothing gate actually blocks adoption. Need data or product input.

### 4. Reminder lead time

How many days before due date does the reminder email fire? (3 days? 7 days?) Needs design input.

---

## Key Technical Insights from Session

- **Push notifications exist** — `is-messages` SDK: `isMsg.pushNotification.alert({ accountId, alert, title })`. Fully built, sends to all merchant devices via Parse/FCM.
- **No magic links needed** — all charge actions happen in authenticated sessions (app or web). Email/push just route to the app.
- **MIT/PCI not a blocker** — all 3 options use same Stripe `off_session: true` pattern as RP. The difference is consent language clarity, not API config. RP already proves the model works.
- **RP handles same failure patterns** — retries (1/3/6 days), max retries → disable, buyer re-vaults. Battle-tested.
- **Option 1 has no unique security concerns** — same as Option 3/RP. Just a different input model.
- **Option 2's "security concern" (Liz)** — not a hard PCI/Stripe blocker. It's about consent copy specificity for dispute evidence. Hotels/mechanics do this daily.

---

## Action Items

| # | What | Who | When |
|---|------|-----|------|
| 1 | Confirm: do merchants need to edit future milestones after vault? (informs schedule-all vs succession) | Liz / Seth / Product | After break |
| 2 | Get data or estimate: how often are milestone intervals < 7 days? | Product / Analytics | After break |
| 3 | Decide reminder lead time (N days before) | Design (Seth) | After break |
| 4 | Update design doc to reflect Option 3 confirmation + lock-after-vault | Lenmor | After break |
| 5 | Decide schedule-all vs succession based on #1 answer | Lenmor + team | After break |
| 6 | Revisit tickets doc once scheduling decision made | Lenmor | After break |

---

## Reference

- Design doc: `2026-05-04-deposits-milestones-auto-payments-design.md`
- Tickets: `2026-05-07-deposits-milestones-auto-payments-tickets.md`
- Confluence: `2026-05-04-confluence-page.md`
- Liz's PRD: https://everpro-tech.atlassian.net/wiki/x/MYBePg
- Slack thread: https://ec-mobile-solutions.slack.com/archives/C0B331AP0BY/p1778699694576969
