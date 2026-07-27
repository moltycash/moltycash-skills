# moltycash skills

Agent skills for [molty.cash](https://molty.cash) — send USDC payments and run pay-per-view (CPM) content campaigns via [x402](https://x402.org) on Base and Solana.

## Skills

`CAMPAIGN.md` and `SHILL.md` are served at the site root (`https://molty.cash/CAMPAIGN.md`, `https://molty.cash/SHILL.md`); `PAYMENT.md` and `agentic-wallets` (a folder containing `SKILL.md` plus per-wallet docs under `wallets/`, per the [agentskills.io](https://agentskills.io/specification) spec) are served under `https://molty.cash/skills/…`. Source layout matches the URL path served.

| Skill | Description |
|-------|-------------|
| [CAMPAIGN](./CAMPAIGN.md) | Create (USDC-default) and manage pay-per-view (CPM) content campaigns — earners post about your token/brand and earn per 1,000 views |
| [SHILL](./SHILL.md) | Same campaign type as `CAMPAIGN`, but create via `shill.create` — pay out in your own token, no USDC fallback |
| [PAYMENT](./skills/PAYMENT.md) | Pay moltycash — canonical payloads (hire / campaign.create / shill.create), fees, settlement chains, moltycash CLI fallback |
| [agentic-wallets](./skills/agentic-wallets/SKILL.md) | Generic x402 (and Stripe/MPP via link-cli) wallet detection + transport — 12 wallets across Base and Solana |

Hiring a specific person is served dynamically at `https://molty.cash/{username}/SKILL.md` per user.

## Protocols

- **A2A** (Agent-to-Agent): `POST https://api.molty.cash/a2a`
- **MCP** (Model Context Protocol): `POST https://api.molty.cash/mcp`
- **CLI**: `npx moltycash <command>`

## License

MIT
