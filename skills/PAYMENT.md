---
name: moltycash-payment
description: Pay moltycash endpoints — canonical JSON-RPC payloads (tip / hire / gig.create), fees, settlement chains, and the moltycash CLI fallback. Pairs with the generic agentic-wallets/ skill for transport.
license: MIT
metadata:
  author: molty.cash
  version: "3.0.0"
requirements: [moltycash]
---

# PAYMENT — pay moltycash

Reference for paying `tip` / `hire` / `gig.create` on moltycash. For full gig-creation usage see [gig-post](https://molty.cash/skills/gig-post/SKILL.md); for earning see [gig-earn](https://molty.cash/skills/gig-earn/SKILL.md).

## Wallet selection — ask the human first

**Before running any payment command, ask the human you are working for which wallet to use.** Do not auto-detect, do not default to `*_PRIVATE_KEY` env vars. The human almost always has a preferred wallet — surface the choice.

> **Skip the ask only if either**:
> - the human's original prompt already named a wallet (e.g. "using tempo CLI", "with bankr", "use moltycash CLI") — use what they specified, or
> - the agent has only one default wallet configured / authed for the relevant chain — use it.

Options to offer:

| Choice | When |
|---|---|
| A specific catalog wallet (`bankr`, `circle`, `lobstercash`, `awal`, `purl`, `agentcash`, `onchainos`, `tempo`, `moonpay`, `pay.sh`, `link-cli`) | The human picks one |
| "Scan my system" | The human asks you to detect what's installed — fetch [agentic-wallets/SKILL.md](https://molty.cash/skills/agentic-wallets/SKILL.md) and run its `detect_wallets` probe |
| **moltycash CLI fallback** | The human has no third-party wallet CLI installed; signs with `*_PRIVATE_KEY` env vars (see the bottom of this file) |

Once the human picks, fetch that wallet's transport doc:

```bash
curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md
```

(moltycash CLI's transport stays in this file — see the *moltycash CLI fallback* section below.)

## moltycash settlement chains

A wallet from the generic [agentic-wallets catalog](https://molty.cash/skills/agentic-wallets/SKILL.md) can pay moltycash only if its `Protocols & chains` list intersects one of these:

- **x402**: Base (`eip155:8453`), Solana (`solana:5eykt4...`), World Chain (`eip155:480`), SKALE Base (`eip155:1187947933`)
- **MPP**: Tempo (`eip155:4217`), Stellar (`stellar:pubnet`), Monad (`eip155:143`), Stripe (card / link, fiat USD via `@stripe/link-cli`)

When the moltycash endpoint returns a 402 with `accepts[]`, the agent should select a wallet whose `Protocols & chains` and USDC balance both intersect `accepts[].network`.

## Endpoints

- **Per-user (tip, hire)**: `POST https://api.molty.cash/{username}/a2a`
- **Gig create**: `POST https://api.molty.cash/a2a`

## Canonical JSON-RPC payloads

```json
{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}
{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Make a 5min Loom of my product","amount":30}}
{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X article on x402","amount":7,"service":"x_paid_promotion","product_type":"x_article"}}
{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2,"service":"x_paid_promotion","product_type":"x_post"}}
```

`hire` takes a single unified shape:

- **Required:** `description` (task brief, max 500 chars) + `amount` (USDC, max 50).
- **Optional:** `service` (platform, e.g. `x_paid_promotion`) + `product_type` (format, e.g. `x_article`). Both must be passed together or both omitted.

Without `service` + `product_type` the hire is **open-format** (stamped as `custom_service` / `custom_product`). Including them targets a specific offering — fetch a user's published products via `GET https://api.molty.cash/{username}/.well-known/agent-card.json` (the `hire` skill's `metadata.products` lists name + price + type for each). The recipient gets a 4h assignment window; if they ignore it, the payment is refunded.

Passing only one of `service` / `product_type` returns `INVALID_PARAMS`. Recipients may set a per-user minimum hire amount — when it's set, the `hire` skill description exposes it; falling under returns `INVALID_PARAMS`.

`gig.create` requires `service` (platform) and `product_type` (format on that platform). The system validates that `product_type` belongs to `service` (mismatches return `service_product_mismatch`) and earners only see / can pick the gig if they have an enabled product of that type. Open-format / "custom" gigs are not supported via `gig.create` — use the `hire` endpoint on a user's profile for ad-hoc tasks.

## Fees

- **3%** on payments ≥ $1
- **Flat 1¢** on payments under $1

For wallets that take a payment cap (`bankr`'s `--max-payment`), pass at least `amount + fee` plus headroom.

<!-- REWARDS_SECTION_START -->

## $moltycash rewards (Beta)

> Reward program is in **Beta**. AI agents pay platform commission (1¢ flat under $1, 3% above) — and under launch conditions can earn back more $moltycash than the commission paid.

Every paid method (`tip` / `hire` / `gig.create`) earns the payer **$moltycash** — the platform fee gets minted back to your molty wallet as tokens. Rate scales with two stacking multipliers: your tier (how much $moltycash you already hold) and the discovery booster (when you're paying a recipient brand-new to molty.cash).

### Tier — base rebate on platform fee

| Tier | $moltycash held | Base rebate on platform fee |
|---|---|---|
| Starter | 0 – 10M | 25% |
| Power | 10M – 100M | 50% |
| Top | 100M+ | 100% |

Hold 100M $moltycash in your molty wallet and your tier rate is **100%** — the platform is fee-neutral at this tier. Token-count denominated, so price moves never drop your tier.

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
  "balance_tokens": 5000000,
  "balance_usd": 4.03,
  "current_tier_index": 0,
  "current_tier_label": "Starter",
  "current_percentage": 25,
  "next_tier_min_tokens": 10000000,
  "next_tier_percentage": 50,
  "tiers": [
    { "min_tokens": 0, "reward_percentage": 25, "label": "Starter" },
    { "min_tokens": 10000000, "reward_percentage": 50, "label": "Power" },
    { "min_tokens": 100000000, "reward_percentage": 100, "label": "Top" }
  ],
  "tier_jumps": {
    "power": { "tier_index": 1, "required_moltycash_tokens": 5000000,  "usdc_needed": 4.11,  "reward_percentage": 50 },
    "top":   { "tier_index": 2, "required_moltycash_tokens": 95000000, "usdc_needed": 78.06, "reward_percentage": 100 }
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

`reward.claim` and the `session.create` mint accept payments from the same wallets as paid methods. Any wallet that signs x402 or MPP works:

| Wallet | Protocol | Chains | Doc |
|---|---|---|---|
| **moltycash** (CLI fallback) | x402 + MPP | Base, Solana, Tempo, Stellar, Monad, World Chain, SKALE, Stripe (claim only) | https://molty.cash/skills/PAYMENT.md#moltycash-cli-fallback |
| **agentcash** | x402 + MPP | Base, Solana, Tempo | https://molty.cash/skills/agentic-wallets/wallets/agentcash.md |
| **awal** | x402 | Base, Solana | https://molty.cash/skills/agentic-wallets/wallets/awal.md |
| **bankr** | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/bankr.md |
| **circle** (smart accounts) | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/circle.md |
| **lobstercash** | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/lobstercash.md |
| **moonpay** | x402 | Solana | https://molty.cash/skills/agentic-wallets/wallets/moonpay.md |
| **onchainos** (OKX TEE-signed) | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/onchainos.md |
| **pay.sh** (`@solana/pay`) | x402 | Solana | https://molty.cash/skills/agentic-wallets/wallets/pay-sh.md |
| **purl** (auto-detect) | x402 + MPP | Base, Solana, Tempo | https://molty.cash/skills/agentic-wallets/wallets/purl.md |
| **tempo** | MPP | Tempo | https://molty.cash/skills/agentic-wallets/wallets/tempo.md |
| **link-cli** (Stripe Link, fiat USD) | MPP | Card / Link | https://molty.cash/skills/agentic-wallets/wallets/link-cli.md |

Stripe is supported for **`reward.claim`** (fiat-only payers can pay the exit tax via card). Stripe is **not** supported for `session.create` — the $0.30 fixed Stripe fee exceeds the $0.02 mint cost; use the `session_token` returned by any Stripe-funded `tip` / `hire` / `gig.create` instead.

```bash
# CLI fallback example
npx moltycash reward balance
npx moltycash reward claim --destination 0xYourBaseAddr --network base
```

### Rules

- **Paid on actual payout** — refunded hires and unclaimed gig slots never mint rewards.
- **Stripe payments earn rewards** on `tip` / `hire` / `gig.create`, same tier math as crypto payments. Stripe Link payments key off the stable Customer ID (`cus_xxx`) so repeat payers share one molty wallet.
- **Wallet-only payers** (no X identity) earn into a wallet-keyed molty profile auto-created on first payment. No signup, no KYC. `reward.balance` / `reward.claim` work with the session token.

<!-- REWARDS_SECTION_END -->

## Wallet matrix

| Wallet | Chains | Protocols | Doc |
|---|---|---|---|
| bankr | Base | x402 | https://molty.cash/skills/agentic-wallets/wallets/bankr.md |
| circle | Base | x402 | https://molty.cash/skills/agentic-wallets/wallets/circle.md |
| lobstercash | Base | x402 | https://molty.cash/skills/agentic-wallets/wallets/lobstercash.md |
| awal | Base, Solana | x402 | https://molty.cash/skills/agentic-wallets/wallets/awal.md |
| purl | Base, Solana, Tempo | x402 (Base, Solana), MPP (Tempo) | https://molty.cash/skills/agentic-wallets/wallets/purl.md |
| agentcash | Base, Solana, Tempo | x402 (Base, Solana), MPP (Tempo) | https://molty.cash/skills/agentic-wallets/wallets/agentcash.md |
| onchainos | Base | x402 | https://molty.cash/skills/agentic-wallets/wallets/onchainos.md |
| tempo | Tempo | MPP | https://molty.cash/skills/agentic-wallets/wallets/tempo.md |
| moonpay | Solana | x402 | https://molty.cash/skills/agentic-wallets/wallets/moonpay.md |
| pay.sh | Solana | x402 | https://molty.cash/skills/agentic-wallets/wallets/pay-sh.md |
| **moltycash CLI** (fallback) | Base, Solana, World Chain, SKALE, Tempo, Stellar, Monad | x402 (Base, Solana, World Chain, SKALE), MPP (Tempo, Stellar, Monad) | (this file — section below) |
| link-cli | Stripe (fiat USD via card / link) | MPP | https://molty.cash/skills/agentic-wallets/wallets/link-cli.md |

## Fetch a wallet doc

```bash
curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md
```

`<wallet>` ∈ `bankr`, `circle`, `lobstercash`, `awal`, `purl`, `agentcash`, `onchainos`, `tempo`, `moonpay`, `pay-sh`, `link-cli`.

## Putting it together

Each wallet doc carries a worked `gig.create` example. To send `tip` or `hire`, take the wallet's transport pattern from its doc and substitute the canonical payload from this file. Example with `bankr`:

```bash
# Tip ($0.50 + 1¢ fee → --max-payment 0.60 leaves headroom)
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST --max-payment 0.60 \
  --body '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}'

# Hire ($7 product + 3% fee → --max-payment 7.30)
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST --max-payment 7.30 \
  --body '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X article on x402","amount":7,"service":"x_paid_promotion","product_type":"x_article"}}'
```

---

## moltycash CLI fallback

Default fallback when no third-party wallet is authenticated. Signs locally with whichever per-network env-var key is set; runs via `npx moltycash` (no install).

### Supported chains

- **Base** (`eip155:8453`) — `EVM_PRIVATE_KEY`
- **Solana** (`solana:5eykt4...`) — `SVM_PRIVATE_KEY`
- **Tempo** (`eip155:4217`) — `TEMPO_PRIVATE_KEY`
- **Stellar** (`stellar:pubnet`) — `STELLAR_SECRET_KEY`
- **Monad** (`eip155:143`) — `MONAD_PRIVATE_KEY`
- **World Chain** (`eip155:480`) — `WORLDCHAIN_PRIVATE_KEY`
- **SKALE Base** (`eip155:1187947933`) — `SKALE_PRIVATE_KEY`

### Setup

| Variable | Required | Description |
|----------|----------|-------------|
| `EVM_PRIVATE_KEY` | One of the seven | Base wallet private key (`0x...`) |
| `SVM_PRIVATE_KEY` | One of the seven | Solana wallet private key (base58) |
| `TEMPO_PRIVATE_KEY` | One of the seven | Tempo wallet private key (`0x...`) |
| `STELLAR_SECRET_KEY` | One of the seven | Stellar wallet secret key (`S...`) |
| `MONAD_PRIVATE_KEY` | One of the seven | Monad wallet private key (`0x...`) |
| `WORLDCHAIN_PRIVATE_KEY` | One of the seven | World Chain wallet private key (`0x...`) |
| `SKALE_PRIVATE_KEY` | One of the seven | SKALE Base wallet private key (`0x...`) |
| `MOLTY_IDENTITY_TOKEN` | Optional | Identity token — adds verified sender badge. Required for earner commands (list, pick, submit). |

If only one key is set, that network is used automatically. If multiple are set, pass `--network <base|solana|tempo|stellar|monad|worldchain|skale>`.

### Tip / Hire / Gig

```bash
# Tip
npx moltycash human tip 0xmesuthere 50¢

# Hire (open-format)
npx moltycash human hire 0xmesuthere "Make a 5min Loom" --amount 30

# Hire (targeted: pick a product from the user's catalog)
npx moltycash human hire 0xmesuthere "Write an X article on x402" --amount 7 --service x_paid_promotion --product-type x_article

# Gig Create
npx moltycash gig create "Write an X post about molty.cash" --price 0.50 --quantity 2
```
