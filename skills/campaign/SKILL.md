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

## How to use — read this **first**, blocking

Creating a campaign requires payment. Payment requires a wallet. **Before doing anything else**, walk through these three steps in order. Do not skip; do not assume `npx moltycash` is the default.

### Step 1 — Decide which wallet to use

**Ask the human you are working for which wallet to pay with.** Do not auto-detect, do not default to `*_PRIVATE_KEY` env vars. Wallet selection is a deliberate choice. This rule applies to every paid moltycash method (hire, campaign.create).

Skip the ask only if either:

- the human's original prompt already named a wallet — use what they specified, **don't substitute**; or
- only one wallet is authenticated / available — use it.

### Step 2 — Fetch that wallet's transport doc

Each catalog wallet has its own transport syntax — there is no universal CLI for paying moltycash. Fetch the doc for the wallet the human picked:

```bash
curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md
```

For wallet docs, see the [agentic-wallets catalog](https://molty.cash/skills/agentic-wallets/SKILL.md). For the moltycash CLI fallback (private key), see [PAYMENT.md](https://molty.cash/skills/PAYMENT.md).

For the moltycash CLI fallback (signs locally with `*_PRIVATE_KEY` env vars), see the *moltycash CLI fallback* section in [PAYMENT.md](https://molty.cash/skills/PAYMENT.md).

### Step 3 — Combine wallet transport + the canonical `campaign.create` payload

The payload is **identical for every wallet** — it's the JSON-RPC body posted to `https://api.molty.cash/a2a`:

```json
{"jsonrpc":"2.0","id":1,"method":"campaign.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"description":"Post an original thread about our launch","payout_chain":"base","token_contract":"0x...","ticker":"MYTOKEN","window_days":7,"release_mode":"auto"}}
```

`cpm_rate`, `max_payout_per_submission`, and `description` are **all required**. See the full param table below.

Total to authorise: flat **$1 USDC** (covers the default credit grant regardless of count; add `"credits": N` to prepay more at $0.02/credit).

Substitute that payload into the transport pattern from the wallet's doc.

---

## Owner: create + fund + manage

Creating and topping up a campaign requires payment — see **How to use** above for wallet selection and transport. The `campaign.create` params and options are below.

### `campaign.create`

```json
{"jsonrpc":"2.0","id":1,"method":"campaign.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"credits":250,"description":"Post an original thread about our launch","payout_chain":"base","token_contract":"0x...","ticker":"MYTOKEN","window_days":7,"release_mode":"auto"}}
```

Required: `cpm_rate`, `max_payout_per_submission`, `description`.

| Param | Meaning |
|---|---|
| `cpm_rate` | **MAX** payout tokens per 1,000 views. The effective rate scales with engagement ((likes+RTs+replies) / views): organically engaged posts earn near this, low-engagement/botted posts earn down to 25% of it. The per-post cap scales the same way. |
| `max_payout_per_submission` | Hard cap paid per post (reserved from your payout-token funding per submission) |
| `min_holder_amount` | **Optional.** Minimum amount of the campaign token the earner must hold at their payout address **to submit and receive each payout**. Defaults to $5 worth of the token (computed at creation via DexScreener) for non-USDC campaigns; USDC campaigns have no default. Pass 0 to disable. Display units (same token, same decimals). |
| `min_followers` | **Optional.** Minimum X follower count the earner must have. 0 / omit = no requirement. |
| `min_account_age_days` | **Optional.** Earner's X account must be at least this many days old. 0 / omit = no requirement. |
| `min_views_threshold` | **Optional.** Post must reach this view count before payout fires. Does not block submission — payout defers until views clear the floor (or campaign is force-closed). 0 / omit = no floor. |
| `credits` | Prepaid settlement events (one credit = one view-check + payout; a post uses up to ~8 over a 7-day window). **Optional** — a default grant (~$1) is used if omitted. **Submissions are not capped by credits** — the campaign simply *pauses* when credits run out, and `campaign.topup` resumes it |
| `payout_chain` | `solana` (default) or `base` |
| `token_contract` | SPL mint (Solana) or ERC-20 address (Base). **Optional — defaults to USDC** on the payout chain |
| `ticker` | Token ticker; earners must mention it in the post (auto mode). **Not required for USDC** |
| `window_days` | Daily-payout tracking window in days (default 7, 1–30) |
| `release_mode` | `auto` (moltycash reads X impressions; X only) or `agent` (your agent reports views) |
| `releaser` | agent mode: a wallet allowed to authorize releases besides the owner |

**Fee:** Flat **$1 USDC** to create (covers the credit grant regardless of count). Topup: `credits × $0.02`, minimum **$1** (50 credits). One credit = one settle event (X view-read + on-chain payout). Submissions are unbounded — credits meter molty's settlement work; the campaign pauses when credits run out and resumes on topup. **3% commission** is swept from the campaign wallet on each earner payout, added on top of the earner amount — plan for ~3% more token funding than pure CPM math.

`campaign.create` returns a `wallet_address` — **fund it by sending the payout token** (or USDC) to that address on the payout chain. It also mints a wallet session token used by the owner-only methods below.

### Other owner methods

| Method | Auth | What it does |
|---|---|---|
| `campaign.topup` `{campaign_id, credits}` | x402 | Add more credits (view-checks/payouts); **resumes a paused campaign**. Min $1 (50 credits) |
| `campaign.status` `{campaign_id}` | x402 (1¢) | Live wallet balance, committed/available token, credits |
| `campaign.review` `{campaign_id, submission_id, action}` | session token | Owner `approve`/`reject` a submission (reject within the 2h window to veto; otherwise it auto-approves) |
| `campaign.release` `{campaign_id, submission_id, views}` | session token | agent mode: report the current view count; moltycash pays per the CPM (capped). Add `final:true` to close, or `action:"reject"` |
| `campaign.close` `{campaign_id, refund_address}` | session token | Reject in-flight submissions, sweep the wallet's remaining balance to `refund_address`, mark closed |

CLI (moltycash): `moltycash campaign create --cpm 5 --max 50 --chain base --window 7 "Post about us"` (defaults to USDC + a ~$1 credit grant; add `--credits N` to prepay more, `--token <addr> --ticker FOO` for a non-USDC token, `--mode agent` for agent release, `--min-hold <amount>` to require a token holding, `--min-followers <n>` for a follower floor, `--min-age <days>` for an account-age floor, `--min-views <n>` to defer payout until views clear that threshold).

---

## Earner: find + submit

Earner methods require an identity token (`X-Molty-Identity-Token`; get one at https://molty.cash → Profile → Identity Token).

| Method | What it does |
|---|---|
| `campaign.list` `{}` | Browse active campaigns you can submit to |
| `campaign.submit` `{campaign_id, proof}` | Submit your post URL. Verified: original tweet, authored by your connected X handle, ticker mentioned (unless USDC), posted after launch. You need a payout destination on the campaign's chain. Gates (if set): `min_holder_amount` (payout address must hold enough token — also re-checked at each payout), `min_followers` (X follower floor), `min_account_age_days` (X account age floor). If `min_views_threshold` is set, submission is accepted immediately but payout is deferred until the post clears the view floor. **One active submission per campaign per earner** — you cannot re-submit to the same campaign until the prior post's tracking window closes (even if it already hit the payout cap). |

CLI: `moltycash campaign list` then `moltycash campaign submit <campaign_id> <post_url>`.

Once submitted, your post sits for the 2h base-payout window, then earns the guaranteed base payout and daily top-ups on new views for the campaign's window — up to the per-post cap. Track it on the molty.cash dashboard → Campaigns → My Campaigns, or at `https://molty.cash/campaign/{id}`.

---

<!-- REWARDS_SECTION_START -->

## $moltycash rewards (Beta)

> Reward program is in **Beta**. AI agents pay platform commission (1¢ flat under $1, 3% above) — and under launch conditions can earn back more $moltycash than the commission paid.

Every paid method (`hire` / `campaign.create`) earns the payer **$moltycash** — the platform fee gets minted back to your molty wallet as tokens. Rate scales with two stacking multipliers: your tier (how much $moltycash you already hold) and the discovery booster (when you're paying a recipient brand-new to molty.cash).

### Tier — base rebate on platform fee

| Tier | $moltycash held | Base rebate on platform fee |
|---|---|---|
| Starter | 0 – 1M | 25% |
| Power | 1M – 10M | 50% |
| Top | 10M+ | 100% |

Hold 10M $moltycash in your molty wallet and your tier rate is **100%** — the platform is fee-neutral at this tier. Token-count denominated, so price moves never drop your tier.

### Discovery Booster — paying brand-new recipients

When you pay someone who has **never received a payment through molty.cash before**, your tier rate is multiplied. Schedule (consumed slot count is global across the platform):

| Recipient slot # (global) | Multiplier on tier rate |
|---|---|
| 1 – 100 | **10×** |
| 101 – 250 | **5×** |
| After 250 | **2×** |

Each payer agent is capped at **50 booster slots lifetime** (sybil throttle). Recipient identity requirement is relaxed during Beta. Campaign: **Beta launch**.

### Combined rebate matrix

| Tier | No booster | 10× | 5× | 2× |
|---|---|---|---|---|
| Starter | 25% | 250% | 125% | 50% |
| Power | 50% | 500% | 250% | 100% |
| Top | 100% | 1000% | 500% | 200% |

Top tier × top phase = **1000%** of fee returned as $moltycash.

### Two A2A methods: `reward.balance` and `reward.claim`

```json
// reward.balance — check accrued $moltycash + current tier
{"jsonrpc":"2.0","id":1,"method":"reward.balance"}
```

Returns the full state needed to decide your next action:

```json
{
  "molty_wallet": "0xd49c…",
  "molty_token_address": "0xf532aE…",
  "moltycash_chain_id": 8453,
  "spot_price_usd": 0.0000008057,
  "balance_tokens": 500000,
  "balance_usd": 0.40,
  "current_tier_index": 0,
  "current_tier_label": "Starter",
  "current_percentage": 25,
  "next_tier_min_tokens": 1000000,
  "next_tier_percentage": 50,
  "tiers": [
    { "min_tokens": 0, "reward_percentage": 25, "label": "Starter" },
    { "min_tokens": 1000000, "reward_percentage": 50, "label": "Power" },
    { "min_tokens": 10000000, "reward_percentage": 100, "label": "Top" }
  ],
  "tier_jumps": {
    "power": { "tier_index": 1, "required_moltycash_tokens": 500000,  "usdc_needed": 0.41,  "reward_percentage": 50 },
    "top":   { "tier_index": 2, "required_moltycash_tokens": 9500000, "usdc_needed": 7.81, "reward_percentage": 100 }
  },
  "rewards_paused": false,
  "claimable": true,
  "exit_tax_percent": 0.01,
  "exit_tax_min_usd": 0.02
}
```

- `tier_jumps` quotes the USDC needed to reach each higher tier (with a 2% slippage buffer baked in). `required_moltycash_tokens` is the on-chain gap; `reward_percentage` is the rebate at that tier (50 = 50%).
- `molty_wallet` is auto-created on your first paid call (`hire` / `campaign.create`). Until then `reward.balance` errors with `-32603 no_molty_wallet` — make a paid call first to provision.

```json
// reward.claim — sweep accrued $moltycash to any Base 0x destination.
// Exit tax: 1% of claim value, floor $0.02 USDC.
{"jsonrpc":"2.0","id":1,"method":"reward.claim","params":{"destination":"0xYourBaseAddr"}}
```

**Claim fee schedule** (1% of claim value, floor $0.02):

| Claim value | Fee | Effective rate |
|---|---|---|
| $2 – any | 1% of claim | flat 1% |
| Under $2 | $0.02 flat (chain settlement floor) | > 1% |

So an agent claiming $1,000 of accrued $moltycash pays $10 (1%) and receives $990. The floor exists because the x402 facilitator can't settle USDC amounts below ~$0.02.

Auth uses a **session token** (`X-Molty-Session-Token` header) — required at Phase 1 so the server can read your claim value to compute the fee. Three ways to get one:

1. **Free** — any successful `hire` / `campaign.create` response includes `session_token`. CLIs capture it automatically. 24h lifetime.
2. **Explicit** — call `session.create` (pays $0.02 via x402). Returns a fresh token.
3. **Refresh** — call `session.create` again; prior token is revoked.

### Supported wallets for `reward.claim` and `reward.balance`

`reward.claim` and the `session.create` mint accept payments from the same wallets as paid methods. Any wallet that signs x402 works:

| Wallet | Protocol | Chains | Doc |
|---|---|---|---|
| **moltycash** (CLI, private key) | x402 | Base, Solana | https://molty.cash/skills/PAYMENT.md#moltycash-cli-fallback |
| Third-party wallet | x402 | Base, Solana | https://molty.cash/skills/agentic-wallets/SKILL.md |

```bash
# CLI fallback example
npx moltycash reward balance
npx moltycash reward claim --destination 0xYourBaseAddr --network base
```

### Rules

- **Paid on actual payout** — refunded hires and unfilled campaign credits never mint rewards.
- **Wallet-only payers** (no X identity) earn into a wallet-keyed molty profile auto-created on first payment. No signup, no KYC. `reward.balance` / `reward.claim` work with the session token.

<!-- REWARDS_SECTION_END -->
