# Session Recap — 2026-06-18

## Legal Questions Doc
Compiled all legal questions for Deposit Optimization into `2026-06-18-legal-questions.md`. Two sections:
- **Liz's final consolidated 16 questions** — sent to Legal (Alex Higgins, expected response week of June 23)
- **Extended research list** — full team questions (Lenmor, Fred, Seth) with notes on what was absorbed/dropped

## Dip's Feedback (Slack + Confluence)
Dip flagged that removing the token means anyone can hit `/approve` on a public estimate URL and convert it to an invoice.

**Key conclusion:** A token doesn't solve the identity problem — it only adds defense-in-depth (unguessable URL, expiry, single-use, revocable). True identity verification requires auth (OTP, client portal login, etc). Both token and plain URL share the same vulnerability: whoever has the link can approve.

Documented in `feedback-tracker.md` and `estimate-auto-accept-link-spec.md`.

## Jackson's Feedback (Slack)
- Public approval flow could **bypass subscription doc limits** (Essentials/Plus users have invoice caps) — the cloud function must check the merchant's doc limit before creating the invoice
- Confirmed `setting.fullyPaid: true` marks an estimate as closed/converted
- Doc limit check reference: `is-web-app/nextjs/app/(authenticated)/(core)/(documents)/actions/convert-document.action.tsx#L45`

Added to the spec as a guard in the Guards table.

## Confluence Inline Comments (8 open, not yet addressed)
Pulled all 8 open inline comments from the Confluence tech plan. **Were about to go through them one by one when session ended.**

| # | Who | Highlighted text | Comment |
|---|-----|---------|---------|
| 1 | Dip | "renders public read-only estimate page" | "where will this live?" |
| 2 | Dip | "or read directly from Mongo?" | "any reasoning to do this?" |
| 3 | Dip | "validates token (NOT consume)" | "what happens if end user decides to forward the same email to multiple people? and they all open the link?" |
| 4 | Dip | "no token" | "if you don't have a token, anybody can just add /approve to the public estimate url and it will be converted to an invoice" |
| 5 | Sonya | "Calls Parse Cloud Function approveEstimate({ documentId, email? })" | "ye it's not good, we need to encode stuff" |
| 6 | Sonya | "Sends silent sync push + visible ALERT push" | "do we have a native mechanic on parse server to do it or need some plumbing?" |
| 7 | Sonya | "Mark estimate fullyPaid: true (→ 'Closed')" | "this fullyPaid check is gonna be nightmare to maintain, cuz merchant can edit amount any time" |
| 8 | Sonya | "/v/{documentId}/approve detects approvedAt is set → shows public invoice with 'Pay Online' button" | "not 100% confident on url here" + "we could at some point add some signing logic or 'workflow' style thing and it will be conflicting" |

## Next Step
Go through the 8 Confluence comments one by one and reply/resolve each.
