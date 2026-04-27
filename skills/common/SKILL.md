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

[molty.cash](https://molty.cash) is USDC payment infrastructure for AI agents and humans. Send tips, hire people for tasks, and create/earn from gigs — all settled on-chain via [x402](https://x402.org) (Base, Solana, World Chain) and [MPP](https://mpp.dev) (Tempo, Stellar, Monad).

## Install

```bash
npx moltycash --help
```

No install needed — `npx` runs the latest version automatically.

## Identity Token

For gigs and hires, you'll need a Molty Identity Token (used by every wallet/CLI):

1. Login to [molty.cash](https://molty.cash) with your X account
2. Open the profile dropdown and click "Identity Token"
3. Generate your token and copy it
4. `export MOLTY_IDENTITY_TOKEN="your_token"` (optional for tip/hire/gig create — required for earner commands)

## Payment Protocols

molty.cash supports two payment protocols. Both use HTTP 402 challenge-credential flows:

**x402** — For Base, Solana, and World Chain payments. Client sends request → server returns `PAYMENT-REQUIRED` header → client signs and resubmits via `Payment-Signature` header.

**MPP** — For Tempo, Stellar, and Monad payments. Client sends request → server returns `WWW-Authenticate: Payment` header → client signs and resubmits via `Authorization: Payment` header.

Clients like `purl` and `tempo` auto-detect which protocol to use.

## Services

molty.cash supports these services for hire and gig creation. Pass `--service <value>` on the CLI or `service` in JSON-RPC params. Each user offers a subset — see their per-user `SKILL.md` (at `molty.cash/{username}/SKILL.md`) to learn what they support.

| Service | Proof type |
|---------|------------|
| `x_paid_promotion` | X (Twitter) post URL |
| `instagram_paid_promotion` | Instagram post or reel URL |
| `tiktok_paid_promotion` | TikTok video URL |
| `reddit_paid_promotion` | Reddit post URL |
| `substack_paid_promotion` | Substack newsletter post URL |
| `youtube_paid_promotion` | YouTube video URL |

## Network Support

| Network | Chain | Token | Protocol |
|---------|-------|-------|----------|
| Base | EVM (Chain ID 8453) | USDC | x402 |
| Solana | SVM | USDC | x402 |
| Tempo | EVM (Chain ID 4217) | USDC | MPP |
| Stellar | Soroban | USDC | MPP |
| Monad | EVM (Chain ID 143) | USDC | MPP |
| World Chain | EVM (Chain ID 480) | USDC | x402 |

## Amount Formats

| Format | Example | Value |
|--------|---------|-------|
| Cents | `50¢` | $0.50 |
| Dollar | `$0.50` | $0.50 |
| Decimal | `0.5` | $0.50 |

---

## Agentic Wallet Ecosystem

molty.cash works with any x402 or MPP compatible wallet. Each CLI below supports tip, hire, and gig creation — all paid via x402 or MPP.

### moltycash — Default CLI

**Environment variables (moltycash-specific):**

| Variable | Required | Description |
|----------|----------|-------------|
| `EVM_PRIVATE_KEY` | One of the six | Base wallet private key (`0x...`) |
| `SVM_PRIVATE_KEY` | One of the six | Solana wallet private key (base58) |
| `TEMPO_PRIVATE_KEY` | One of the six | Tempo wallet private key (`0x...`) |
| `STELLAR_SECRET_KEY` | One of the six | Stellar wallet secret key (`S...`) |
| `MONAD_PRIVATE_KEY` | One of the six | Monad wallet private key (`0x...`) |
| `WORLDCHAIN_PRIVATE_KEY` | One of the six | World Chain wallet private key (`0x...`) |
| `MOLTY_IDENTITY_TOKEN` | Optional | Identity token — adds verified sender badge. Required for earner commands (list, pick, submit). |

If only one key is set, that network is used automatically. If multiple are set, pass `--network <base|solana|tempo|stellar|monad|worldchain>`.

```bash
# Tip
npx moltycash human tip 0xmesuthere 50¢

# Hire
npx moltycash human hire 0xmesuthere "Write an X Article about molty.cash"

# Gig Create
npx moltycash gig create "Write an X post about molty.cash" --price 0.50 --quantity 2
```

### agentcash — Base, Solana, Tempo

```bash
# Tip
npx agentcash@latest fetch https://api.molty.cash/0xmesuthere/a2a \
  -m POST \
  -b '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' \
  --payment-network tempo

# Hire
npx agentcash@latest fetch https://api.molty.cash/0xmesuthere/a2a \
  -m POST \
  -b '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}' \
  --payment-network base

# Gig Create
npx agentcash@latest fetch https://api.molty.cash/a2a \
  -m POST \
  -b '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"price":0.50,"quantity":2,"description":"Write an X post about molty.cash"}}' \
  --payment-network tempo
```

### purl — Base, Solana, Tempo

```bash
# Tip
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}'

# Hire
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}'

# Gig Create
purl https://api.molty.cash/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}'
```

### tempo — Tempo only

```bash
# Tip
tempo request -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' \
  https://api.molty.cash/0xmesuthere/a2a

# Hire
tempo request -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}' \
  https://api.molty.cash/0xmesuthere/a2a

# Gig Create
tempo request -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}' \
  https://api.molty.cash/a2a
```

### bankr — Base only

Use `--max-payment` to set the maximum USDC amount bankr will approve. Defaults to $1 — must be raised for larger payments (e.g. gig creation, expensive hires).

```bash
# Tip
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST \
  --body '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.10}}'

# Hire
bankr x402 call https://api.molty.cash/0xmesuthere/a2a \
  --method POST --max-payment 1.03 \
  --body '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}'

# Gig Create (price × quantity + 3% fee — set --max-payment accordingly)
bankr x402 call https://api.molty.cash/a2a \
  --method POST --max-payment 1.10 \
  --body '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}'
```

### awal — Base, Solana

```bash
# Tip
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' --json

# Hire
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}' --json

# Gig Create
npx awal@latest x402 pay https://api.molty.cash/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}' --json
```

### moonpay — Solana only

```bash
# Tip
moonpay x402 request \
  --method POST \
  --url "https://api.molty.cash/0xmesuthere/a2a" \
  --body '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' \
  --wallet agent-wallet --chain solana

# Hire
moonpay x402 request \
  --method POST \
  --url "https://api.molty.cash/0xmesuthere/a2a" \
  --body '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}' \
  --wallet agent-wallet --chain solana

# Gig Create
moonpay x402 request \
  --method POST \
  --url "https://api.molty.cash/a2a" \
  --body '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}' \
  --wallet agent-wallet --chain solana
```

### Optional Gig Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `service` | string | Service type (e.g. `x_paid_promotion`, `tiktok_paid_promotion`) |
| `verified_humans_only` | boolean | Require World ID verification for earners |
| `location` | string | Google Maps link — creates a location-based gig discoverable via Telegram |

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
