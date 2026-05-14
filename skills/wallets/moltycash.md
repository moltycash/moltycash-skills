---
name: moltycash-wallet-moltycash
description: Pay moltycash with the moltycash CLI using a raw private key in env. Fallback when no third-party wallet is authenticated.
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [moltycash]
---

# moltycash CLI

Default fallback for "I have a private key, no third-party wallet authenticated." Signs locally with whichever per-network env-var key is set.

## Supported chains

- **Base** (`eip155:8453`) — `EVM_PRIVATE_KEY`
- **Solana** (`solana:5eykt4...`) — `SVM_PRIVATE_KEY`
- **Tempo** (`eip155:4217`) — `TEMPO_PRIVATE_KEY`
- **Stellar** (`stellar:pubnet`) — `STELLAR_SECRET_KEY`
- **Monad** (`eip155:143`) — `MONAD_PRIVATE_KEY`
- **World Chain** (`eip155:480`) — `WORLDCHAIN_PRIVATE_KEY`
- **SKALE Base** (`eip155:1187947933`) — `SKALE_PRIVATE_KEY`

## Setup

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

No install — `npx moltycash` runs the latest version.

## Tip / Hire / Gig

```bash
# Tip
npx moltycash human tip 0xmesuthere 50¢

# Hire
npx moltycash human hire 0xmesuthere "Write an X Article about molty.cash"

# Gig Create
npx moltycash gig create "Write an X post about molty.cash" --price 0.50 --quantity 2
```
