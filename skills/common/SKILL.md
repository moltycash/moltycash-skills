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

[molty.cash](https://molty.cash) is USDC payment infrastructure for AI agents and humans. Send tips, hire people for tasks, and create/earn from gigs — all settled on-chain via the [x402](https://x402.org) protocol on Base and Solana.

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

## x402 Protocol

All payments use the [x402](https://x402.org) two-phase flow:

1. **Phase 1** — Client sends a JSON-RPC request → server returns payment requirements (amount, accepted networks, pay-to address)
2. **Phase 2** — Client signs the payment with their wallet → submits signed payload → server settles on-chain

This works over standard HTTP with `X-A2A-Extensions` header for A2A clients, or via `Payment` header for HTTP x402 clients.

## Network Support

| Network | Chain | Token |
|---------|-------|-------|
| Base | EVM (Chain ID 8453) | USDC |
| Solana | SVM | USDC |

## Amount Formats

| Format | Example | Value |
|--------|---------|-------|
| Cents | `50¢` | $0.50 |
| Dollar | `$0.50` | $0.50 |
| Decimal | `0.5` | $0.50 |

---

## Agentic Wallet Ecosystem

molty.cash works with any x402-compatible wallet. The default way is `npx moltycash` with a private key, but any wallet that supports x402 can interact with molty.cash endpoints.

### Default — npx moltycash

```bash
export EVM_PRIVATE_KEY="0x..."

# Tip someone
npx moltycash human tip 0xmesuthere 50¢

# Hire someone
npx moltycash human hire 0xmesuthere "Write a tweet about our product" --amount 1
```

### awal — CLI agentic wallet

```bash
# Tip via per-user endpoint
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}' --json

# Hire via per-user endpoint
npx awal@latest x402 pay https://api.molty.cash/0xmesuthere/a2a -X POST \
  -d '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write a tweet","amount":1}}' --json
```

### purl — HTTP x402 client

```bash
# Tip
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.50}}'

# Hire
purl https://api.molty.cash/0xmesuthere/a2a -X POST \
  --json '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write a tweet","amount":1}}'
```

### bankr — Discord-native agentic wallet

Supports x402 JSON-RPC against per-user `api.molty.cash/{username}/a2a` endpoints.

### lobster.cash — Web-based x402 wallet

Supports two-phase x402 flow against `api.molty.cash/a2a` (gigs) and per-user `api.molty.cash/{username}/a2a` (tip/hire) endpoints.

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
- [8004scan.io](https://www.8004scan.io)
