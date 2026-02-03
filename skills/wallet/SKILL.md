---
name: Agent Wallet
description: The agent's wallet. Use this skill to safely create a wallet the agent can use for transfers, swaps, and any EVM chain transaction.
---

# Agent Wallet

Use this skill to safely create a wallet the agent can use for transfers, swaps, and any EVM chain transaction without ever exposing private keys to the agent. Create a wallet, set spending policies, and your agent can transfer tokens, do swaps, and interact with smart contracts within the boundaries you define.

**The agent never sees the private key.** All transactions are executed server-side through a smart account. The wallet owner controls what the agent can do via configurable policies.

## Configuration

All API requests require a Bearer token (the API key returned when creating a wallet). If you're an openclaw instance, store and retrieve it from `~/.openclaw/credentials/agentwallet/<API_KEY_ID>.json`. Otherwise, you can store it in your current working directory at `agentwallet/<API_KEY_ID>.json`.

```
Authorization: Bearer <API_KEY>
```

## Quick Start

### 1. Create a Wallet

Create a new smart account wallet for your agent. This generates a private key server-side (you never see it), creates a ZeroDev smart account, and returns an API key for the agent plus a claim URL for the wallet owner.

```bash
curl -X POST "https://heyvincent.ai/api/secrets" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "EVM_WALLET",
    "memo": "My agent wallet",
    "chainId": 84532
  }'
```

Response includes:

- `apiKey` -- store this securely; use it as the Bearer token for all future requests
- `claimUrl` -- share this with the user so they can claim the wallet and set policies
- `address` -- the smart account address

After creating, tell the user:

> "Here is your wallet claim URL: `<claimUrl>`. Use this to claim ownership, set spending policies, and monitor your agent's wallet activity."

### 2. Get Wallet Address

```bash
curl -X GET "https://heyvincent.ai/api/skills/evm-wallet/address" \
  -H "Authorization: Bearer <API_KEY>"
```

### 3. Check Balances

```bash
# Get all token balances across all supported chains (ETH, WETH, USDC, etc.)
curl -X GET "https://heyvincent.ai/api/skills/evm-wallet/balances" \
  -H "Authorization: Bearer <API_KEY>"

# Filter to specific chains (comma-separated chain IDs)
curl -X GET "https://heyvincent.ai/api/skills/evm-wallet/balances?chainIds=1,137,42161" \
  -H "Authorization: Bearer <API_KEY>"
```

Returns all ERC-20 tokens and native balances with symbols, decimals, logos, and USD values.

### 4. Transfer ETH or Tokens

```bash
# Transfer native ETH
curl -X POST "https://heyvincent.ai/api/skills/evm-wallet/transfer" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "0xRecipientAddress",
    "amount": "0.01"
  }'

# Transfer ERC-20 token
curl -X POST "https://heyvincent.ai/api/skills/evm-wallet/transfer" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "0xRecipientAddress",
    "amount": "100",
    "token": "0xTokenContractAddress"
  }'
```

### 5. Swap Tokens

Swap one token for another using DEX liquidity (powered by 0x).

```bash
# Preview a swap (no execution, just pricing)
curl -X POST "https://heyvincent.ai/api/skills/evm-wallet/swap/preview" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "sellToken": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE",
    "buyToken": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "sellAmount": "0.1",
    "chainId": 1
  }'

# Execute a swap
curl -X POST "https://heyvincent.ai/api/skills/evm-wallet/swap/execute" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "sellToken": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE",
    "buyToken": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "sellAmount": "0.1",
    "chainId": 1,
    "slippageBps": 100
  }'
```

- `sellToken` / `buyToken`: Token contract addresses. Use `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` for native ETH.
- `sellAmount`: Human-readable amount to sell (e.g. `"0.1"` for 0.1 ETH).
- `chainId`: The chain to swap on (1 = Ethereum, 137 = Polygon, 42161 = Arbitrum, 10 = Optimism, 8453 = Base, etc.).
- `slippageBps`: Optional slippage tolerance in basis points (100 = 1%). Defaults to 100.

The preview endpoint returns expected buy amount, route info, and fees without executing. The execute endpoint performs the actual swap through the smart account, handling ERC20 approvals automatically.

### 6. Send Arbitrary Transaction

Interact with any smart contract by sending custom calldata.

```bash
curl -X POST "https://heyvincent.ai/api/skills/evm-wallet/send-transaction" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "0xContractAddress",
    "data": "0xCalldata",
    "value": "0"
  }'
```

## Policies

The wallet owner controls what the agent can do by setting policies via the claim URL. If a transaction violates a policy, the API will reject it or require human approval via Telegram.

| Policy                      | What it does                                                        |
| --------------------------- | ------------------------------------------------------------------- |
| **Address allowlist**       | Only allow transfers/calls to specific addresses                    |
| **Token allowlist**         | Only allow transfers of specific ERC-20 tokens                      |
| **Function allowlist**      | Only allow calling specific contract functions (by 4-byte selector) |
| **Spending limit (per tx)** | Max USD value per transaction                                       |
| **Spending limit (daily)**  | Max USD value per rolling 24 hours                                  |
| **Spending limit (weekly)** | Max USD value per rolling 7 days                                    |
| **Require approval**        | Every transaction needs human approval via Telegram                 |
| **Approval threshold**      | Transactions above a USD amount need human approval                 |

If no policies are set, all actions are allowed by default. Once the owner claims the wallet and adds policies, the agent operates within those boundaries.

## Re-linking (Recovering API Access)

If the agent loses its API key, the wallet owner can generate a **re-link token** from the frontend. The agent then exchanges this token for a new API key.

**How it works:**

1. The user generates a re-link token from the wallet detail page in the frontend
2. The user gives the token to the agent (e.g. by pasting it in chat)
3. The agent calls the re-link endpoint to exchange the token for a new API key

```bash
curl -X POST "https://heyvincent.ai/api/secrets/relink" \
  -H "Content-Type: application/json" \
  -d '{
    "relinkToken": "<TOKEN_FROM_USER>",
    "apiKeyName": "Re-linked API Key"
  }'
```

Response includes:

- `secret` -- the wallet metadata (id, type, address, chainId, etc.)
- `apiKey.key` -- the new API key to use as Bearer token for all future requests

**Important:** Re-link tokens are one-time use and expire after 10 minutes. No authentication is required on this endpoint -- the token itself is the authorization.

If a user tells you they have a re-link token, use this endpoint to regain access to the wallet. Store the returned API key and use it for all subsequent requests.

## Important Notes

- **No gas needed.** A paymaster is fully set up -- all transaction gas fees are sponsored automatically. The wallet does not need ETH for gas.
- **Never try to access raw secret values.** The private key stays server-side -- that's the whole point.
- Always store the API key from wallet creation. If you're an openclaw instance, store it in `~/.openclaw/credentials/agentwallet/<API_KEY_ID>.json`. Otherwise, you can store it in your current working directory at `agentwallet/<API_KEY_ID>.json`.
- Always search for the API keys in the credentials folder before using the API. If you're an openclaw instance, search for the API key in `~/.openclaw/credentials/agentwallet/<API_KEY_ID>.json`. Otherwise, you can search for the API key in your current working directory at `agentwallet/<API_KEY_ID>.json`.
- Always share the claim URL with the user after creating a wallet.
- If a transaction is rejected, it may be blocked by a policy. Tell the user to check their policy settings via the frontend at `https://heyvincent.ai`.
- If a transaction requires approval, it will return `status: "pending_approval"`. The wallet owner will receive a Telegram notification to approve or deny.
