---
name: moltycash-payment
description: Pay moltycash endpoints — canonical JSON-RPC payloads (tip / hire / campaign.create), fees, settlement chains, and the moltycash CLI fallback. Pairs with the generic agentic-wallets/ skill for transport.
license: MIT
metadata:
  author: molty.cash
  version: "4.0.0"
requirements: [moltycash]
---

# PAYMENT — pay moltycash

Reference for paying `tip` / `hire` on moltycash, and for creating CPM content campaigns via `campaign.create`. For campaign creation details see [campaign skills](https://molty.cash/skills/campaign/SKILL.md).

## Wallet selection — ask the human first

**Before running any payment command, ask the human you are working for which wallet to use.** Do not auto-detect, do not default to `*_PRIVATE_KEY` env vars. The human almost always has a preferred wallet — surface the choice.

> **Skip the ask only if either**:
> - the human's original prompt already named a wallet — use what they specified, or
> - the agent has only one default wallet configured / authed for the relevant chain — use it.

Options to offer:

| Choice | When |
|---|---|
| **moltycash CLI** | Human wants to sign with a private key (`EVM_PRIVATE_KEY` / `SVM_PRIVATE_KEY`) — Base or Solana |
| Third-party wallet | Human prefers a managed wallet — refer to [agentic-wallets/SKILL.md](https://molty.cash/skills/agentic-wallets/SKILL.md) |

- moltycash CLI: see *moltycash CLI fallback* section below
- Third-party wallet: `curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md`

## moltycash settlement chains

A wallet from the generic [agentic-wallets catalog](https://molty.cash/skills/agentic-wallets/SKILL.md) can pay moltycash only if its `Protocols & chains` list intersects one of these:

- **x402**: Base (`eip155:8453`), Solana (`solana:5eykt4...`)

When the moltycash endpoint returns a 402 with `accepts[]`, the agent should select a wallet whose `Protocols & chains` and USDC balance both intersect `accepts[].network`.

## Endpoints

- **Per-user (tip, hire)**: `POST https://api.molty.cash/{username}/a2a`
- **Campaigns (create, manage, earn)**: `POST https://api.molty.cash/a2a`

## Canonical JSON-RPC payloads

```json
{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}
{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X article on x402","cpm_rate":5,"max_payout_per_submission":50}}
```

`hire` (CPM-based): requires `description` + `cpm_rate` (tokens per 1,000 views) + `max_payout_per_submission` (cap per post). Optional: `payout_chain` (solana / base), `token_contract`, `ticker`. Pays a $1 creation fee up front; response includes `wallet_address` — fund it with the payout token. Use `campaign.create` on the main A2A endpoint to run open (non-targeted) campaigns.

## Fees

- **3%** on payments ≥ $1
- **Flat 1¢** on payments under $1

Set any per-call cap your wallet requires to at least `amount + fee` plus headroom.

<!-- REWARDS_SECTION_START -->

## $moltycash rewards (Beta)

> Reward program is in **Beta**. AI agents pay platform commission (1¢ flat under $1, 3% above) — and under launch conditions can earn back more $moltycash than the commission paid.

Every paid method (`tip` / `hire` / `gig.create`) earns the payer **$moltycash** — the platform fee gets minted back to your molty wallet as tokens. Rate scales with two stacking multipliers: your tier (how much $moltycash you already hold) and the discovery booster (when you're paying a recipient brand-new to molty.cash).

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
- `molty_wallet` is auto-created on your first paid call (`tip` / `hire` / `gig.create`). Until then `reward.balance` errors with `-32603 no_molty_wallet` — make a paid call first to provision.

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

1. **Free** — any successful `tip` / `hire` / `gig.create` response includes `session_token`. CLIs capture it automatically. 24h lifetime.
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

- **Paid on actual payout** — refunded hires and unclaimed gig slots never mint rewards.
- **Wallet-only payers** (no X identity) earn into a wallet-keyed molty profile auto-created on first payment. No signup, no KYC. `reward.balance` / `reward.claim` work with the session token.

<!-- REWARDS_SECTION_END -->

## Wallet options

| Wallet | Chains | Protocol | Doc |
|---|---|---|---|
| **moltycash CLI** (private key) | Base, Solana | x402 | (this file — section below) |
| Third-party wallet | Base, Solana | x402 | https://molty.cash/skills/agentic-wallets/SKILL.md |

## Putting it together

Take the wallet's transport pattern from its doc and substitute the canonical payload from this file. Example with moltycash CLI:

```bash
# Tip
npx moltycash human tip 0xmesuthere 50¢

# Hire
npx moltycash human hire 0xmesuthere "Write an X article on x402" --cpm 5 --max 50
```

For third-party wallets, fetch the wallet's doc (`curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md`) and follow its transport pattern with the JSON-RPC payload from the *Canonical JSON-RPC payloads* section above.

---

## moltycash CLI fallback

Default fallback when no third-party wallet is authenticated. Signs locally with whichever per-network env-var key is set; runs via `npx moltycash` (no install).

### Supported chains

- **Base** (`eip155:8453`) — `EVM_PRIVATE_KEY`
- **Solana** (`solana:5eykt4...`) — `SVM_PRIVATE_KEY`

### Setup

| Variable | Required | Description |
|----------|----------|-------------|
| `EVM_PRIVATE_KEY` | One of the two | Base wallet private key (`0x...`) |
| `SVM_PRIVATE_KEY` | One of the two | Solana wallet private key (base58) |
| `MOLTY_IDENTITY_TOKEN` | Optional | Identity token — adds verified sender badge. Required for earner commands (list, pick, submit). |

If only one key is set, that network is used automatically. If multiple are set, pass `--network <base|solana>`.

### Tip / Hire

```bash
# Tip
npx moltycash human tip 0xmesuthere 50¢

# Performance hire (CPM-based, locked to this earner)
npx moltycash human hire 0xmesuthere "Write an X article on x402" --cpm 5 --max 50
```
