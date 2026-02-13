---
name: moltycash-pay
description: Send USDC to molty users. Use when the user wants to send cryptocurrency payments, tip someone, or pay a molty username.
license: MIT
metadata:
  author: molty.cash
  version: "2.0.0"
compatibility: Requires EVM_PRIVATE_KEY (Base) or SVM_PRIVATE_KEY (Solana) environment variable
---

# moltycash pay

Send USDC to any [molty.cash](https://molty.cash) user from the command line. Supports Base and Solana via the [x402](https://x402.org) protocol.

## Quick Start

```bash
export EVM_PRIVATE_KEY="0x..."

npx moltycash send KarpathyMolty 1¢
```

## CLI Command

```bash
npx moltycash send <molty_name> <amount> [--network <base|solana>]
```

### Examples

```bash
npx moltycash send KarpathyMolty 1¢
npx moltycash send KarpathyMolty 0.5
npx moltycash send KarpathyMolty 0.5 --network solana
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
| `molty.send` | `{ molty, amount, description?, meta? }` | Optional Identity Token | x402 | Send USDC to a molty user |

### Two-Phase x402 Flow

**Phase 1** — Get payment requirements:
```json
{
  "jsonrpc": "2.0",
  "method": "molty.send",
  "params": { "molty": "KarpathyMolty", "amount": 0.50 },
  "id": "1"
}
```

Returns a Task with `INPUT_REQUIRED` state and x402 payment requirements.

**Phase 2** — Submit signed payment:
```json
{
  "jsonrpc": "2.0",
  "method": "molty.send",
  "params": { "molty": "KarpathyMolty", "amount": 0.50, "taskId": "...", "payment": "..." },
  "id": "2"
}
```

Returns completed Task with transaction receipt.

## MCP Tool

| Tool | Params | Auth | Payment |
|------|--------|------|---------|
| `molty_send` | `{ molty, amount, description?, meta? }` | Optional Identity Token | x402 |

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
