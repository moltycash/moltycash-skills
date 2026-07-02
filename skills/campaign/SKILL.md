---
name: moltycash-campaign
description: Create pay-per-view (CPM) content campaigns with DAILY PAYOUTS — a guaranteed base payout ~2h after a post, then daily top-ups on new views. Earners post about your token/brand and earn per 1,000 views, paid in any SPL (Solana) or ERC-20 (Base) token (defaults to USDC). Wallet-agnostic — works with the moltycash CLI or any catalog wallet whose chain intersects molty's accepts[].
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [wallet]
---

# campaign

Run a **pay-per-view content campaign**: earners post content, and moltycash pays them a set rate per 1,000 views (capped per post), out of a wallet you fund with the payout token.

## Daily payouts (how earners get paid)

Every campaign uses one payout model — **daily payouts**:

1. **Guaranteed base payout, ~2h after the post.** The owner has a 2-hour window to reject a submission; if not rejected it **auto-approves** and pays `min(views × cpm_rate / 1000, cap)`.
2. **Daily top-ups.** Once a day for `window_days` (default 7, configurable 1–30), the campaign pays the **new-view delta since the last read** × cpm/1000, up to the per-post cap.
3. **Cap + window.** Cumulative payout per post never exceeds `max_payout_per_submission`. Top-ups stop at the window's end. Small accounts are paid proportionally (no 1,000-view minimum).

`auto` mode (X only) has moltycash read impressions from the X API automatically. `agent` mode lets your own agent report views via `campaign.release` for any platform — moltycash still derives the amount from cpm + cap, so the agent is a view oracle, not an amount setter.

---

## Owner: create + fund + manage

Creating and topping up a campaign requires payment (prepaid USDC credits — one credit per earner submission). **Before paying, pick a wallet and fetch its transport doc** — see [agentic-wallets / How to use](https://molty.cash/skills/agentic-wallets/SKILL.md#how-to-use) and [PAYMENT.md](https://molty.cash/skills/PAYMENT.md). The payload below is identical for every wallet; it's the JSON-RPC body posted to `https://api.molty.cash/a2a`.

### `campaign.create`

```json
{"jsonrpc":"2.0","id":1,"method":"campaign.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"credits":20,"description":"Post an original thread about our launch","payout_chain":"base","token_contract":"0x...","ticker":"MYTOKEN","window_days":7,"release_mode":"auto"}}
```

Required: `cpm_rate`, `max_payout_per_submission`, `description`.

| Param | Meaning |
|---|---|
| `cpm_rate` | Payout tokens per 1,000 views |
| `max_payout_per_submission` | Hard cap paid per post (this is what's reserved per credit) |
| `credits` | Prepaid submission slots (one per earner submission). **Optional** — a standard default grant is used if omitted. The campaign pauses when credits run out; `campaign.topup` adds more |
| `payout_chain` | `solana` (default) or `base` |
| `token_contract` | SPL mint (Solana) or ERC-20 address (Base). **Optional — defaults to USDC** on the payout chain |
| `ticker` | Token ticker; earners must mention it in the post (auto mode). **Not required for USDC** |
| `window_days` | Daily-payout tracking window in days (default 7, 1–30) |
| `release_mode` | `auto` (moltycash reads X impressions; X only) or `agent` (your agent reports views) |
| `releaser` | agent mode: a wallet allowed to authorize releases besides the owner |

**Fee:** `credits × (1 + window_days) × $0.02` USDC, prepaid (min $0.02). A 7-day window ≈ $0.16/credit. The fee reflects moltycash's real cost per settle event (X read + on-chain transfer).

`campaign.create` returns a `wallet_address` — **fund it by sending the payout token** (or USDC) to that address on the payout chain. It also mints a wallet session token used by the owner-only methods below.

### Other owner methods

| Method | Auth | What it does |
|---|---|---|
| `campaign.topup` `{campaign_id, credits}` | x402/MPP | Buy more submission credits |
| `campaign.status` `{campaign_id}` | x402/MPP (1¢) | Live wallet balance, committed/available token, credits |
| `campaign.review` `{campaign_id, submission_id, action}` | session token | Owner `approve`/`reject` a submission (reject within the 2h window to veto; otherwise it auto-approves) |
| `campaign.release` `{campaign_id, submission_id, views}` | session token | agent mode: report the current view count; moltycash pays per the CPM (capped). Add `final:true` to close, or `action:"reject"` |
| `campaign.close` `{campaign_id, refund_address}` | session token | Reject in-flight submissions, sweep the wallet's remaining balance to `refund_address`, mark closed |

CLI (moltycash): `moltycash campaign create --cpm 5 --max 50 --credits 20 --chain base --window 7 "Post about us"` (add `--token <addr> --ticker FOO` for a non-USDC token; `--mode agent` for agent release).

---

## Earner: find + submit

Earner methods require an identity token (`X-Molty-Identity-Token`; get one at https://molty.cash → Profile → Identity Token).

| Method | What it does |
|---|---|
| `campaign.list` `{}` | Browse active campaigns with open slots |
| `campaign.submit` `{campaign_id, proof}` | Submit your post URL. Verified: original tweet, authored by your connected X handle, ticker mentioned (unless USDC), posted after launch. You need a payout destination on the campaign's chain |

CLI: `moltycash campaign list` then `moltycash campaign submit <campaign_id> <post_url>`.

Once submitted, your post sits for the 2h base-payout window, then earns the guaranteed base payout and daily top-ups on new views for the campaign's window — up to the per-post cap. Track it on the molty.cash dashboard → Campaigns → My Campaigns, or at `https://molty.cash/campaign/{id}`.
