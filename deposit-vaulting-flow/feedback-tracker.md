# Design Decisions & Feedback - Deposit Optimization

## TLDR

- Securing approval to only one person can only be done fully with auth (client portal login, OTP, etc.)
- A token adds defense-in-depth (unguessable URL, expiry, single-use, revocable) but the link with the token is still public — it doesn't solve the identity/auth problem
- Without auth, both token and plain URL have the same vulnerability: whoever has the link can approve
- The question is whether approval needs identity verification at all, or if checkout (card entry) is the real security boundary

---

## 1. Public estimate URL for approval (no token, no new service)

### Decision

Reuse the existing public estimate preview link (`/v/{documentId}`) with an "Approve" button. No token, no new microservice, no expiry. Approval converts estimate → invoice.

### Rationale

- Simplifies tech: no token table, no token service, no expiry/cleanup logic
- The estimate preview is already public — we're just adding a button
- No money moves at approval; checkout (card entry) is the real security boundary
- Faster to ship, fewer moving parts

### Opposition

**Dip Shah (2026-06-18):**
- "Big risk" — anyone can append `/approve` to a public estimate URL and instantly convert it to an invoice
- Removing the token is "cutting corners to move faster"
- A new service isn't much work (one-shot with CC, only overhead is devops URL config)
- Should run design by Jackson's team (they own estimate→invoice conversion)

**Worst-case scenarios raised:**
1. Merchant gets false approval signal → starts work → client disputes → merchant eats cost
2. Bot mass-converts estimates (if ObjectIds are guessable)
3. Vaulting chain: fraudster approves → pays with stolen card → card vaulted for future charges
4. Social engineering: victim tricked into clicking "approve" thinking they're just viewing

### Response

A token doesn't actually solve the core concern (identity). A token adds:
- Unguessable URL (vs. 24-hex ObjectId)
- Expiry (e.g. 7 days)
- Single-use (consumed on approval)
- Revocable by merchant

A token does NOT add:
- Identity verification — wrong person can still click a token link
- Bot protection — bot with the token link still works
- Forwarding protection — same link, same access

The identity problem is only solved by **authentication** (OTP, login) or by **accepting that approval is a low-stakes intent signal** (checkout is the real gate).

### Status

**Pending.** Legal consulted on how public/secure the approval should be (Legal Q16). Product deciding on traceability (does it matter that the named client is the one who approves). Response sent to Dip 2026-06-19.

### Fallback

If Legal/Product requires identity verification at approval, options are:
- Token-based link (defense-in-depth, but not true identity)
- OTP at approval step (true identity, but adds friction)
- Accept risk + confirmation step ("Are you sure you want to approve?")

---

## 2. (Placeholder — add future design decisions here)

---

## Action Items

- [ ] Loop in Jackson's team on estimate→invoice conversion design (Dip request, 2026-06-18)
- [ ] Update this doc after Legal responds (expected week of June 23)
