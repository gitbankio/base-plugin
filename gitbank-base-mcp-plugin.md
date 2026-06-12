---
title: "Gitbank Plugin"
description: "Skill plugin reference for managing GitHub-linked soul-bound vaults on Base Mainnet via Coinbase Wallet (send_calls mode). Read operations are plain GET requests. Write operations queue a pending request, require GitHub identity confirmation, then return EIP-5792 calldata for the user to submit via wallet_sendCalls."
---

# Gitbank Plugin

Gitbank is an IssueOps platform for Web3 dev teams. Every GitHub account gets a soul-bound vault on Base Mainnet, anchored to the account's permanent GitHub user ID. Vaults hold USDC and WETH.

**This plugin uses send_calls mode by default.** No MCP server connection is required. Read operations are plain GET requests. Write operations (deposit, withdraw, swap) queue a pending command and return a confirm code. The user authorizes by posting one comment on GitHub. The bot verifies the commenter's identity: **only the exact GitHub account whose username was used in the prepare request can confirm it.** After confirmation, the bot posts an execute token URL in the GitHub thread. Your AI assistant fetches that URL to retrieve the EIP-5792 `wallet_sendCalls` payload, which the user submits via their Coinbase Wallet or Base Account.

**Chain:** Base Mainnet (`8453`)

**Supported tokens:** USDC (`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`), WETH (`0x4200000000000000000000000000000000000006`)

**Gitbank API base URL:** `https://gitbank.io/api/public`

---

## How it works (send_calls mode)

```
User: "Swap 50 USDC to WETH, GitHub username: alice"

1. AI calls GET /vault/by-github/alice
   -> vault deployed, USDC balance: 250.00

2. AI calls GET /prepare/swap?username=alice&amount=50&from_token=USDC&to_token=WETH&mode=send_calls
   -> { confirm_code: "mcp1a2b3c4d", confirm_url: "https://github.com/gitbankio/playground/discussions/4#new_comment_form" }

3. AI shows the user:
   "To authorize, open: https://github.com/gitbankio/playground/discussions/4#new_comment_form
    And post: @gitbankbot confirm mcp1a2b3c4d
    (Expires in 10 minutes. Only @alice can confirm it.)"

4. User opens the link and posts the comment on GitHub as @alice.

5. Gitbank bot detects the comment.
   SECURITY CHECK: If the commenter is NOT @alice, the bot rejects it:
   "This command was requested by @alice. Only they can confirm it."

6. If identity confirmed: bot posts in GitHub thread:
   "Identity confirmed. Your signed transaction is ready.
    GET https://gitbank.io/api/public/execute/exec9f8e7d6c5b4a"

7. AI calls GET /execute/exec9f8e7d6c5b4a
   -> { calls: [{ to: "0x...", data: "0x...", value: "0x0" }] }

8. AI shows the user the calls array and instructs them to submit via wallet_sendCalls
   from their Coinbase Wallet or Base Account.
```

**Identity guarantee:** GitHub webhook payloads are HMAC-signed by GitHub. The bot reads sender identity from the signed payload only. It cannot be spoofed by calling the API directly.

---

## Read Endpoints

### `GET /api/public/vault/by-github/:github_username`

Returns the vault address and current USDC + WETH balances. **Always call this first** to confirm the vault is deployed and to check available balance.

```
GET https://gitbank.io/api/public/vault/by-github/alice
```

Response (vault deployed):

```json
{
  "github_username": "alice",
  "vault_address": "0x...",
  "vault_deployed": true,
  "balances": {
    "USDC": "250.00",
    "WETH": "0.050000"
  },
  "chain": "base",
  "chain_id": 8453
}
```

Response (vault not yet deployed):

```json
{
  "github_username": "alice",
  "vault_deployed": false,
  "balances": {}
}
```

If `vault_deployed` is `false`, proceed normally. The vault auto-deploys on the first prepare request (free, on-chain).

### `GET /api/public/vault/:vault_address`

Returns balances for a known vault address directly.

```
GET https://gitbank.io/api/public/vault/0x70Bf2cac89926f6E1a5592BB4b2645377e097495
```

---

## Prepare Endpoints

Prepare endpoints queue a pending vault operation and return a confirm code. The operation is not signed or executed until the user confirms on GitHub as the correct account. Codes expire after 10 minutes.

**Always append `&mode=send_calls` to every prepare request.** This ensures the user submits the transaction themselves via Coinbase Wallet instead of the relayer.

> [!NOTE]
> If `gitbank.io` is not on your client's fetch allowlist, construct the full URL, show it to the user, ask them to paste the JSON response back into chat, then read the `instructions` field and show it to the user.

### `GET /api/public/prepare/deposit`

Queues a deposit of USDC or WETH into the vault.

```
GET https://gitbank.io/api/public/prepare/deposit?username=alice&amount=50&token=USDC&mode=send_calls
```

| Param | Required | Description |
|-------|----------|-------------|
| `username` | yes | GitHub username |
| `amount` | yes | Human-decimal amount (e.g. `50` for 50 USDC, `0.001` for 0.001 WETH) |
| `token` | yes | `USDC` or `WETH` |
| `mode` | yes | Always `send_calls` for this plugin |

### `GET /api/public/prepare/withdraw`

Queues a withdrawal from the vault to a wallet address.

```
GET https://gitbank.io/api/public/prepare/withdraw?username=alice&amount=50&token=USDC&to=0x...&mode=send_calls
```

| Param | Required | Description |
|-------|----------|-------------|
| `username` | yes | GitHub username |
| `amount` | yes | Human-decimal amount |
| `token` | yes | `USDC` or `WETH` |
| `to` | yes | Destination wallet address |
| `mode` | yes | Always `send_calls` for this plugin |

A 0.1% protocol fee applies.

### `GET /api/public/prepare/swap`

Queues a Uniswap v3 swap inside the vault.

```
GET https://gitbank.io/api/public/prepare/swap?username=alice&amount=50&from_token=USDC&to_token=WETH&mode=send_calls
```

| Param | Required | Description |
|-------|----------|-------------|
| `username` | yes | GitHub username |
| `amount` | yes | Human-decimal amount of `from_token` |
| `from_token` | yes | `USDC` or `WETH` |
| `to_token` | yes | `USDC` or `WETH` (must differ from `from_token`) |
| `mode` | yes | Always `send_calls` for this plugin |

A 0.3% protocol fee applies.

**All prepare endpoints return the same shape:**

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

**Always show the `instructions` field verbatim to the user.** It contains the exact comment they need to post.

---

## Execute Endpoint (send_calls mode)

After the user confirms on GitHub and the bot posts the execute token URL in the thread, fetch the EIP-5792 calldata:

### `GET /api/public/execute/:token`

```
GET https://gitbank.io/api/public/execute/exec9f8e7d6c5b4a
```

Returns:

```json
{
  "ok": true,
  "command": "swap",
  "calls": [
    { "to": "0x...", "data": "0x...", "value": "0x0" }
  ]
}
```

Pass the `calls` array to `wallet_sendCalls` (EIP-5792) from the user's Coinbase Wallet or Base Account.

**Token is single-use and expires 10 minutes after confirmation.**

---

## Orchestration Pattern

```
1. GET /vault/by-github/:username
   -> note vault_address and balances (vault auto-deploys if not yet deployed)

2. GET /prepare/<deposit|withdraw|swap>?username=<username>&...&mode=send_calls
   -> confirm_code, instructions, confirm_url

3. Show the user the instructions field verbatim.
   Tell them to open confirm_url and post the shown comment.
   Remind them: only their GitHub account can confirm this.

4. Wait for the user to say they confirmed on GitHub.

5. Tell the user: "Once the bot verifies your identity on GitHub, it will post an
   execute token URL in the thread. Paste that URL here and I will fetch the calldata."

6. User pastes the execute URL (or the bot auto-posts it in the GitHub thread).

7. GET /execute/:token -> calls array

8. Show the user the calls and instruct them to submit via wallet_sendCalls
   from their Coinbase Wallet or Base Account on Base Mainnet (chainId 8453).
```

---

## Example Sessions

**Check vault balance**

```
What's in my Gitbank vault? My GitHub username is alice.
```

1. `GET /vault/by-github/alice` -> show USDC and WETH balances.

---

**Swap 50 USDC to WETH (send_calls)**

```
Swap 50 USDC to WETH in my Gitbank vault. GitHub: alice.
```

1. `GET /vault/by-github/alice` -> vault_address, confirm USDC balance >= 50.
2. `GET /prepare/swap?username=alice&amount=50&from_token=USDC&to_token=WETH&mode=send_calls` -> confirm_code.
3. Show user the instructions (open GitHub link, post the confirm comment as @alice).
4. Wait. Bot verifies @alice posted the comment. If identity mismatch, bot rejects.
5. Bot posts execute URL in GitHub thread.
6. `GET /execute/:token` -> calls array.
7. User submits via wallet_sendCalls.

---

**Withdraw 50 USDC to wallet**

```
Withdraw 50 USDC from alice's Gitbank vault to 0x1234...
```

1. `GET /vault/by-github/alice` -> vault_address, confirm USDC balance >= 50.
2. `GET /prepare/withdraw?username=alice&amount=50&token=USDC&to=0x1234...&mode=send_calls` -> confirm_code.
3. Show user the instructions.
4. Bot verifies identity. Bot posts execute URL.
5. `GET /execute/:token` -> calls array. User submits.

---

**Deposit 100 USDC**

```
Deposit 100 USDC into my Gitbank vault. GitHub: alice.
```

1. `GET /vault/by-github/alice` -> vault_address.
2. `GET /prepare/deposit?username=alice&amount=100&token=USDC&mode=send_calls` -> confirm_code.
3. Show user the instructions.
4. Bot verifies identity. Bot posts execute URL.
5. `GET /execute/:token` -> calls array. User submits.

---

## Operation Summary

| Operation | Prepare endpoint | Auth method | Who pays gas |
|-----------|-----------------|-------------|-------------|
| Deposit USDC/WETH | `/prepare/deposit?...&mode=send_calls` | GitHub confirm (identity verified) | User (Coinbase Wallet) |
| Withdraw USDC/WETH | `/prepare/withdraw?...&mode=send_calls` | GitHub confirm (identity verified) | User (Coinbase Wallet) |
| Swap USDC to WETH | `/prepare/swap?...&mode=send_calls` | GitHub confirm (identity verified) | User (Coinbase Wallet) |

---

## Security Model

- **GitHub identity is mandatory.** The confirm code is bound to a specific GitHub username. Only that account can post the confirm comment. If a different account tries, the bot rejects with: "This command was requested by @alice. Only they can confirm it."
- **No calldata before identity check.** The prepare endpoint returns only a confirm code. No transaction is signed or calldata generated until the Gitbank bot verifies the commenter via HMAC-signed GitHub webhook.
- **Destination address is locked.** For withdrawals, the destination is embedded in the ownerSig verified by the vault contract. Calldata cannot redirect funds to a different address.
- **Execute token is single-use and atomic.** The token is claimed via a single atomic database UPDATE. Two simultaneous requests cannot both succeed. Token expires in 10 minutes.

---

## Error Handling

| HTTP | `error` field | Meaning |
|------|--------------|---------|
| 400 | `"username, amount, and token are required"` | Missing query param |
| 400 | `"Invalid destination address"` | `to` is not a valid EVM address |
| 400 | `"Unsupported token. Use USDC or WETH"` | Unknown token symbol |
| 400 | `"from_token and to_token must differ"` | Swap source equals destination |
| 404 | `"User not found"` | GitHub username not in Gitbank |
| 404 | `"Execute token not found or already used"` | Token claimed or expired |

---

## Notes

- GitHub username lookup is case-insensitive.
- Vaults auto-deploy on the first prepare request. No prior setup at gitbank.io is required.
- Only the GitHub user whose username was used in the prepare request can confirm it on GitHub. The bot verifies the commenter's identity via HMAC-signed GitHub webhook payload.
- Confirm codes expire in 10 minutes. If expired, repeat the prepare request to get a fresh code.
- Execute tokens expire 10 minutes after the GitHub confirmation and are single-use.
- GitVaultFactory on Base Mainnet: `0xAA0a4ff46733EBaE8E658642A1314f18980fc77B`
- Basescan: `https://basescan.org/tx/<txHash>`
- For relayer mode (Gitbank pays gas, no wallet needed), use the MCP server at `https://gitbank.io/api/mcp` instead.
