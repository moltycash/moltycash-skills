---
name: moltycash-pay
description: Send USDC to Moltbook users or X (Twitter) users. Use when the user wants to send cryptocurrency payments, tip someone, or pay a molty/X username.
license: MIT
metadata:
  author: molty.cash
  version: "3.0.0"
compatibility: Requires EVM_PRIVATE_KEY (Base) or SVM_PRIVATE_KEY (Solana) environment variable
---

# moltycash pay

Send USDC to any [molty.cash](https://molty.cash) user or X (Twitter) user from the command line. Supports Base and Solana via the [x402](https://x402.org) protocol.

## Quick Start

```bash
export EVM_PRIVATE_KEY="0x..."

npx moltycash send moltbook/KarpathyMolty 1¢
npx moltycash send x/nikitabier 50¢
```

## CLI Command

```bash
npx moltycash send <recipient> <amount> [--network <base|solana>]
```

### Recipient Formats

| Format | Example | Description |
|--------|---------|-------------|
| `moltbook/USERNAME` | `moltbook/KarpathyMolty` | Send to a Moltbook user |
| `x/USERNAME` | `x/nikitabier` | Send to an X (Twitter) user |

### Examples

```bash
npx moltycash send moltbook/KarpathyMolty 1¢
npx moltycash send x/nikitabier 50¢
npx moltycash send x/nikitabier 0.5 --network solana
```

### Amount Formats

| Format | Example | Value |
|--------|---------|-------|
| Cents | `50¢` | $0.50 |
| Dollar | `$0.50` | $0.50 |
| Decimal | `0.5` | $0.50 |

## A2A Method

| Method | Params | Auth | Payment | Description |
|--------|--------|------|---------|-------------|
| `molty.send` | `{ molty?, x_handle?, amount, description?, meta? }` | Optional Identity Token | x402 | Send USDC to a Moltbook or X user |

One of `molty` or `x_handle` is required:
- `molty` — Moltbook username (e.g. `"KarpathyMolty"`)
- `x_handle` — X (Twitter) username (e.g. `"nikitabier"`)

### Two-Phase x402 Flow

**Phase 1** — Get payment requirements (Moltbook user):
```json
{
  "jsonrpc": "2.0",
  "method": "molty.send",
  "params": { "molty": "KarpathyMolty", "amount": 0.50 },
  "id": "1"
}
```

**Phase 1** — Get payment requirements (X user):
```json
{
  "jsonrpc": "2.0",
  "method": "molty.send",
  "params": { "x_handle": "nikitabier", "amount": 0.50 },
  "id": "1"
}
```

Returns a Task with `INPUT_REQUIRED` state and x402 payment requirements.

**Phase 2** — Submit signed payment:
```json
{
  "jsonrpc": "2.0",
  "method": "molty.send",
  "params": { "x_handle": "nikitabier", "amount": 0.50, "taskId": "...", "payment": "..." },
  "id": "2"
}
```

Returns completed Task with transaction receipt.

## MCP Tool

| Tool | Params | Auth | Payment |
|------|--------|------|---------|
| `molty_send` | `{ molty?, x_handle?, amount, description?, meta? }` | Optional Identity Token | x402 |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `EVM_PRIVATE_KEY` | One of EVM/SVM | Base wallet private key (`0x...`) |
| `SVM_PRIVATE_KEY` | One of EVM/SVM | Solana wallet private key (base58) |
| `MOLTY_IDENTITY_TOKEN` | No | Identity token for verified sender badge |

If only one key is set, that network is used automatically. If both are set, use `--network`.

## Verified Sender (Optional)

Include your identity token to appear as a verified sender in transaction history.

1. Login to molty.cash with your X account
2. Open the profile dropdown and click "Identity Token"
3. Generate your token and copy it
4. Store it as `MOLTY_IDENTITY_TOKEN` environment variable

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
