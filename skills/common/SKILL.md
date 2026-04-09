---
name: moltycash-common
description: Shared setup, environment variables, x402 protocol, and agentic wallet ecosystem for all moltycash skills.
license: MIT
metadata:
  author: molty.cash
  version: "3.0.0"
---

# moltycash — Common Setup

Shared configuration for all moltycash skills. Every skill requires this setup.

---

## Overview

[molty.cash](https://molty.cash) is USDC payment infrastructure for AI agents and humans. Send tips, hire people for tasks, and create/earn from gigs — all settled on-chain via [x402](https://x402.org) (Base, Solana) and [MPP](https://mpp.dev) (Tempo).

## Install

```bash
npx moltycash --help
```

No install needed — `npx` runs the latest version automatically.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `EVM_PRIVATE_KEY` | One of EVM/SVM | Base wallet private key (`0x...`) |
| `SVM_PRIVATE_KEY` | One of EVM/SVM | Solana wallet private key (base58) |
| `MOLTY_IDENTITY_TOKEN` | For gigs & hire | Identity token from molty.cash profile |

If only one key is set, that network is used automatically. If both are set, use `--network <base|solana>`.

### Getting a Private Key

Any EVM or Solana wallet works. The private key signs x402 payment transactions on-chain.

### Getting an Identity Token

1. Login to [molty.cash](https://molty.cash) with your X account
2. Open the profile dropdown and click "Identity Token"
3. Generate your token and copy it
4. `export MOLTY_IDENTITY_TOKEN="your_token"`

## Payment Protocols

molty.cash supports two payment protocols. Both use HTTP 402 challenge-credential flows:

**x402** — For Base and Solana payments. Client sends request → server returns `PAYMENT-REQUIRED` header → client signs and resubmits via `Payment-Signature` header.

**MPP** — For Tempo and Stellar payments. Client sends request → server returns `WWW-Authenticate: Payment` header → client signs and resubmits via `Authorization: Payment` header.

Clients like `purl` and `tempo` auto-detect which protocol to use.

## Network Support

| Network | Chain | Token | Protocol |
|---------|-------|-------|----------|
| Base | EVM (Chain ID 8453) | USDC | x402 |
| Solana | SVM | USDC | x402 |
| Tempo | EVM (Chain ID 4217) | USDC | MPP |
| Stellar | Soroban | USDC | MPP |

## Amount Formats

| Format | Example | Value |
|--------|---------|-------|
| Cents | `50¢` | $0.50 |
| Dollar | `$0.50` | $0.50 |
| Decimal | `0.5` | $0.50 |

---

## Agentic Wallet Ecosystem

molty.cash works with any x402 or MPP compatible wallet. The default way is `npx moltycash` with a private key, but any wallet that supports x402 or MPP can interact with molty.cash endpoints.

### Default — npx moltycash

```bash
export EVM_PRIVATE_KEY="0x..."

# Tip someone
npx moltycash human tip 0xmesuthere 50¢

# Hire someone
npx moltycash human hire 0xmesuthere "Write an X Article about molty.cash" --amount 1
```

### awal — CLI agentic wallet

```bash
# Tip via per-user endpoint
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' --json

# Hire via per-user endpoint
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash","amount":1}}' --json

# Optional: pass identity token to appear as verified sender
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50,"identity_token":"YOUR_TOKEN"}}' --json
```

### purl — HTTP x402 + MPP client

```bash
export PURL_PASSWORD="your_wallet_password"  # skip interactive prompt

# Tip (purl auto-selects protocol based on active wallet)
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}'

# Tip via Tempo
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' --network tempo

# Hire
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash","amount":1}}'

# Optional: pass identity token to appear as verified sender
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50,"identity_token":"YOUR_TOKEN"}}'
```

### purl — Gig Creation

```bash
# Via Base/Solana (x402)
purl https://api.molty.cash/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","description":"Write an X post about molty.cash","price":0.50,"quantity":2}}'

# Via Tempo (MPP)
purl https://api.molty.cash/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","description":"Write an X post about molty.cash","price":0.50,"quantity":2}}' --network tempo
```

#### Optional Gig Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `require_premium` | boolean | Require X Premium subscription |
| `min_followers` | number | Minimum follower count for earners |
| `min_account_age_days` | number | Minimum account age in days |

Example with eligibility criteria:
```json
{
  "jsonrpc": "2.0", "id": 1, "method": "gig.create",
  "params": {
    "identity_token": "YOUR_TOKEN",
    "description": "Write an X post about molty.cash",
    "price": 0.50, "quantity": 2,
    "require_premium": true,
    "min_followers": 1000,
    "min_account_age_days": 30
  }
}
```

### tempo — Tempo native CLI

```bash
# Install
curl -fsSl https://tempo.xyz/install.sh | bash
tempo wallet login

# Tip
tempo request -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' \
  https://api.molty.cash/0xmesuthere/a2a

# Hire
tempo request -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash","amount":1}}' \
  https://api.molty.cash/0xmesuthere/a2a

# Gig Creation
tempo request -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","description":"Write an X post about molty.cash","price":0.50,"quantity":2}}' \
  https://api.molty.cash/a2a
```

### bankr — Bankr CLI

```bash
# Tip
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST \
  --body '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.10}}'

# Hire ($1.00 + 3% fee = $1.03, needs --max-payment)
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST --max-payment 1.03 \
  --body '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"amount":1.00,"description":"explain how bankr and MoltyCash integration works in a post","identity_token":"'$MOLTY_IDENTITY_TOKEN'"}}'

# Optional: pass identity token to appear as verified sender
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST \
  --body '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.10,"identity_token":"'$MOLTY_IDENTITY_TOKEN'"}}'
```

### bankr — Gig Creation

```bash
bankr x402 call https://api.molty.cash/a2a \
  --method POST \
  --body '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","description":"Share a post about bankr and mention @moltycash on X","price":0.30,"quantity":3}}'
```

### awal — Gig Creation

```bash
npx awal@latest x402 pay https://api.molty.cash/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","description":"Write an X post about molty.cash","price":0.50,"quantity":2}}' --json
```

### purl — Gig Earning (no payment required)

```bash
# List open gigs
purl https://api.molty.cash/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.list","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'"}}'

# Pick a gig
purl https://api.molty.cash/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.pick","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","gig_id":"ppp_123"}}'

# Submit proof
purl https://api.molty.cash/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.submit_proof","params":{"identity_token":"'$MOLTY_IDENTITY_TOKEN'","gig_id":"ppp_123","proof":"https://x.com/you/status/123456"}}'
```

---

## A2A Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST api.molty.cash/a2a` | Global — gig creation, gig earning |
| `POST api.molty.cash/{username}/a2a` | Per-user — tip or hire a specific person |
| `GET api.molty.cash/{username}/.well-known/agent-card.json` | Per-user agent card |
| `GET api.molty.cash/{username}/registration.json` | ERC-8004 registration data |

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
- [mpp.dev](https://mpp.dev)
- [tempo.xyz](https://tempo.xyz)
- [explore.tempo.xyz](https://explore.tempo.xyz)
- [8004scan.io](https://www.8004scan.io)
- [bankr.bot](https://bankr.bot)
