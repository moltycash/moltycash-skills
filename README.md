# moltycash skills

Agent skills for [molty.cash](https://molty.cash) — send USDC payments and run pay-per-view (CPM) content campaigns via [x402](https://x402.org) on Base and Solana.

## Skills

Each skill is a folder containing `SKILL.md` (and optional `references/`, `scripts/`, `assets/`) per the [agentskills.io](https://agentskills.io/specification) spec. Source layout matches the URL path served at `https://molty.cash/skills/…`.

| Skill | Description |
|-------|-------------|
| [campaign](./skills/campaign/SKILL.md) | Create (USDC-default) and manage pay-per-view (CPM) content campaigns — earners post about your token/brand and earn per 1,000 views |
| [shill](./skills/shill/SKILL.md) | Same campaign type as `campaign`, but create via `shill.create` — pay out in your own token, no USDC fallback |
| [PAYMENT](./skills/PAYMENT.md) | Pay moltycash — canonical payloads (hire / campaign.create / shill.create), fees, settlement chains, moltycash CLI fallback |
| [agentic-wallets](./skills/agentic-wallets/SKILL.md) | Generic x402 (and Stripe/MPP via link-cli) wallet detection + transport — 12 wallets across Base and Solana |

Hiring a specific person is served dynamically at `https://molty.cash/{username}/SKILL.md` per user.

## Protocols

- **A2A** (Agent-to-Agent): `POST https://api.molty.cash/a2a`
- **MCP** (Model Context Protocol): `POST https://api.molty.cash/mcp`
- **CLI**: `npx moltycash <command>`

## License

MIT
