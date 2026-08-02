---
name: moltycash-shill
description: Create pay-per-view (CPM) content campaigns that pay out in YOUR OWN token via shill.create — token_contract is mandatory (no USDC fallback). Earners post about your token and earn per 1,000 views, paid in the SPL (Solana) or ERC-20 (Base) token you specify. Same underlying campaign as moltycash-campaign, managed with the same methods — see that skill for status/review/release/close/list. Wallet-agnostic — works with the moltycash CLI or any catalog wallet whose chain intersects molty's accepts[].
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [wallet]
---

# shill

Run a **pay-per-view content campaign that pays out in your own token**: earners post content, and moltycash pays them a set rate per 1,000 views (capped per post), out of a wallet you fund with your token.

`shill.create` is the token-mandatory sibling of `campaign.create` (see [CAMPAIGN.md](https://molty.cash/CAMPAIGN.md)) — both create the exact same campaign type and are managed identically afterward. The only difference is at creation time:

| Method | `token_contract` | Use when |
|---|---|---|
| `shill.create` (this skill) | **Mandatory** — rejected if omitted | You specifically want to pay in your own token and want a typo'd/omitted token to fail loudly instead of silently defaulting to USDC |
| `campaign.create` | Optional — omit to pay in USDC | You don't have a token, or you're fine with USDC as the default |

If you want the USDC-default behavior, use [CAMPAIGN.md](https://molty.cash/CAMPAIGN.md) instead — the rest of this doc assumes you have a specific token to shill.

## Daily payouts (how earners get paid)

Every campaign uses one payout model — **daily payouts**:

1. **Guaranteed base payout, ~2h after the post.** The owner has a 2-hour window to reject a submission; if not rejected it **auto-approves** and pays `min(views × cpm_rate / 1000, cap)`.
2. **Daily top-ups.** Once a day for `window_days` (default 2, configurable 1–30), the campaign pays the **new-view delta since the last read** × cpm/1000, up to the per-post cap.
3. **Cap + window.** Cumulative payout per post never exceeds `max_payout_per_submission`. Top-ups stop at the window's end. Small accounts are paid proportionally (no 1,000-view minimum).

`auto` mode (X only) has moltycash read impressions from the X API automatically. `agent` mode lets your own agent report views via `campaign.release` for any platform — moltycash still derives the amount from cpm + cap, so the agent is a view oracle, not an amount setter.

---

## How to use — read this **first**, blocking

Creating a campaign requires payment. Payment requires a wallet. **Before doing anything else**, walk through these three steps in order. Do not skip; do not assume `npx moltycash` is the default.

### Step 1 — Decide which wallet to use

**Ask the human you are working for which wallet to pay with.** Do not auto-detect, do not default to `*_PRIVATE_KEY` env vars. Wallet selection is a deliberate choice. This rule applies to every paid moltycash method (hire, campaign.create, shill.create).

Skip the ask only if either:

- the human's original prompt already named a wallet — use what they specified, **don't substitute**; or
- only one wallet is authenticated / available — use it.

### Step 2 — Fetch that wallet's transport doc

Each catalog wallet has its own transport syntax — there is no universal CLI for paying moltycash. Fetch the doc for the wallet the human picked:

```bash
curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md
```

For wallet docs, see the [agentic-wallets catalog](https://molty.cash/skills/agentic-wallets/SKILL.md). For the moltycash CLI fallback (private key), see [PAYMENT.md](https://molty.cash/skills/PAYMENT.md).

### Step 3 — Combine wallet transport + the canonical `shill.create` payload

The payload is **identical for every wallet** — it's the JSON-RPC body posted to `https://api.molty.cash/a2a`:

```json
{"jsonrpc":"2.0","id":1,"method":"shill.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"description":"Post an original thread about our launch","token_contract":"0x...","ticker":"MYTOKEN","window_days":2,"release_mode":"auto"}}
```

`description` and `token_contract` are **always required** for `shill.create`. `cpm_rate`/`max_payout_per_submission` are optional together — pass both, or omit both to auto-price `cpm_rate` at $1 worth of the token (`max_payout_per_submission` then defaults to `cpm_rate` × 10). Passing `max_payout_per_submission` without `cpm_rate` is rejected. See the full param table below.

Total to authorise: flat **$1 USDC**. Billing is commission-only — nothing else to prepay (see the Fee section below).

Substitute that payload into the transport pattern from the wallet's doc.

---

## Create + fund

### `shill.create`

```json
{"jsonrpc":"2.0","id":1,"method":"shill.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"description":"Post an original thread about our launch","token_contract":"0x...","ticker":"MYTOKEN","window_days":2,"release_mode":"auto"}}
```

Required: `description`, `token_contract`.

| Param | Meaning |
|---|---|
| `token_contract` | **Required.** SPL mint (Solana) or ERC-20 address (Base) — no default, no USDC fallback. The payout chain is inferred from the address format (`0x...` = Base, base58 = Solana) |
| `cpm_rate` | **MAX** payout tokens per 1,000 views. The effective rate scales with engagement ((likes+RTs+replies) / views): organically engaged posts earn near this, low-engagement/botted posts earn down to 25% of it. The per-post cap scales the same way. **Optional together with `max_payout_per_submission`** — pass both, or omit both to auto-price `cpm_rate` at $1 worth of the payout token (DexScreener-priced). Passing `max_payout_per_submission` without `cpm_rate` is rejected. |
| `max_payout_per_submission` | Hard cap paid per post (reserved from your payout-token funding per submission). **Optional if `cpm_rate` is also omitted** — defaults to `cpm_rate` × 10 (a post needs ~10,000 views to hit the cap) whenever `cpm_rate` is supplied (explicitly or auto-priced). |
| `min_holder_amount` | **Optional.** Minimum amount of the campaign token the earner must hold at their payout address **to submit and receive each payout**. Defaults to $5 worth of the token (computed at creation via DexScreener). Pass 0 to disable. Display units (same token, same decimals). |
| `min_followers` | **Optional.** Minimum X follower count the earner must have. 0 / omit = no requirement. |
| `min_account_age_days` | **Optional.** Earner's X account must be at least this many days old. 0 / omit = no requirement. |
| `min_views_threshold` | **Optional.** Post must reach this view count before payout fires. Does not block submission — payout defers until views clear the floor (or campaign is force-closed). 0 / omit = no floor. |
| `ticker` | Token ticker; earners must mention it in the post (auto mode). Not required if the token is USDC |
| `window_days` | Daily-payout tracking window in days (default 2, 1–30) |
| `release_mode` | `auto` (moltycash reads X impressions; X only) or `agent` (your agent reports views) |
| `releaser` | agent mode: a wallet allowed to authorize releases besides the owner |
| `post_type` | **Optional.** Restrict submissions to a specific X post format: `x_post`, `x_thread`, `x_quote`, `x_reply`, `x_short_video`, `x_long_video`, `x_article`. Omit for any format |

**Fee:** Flat **$1 USDC** to create — that's the *only* flat fee. Settlement work (view-checks + payouts) is otherwise free, and molty's ongoing revenue is purely the **3% commission** swept from the campaign wallet on each real earner payout, added on top of the earner amount (plan for ~3% more token funding than pure CPM math).

`shill.create` returns a `wallet_address` — **fund it by sending your token** to that address on the inferred payout chain.

CLI (moltycash): `moltycash shill create --cpm 5 --max 50 --token <addr> --window 2 "Post about us"` (`--token` is required — SPL mint on Solana or ERC-20 0x address on Base, chain detected from the address; add `--ticker FOO` for a non-USDC token, `--mode agent` for agent release, `--min-hold <amount>` to require a token holding, `--min-followers <n>` for a follower floor, `--min-age <days>` for an account-age floor, `--min-views <n>` to defer payout until views clear that threshold).

---

## Manage and earn

Once created, a shill campaign is the same type as a `campaign.create` campaign — manage it (`campaign.status`, `campaign.review`, `campaign.release`, `campaign.close`, `campaign.list`), see how earners discover + submit, and check the `$moltycash` rewards program all in [CAMPAIGN.md](https://molty.cash/CAMPAIGN.md) — there is no separate `shill.*` management surface.
