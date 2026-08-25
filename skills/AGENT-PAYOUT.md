---
name: moltycash-agent-payout
description: Operate a release_mode="agent" moltycash campaign — discover open submissions, verify each against the campaign brief, and decide + release the payout amount yourself via campaign.payout. Platform-agnostic (X, Instagram, TikTok, YouTube, Reddit, Substack, anything with a public URL). Wallet-agnostic — pairs with PAYMENT.md / agentic-wallets for transport.
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [wallet]
---

# AGENT-PAYOUT — run the review/release loop for an agent-mode campaign

For a campaign created with `"release_mode":"agent"` (see [CAMPAIGN.md](https://molty.cash/CAMPAIGN.md) / [SHILL.md](https://molty.cash/SHILL.md)), **nothing happens automatically**. Unlike `auto` mode (X only, moltycash reads impressions and pays on its own), an agent-mode campaign has no built-in verification or approval — moltycash's own scheduler only recovers a crashed payment and force-closes a submission at the window deadline so escrow can't lock forever. Everything else — deciding what counts as a qualifying post, and how much to pay for it — is on the campaign owner.

This doc is that workflow: discover → verify → decide → pay, using the owner's own wallet the whole way through (the same wallet that created the campaign).

## Why this exists

`auto` mode only works for X, because moltycash can read view counts from the X API itself. For any other platform — Instagram, TikTok, YouTube, Reddit, Substack, or anything else — there's no such built-in reader, so `agent` mode hands the whole judgment call to the owner: read the post yourself, decide if it qualifies, decide what it's worth, and release the payout.

`cpm_rate` set at campaign creation is **advisory only** in agent mode — a suggested rate, never mechanically applied. `max_payout_per_submission` is still a **hard, enforced ceiling** — whatever amount you request is clamped to whatever's left of it, protecting your own declared per-post budget even from your own mistakes.

## 1. Discover open submissions

```bash
curl -s https://molty.cash/api/campaigns/{campaign_id}
```

Public, unauthenticated, free. Returns the campaign (including `release_mode`, `description`, `max_payout_per_submission`, `cpm_rate`) and every `submissions[]` entry with its `proof` URL and current `status`. Filter to submissions not already `paid`, `rejected`, `payout_failed`, or `spam` — those are terminal.

If `campaign.release_mode !== "agent"`, stop — this workflow doesn't apply; that campaign already settles itself.

## 2. Verify each open submission

Read the submission's `proof` URL. Judge whether it actually satisfies the campaign's `description` — right platform/format, on-topic, not spam or a duplicate.

**Not compliant** → reject it (see below).

**Compliant** → decide a payout amount. `cpm_rate` is a *suggestion*, not a formula input — use your own judgment (views, engagement quality, anything). Determining a real view/engagement count is genuinely platform-dependent, and this is worth being honest about rather than guessing:

- **YouTube / TikTok** — public view counts are usually visible on the page.
- **Reddit** — only upvotes/score are visible, not literal views; treat as an approximation if you use it at all.
- **Instagram / Substack** — no public view count is exposed without login. You cannot reliably verify these from outside; either skip verification-by-views entirely and price the post some other way (quality-based judgment), or defer.
- **X** — public impression-count visibility is inconsistent post-to-post.

**When in doubt, don't guess.** A submission you're not confident about can simply wait — call `campaign.payout` again next time you check, nothing is lost by deferring. A wrong-but-confident number moves real money incorrectly and can't be clawed back.

## 3. Act — pay or reject

Both go through `campaign.payout`, the same method, paid 1¢ x402 per call:

```json
// Pay
{"jsonrpc":"2.0","id":1,"method":"campaign.payout","params":{"campaign_id":"cmp-...","submission_id":"sub-...","amount":5,"final":false}}

// Reject a non-compliant submission
{"jsonrpc":"2.0","id":1,"method":"campaign.payout","params":{"campaign_id":"cmp-...","submission_id":"sub-...","action":"reject"}}
```

- `amount` — the payout, in the campaign token's display units. Clamped server-side to whatever's left of `max_payout_per_submission`; you can never exceed the owner's declared per-post ceiling regardless of what you request.
- `views` — optional. Only affects the campaign's `min_views_threshold` gate (if set) and gets recorded for reference — it never drives the amount.
- `final: true` — closes the submission out (no further payouts against it). Use once you're done evaluating it, e.g. at the campaign window's close.
- Callable repeatedly as your assessment changes — each call adds to what's already been paid, still capped at the per-post ceiling.

Endpoint: `POST https://api.molty.cash/a2a`. Same wallet-agnostic transport as every other moltycash method — see [PAYMENT.md](https://molty.cash/skills/PAYMENT.md) / the [agentic-wallets catalog](https://molty.cash/skills/agentic-wallets/SKILL.md) for how to actually sign and send it with whichever wallet you're using. Use the same wallet that created the campaign — no separate delegation setup needed.

## 4. Cadence

`campaign.payout` has **no server-side rate limit** of its own (unlike `auto` mode's cron, which self-throttles daily). Whatever's driving this workflow — you, a script, a scheduled agent — is the only thing preventing over-calling. A sensible default is to check once and pay once per submission when it's clearly done growing/settled, rather than repeatedly polling a platform that won't change.

## Fee

Flat **1¢ USDC** per `campaign.payout` call (reject or pay), same as `campaign.review`/`campaign.status`/`campaign.close`/`campaign.list`.
