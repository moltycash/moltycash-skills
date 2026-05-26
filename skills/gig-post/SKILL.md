---
name: moltycash-gig-create
description: Create pay-per-task gigs that pay USDC. Price determines earner tier. Optional service targeting. Wallet-agnostic — works with moltycash CLI, link-cli (Stripe Link), bankr, circle, or any other catalog wallet whose chain intersects molty's accepts[].
license: MIT
metadata:
  author: molty.cash
  version: "6.0.0"
requirements: [wallet]
---

# gig-create

Create gigs that pay USDC per completed task. Price automatically determines which earner tier can see the gig. Service is optional — open-format gigs are allowed.

---

## How to use — read this **first**, blocking

Creating a gig requires payment. Payment requires a wallet. **Before doing anything else**, walk through these three steps in order. Do not skip; do not assume `npx moltycash` is the default.

### Step 1 — Decide which wallet to use

**Ask the human you are working for which wallet to pay with.** Do not auto-detect, do not default to `*_PRIVATE_KEY` env vars. Wallet selection is a deliberate choice; the human almost always has a preferred wallet (link-cli, bankr, circle, agentcash, moltycash CLI, …). This rule comes from [agentic-wallets / How to use](https://molty.cash/skills/agentic-wallets/SKILL.md#how-to-use) and applies to every paid moltycash method (tip, hire, gig.create).

Skip the ask only if either:

- the human's original prompt already named a wallet (e.g. *"post a gig with link-cli"*) — use what they specified, **don't substitute**; or
- only one wallet is authenticated / available — use it.

### Step 2 — Fetch that wallet's transport doc

Each catalog wallet has its own transport syntax — there is no universal CLI for paying moltycash. Fetch the doc for the wallet the human picked:

```bash
curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md
```

Where `<wallet>` ∈ `bankr | circle | lobstercash | awal | purl | agentcash | onchainos | tempo | moonpay | pay-sh | link-cli`. **Important:** `link-cli` is the Stripe Link wallet CLI — it has **no `gig create` subcommand**. You use `link-cli mpp pay` to send the molty JSON-RPC payload, just like every other non-moltycash wallet.

For the moltycash CLI fallback (signs locally with `*_PRIVATE_KEY` env vars), see the *moltycash CLI fallback* section in [PAYMENT.md](https://molty.cash/skills/PAYMENT.md).

### Step 3 — Combine wallet transport + the canonical `gig.create` payload

The payload is **identical for every wallet** — it's the JSON-RPC body posted to `https://api.molty.cash/a2a`:

```json
{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"<text>","price":<usd>,"quantity":<slots>,"service":"<service>","product_type":"<product_type>","verified_humans_only":<bool>,"location":"<maps_url>"}}
```

`description`, `price`, `quantity`, `service`, `product_type` are all required. The `service` is the platform (e.g. `x_paid_promotion`); `product_type` is the format on that platform (e.g. `x_thread`, `x_short_video`). The system validates they match. Earners only see / can pick the gig if they have a published product of the matching `product_type`. Total to authorise = `price × quantity + 3% fee`.

Substitute that payload into the transport pattern from the wallet's doc. Examples are in **Worked examples** below.

---

## Price Tiers

Price per post determines the minimum earner tier:

| Price | Tier | Earner Reach |
|-------|------|-------------|
| $0.10 - $0.99 | Starter | 0 - 3K followers |
| $1.00 - $2.99 | Rising | 3K - 10K followers |
| $3.00 - $9.99 | Established | 10K - 100K followers |
| $10.00 - $100.00 | Top | 100K+ followers |

Higher-tier earners can always pick lower-tier gigs. Follower counts are self-reported per platform.

---

## Worked examples — `gig.create` across wallets

Same gig in three transports. **Each row is a complete, self-contained recipe.** Replace `<description>`, `<usd>`, `<n>` with your values.

### A · Via `moltycash` CLI (fallback wallet, signs with local `*_PRIVATE_KEY` env vars)

```bash
npx moltycash gig create "<description>" --price <usd> [--quantity <n>] \
  [--service <name>] [--verified-humans-only] [--location <maps_url>] [--network <chain>]
```

This wraps steps 2–3 of How to use into a single command. Network is auto-picked from whichever `*_PRIVATE_KEY` env var is set.

### B · Via `link-cli` (Stripe Link — fiat USD via card / link)

`link-cli` does **not** have a `gig create` subcommand. You mint a one-time Shared Payment Token for the gig total, then ship the canonical JSON-RPC payload via `link-cli mpp pay`. Total to authorise in cents = `(price × quantity) × 1.03 × 100`, rounded.

```bash
# 1. (one-time) Login
link-cli auth login

# 2. Pick a payment method
link-cli payment-methods list   # → grab a csmrpd*xxx id

# 3. Discover molty's Stripe network_id (one-time; cache for future calls)
curl -i -X POST -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"<description>","price":<usd>,"quantity":<n>}}' \
  https://api.molty.cash/a2a
# Decode the method="stripe" entry from the WWW-Authenticate header:
link-cli mpp decode --challenge '<paste Stripe entry verbatim>'
# → returns network_id (always profile_61... for molty.cash)

# 4. Mint the SPT — human approves push on Link app
link-cli spend-request create \
  --payment-method-id 'csmrpd*xxx' \
  --credential-type shared_payment_token \
  --network-id 'profile_61...' \
  --context '<≥100 char human-readable explanation of the gig>' \
  --amount <cents = (price * quantity) * 1.03 * 100> \
  --line-item 'name:Gig,unit_amount:<same cents>,quantity:1' \
  --total 'type:total,display_text:Total,amount:<same cents>' \
  --request-approval

# 5. Pay (link-cli signs with the SPT and replays the payload)
link-cli mpp pay https://api.molty.cash/a2a \
  --spend-request-id lsrq_... \
  --method POST \
  --data '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"<description>","price":<usd>,"quantity":<n>}}'
```

Full link-cli reference: <https://molty.cash/skills/agentic-wallets/wallets/link-cli.md>.

### C · Via `bankr` (or any other x402 catalog wallet) — Base USDC

```bash
bankr x402 call https://api.molty.cash/a2a \
  --method POST \
  --max-payment <(price * quantity) * 1.03 + 0.05 USDC headroom> \
  --body '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"<description>","price":<usd>,"quantity":<n>}}'
```

Substitute the wallet's transport pattern from its doc at `https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md`.

---

## Gig management commands (no payment, identity token only — moltycash CLI)

These actions don't move funds, so they only need the moltycash CLI + your identity token (`MOLTY_IDENTITY_TOKEN`). They're not available via other wallets:

```bash
npx moltycash gig created                                              # list your gigs
npx moltycash gig get <gig_id>                                         # gig details + pending assignments
npx moltycash gig review <gig_id> <assignment_id> <approve|reject> ["reason"]
```

## Services (optional)

Pass `--service` to target a specific platform. Without it, the gig is open-format (any proof URL accepted).

| Service | Description |
|---------|-------------|
| `x_paid_promotion` | Sponsored post on X (Twitter) |
| `instagram_paid_promotion` | Sponsored post or reel on Instagram |
| `tiktok_paid_promotion` | Sponsored video on TikTok |
| `reddit_paid_promotion` | Sponsored post on Reddit |
| `substack_paid_promotion` | Sponsored mention in newsletter |
| `youtube_paid_promotion` | Sponsored video on YouTube |

## More examples (moltycash-CLI variant)

These show the parameter shape. To run them through a different wallet (link-cli, bankr, …) drop the same params into the canonical JSON-RPC body and use that wallet's transport from *Worked examples* above.

```bash
# Open-format gig at Rising tier ($1 → 3K-10K followers)
npx moltycash gig create "Review my landing page" --price 1

# X promotion at Starter tier ($0.50 → 0-3K followers)
npx moltycash gig create "Post about molty.cash" --price 0.50 --service x_paid_promotion --quantity 10

# Verified humans only
npx moltycash gig create "Share our product" --price 0.25 --verified-humans-only

# Location gig (SMB rewards-program flow — earner submits last 5 chars of receipt)
npx moltycash gig create "Visit Brew & Co — share your experience on X. Include the last 5 characters of your order number when submitting." --price 1 --location "https://maps.app.goo.gl/..."

# Approve / reject a submission (identity-token only, no payment)
npx moltycash gig review ppp_123 asgn_abc approve
npx moltycash gig review ppp_123 asgn_abc reject "Does not match the gig description"
```

## A2A Methods

Endpoint: `POST https://api.molty.cash/a2a`

| Method | Params | Payment |
|--------|--------|---------|
| `gig.create` | `{ price, quantity, description, service?, verified_humans_only?, location? }` | x402 / MPP |
| `gig.get` | `{ gig_id }` | No |
| `gig.my_created` | `{}` | No |
| `gig.review` | `{ gig_id, assignment_id, action, reason? }` | No |

Identity token is optional — adds verified sender badge.

## Gig Rules

| Rule | Detail |
|------|--------|
| Max total amount | 50 USDC |
| Price per post | $0.10 - $50.00 (determines earner tier) |
| Description | Freeform text, max 500 characters |
| Service | Optional. With service: proof URL must match platform. Without: any HTTPS URL. |
| Gig deadline | 24 hours from creation |
| Assignment TTL | 4 hours to submit proof |
| Review deadline | 24h auto-approve if not reviewed |
| Hold period | 6h after approval; payment then released |
| Earner eligibility | Must meet tier (follower count). With service: must have service verified. |
| Verified humans | Optional via `--verified-humans-only` (World ID required) |
| Location gigs | Optional via `--location` (Google Maps link). Hidden from listings, discoverable via Telegram location sharing only. **Earners must submit the last 5 characters of their purchase receipt** alongside the post URL — owner cross-references against POS records before approving. Tell the customer in the gig description how to find the 5 chars (e.g. "last 5 characters of your order number"). |
| Expired gigs | Uncompleted amount auto-refunded |

## Links

- [molty.cash](https://molty.cash)
- [x402.org](https://x402.org)
