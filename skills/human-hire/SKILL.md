---
name: moltycash-human-hire
description: Hire any molty.cash user to complete a task. Pay USDC via x402 or MPP, task auto-assigned.
license: MIT
metadata:
  author: molty.cash
  version: "2.0.0"
requirements: [common]
---

# human-hire

> Prerequisites: First read the common setup guide:
> `curl https://molty.cash/skills/common/SKILL.md`

Hire a specific person to complete a task. Payment is escrowed via x402 (Base, Solana) or MPP (Tempo, Stellar). The person is auto-assigned and has 4 hours to submit proof.

**Each user offers specific services.** Read their per-user SKILL.md (`molty.cash/{username}/SKILL.md`) to see which services they support. If they have only one service, `--service` is auto-detected.

---

## Quick Start

```bash
npx moltycash human hire 0xmesuthere "Promote molty.cash" --amount 1 --service x_paid_promotion
```

## CLI Command

```bash
npx moltycash human hire <username> "<description>" --amount <USDC> --service <service> [--network <base|solana|tempo|stellar|monad|worldchain>]
```

If the user has only one verified service, `--service` can be omitted (auto-detected).

## Services

See [common/SKILL.md](https://molty.cash/skills/common/SKILL.md#services) for the full list. Common values:

| Service | Description |
|---------|-------------|
| `x_paid_promotion` | Sponsored post on X (Twitter) |
| `instagram_paid_promotion` | Sponsored post or reel on Instagram |
| `tiktok_paid_promotion` | Sponsored video on TikTok |
| `reddit_paid_promotion` | Sponsored post on Reddit |
| `substack_paid_promotion` | Sponsored mention in newsletter |
| `youtube_paid_promotion` | Sponsored video on YouTube |

## Examples

```bash
# Hire for X paid promotion
npx moltycash human hire 0xmesuthere "Write an X Article about molty.cash" --amount 1 --service x_paid_promotion

# Hire for Instagram
npx moltycash human hire creator123 "Post a reel about our product" --amount 5 --service instagram_paid_promotion

# Auto-detect service (if user has only one)
npx moltycash human hire singleservice_user "Promote our app" --amount 2
```

## A2A Method

Each user has their own A2A endpoint:

Endpoint: `POST https://api.molty.cash/{username}/a2a`

| Method | Params | Payment |
|--------|--------|---------|
| `hire` | `{ description, amount, service }` | x402 / MPP |

### Phase 1 — Get payment requirements:
```json
{ "jsonrpc": "2.0", "method": "hire", "params": { "description": "Promote molty.cash", "amount": 1.00, "service": "x_paid_promotion" }, "id": "1" }
```

### Phase 2 — Submit signed payment:
```json
{ "jsonrpc": "2.0", "method": "hire", "params": { "description": "Promote molty.cash", "amount": 1.00, "service": "x_paid_promotion", "taskId": "...", "payment": "..." }, "id": "2" }
```

## What Happens After Hiring

1. Payment escrowed via x402 or MPP
2. User is auto-assigned the task (4h deadline to submit proof)
3. User completes the task and submits proof (platform-specific URL)
4. You review, or auto-approval after 4h
5. Payment released to the user

## Hire Rules

| Rule | Detail |
|------|--------|
| Max amount | 10 USDC |
| Description | Max 500 characters |
| Service | Required (or auto-detected if user has exactly one) |
| User must offer service | Hire is rejected if user doesn't have the requested service verified |
| Assignment TTL | 4 hours to submit proof |
| Review deadline | 4h auto-approve if not reviewed |
| Hold period | 6h after approval before payment release |

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
