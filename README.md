# gitbank-base-plugin

Gitbank plugin for AI clients. Lets any AI assistant with HTTP access read balances and queue vault operations (deposit, withdraw, swap) on GitHub-linked soul-bound vaults on Base Mainnet. Write operations are authorized by posting one comment on GitHub.

No MCP server connection required. No API key. No Base Account or gas required.

---

## How it works

```
AI reads vault state
  GET https://gitbank.io/api/public/vault/by-github/:username

AI queues the operation
  GET https://gitbank.io/api/public/prepare/deposit|withdraw|swap?username=...&amount=...

AI shows the user a confirm code and GitHub link
  "Open https://github.com/gitbankio/playground/discussions/4#new_comment_form
   And post: @gitbankbot confirm mcp1a2b3c4d"

User posts one comment on GitHub as themselves
  Gitbank bot verifies identity, signs, and submits via relayer

Bot replies on GitHub with tx hash and Basescan link
```

Nothing is signed or submitted until the user authorizes on GitHub. The bot verifies that the GitHub account posting the confirm comment matches the vault owner. Gitbank's relayer covers all gas costs.

---

## What this repo contains

| File | Purpose |
|------|---------|
| `gitbank-base-mcp-plugin.md` | Full plugin spec. Load or paste this into your AI session to enable all Gitbank operations. |
| `SKILL.md` | Condensed skill spec with frontmatter, endpoint reference, orchestration pattern, and example flows. |

---

## Quick start

Two ways to load the plugin:

### Option A — Upload plugin (permanent, Claude only)

Works in Claude Desktop and claude.ai. Install once, active in every session automatically.

1. Download [gitbank-base-mcp-plugin.md](https://gitbank.io/api/public/plugin/download)
2. Open Claude Desktop or claude.ai
3. Click **Customize** → **Personal plugins** → **Create plugin** → **Upload plugin**
4. Upload the file

Done. Gitbank is now a permanent plugin in Claude.

### Option B — Paste URL (any AI client, per session)

Works in Claude, Cursor, ChatGPT, or any AI with HTTP access. Paste at the start of each chat:

```
Read this skill spec and follow it for all Gitbank operations:
https://raw.githubusercontent.com/gitbankio/base-plugin/main/gitbank-base-mcp-plugin.md
```

---

## API reference

Base URL: `https://gitbank.io/api/public`

All endpoints are public GET requests. No authentication required.

### Read

| Endpoint | Description |
|----------|-------------|
| `GET /vault/by-github/:username` | Vault address and USDC + WETH balances by GitHub username |
| `GET /vault/:vault_address` | Balances by vault address |

### Prepare (queue and confirm on GitHub)

| Endpoint | Required params | Fee |
|----------|----------------|-----|
| `GET /prepare/deposit` | `username`, `amount`, `token` | none |
| `GET /prepare/withdraw` | `username`, `amount`, `token`, `to` | 0.1% |
| `GET /prepare/swap` | `username`, `amount`, `from_token`, `to_token` | 0.3% |

All prepare responses include:

```json
{
  "ok": true,
  "command": "swap",
  "username": "alice",
  "vault_address": "0x...",
  "confirm_code": "mcp1a2b3c4d",
  "instructions": "...",
  "confirm_url": "https://github.com/gitbankio/playground/discussions/4#new_comment_form",
  "expires_in_seconds": 600
}
```

Show the `instructions` field verbatim to the user. The confirm code expires in 10 minutes.

---

## Supported operations and tokens

| Operation | Token(s) |
|-----------|---------|
| Deposit | USDC, WETH |
| Withdraw | USDC, WETH |
| Swap | USDC to WETH, WETH to USDC |

---

## Chain and contracts

| Item | Address |
|------|---------|
| Chain | Base Mainnet (chainId 8453) |
| GitVaultFactory | `0xAA0a4ff46733EBaE8E658642A1314f18980fc77B` |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| WETH | `0x4200000000000000000000000000000000000006` |

Contracts are verified on Basescan:
- Factory: https://basescan.org/address/0xAA0a4ff46733EBaE8E658642A1314f18980fc77B#code
- Vault impl: https://basescan.org/address/0x3602197A1b445AA4746c47C9D69436d9B7cF5dc9#code

---

## Security model

Write operations are never signed until the user confirms on GitHub. The Gitbank bot verifies that the GitHub account posting the confirm comment matches the vault owner. This means:

- An attacker who calls a prepare endpoint cannot execute anything without access to the victim's GitHub account.
- The confirm code is single-use and expires in 10 minutes.
- Gitbank vaults are anchored to GitHub permanent user IDs (not Ethereum addresses). The vault contract validates `ownerSig` and `relayerSig` independently on-chain.

---

## License

Apache-2.0. See [LICENSE](LICENSE).
