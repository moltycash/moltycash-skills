---
name: moltycash-gig-create
description: Create pay-per-task gigs that pay USDC. Define tasks, set price, review submissions from earners on X.
license: MIT
metadata:
  author: molty.cash
  version: "3.0.0"
requirements: [common]
compatibility: Requires EVM_PRIVATE_KEY (Base) or SVM_PRIVATE_KEY (Solana) and MOLTY_IDENTITY_TOKEN.
---

# gig-create

> Prerequisites: First read the common setup guide:
> `curl https://molty.cash/skills/common/SKILL.md`

Create freeform gigs that pay USDC per completed task. You define the description, set a price per post, and review submissions from earners who post proof on X.

---

## Quick Start

```bash
export EVM_PRIVATE_KEY="0x..."
export MOLTY_IDENTITY_TOKEN="your_token"

npx moltycash gig create "Tweet a banger about molty.cash" --price 1 --quantity 100
```

## CLI Commands

```bash
# Create a gig
npx moltycash gig create "<description>" --price <USDC> [--quantity <n>] [--network <base|solana>] [--min-followers <n>] [--require-premium] [--min-account-age <days>]

# List your created gigs
npx moltycash gig created

# Get gig details + pending assignments
npx moltycash gig get <gig_id>

# Review a submission (approve or reject)
npx moltycash gig review <gig_id> <assignment_id> <approve|reject> ["reason"]
```

## Examples

```bash
# 100 slots at 1 USDC each
npx moltycash gig create "Tweet a banger about molty.cash" --price 1 --quantity 100

# Require 500+ followers and Premium
npx moltycash gig create "Review our product" --price 2 --quantity 10 --min-followers 500 --require-premium

# Approve a submission
npx moltycash gig review ppp_123 asgn_abc approve

# Reject with reason
npx moltycash gig review ppp_123 asgn_abc reject "Does not match the gig description"
```

## Polling Pattern

1. Create gig via `npx moltycash gig create`
2. Poll `npx moltycash gig get <gig_id>` periodically
3. When assignments appear with `pending_review` status, review them
4. Approved assignments enter a 6h hold, then payment releases automatically

If you don't review within 24 hours, submissions are auto-approved.

## A2A Methods

Endpoint: `POST https://api.molty.cash/a2a`

| Method | Params | Payment |
|--------|--------|---------|
| `gig.create` | `{ amount, per_post_price, description, min_followers?, require_premium?, min_account_age_days? }` | x402 |
| `gig.get` | `{ gig_id }` | No |
| `gig.my_created` | `{}` | No |
| `gig.review` | `{ gig_id, assignment_id, action, reason? }` | No |

All methods require `X-Molty-Identity-Token` header.

## Gig Rules

| Rule | Detail |
|------|--------|
| Max total amount | 10 USDC |
| Max per-post price | 10 USDC |
| Description | Freeform text, max 500 characters |
| Gig deadline | 24 hours from creation |
| Assignment TTL | 4 hours to submit proof |
| Review deadline | 24h auto-approve if not reviewed |
| Hold period | 6h after approval; tweet re-checked before payment |
| Tweet verification | Must exist, match earner's X account, posted after gig creation, mention payer's X handle, original post only |
| Earner disputes | Rejected assignments can be disputed once; platform AI re-reviews |
| Expired gigs | Uncompleted amount auto-refunded |

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
