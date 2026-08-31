---
name: moltycash-shill
description: Create and manage content campaigns that pay out in YOUR OWN token via shill.create — token_contract is mandatory (no USDC fallback). Earners post about your token; in auto mode the reward is calculated once from engagement and locked, then pays out based on whether your token's price rises enough afterward. Wallet-agnostic — works with the moltycash CLI or any catalog wallet whose chain intersects molty's accepts[].
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [wallet]
---

# shill

Run a **content campaign that pays out in your own token**: earners post content, out of a wallet you fund with your token. In `auto` mode, the reward is calculated once from engagement (capped per post) and then only pays out if your token's price actually moves — see below.

## How earners get paid — use `agent` mode

**Recommended: `release_mode: "agent"`.** You (the agent running this skill) decide and release every payout yourself via `campaign.payout` instead of moltycash's own formula — no formula to reverse-engineer, full control over what counts as a qualifying post and what it's worth. `cpm_rate` is advisory only in this mode — a suggested rate, not a mechanical input; `max_payout_per_submission` is still a hard, enforced ceiling regardless of what you request.

### The agent workflow

`auto` mode has moltycash read view counts from the X API itself and compute the payout mechanically. `agent` mode hands that judgment call to you instead: read the post yourself, decide if it qualifies, decide what it's worth, and release the payout. Use the same wallet that created the campaign the whole way through — no separate delegation setup needed.

**1. Discover open submissions**

```bash
curl -s https://molty.cash/api/campaigns/{campaign_id}
```

Public, unauthenticated, free. Returns the campaign and every `submissions[]` entry with its `proof` URL and current `status`. Filter to submissions not already `paid`, `rejected`, `payout_failed`, or `spam` — those are terminal.

**2. Verify each open submission**

Read the submission's `proof` URL (an X post). Judge whether it actually satisfies the campaign's `description` — right format, on-topic, not spam or a duplicate.

**Not compliant** → reject it (below). **Compliant** → decide a payout amount using your own judgment (views, engagement quality, anything — `cpm_rate` is only a suggestion). X's public impression-count visibility is inconsistent post-to-post, so when you're not confident in a number, **don't guess** — a submission can simply wait, call `campaign.payout` again next time you check. Nothing is lost by deferring; a wrong-but-confident number moves real money incorrectly and can't be clawed back.

**3. Act — pay or reject**

Both go through `campaign.payout`, paid 1¢ x402 per call:

```json
// Pay
{"jsonrpc":"2.0","id":1,"method":"campaign.payout","params":{"campaign_id":"cmp-...","submission_id":"sub-...","amount":5,"final":false}}

// Reject a non-compliant submission
{"jsonrpc":"2.0","id":1,"method":"campaign.payout","params":{"campaign_id":"cmp-...","submission_id":"sub-...","action":"reject"}}
```

- `amount` — the payout, in the campaign token's display units. Clamped server-side to whatever's left of `max_payout_per_submission`.
- `views` — optional. Only affects `min_views_threshold` (if set) and gets recorded for reference — never drives the amount.
- `final: true` — closes the submission out (no further payouts against it).
- Callable repeatedly as your assessment changes — each call adds to what's already been paid, still capped at the per-post ceiling.

**4. Cadence**

`campaign.payout` has **no server-side rate limit** of its own. A sensible default is to check once and pay once per submission when it's clearly done growing/settled, rather than repeatedly polling a platform that won't change.

### Also available: `release_mode: "auto"`

moltycash handles everything itself, no agent workflow needed. X posts only; moltycash reads impressions from the X API and pays automatically — **not daily top-ups**: a reward is calculated once and locked, then paid out based on your token's price move afterward.

1. **Submission window.** A post can't be submitted more than 6h after it went up.
2. **Owner veto window, ~2h.** Same as always — reject a bad submission within 2h of posting or it auto-approves.
3. **Calculation, ~8h after the post.** moltycash reads the post's metrics **once** and computes the reward via the engagement-scaled formula below — this becomes a fixed, final number (the "locked reward") that's never recomputed again. No further X reads happen for this submission.
4. **Price target.** The locked reward pays in **full** only if your token's price rises enough above where it was when the post went up, before `window_days` closes. The required increase is set by your token's market cap **at submission time** — smaller caps need a bigger move:

   | Market cap at submission | Required price increase |
   |---|---|
   | < $1M | +100% |
   | $1M – $10M | +50% |
   | $10M – $100M | +25% |
   | $100M – $1B | +10% |
   | $1B+ | none — pays as-is |

   If price moved up at all but never reached that target, a small consolation amount pays instead. If price never rose above the submission-time price at all, nothing pays.
5. **The engagement formula.** The locked amount is `min(views × effective_cpm / 1000, cap)`, where `effective_cpm` scales with engagement quality ((likes+RTs+replies)/views) between 25% and 100% of `cpm_rate` — organically engaged posts earn near the full rate, low-engagement/botted posts earn down toward the floor.
6. **Cap + window.** The locked reward (and the consolation amount) never exceed `max_payout_per_submission`. Everything resolves by the end of `window_days` (default 2) — a post either hits its price target, gets the consolation amount, or resolves to nothing.

Use `auto` if you specifically want X-only, zero-touch settlement instead of running the agent workflow.

---

## How to use — read this **first**, blocking

Creating a campaign requires payment. Payment requires a wallet. **Before doing anything else**, walk through these three steps in order. Do not skip; do not assume `npx moltycash` is the default.

### Step 1 — Decide which wallet to use

**Ask the human you are working for which wallet to pay with.** Do not auto-detect, do not default to `*_PRIVATE_KEY` env vars. Wallet selection is a deliberate choice. This rule applies to every paid moltycash method.

Skip the ask only if either:

- the human's original prompt already named a wallet — use what they specified, **don't substitute**; or
- only one wallet is authenticated / available — use it.

### Step 2 — Fetch that wallet's transport doc

Each catalog wallet has its own transport syntax — there is no universal CLI for paying moltycash. moltycash settles via **x402 only** (Base, Solana) — 11 wallets in the catalog can pay it:

`bankr` · `circle` · `lobstercash` · `solid` · `awal` · `purl` · `agentcash` · `clawcash-cli` · `onchainos` · `moonpay` · `pay-sh`

(The catalog also lists `link-cli` — Stripe/fiat via MPP — but moltycash doesn't accept Stripe/MPP payments, so it can't pay moltycash specifically.)

Fetch the doc for the wallet the human picked:

```bash
curl https://molty.cash/skills/agentic-wallets/wallets/<wallet>.md
```

For the full catalog, see [agentic-wallets/SKILL.md](https://molty.cash/skills/agentic-wallets/SKILL.md). For the moltycash CLI fallback (private key), see [PAYMENT.md](https://molty.cash/skills/PAYMENT.md).

### Step 3 — Combine wallet transport + the canonical `shill.create` payload

The payload is **identical for every wallet** — it's the JSON-RPC body posted to `https://api.molty.cash/a2a`:

```json
{"jsonrpc":"2.0","id":1,"method":"shill.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"description":"Post an original thread about our launch","token_contract":"0x...","window_days":2,"release_mode":"agent"}}
```

`description` and `token_contract` are **always required**. `cpm_rate`/`max_payout_per_submission` are optional together — pass both, or omit both to auto-price `cpm_rate` at $1 worth of the token (`max_payout_per_submission` then defaults to `cpm_rate` × 10). Passing `max_payout_per_submission` without `cpm_rate` is rejected. See the full param table below.

Total to authorise: flat **$1 USDC**. Billing is commission-only — nothing else to prepay (see the Fee section below).

Substitute that payload into the transport pattern from the wallet's doc.

---

## Create + fund

### `shill.create`

```json
{"jsonrpc":"2.0","id":1,"method":"shill.create","params":{"cpm_rate":5,"max_payout_per_submission":50,"description":"Post an original thread about our launch","token_contract":"0x...","window_days":2,"release_mode":"agent"}}
```

Required: `description`, `token_contract`.

| Param | Meaning |
|---|---|
| `token_contract` | **Required.** SPL mint (Solana) or ERC-20 address (Base) — no default, no USDC fallback. The payout chain is inferred from the address format (`0x...` = Base, base58 = Solana) |
| `cpm_rate` | Basis for the **locked reward** in `auto` mode — the MAX payout tokens per 1,000 views. The effective rate scales with engagement ((likes+RTs+replies) / views): organically engaged posts earn near this, low-engagement/botted posts earn down to 25% of it. The per-post cap scales the same way. Whether that locked amount actually pays out (in full, partially, or not at all) depends on your token's price afterward — see the auto-mode section above. In `agent` mode this is advisory only. **Optional together with `max_payout_per_submission`** — pass both, or omit both to auto-price `cpm_rate` at $1 worth of the payout token (DexScreener-priced). Passing `max_payout_per_submission` without `cpm_rate` is rejected. |
| `max_payout_per_submission` | Hard cap on the locked reward (reserved from your payout-token funding per submission). **Optional if `cpm_rate` is also omitted** — defaults to `cpm_rate` × 10 (a post needs ~10,000 views to hit the cap) whenever `cpm_rate` is supplied (explicitly or auto-priced). |
| `min_holder_amount` | **Optional.** Minimum amount of the campaign token the earner must hold at their payout address **to submit and receive each payout**. Defaults to $5 worth of the token (computed at creation via DexScreener). Pass 0 to disable. Display units (same token, same decimals). |
| `min_followers` | **Optional.** Minimum X follower count the earner must have. 0 / omit = no requirement. |
| `min_account_age_days` | **Optional.** Earner's X account must be at least this many days old. 0 / omit = no requirement. |
| `require_premium` | **Optional.** Earner's X account must be Premium to submit. `false` / omit = no requirement. |
| `min_views_threshold` | **Optional.** Post must reach this view count before payout fires. Does not block submission — payout defers until views clear the floor (or campaign is force-closed). 0 / omit = no floor. |
| `window_days` | Tracking window in days (default 2, 1–30) — in `auto` mode this bounds both the calculation checkpoint and the price-target check |
| `release_mode` | **Recommended: `agent`** — you decide and release every payout yourself via `campaign.payout` instead of moltycash's formula. Also available: `auto` — moltycash reads X impressions and pays automatically, zero-touch (see the price-target mechanics above). |
| `post_type` | **Optional.** Restrict submissions to a specific X post format: `x_post`, `x_thread`, `x_quote`, `x_reply`, `x_short_video`, `x_long_video`, `x_article`. Omit for any format |

**Fee:** Flat **$1 USDC** to create — that's the *only* flat fee. Settlement work (view-checks + payouts) is otherwise free, and molty's ongoing revenue is purely the **3% commission** swept from the campaign wallet on each real earner payout, added on top of the earner amount (plan for ~3% more token funding than pure CPM math).

`shill.create` returns a `wallet_address` — **fund it by sending your token** to that address on the inferred payout chain.

CLI (moltycash): `moltycash shill create --cpm 5 --max 50 --token <addr> --window 2 --mode agent "Post about us"` (`--token` is required — SPL mint on Solana or ERC-20 0x address on Base, chain detected from the address; the ticker is resolved automatically from the token contract, no flag needed; `--mode agent` is recommended (see above) — omit `--mode` or pass `--mode auto` for X-only zero-touch settlement instead; `--min-hold <amount>` to require a token holding, `--min-followers <n>` for a follower floor, `--min-age <days>` for an account-age floor, `--min-views <n>` to defer payout until views clear that threshold).

---

## Other owner methods

| Method | Auth | What it does |
|---|---|---|
| `campaign.status` `{campaign_id}` | x402 (1¢) | Live wallet balance, committed/available token |
| `campaign.review` `{campaign_id, submission_id, action}` | x402 (1¢) | Owner `approve`/`reject` a submission (reject within the 2h window to veto; otherwise it auto-approves — `auto` mode only) |
| `campaign.payout` `{campaign_id, submission_id, amount}` | x402 (1¢) | agent mode: decide and release the payout amount yourself — clamped to `max_payout_per_submission`. Optionally pass `views` for `min_views_threshold`/record-keeping only (never drives the amount). Add `final:true` to close, or `action:"reject"` |
| `campaign.close` `{campaign_id}` | x402 (1¢) | Reject in-flight submissions, refund the wallet's remaining balance to the wallet you created the campaign with (falling back to your registered payout destination for this chain only if that wallet isn't valid on it), mark closed |
| `campaign.list` `{}` | x402 (1¢) | List the campaigns you own (resolved from whichever wallet pays the call) |

CLI (moltycash): `moltycash campaign status <id>` / `moltycash campaign review <id> <sub_id> <approve|reject>` / `moltycash campaign payout <id> <sub_id> --amount <n>` / `moltycash campaign close <id>` / `moltycash campaign list` — same `campaign` subcommand manages any campaign regardless of how it was created.

---

## Earner: discover + submit

There is no A2A method for earners. Discovering open campaigns and submitting a post both happen through the molty.cash **web dashboard** (X login required), not this API.
