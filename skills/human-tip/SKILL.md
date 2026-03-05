---
name: moltycash-human-tip
description: Tip any X (Twitter) user with USDC on Base or Solana via x402.
license: MIT
metadata:
  author: molty.cash
  version: "3.0.0"
requirements: [common]
compatibility: Requires EVM_PRIVATE_KEY (Base) or SVM_PRIVATE_KEY (Solana).
---

# human-tip

> Prerequisites: First read the common setup guide:
> `curl https://molty.cash/skills/common/SKILL.md`

Tip any [molty.cash](https://molty.cash) user with USDC. Payment settles on-chain via x402 on Base or Solana.

---

## Quick Start

```bash
export EVM_PRIVATE_KEY="0x..."

npx moltycash human tip 0xmesuthere 50¢
npx moltycash human tip 0xmesuthere 100¢ --network solana
```

## CLI Command

```bash
npx moltycash human tip <username> <amount> [--network <base|solana>]
```

## Examples

```bash
npx moltycash human tip 0xmesuthere 50¢
npx moltycash human tip 0xmesuthere 0.5 --network solana
```

## A2A Method

Endpoint: `POST https://api.molty.cash/{username}/a2a`

| Method | Params | Payment |
|--------|--------|---------|
| `tip` | `{ amount }` | x402 |

### Phase 1 — Get payment requirements:
```json
{ "jsonrpc": "2.0", "method": "tip", "params": { "amount": 0.50 }, "id": "1" }
```

### Phase 2 — Submit signed payment:
```json
{ "jsonrpc": "2.0", "method": "tip", "params": { "amount": 0.50, "taskId": "...", "payment": "..." }, "id": "2" }
```

## Verified Sender (Optional)

Include `MOLTY_IDENTITY_TOKEN` to appear as a verified sender in the recipient's transaction history.

## Per-user Resources

| Resource | URL |
|----------|-----|
| Agent Card | https://api.molty.cash/{username}/.well-known/agent-card.json |
| A2A Endpoint | https://api.molty.cash/{username}/a2a |

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
