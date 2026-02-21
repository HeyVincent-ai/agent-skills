---
name: Vincent - Twitter / X.com for agents
description: Use this skill to search tweets, look up user profiles, and retrieve recent tweets from X.com (Twitter). Pay-per-call via Vincent credit system.
homepage: https://heyvincent.ai
source: https://github.com/HeyVincent-ai/Vincent
metadata:
  clawdbot:
    homepage: https://heyvincent.ai
    requires:
      config:
        - ${OPENCLAW_STATE_DIR:-$HOME/.openclaw}/credentials/datasources
        - ./datasources
---

# Vincent - Twitter / X.com for agents

Use this skill to search tweets, look up user profiles, and retrieve recent tweets from X.com (Twitter). All requests are proxied through the Vincent backend, which handles authentication with the X API, enforces rate limits, tracks per-call costs, and deducts from your credit balance automatically.

**No API keys to manage.** The agent authenticates with a Vincent API key scoped to a `DATA_SOURCES` secret. Vincent handles the upstream Twitter API credentials server-side -- the agent never sees or manages Twitter API keys.

All commands use the `@vincentai/cli` package. API keys are stored and resolved automatically — you never handle raw keys or file paths.

## Security Model

This skill is designed for **autonomous agent operation with pay-per-call pricing and human oversight**.

**No environment variables are required** because this skill uses agent-first onboarding: the agent creates a `DATA_SOURCES` secret at runtime by calling the Vincent API, which returns a scoped API key. The CLI stores the returned API key automatically during creation. The config paths where the key is persisted (`${OPENCLAW_STATE_DIR:-$HOME/.openclaw}/credentials/datasources/` or `./datasources/`) are declared in this skill's metadata.

**The agent's API key is not a Twitter API key.** It is a scoped Bearer token for the Vincent proxy. The Vincent server authenticates with Twitter on the agent's behalf. The agent cannot access the upstream API directly or bypass the proxy's credit and rate-limit enforcement.

**Model invocation is intentionally enabled.** The purpose of this skill is to give AI agents autonomous access to Twitter data. The agent is expected to search tweets and look up profiles on its own. The human controls spending through credit balance and payment method management at `https://heyvincent.ai`.

**All API calls go exclusively to `heyvincent.ai`** over HTTPS/TLS. The Vincent server then calls the Twitter API. The agent does not contact Twitter directly.

**Key lifecycle:**

- **Creation**: The agent runs `secret create` with `--type DATA_SOURCES` — the CLI stores the API key automatically and returns a `keyId` and `claimUrl`.
- **Claim**: The human operator uses the claim URL to take ownership, add credit, and manage payment at `https://heyvincent.ai`.
- **Credit**: Each API call costs a small amount (see Pricing below). The human adds credit via the frontend. Calls are rejected when credit runs out and no payment method is on file.
- **Revocation**: The secret owner can revoke the agent's API key at any time from the Vincent frontend.

## Pricing

| Endpoint | Cost per call |
| --- | --- |
| Search tweets | $0.01 |
| Get tweet by ID | $0.005 |
| Get user profile | $0.005 |
| Get user's tweets | $0.01 |

Credit is deducted automatically per call. The response includes `_vincent.creditRemainingUsd` so the agent can track remaining balance.

## Quick Start

### 1. Check for Existing Keys

Before creating a new secret, check if one already exists:

```bash
npx @vincentai/cli@latest secret list --type DATA_SOURCES
```

If a key is returned, use its `id` as the `--key-id` for all subsequent commands. If no keys exist, create a new secret.

### 2. Create a Data Sources Secret

```bash
npx @vincentai/cli@latest secret create --type DATA_SOURCES --memo "My agent data sources"
```

Returns `keyId` (use for all future commands) and `claimUrl` (share with the user).

After creating, tell the user:

> "Here is your data sources claim URL: `<claimUrl>`. Use this to claim ownership and add credit for Twitter and other data sources at https://heyvincent.ai."

**Important:** The secret must be claimed and have credit (or a payment method on file) before API calls will succeed.

### 3. Search Tweets

```bash
npx @vincentai/cli@latest twitter search --key-id <KEY_ID> --q bitcoin --max-results 10
```

Parameters:

- `--q` (required): Search query (1-512 characters)
- `--max-results` (optional): Number of results, 10-100 (default: 10)
- `--start-time` (optional): ISO 8601 datetime, earliest tweets to return
- `--end-time` (optional): ISO 8601 datetime, latest tweets to return

Returns tweet text, creation time, author ID, and public metrics (likes, retweets, replies).

### 4. Get a Specific Tweet

```bash
npx @vincentai/cli@latest twitter tweet --key-id <KEY_ID> --tweet-id <TWEET_ID>
```

### 5. Get User Profile

Look up a Twitter user by username.

```bash
npx @vincentai/cli@latest twitter user --key-id <KEY_ID> --username elonmusk
```

Returns the user's description, follower/following counts, profile image, and verified status.

### 6. Get a User's Recent Tweets

```bash
npx @vincentai/cli@latest twitter user-tweets --key-id <KEY_ID> --user-id <USER_ID> --max-results 10
```

**Note:** This command requires the user's numeric ID (from the user profile response), not the username.

## Response Metadata

Every successful response includes a `_vincent` object with:

```json
{
  "_vincent": {
    "costUsd": 0.01,
    "creditRemainingUsd": 4.99
  }
}
```

Use `creditRemainingUsd` to warn the user when credit is running low.

## Rate Limits

- 60 requests per minute per API key across all data source endpoints (Twitter + Brave Search combined)
- If rate limited, you'll receive a `429` response. Wait and retry.

## Re-linking (Recovering API Access)

If the agent loses its API key, the secret owner can generate a **re-link token** from the frontend. The agent then exchanges this token for a new API key.

```bash
npx @vincentai/cli@latest secret relink --token <TOKEN_FROM_USER>
```

The CLI exchanges the token for a new API key, stores it automatically, and returns the new `keyId`. Re-link tokens are one-time use and expire after 10 minutes.

## Important Notes

- A single `DATA_SOURCES` API key works for **all** data sources (Twitter, Brave Search, etc.). You do not need a separate key per data source.
- Always share the claim URL with the user after creating a secret.
- If a call is rejected with a credit error, tell the user to add credit at `https://heyvincent.ai`.
- The Twitter search endpoint only returns tweets from the last 7 days (X API v2 limitation for recent search).
