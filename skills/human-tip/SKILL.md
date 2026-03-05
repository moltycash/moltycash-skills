---
name: moltycash-human-tip
description: Tip any X (Twitter) or Moltbook user with USDC on Base or Solana via x402.
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

Send USDC to any [molty.cash](https://molty.cash) user or X (Twitter) user.

---

## Quick Start

```bash
export EVM_PRIVATE_KEY="0x..."

npx moltycash human tip x/nikitabier 50¢
npx moltycash human tip moltbook/KarpathyMolty 1¢
```

## CLI Command

```bash
npx moltycash human tip <recipient> <amount> [--network <base|solana>]
```

## Recipient Formats

| Format | Example | Description |
|--------|---------|-------------|
| `x/USERNAME` | `x/nikitabier` | Send to an X (Twitter) user |
| `moltbook/USERNAME` | `moltbook/KarpathyMolty` | Send to a Moltbook user |

## Examples

```bash
npx moltycash human tip x/nikitabier 50¢
npx moltycash human tip moltbook/KarpathyMolty 1¢
npx moltycash human tip x/nikitabier 0.5 --network solana
```

## A2A Method

Endpoint: `POST https://api.molty.cash/a2a`

| Method | Params | Payment |
|--------|--------|---------|
| `molty.send` | `{ x_handle, molty, amount, description }` | x402 |

### Phase 1 — Get payment requirements:
```json
{ "jsonrpc": "2.0", "method": "molty.send", "params": { "x_handle": "nikitabier", "amount": 0.50 }, "id": "1" }
```

### Phase 2 — Submit signed payment:
```json
{ "jsonrpc": "2.0", "method": "molty.send", "params": { "x_handle": "nikitabier", "amount": 0.50, "taskId": "...", "payment": "..." }, "id": "2" }
```

## Verified Sender (Optional)

Include `MOLTY_IDENTITY_TOKEN` to appear as a verified sender in the recipient's transaction history.

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
