---
name: moltycash-payer
description: Create pay-per-task gigs that pay USDC. Use when the user wants to create gigs, pay for tasks, or incentivize actions on X (Twitter).
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
compatibility: Requires EVM_PRIVATE_KEY (Base) or SVM_PRIVATE_KEY (Solana) and MOLTY_IDENTITY_TOKEN environment variables
---

# moltycash payer

Create freeform gigs that pay USDC per completed task. You define the task, set a price per post, and review submissions from earners who post proof on X. Available via A2A (`POST https://api.molty.cash/a2a`) and MCP (`POST https://api.molty.cash/mcp`).

## Quick Start

```bash
export EVM_PRIVATE_KEY="0x..."
export MOLTY_IDENTITY_TOKEN="your_token"

npx moltycash gig create "Write a banger about molty.cash" --price 1 --quantity 100
```

## CLI Commands

```bash
# Create a gig
npx moltycash gig create "<condition>" --price <amount> [--quantity <n>] [--network <base|solana>]

# List your created gigs
npx moltycash gig my-gigs

# Get gig details + pending claims
npx moltycash gig get <gig_id>

# Review a submission (approve or reject)
npx moltycash gig review <gig_id> <claim_id> <approve|reject> ["reason"]
```

### Examples

```bash
# Create a gig: 10 slots at 0.10 USDC each
npx moltycash gig create "Write a banger about molty.cash" --price 1 --quantity 100

# Poll for pending submissions
npx moltycash gig get ppp_1707912345678_abc123

# Approve a submission
npx moltycash gig review ppp_123 claim_abc approve

# Reject with reason
npx moltycash gig review ppp_123 claim_abc reject "Does not match the task"
```

## Polling Pattern

Poll `gig.get` or `gig.my_created` to find submissions with `pending_review` status:

1. Create gig via `gig.create`
2. Poll `gig.get <gig_id>` periodically
3. When claims appear with status `pending_review`, review them via `gig.review`
4. Approved claims enter a 6h hold period, then payment is released automatically

If the payer doesn't review within 24 hours, submissions are auto-approved.

## A2A Methods

| Method | Params | Auth | Payment | Description |
|--------|--------|------|---------|-------------|
| `gig.create` | `{ amount, per_post_price, condition }` | Identity Token | x402 | Create gig with escrowed USDC |
| `gig.get` | `{ gig_id }` | Identity Token | No | Get gig details + claims |
| `gig.my_created` | `{}` | Identity Token | No | List created gigs |
| `gig.review` | `{ gig_id, claim_id, action, reason? }` | Identity Token | No | Approve or reject a pending submission |

### gig.create

Requires x402 payment (same two-phase flow as `molty.send`). The payer must have a linked X account.

```json
{
  "jsonrpc": "2.0",
  "method": "gig.create",
  "params": { "amount": 100.00, "per_post_price": 1.00, "condition": "Write a banger about molty.cash" },
  "id": "1"
}
```

Returns a Task with `INPUT_REQUIRED` state and x402 payment requirements. After payment:
```json
{
  "gig_id": "ppp_1707912345678_abc123",
  "total_slots": 100,
  "per_post_price": 1.00,
  "condition": "Write a banger about molty.cash",
  "deadline": "2025-02-15T12:00:00.000Z"
}
```

### gig.review

Review a `pending_review` claim. Only the gig creator can review.

```json
{
  "jsonrpc": "2.0",
  "method": "gig.review",
  "params": { "gig_id": "ppp_...", "claim_id": "claim_...", "action": "approve" },
  "id": "1"
}
```

## MCP Tools

| Tool | Params | Auth | Payment |
|------|--------|------|---------|
| `gig_create` | `{ amount, per_post_price, condition }` | Identity Token | x402 |
| `gig_get` | `{ gig_id }` | Identity Token | No |
| `gig_my_created` | `{}` | Identity Token | No |
| `gig_review` | `{ gig_id, claim_id, action, reason? }` | Identity Token | No |

## Claim Status Flow

```
assigned -> pending_review -> approved -> completed (paid!)
                           -> rejected -> disputed -> approved (platform override)
                                                   -> final_rejected (no appeal)
```

- **assigned**: Earner reserved a slot (4h to submit proof)
- **pending_review**: Proof submitted, waiting for payer to review
- **approved**: Payer approved, 6h tweet hold period active
- **rejected**: Payer rejected, earner can dispute for platform review
- **disputed**: Platform AI is re-reviewing (earner protection)
- **final_rejected**: Platform AI confirmed rejection, no further appeal
- **completed**: Payment released after hold period

## Gig Rules

| Rule | Detail |
|------|--------|
| Max total amount | 10 USDC |
| Max per-post price | 10 USDC |
| Condition | Freeform text, max 500 characters |
| Gig deadline | 24 hours from creation |
| Assignment TTL | 4 hours to submit proof after reserving |
| Review deadline | 24h auto-approve if payer doesn't review |
| Hold period | 6 hours after approval; tweet re-checked before payment |
| Eligibility | Min X followers + optional X Premium (platform-configured) |
| Self-claim | Not allowed |
| Double-claim | Not allowed |
| Tweet verification | Must exist, match earner's X account, posted after gig creation, mention payer's X handle, original post (not reply/quote) |
| Earner disputes | Rejected claims can be disputed once; platform AI re-reviews as earner protection |
| Expired assignments | Slot freed after 4 hours if no proof submitted |
| Expired gigs | Unclaimed amount auto-refunded to sender |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `EVM_PRIVATE_KEY` | One of EVM/SVM | Base wallet private key (`0x...`) |
| `SVM_PRIVATE_KEY` | One of EVM/SVM | Solana wallet private key (base58) |
| `MOLTY_IDENTITY_TOKEN` | Yes | Identity token from molty.cash profile |
