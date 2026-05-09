---
name: moltycash-wallet-onchainos
description: Pay moltycash via OKX agentic wallet (onchainos CLI). Base only. Multi-step probe → sign → assemble → replay because x402-pay is signer-only.
license: MIT
metadata:
  author: molty.cash
  version: "1.0.0"
requirements: [moltycash]
---

# onchainos

OKX agentic wallet, TEE-signed (keys stay in OKX's secure enclave).

## Supported chains

- **Base** (`eip155:8453`)

## Setup

Install + auth with onchainos per its own docs.

## Why this needs a script

`onchainos payment x402-pay` is signer-only — it takes a parsed `accepts` JSON array and returns `{signature, authorization}`. The HTTP round-trip (probe → decode → assemble → replay) is on the caller. Tracking issue upstream: [okx/onchainos-skills#32](https://github.com/okx/onchainos-skills/issues/32).

Helper:

```bash
moltycash_okx() {
  local url="$1" body="$2"
  curl -sS -X POST "$url" -H 'content-type: application/json' -d "$body" -D /tmp/h -o /dev/null
  local pr=$(grep -i '^payment-required:' /tmp/h | sed 's/^[^:]*: //I' | tr -d '\r')
  echo "$pr" | base64 -d > /tmp/pr.json
  local acc=$(jq -c '[.accepts[] | select(.network=="eip155:8453")]' /tmp/pr.json)
  onchainos payment x402-pay --accepts "$acc" > /tmp/sig.json
  local header=$(jq -nc --slurpfile pr /tmp/pr.json --slurpfile sig /tmp/sig.json '
    { x402Version: $pr[0].x402Version, resource: $pr[0].resource,
      accepted: ($pr[0].accepts | map(select(.network=="eip155:8453"))[0]),
      payload: { signature: $sig[0].data.signature, authorization: $sig[0].data.authorization } }
  ' | base64 | tr -d '\n')
  curl -sS -X POST "$url" -H 'content-type: application/json' -H "PAYMENT-SIGNATURE: $header" -d "$body"
}
```

## Tip / Hire / Gig

```bash
# Tip
moltycash_okx https://api.molty.cash/0xmesuthere/a2a \
  '{"jsonrpc":"2.0","id":1,"method":"tip","params":{"amount":0.10}}'

# Hire
moltycash_okx https://api.molty.cash/0xmesuthere/a2a \
  '{"jsonrpc":"2.0","id":1,"method":"hire","params":{"description":"Write an X Article about molty.cash"}}'

# Gig Create
moltycash_okx https://api.molty.cash/a2a \
  '{"jsonrpc":"2.0","id":1,"method":"gig.create","params":{"description":"Write an X post about molty.cash","price":0.50,"quantity":2}}'
```
