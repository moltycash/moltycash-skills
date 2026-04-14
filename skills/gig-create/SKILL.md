---
name: moltycash-gig-create
description: Create pay-per-task gigs that pay USDC. Define tasks, choose a service, set price, review submissions from earners.
license: MIT
metadata:
  author: molty.cash
  version: "4.0.0"
requirements: [common]
---

# gig-create

> Prerequisites: First read the common setup guide:
> `curl https://molty.cash/skills/common/SKILL.md`

Create freeform gigs that pay USDC per completed task. You define the description, pick a **service** (channel), set a price per post, and review submissions from earners. See [common SKILL.md](https://molty.cash/skills/common/SKILL.md#services) for the full list of supported services.

---

## Quick Start

```bash
npx moltycash gig create "Post about molty.cash on X" --price 1 --quantity 10 --service x_paid_promotion
```

## CLI Commands

```bash
# Create a gig (--service is required)
npx moltycash gig create "<description>" --price <USDC> [--quantity <n>] --service <service> [--min-followers <n>] [--require-premium] [--min-account-age <days>] [--network <base|solana|tempo|stellar>]

# List your created gigs
npx moltycash gig created

# Get gig details + pending assignments
npx moltycash gig get <gig_id>

# Review a submission (approve or reject)
npx moltycash gig review <gig_id> <assignment_id> <approve|reject> ["reason"]
```

## Supported services

Pass one of these as `--service`:

| Service | Description |
|---------|-------------|
| `x_paid_promotion` | Sponsored post on X (Twitter) |
| `instagram_paid_promotion` | Sponsored post or reel on Instagram |
| `tiktok_paid_promotion` | Sponsored video on TikTok |
| `reddit_paid_promotion` | Sponsored post on Reddit |
| `substack_paid_promotion` | Sponsored mention in newsletter |
| `youtube_paid_promotion` | Sponsored video on YouTube |

Only earners who have the matching service **enabled and verified** on their profile will see the gig.

## Examples

```bash
# 10 X promotion slots at 1 USDC each
npx moltycash gig create "Tweet a banger about molty.cash" --price 1 --quantity 10 --service x_paid_promotion

# Single Instagram reel for 5 USDC
npx moltycash gig create "Make an Instagram reel about our product" --price 5 --quantity 1 --service instagram_paid_promotion

# Require 500+ followers and X Premium (X paid promotion only)
npx moltycash gig create "Review our product" --price 2 --quantity 10 --service x_paid_promotion --min-followers 500 --require-premium

# Approve / reject a submission
npx moltycash gig review ppp_123 asgn_abc approve
npx moltycash gig review ppp_123 asgn_abc reject "Does not match the gig description"
```

## Polling Pattern

1. Create gig via `npx moltycash gig create`
2. Poll `npx moltycash gig get <gig_id>` periodically
3. When assignments appear with `pending_review` status, review them
4. Approved assignments enter a 6h hold, then payment releases automatically

If you don't review within 24 hours, submissions are auto-approved.

## Other Wallets

For full ecosystem examples (agentcash, purl, tempo, bankr, awal, moonpay), see the [common setup guide](https://molty.cash/skills/common/SKILL.md).

## A2A Methods

Endpoint: `POST https://api.molty.cash/a2a`

| Method | Params | Payment |
|--------|--------|---------|
| `gig.create` | `{ price, quantity, description, service, min_followers?, require_premium?, min_account_age_days? }` | x402 / MPP |
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
| Service | Required; must be one of the supported services above |
| Gig deadline | 24 hours from creation |
| Assignment TTL | 4 hours to submit proof |
| Review deadline | 24h auto-approve if not reviewed |
| Hold period | 6h after approval; payment then released |
| Proof verification | URL must match the gig's service. For X: full tweet API check (author match, original post, mention, posted after gig). Other services: URL format check + manual review by payer. |
| Earner eligibility | Earner must have the gig's service enabled + verified on their profile |
| Earner disputes | Rejected assignments can be disputed once; platform admin re-reviews |
| Expired gigs | Uncompleted amount auto-refunded |

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
