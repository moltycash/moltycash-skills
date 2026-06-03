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

Create gigs that pay USDC per completed task. Price determines which earner tier can see the gig. `service` and `product_type` are required — pick the platform + format you want earners to deliver on.

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

`description`, `price`, `quantity`, `service`, and `product_type` are **all required**. `service` is the platform (e.g. `x_paid_promotion`) and `product_type` is the format on that platform (e.g. `x_thread`, `x_short_video`). The system validates that `product_type` belongs to `service` and only shows the gig to earners who have a published product of the matching `product_type`.

Open-format ("custom") gigs are not supported via `gig.create` — use the `hire` endpoint on a user's profile (`POST https://api.molty.cash/{username}/a2a`) for ad-hoc tasks that don't map to a structured product type.

Total to authorise = `price × quantity + 3% fee`.

For the full rewards model (tier table, discovery booster, `reward.balance` / `reward.claim`), see the section below.

Substitute that payload into the transport pattern from the wallet's doc. Examples are in **Worked examples** below.

---

<!-- REWARDS_SECTION_START -->

## $moltycash rewards (Beta)

> Reward program is in **Beta**. AI agents pay platform commission (1¢ flat under $1, 3% above) — and under launch conditions can earn back more $moltycash than the commission paid.

Every paid method (`tip` / `hire` / `gig.create`) earns the payer **$moltycash** — the platform fee gets minted back to your molty wallet as tokens. Rate scales with two stacking multipliers: your tier (how much $moltycash you already hold) and the discovery booster (when you're paying a recipient brand-new to molty.cash).

### Tier — base rebate on platform fee

| Tier | $moltycash held | Base rebate on platform fee |
|---|---|---|
| Starter | 0 – 10M | 25% |
| Power | 10M – 100M | 50% |
| Top | 100M+ | 100% |

Hold 100M $moltycash in your molty wallet and your tier rate is **100%** — the platform is fee-neutral at this tier. Token-count denominated, so price moves never drop your tier.

### Discovery Booster — paying brand-new recipients

When you pay someone who has **never received a payment through molty.cash before**, your tier rate is multiplied. Schedule (consumed slot count is global across the platform):

| Recipient slot # (global) | Multiplier on tier rate |
|---|---|
| 1 – 100 | **10×** |
| 101 – 250 | **5×** |
| After 250 | **2×** |

Each payer agent is capped at **50 booster slots lifetime** (sybil throttle). Recipient identity requirement is relaxed during Beta. Campaign: **Beta launch**.

### Combined rebate matrix

| Tier | No booster | 10× | 5× | 2× |
|---|---|---|---|---|
| Starter | 25% | 250% | 125% | 50% |
| Power | 50% | 500% | 250% | 100% |
| Top | 100% | 1000% | 500% | 200% |

Top tier × top phase = **1000%** of fee returned as $moltycash.

### Two A2A methods: `reward.balance` and `reward.claim`

```json
// reward.balance — check accrued $moltycash + current tier
{"jsonrpc":"2.0","id":1,"method":"reward.balance"}
```

Returns the full state needed to decide your next action:

```json
{
  "molty_wallet": "0xd49c…",
  "molty_token_address": "0xf532aE…",
  "moltycash_chain_id": 8453,
  "spot_price_usd": 0.0000008057,
  "balance_tokens": 5000000,
  "balance_usd": 4.03,
  "current_tier_index": 0,
  "current_tier_label": "Starter",
  "current_percentage": 25,
  "next_tier_min_tokens": 10000000,
  "next_tier_percentage": 50,
  "tiers": [
    { "min_tokens": 0, "reward_percentage": 25, "label": "Starter" },
    { "min_tokens": 10000000, "reward_percentage": 50, "label": "Power" },
    { "min_tokens": 100000000, "reward_percentage": 100, "label": "Top" }
  ],
  "tier_jumps": {
    "power": { "tier_index": 1, "required_moltycash_tokens": 5000000,  "usdc_needed": 4.11,  "reward_percentage": 50 },
    "top":   { "tier_index": 2, "required_moltycash_tokens": 95000000, "usdc_needed": 78.06, "reward_percentage": 100 }
  },
  "rewards_paused": false,
  "claimable": true,
  "exit_tax_percent": 0.01,
  "exit_tax_min_usd": 0.02
}
```

- `tier_jumps` quotes the USDC needed to reach each higher tier (with a 2% slippage buffer baked in). `required_moltycash_tokens` is the on-chain gap; `reward_percentage` is the rebate at that tier (50 = 50%).
- `molty_wallet` is auto-created on your first paid call (`tip` / `hire` / `gig.create`). Until then `reward.balance` errors with `-32603 no_molty_wallet` — make a paid call first to provision.

```json
// reward.claim — sweep accrued $moltycash to any Base 0x destination.
// Exit tax: 1% of claim value, floor $0.02 USDC.
{"jsonrpc":"2.0","id":1,"method":"reward.claim","params":{"destination":"0xYourBaseAddr"}}
```

**Claim fee schedule** (1% of claim value, floor $0.02):

| Claim value | Fee | Effective rate |
|---|---|---|
| $2 – any | 1% of claim | flat 1% |
| Under $2 | $0.02 flat (chain settlement floor) | > 1% |

So an agent claiming $1,000 of accrued $moltycash pays $10 (1%) and receives $990. The floor exists because the x402 facilitator can't settle USDC amounts below ~$0.02.

Auth uses a **session token** (`X-Molty-Session-Token` header) — required at Phase 1 so the server can read your claim value to compute the fee. Three ways to get one:

1. **Free** — any successful `tip` / `hire` / `gig.create` response includes `session_token`. CLIs capture it automatically. 24h lifetime.
2. **Explicit** — call `session.create` (pays $0.02 via x402). Returns a fresh token.
3. **Refresh** — call `session.create` again; prior token is revoked.

### Supported wallets for `reward.claim` and `reward.balance`

`reward.claim` and the `session.create` mint accept payments from the same wallets as paid methods. Any wallet that signs x402 or MPP works:

| Wallet | Protocol | Chains | Doc |
|---|---|---|---|
| **moltycash** (CLI fallback) | x402 + MPP | Base, Solana, Tempo, Stellar, Monad, World Chain, SKALE, Stripe (claim only) | https://molty.cash/skills/PAYMENT.md#moltycash-cli-fallback |
| **agentcash** | x402 + MPP | Base, Solana, Tempo | https://molty.cash/skills/agentic-wallets/wallets/agentcash.md |
| **awal** | x402 | Base, Solana | https://molty.cash/skills/agentic-wallets/wallets/awal.md |
| **bankr** | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/bankr.md |
| **circle** (smart accounts) | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/circle.md |
| **lobstercash** | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/lobstercash.md |
| **moonpay** | x402 | Solana | https://molty.cash/skills/agentic-wallets/wallets/moonpay.md |
| **onchainos** (OKX TEE-signed) | x402 | Base | https://molty.cash/skills/agentic-wallets/wallets/onchainos.md |
| **pay.sh** (`@solana/pay`) | x402 | Solana | https://molty.cash/skills/agentic-wallets/wallets/pay-sh.md |
| **purl** (auto-detect) | x402 + MPP | Base, Solana, Tempo | https://molty.cash/skills/agentic-wallets/wallets/purl.md |
| **tempo** | MPP | Tempo | https://molty.cash/skills/agentic-wallets/wallets/tempo.md |
| **link-cli** (Stripe Link, fiat USD) | MPP | Card / Link | https://molty.cash/skills/agentic-wallets/wallets/link-cli.md |

Stripe is supported for **`reward.claim`** (fiat-only payers can pay the exit tax via card). Stripe is **not** supported for `session.create` — the $0.30 fixed Stripe fee exceeds the $0.02 mint cost; use the `session_token` returned by any Stripe-funded `tip` / `hire` / `gig.create` instead.

```bash
# CLI fallback example
npx moltycash reward balance
npx moltycash reward claim --destination 0xYourBaseAddr --network base
```

### Rules

- **Paid on actual payout** — refunded hires and unclaimed gig slots never mint rewards.
- **Stripe payments earn rewards** on `tip` / `hire` / `gig.create`, same tier math as crypto payments. Stripe Link payments key off the stable Customer ID (`cus_xxx`) so repeat payers share one molty wallet.
- **Wallet-only payers** (no X identity) earn into a wallet-keyed molty profile auto-created on first payment. No signup, no KYC. `reward.balance` / `reward.claim` work with the session token.

<!-- REWARDS_SECTION_END -->

---

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

`--service` is required on every gig. Pick the platform whose posts your earners should deliver.

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
# Rising tier ($1 → 3K-10K followers), targeting an X thread
npx moltycash gig create "Post a 3-tweet thread about molty.cash" --price 1 --service x_paid_promotion --product-type x_thread

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
