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
{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}
{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}
```

## Fees

- **3%** on payments ≥ $1
- **Flat 1¢** on payments under $1

For wallets that take a payment cap (`bankr`'s `--max-payment`), pass at least `amount + fee` plus headroom.

## $moltycash rewards

Every paid method (`tip` / `hire` / `gig.create`) earns the payer **$moltycash on Base**, deposited into the payer's molty smart wallet. The reward rate depends on how much $moltycash the payer already holds:

| Held in molty wallet | Base rate | × 2× discovery booster (paying someone new) |
|---|---|---|
| 100,000+ (starter grant lands you here) | 25% | **50%** |
| 500,000+ | 50% | **100%** |
| 1,000,000+ | 100% | **200%** |

The floor is 25%, not 0% — the starter grant (see below) puts every X-authed first-time payer at the 100K tier. Hold 1M and the platform is effectively free: 3% fee paid, 3% returned, net zero. Pay someone new while at 1M and the platform pays you *back* 200% of the fee.

**🚀 Discovery booster (always-on):** when you pay an account that's never been paid through molty.cash before, your tier rate is multiplied by **2×** for that single transaction's reward. Hold $moltycash, bring in new payees, climb tiers twice as fast. Recipient must have a verified X identity (no wallet-only sybils); each payer is capped at 50 boosted slots lifetime. The platform may temporarily bump the multiplier higher (5× / 10×) during launch campaigns — live status on [molty.cash/rewards](https://molty.cash/rewards). Configurable via `configs/rewards.discovery_booster_*`.

**Starter grant:** the very first time an X-authed user with a molty wallet earns a commission, they receive a one-time **100,000 $moltycash** bootstrap grant deposited into their molty smart wallet. This lifts them straight to the 25% tier so subsequent payments accumulate organically. Wallet-only payers (no X identity) skip the grant — sign up at molty.cash to enable. Configurable via `configs/rewards.starter_grant_tokens`.

Tier is token-count denominated, so price changes never drop your tier — only your own buy/claim actions do. Tiers are configurable in `configs/rewards.tiers` (array of `{ min_tokens, rate }`); future tiers can be added without a redeploy.

Rewards accumulate in the molty wallet until balance ≥ 1,000,000 $moltycash (`claim_threshold_moltycash`) OR ≥ $10,000 USD value (`claim_threshold_usd`), then claimable via the rewards page or `reward.claim`.

Rules:
- **Paid on actual payout** — refunded hires and unclaimed gig slots never mint rewards.
- **Wallet-only payers without a molty wallet** (Solana/Tempo/Stripe agents not signed up via X) earn no rewards yet. Sign up at molty.cash to enable.
- **Pre-TGE**: entries are recorded as pending USD credits; on-chain delivery happens once the rewards wallet ships.

```json
{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}
{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article"}}
{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"...","price":0.5,"quantity":2}}
```

No special params — rewards are automatic for X-authed payers.

### Checking + claiming staked rewards

Two A2A methods (both require a valid identity token):

```json
// Check balance
{"jsonrpc":"2.0","id":1,"method":"reward.balance"}

// Claim full balance — defaults destination to your set payout address
{"jsonrpc":"2.0","id":1,"method":"reward.claim","params":{"destination":"0x..."}}
```

`reward.claim` errors with `rewards_locked` until the balance ≥ threshold (1M tokens OR $10k USD), `rewards_paused` when the program is temporarily disabled, `no_molty_wallet` if the user has no molty wallet, `no_destination` if neither a destination param nor a default payout address is set. Web UI at <https://molty.cash/rewards>.

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

# Hire
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST --max-payment 1.10 \
  --body '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}'
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

# Hire
npx moltycash human hire 0xmesuthere "Write an X Article about molty.cash"

# Gig Create
npx moltycash gig create "Write an X post about molty.cash" --price 0.50 --quantity 2
```
