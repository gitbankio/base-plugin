---
name: gitbank
description: "Execute vault operations (deposit, withdraw, swap) and check balances on Gitbank GitHub-linked soul-bound vaults on Base Mainnet. Write operations are queued via the prepare endpoints and authorized by the user posting a confirm code on GitHub. No MCP server connection required."
---

# Gitbank Skill

Gitbank is an IssueOps platform for Web3 dev teams. Every GitHub account gets a soul-bound vault on Base Mainnet, anchored to the account's permanent GitHub user ID. The vault holds USDC and WETH on Base Mainnet.

This skill teaches the AI how to:
1. Call Gitbank's public REST API to read vault state.
2. Queue vault operations and return a GitHub confirm code to the user.
3. The user authorizes by posting one comment on GitHub. Gitbank's relayer then executes the transaction.

**Chain:** Base Mainnet (chainId 8453)
**Supported tokens:** USDC, WETH
**API base URL:** `https://gitbank.io/api/public`
**No API key required. No MCP server connection required.**

---

## Read endpoints

### GET /api/public/vault/by-github/:username

Returns the vault address and current USDC and WETH balances for a GitHub username.

Always call this first to confirm the vault is deployed before any operation.

```json
{
  "github_username": "alice",
  "vault_address": "0x...",
  "vault_deployed": true,
  "balances": { "USDC": "250.00", "WETH": "0.050000" },
  "chain": "base",
  "chain_id": 8453
}
```

If `vault_deployed` is `false`, proceed normally. The vault auto-deploys on the first prepare request (free, on-chain, no gitbank.io required).

### GET /api/public/vault/:vault_address

Returns USDC and WETH balances for a known vault address directly.

---

## Prepare endpoints

Prepare endpoints queue a pending vault operation and return a confirm code. Nothing is signed or submitted until the user authorizes on GitHub. The code expires in 10 minutes.

### GET /api/public/prepare/deposit

Query params:

| Param | Required | Description |
|-------|----------|-------------|
| `username` | yes | GitHub username |
| `amount` | yes | Human-decimal amount (e.g. `100` for 100 USDC, `0.001` for 0.001 WETH) |
| `token` | yes | `USDC` or `WETH` |

### GET /api/public/prepare/withdraw

Query params:

| Param | Required | Description |
|-------|----------|-------------|
| `username` | yes | GitHub username |
| `amount` | yes | Human-decimal amount |
| `token` | yes | `USDC` or `WETH` |
| `to` | yes | Destination wallet address |

A 0.1% protocol fee applies.

### GET /api/public/prepare/swap

Query params:

| Param | Required | Description |
|-------|----------|-------------|
| `username` | yes | GitHub username |
| `amount` | yes | Human-decimal amount of `from_token` |
| `from_token` | yes | `USDC` or `WETH` |
| `to_token` | yes | `USDC` or `WETH` (must differ from `from_token`) |

A 0.3% protocol fee applies. Routes through Uniswap v3 inside the vault.

**Prepare response shape:**

```json
{
  "ok": true,
  "command": "swap",
  "username": "alice",
  "vault_address": "0x...",
  "amount": 50,
  "from_token": "USDC",
  "to_token": "WETH",
  "confirm_code": "mcp1a2b3c4d",
  "instructions": "Swap 50 USDC to WETH in @alice's vault queued.\n\nTo authorize, open:\nhttps://github.com/gitbankio/playground/discussions/4#new_comment_form\n\nAnd post this comment:\n@gitbankbot confirm mcp1a2b3c4d\n\n(Expires in 10 minutes. Only @alice can confirm it.)",
  "confirm_url": "https://github.com/gitbankio/playground/discussions/4#new_comment_form",
  "expires_in_seconds": 600
}
```

---

## Orchestration pattern

```
1. GET /api/public/vault/by-github/:username
   Note vault_address and balances (vault auto-deploys if not yet deployed).

2. GET /api/public/prepare/<deposit|withdraw|swap>?username=<username>&...
   Receive { confirm_code, instructions, confirm_url }.

3. Show the user the instructions field verbatim.
   Tell them to open confirm_url and post the comment shown.

4. Wait for the user to say they confirmed.

5. Tell the user the transaction will execute within 30 seconds.
   They can check their vault balance or Gitbank dashboard for the result.
```

---

## Example prompts and tool flows

**Check vault balance**
```
User: What is in my Gitbank vault? My GitHub username is alice.
AI:   GET /api/public/vault/by-github/alice
      -> vault_address: 0x93A3...899D
         USDC: 250.00, WETH: 0.050000
```

**Swap 50 USDC to WETH**
```
User: Swap 50 USDC to WETH in my vault. GitHub: alice.
AI:   GET /api/public/vault/by-github/alice         (confirm USDC balance >= 50)
      GET /api/public/prepare/swap?username=alice&amount=50&from_token=USDC&to_token=WETH
      -> confirm_code: mcp1a2b3c4d
      Show user the instructions (open GitHub link, post @gitbankbot confirm mcp1a2b3c4d)
      Wait for user to confirm
      -> "Transaction queued. The relayer will execute within 30 seconds."
```

**Deposit 100 USDC**
```
User: Deposit 100 USDC into my vault. GitHub: alice.
AI:   GET /api/public/vault/by-github/alice
      GET /api/public/prepare/deposit?username=alice&amount=100&token=USDC
      -> Show confirm instructions
```

**Withdraw 50 USDC to wallet**
```
User: Withdraw 50 USDC from my vault to 0x1234...abcd. GitHub: alice.
AI:   GET /api/public/vault/by-github/alice         (confirm USDC balance >= 50)
      GET /api/public/prepare/withdraw?username=alice&amount=50&token=USDC&to=0x1234...abcd
      -> Show confirm instructions to user
```

---

## Optional: send_calls mode

Add `?mode=send_calls` to any prepare endpoint to get an EIP-5792 `wallet_sendCalls` payload instead of relayer execution. Use this only when the user explicitly wants to sign and submit via Coinbase Wallet or Base Account (they pay gas). Default relayer mode (zero gas cost) is correct for most users.

After GitHub confirm, the bot posts an execute token URL in the GitHub thread. Call:

```
GET https://gitbank.io/api/public/execute/:token
```

Returns `{ ok: true, command, calls: [{ to, data, value }] }`. Pass `calls` to `wallet_sendCalls`. Token is single-use, expires in 10 minutes.

---

## Notes

- GitHub username lookup is case-insensitive.
- Vaults auto-deploy on the first prepare request. No prior setup at gitbank.io is required.
- The confirm code expires in 10 minutes. If expired, repeat the prepare request to get a fresh code.
- Only the GitHub user whose username was used to prepare the operation can confirm it. The bot checks this.
- After confirmation, Gitbank's relayer signs and submits the transaction. The user pays no gas.
- The confirm link goes to: `https://github.com/gitbankio/playground/discussions/4#new_comment_form`
- GitVaultFactory on Base Mainnet: `0xAA0a4ff46733EBaE8E658642A1314f18980fc77B`
- USDC on Base Mainnet: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- WETH on Base Mainnet: `0x4200000000000000000000000000000000000006`
