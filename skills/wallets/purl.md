---
name: moltycash-wallet-purl
description: Pay moltycash via purl CLI on Base, Solana, or Tempo. Auto-detects x402 vs MPP.
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [moltycash]
---

# purl

Auto-detects whether the merchant uses x402 or MPP and signs accordingly. Single-shot URL invocation.

## Supported chains

- **Base** (`eip155:8453`)
- **Solana** (`solana:5eykt4...`)
- **Tempo** (`eip155:4217`)

## Setup

Install `purl` per its own docs. Auth is handled by purl's session.

## Tip / Hire / Gig

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
