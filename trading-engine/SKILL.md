# Vincent Trading Engine - Strategy-Driven Automated Trading

Use this skill to create and manage automated trading strategies for Polymarket prediction markets. The Trading Engine combines data monitoring (web search, Twitter, price feeds) with LLM-powered decision-making to automatically trade based on your thesis. It also includes standalone stop-loss, take-profit, and trailing stop rules that work without the LLM.

All commands use the `@vincentai/cli` package.

## How It Works

**The Trading Engine is a unified system with two modes:**

1. **LLM-Powered Strategies** — Create a versioned strategy with monitors (web search keywords, Twitter accounts, price triggers, newswire feeds). When a monitor detects new information, an LLM (Claude via OpenRouter) evaluates it against your thesis and decides whether to trade, set protective orders, or alert you.
2. **Standalone Trade Rules** — Set stop-loss, take-profit, and trailing stop rules on positions. These execute automatically when price conditions are met — no LLM involved.

**Architecture:**
- Integrated into the Vincent backend (no separate service to run)
- Strategy endpoints under `/api/skills/polymarket/strategies/...`
- Trade rule endpoints under `/api/skills/polymarket/rules/...`
- Uses the same API key as the Polymarket skill
- All trades go through Vincent's policy-enforced pipeline
- LLM costs are metered and deducted from the user's credit balance
- Every LLM invocation is recorded with full audit trail (tokens, cost, actions, duration)

## Security Model

- **LLM cannot bypass policies** — all trades go through `polymarketSkill.placeBet()` which enforces spending limits, approval thresholds, and allowlists
- **Backend-side LLM key** — the OpenRouter API key never leaves the server. Agents and users cannot invoke the LLM directly
- **Credit gating** — no LLM invocation without sufficient credit balance
- **Tool constraints** — the LLM's available tools are controlled by the strategy's `config.tools` settings. If `canTrade: false`, the trade tool is not provided
- **Rate limiting** — max concurrent LLM invocations is capped to prevent runaway costs
- **Audit trail** — every invocation is recorded with full prompt, response, actions, cost, and duration
- **No private keys** — the Trading Engine uses the Vincent API for all trades. Private keys stay on Vincent's servers

## Part 1: LLM-Powered Strategies

### Strategy Lifecycle

Strategies follow a versioned lifecycle: `DRAFT` → `ACTIVE` → `PAUSED` → `ARCHIVED`

- **DRAFT**: Can be edited. Not yet monitoring or invoking the LLM.
- **ACTIVE**: Monitors are running. New data triggers LLM invocations.
- **PAUSED**: Monitoring is stopped. Can be resumed.
- **ARCHIVED**: Permanently stopped. Cannot be reactivated.

To iterate on a strategy, duplicate it as a new version (creates a new DRAFT with incremented version number and the same config).

### Create a Strategy

```bash
npx @vincentai/cli@latest trading-engine create-strategy \
  --key-id <KEY_ID> \
  --name "AI Token Momentum" \
  --alert-prompt "AI tokens are about to re-rate as funding accelerates. Buy dips in AI-related prediction markets. Sell if the thesis breaks." \
  --poll-interval 15 \
  --web-keywords "AI tokens,GPU shortage,AI funding" \
  --twitter-accounts "@DeepSeek,@nvidia,@OpenAI" \
  --newswire-topics "artificial intelligence,GPU,semiconductor" \
  --can-trade \
  --can-set-rules \
  --max-trade-usd 50
```

**Parameters:**
- `--name`: User-friendly name for the strategy
- `--alert-prompt`: Your thesis and instructions for the LLM. This is the most important part — be specific about what information matters and what actions to take.
- `--poll-interval`: How often (in minutes) to check periodic monitors (default: 15)
- `--web-keywords`: Comma-separated keywords for Brave web search monitoring
- `--twitter-accounts`: Comma-separated Twitter handles to monitor (with or without @)
- `--newswire-topics`: Comma-separated keywords for Finnhub market news monitoring (headlines matching any keyword trigger the LLM)
- `--can-trade`: Allow the LLM to place trades (omit to restrict to alerts only)
- `--can-set-rules`: Allow the LLM to create stop-loss/take-profit/trailing stop rules
- `--max-trade-usd`: Maximum USD per trade the LLM can place

### List Strategies

```bash
npx @vincentai/cli@latest trading-engine list-strategies --key-id <KEY_ID>
```

### Get Strategy Details

```bash
npx @vincentai/cli@latest trading-engine get-strategy --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### Activate a Strategy

Starts monitoring and LLM invocations. Strategy must be in DRAFT status.

```bash
npx @vincentai/cli@latest trading-engine activate --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### Pause a Strategy

Stops monitoring. Strategy must be ACTIVE.

```bash
npx @vincentai/cli@latest trading-engine pause --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### Resume a Strategy

Resumes monitoring. Strategy must be PAUSED.

```bash
npx @vincentai/cli@latest trading-engine resume --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

Note: The `resume` command uses the same `activate` command endpoint internally with the PAUSED → ACTIVE transition handled server-side.

### Archive a Strategy

Permanently stops a strategy. Cannot be undone.

```bash
npx @vincentai/cli@latest trading-engine archive --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### Duplicate a Strategy (New Version)

Creates a new DRAFT with the same config, incremented version number, and a link to the parent version.

```bash
npx @vincentai/cli@latest trading-engine duplicate-strategy --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### View Version History

See all versions of a strategy lineage.

```bash
npx @vincentai/cli@latest trading-engine versions --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### View LLM Invocation History

See the LLM decision log for a strategy — what data triggered it, what the LLM decided, what actions were taken, and the cost.

```bash
npx @vincentai/cli@latest trading-engine invocations --key-id <KEY_ID> --strategy-id <STRATEGY_ID> --limit 20
```

### View Cost Summary

See aggregate LLM costs for all strategies under a secret.

```bash
npx @vincentai/cli@latest trading-engine costs --key-id <KEY_ID>
```

### Monitor Configuration

#### Web Search Monitors

Add `--web-keywords` when creating a strategy. The engine periodically searches Brave for these keywords and triggers the LLM when new results appear.

```bash
--web-keywords "AI tokens,GPU shortage,prediction market regulation"
```

Each keyword is searched independently. Results are deduplicated — the same URLs won't trigger the LLM twice.

#### Twitter Monitors

Add `--twitter-accounts` when creating a strategy. The engine periodically checks these accounts for new tweets and triggers the LLM when new tweets appear.

```bash
--twitter-accounts "@DeepSeek,@nvidia,@OpenAI"
```

Tweets are deduplicated by tweet ID — only genuinely new tweets trigger the LLM.

#### Newswire Monitors (Finnhub)

Add `--newswire-topics` when creating a strategy. The engine periodically polls Finnhub's market news API (general + crypto categories) and triggers the LLM when new headlines matching your topic keywords appear.

```bash
--newswire-topics "artificial intelligence,GPU shortage,semiconductor"
```

Each topic string can contain comma-separated keywords. Headlines and summaries are matched case-insensitively. Articles are deduplicated by headline hash with a sliding window of 100 entries per topic.

**Note:** Requires a `FINNHUB_API_KEY` env var on the server. Finnhub's free tier allows 60 API calls/min — more than sufficient for strategy monitoring. No per-call credit deduction (Finnhub free tier has no cost).

#### Price Triggers

Price triggers are configured in the strategy's JSON config and evaluated in real-time via the Polymarket WebSocket feed. When a price condition is met, the LLM is invoked with the price data.

Trigger types:
- `ABOVE` — triggers when price exceeds a threshold
- `BELOW` — triggers when price drops below a threshold
- `CHANGE_PCT` — triggers on a percentage change from reference price

Price triggers are one-shot: once fired, they're marked as consumed. The LLM can create new triggers if needed.

### Alert Prompt Best Practices

The alert prompt is your instructions to the LLM. Good prompts are:

1. **Specific about the thesis**: "I believe AI tokens will rally because GPU demand is increasing. Buy any AI-related prediction market position below 40 cents."
2. **Clear about action criteria**: "Only trade if the new information directly supports or contradicts the thesis. If ambiguous, alert me instead."
3. **Explicit about risk**: "Never allocate more than $50 to a single position. Set a 15% trailing stop on any new position."
4. **Contextual**: "Ignore routine corporate announcements. Focus on regulatory actions, major product launches, and competitive threats."

### LLM Available Tools

When the LLM is invoked, it can use these tools (depending on strategy config):

| Tool | Description | Requires |
|---|---|---|
| `place_trade` | Buy or sell a position | `canTrade: true` |
| `set_stop_loss` | Set a stop-loss rule on a position | `canSetRules: true` |
| `set_take_profit` | Set a take-profit rule | `canSetRules: true` |
| `set_trailing_stop` | Set a trailing stop | `canSetRules: true` |
| `alert_user` | Send an alert without trading | Always available |
| `no_action` | Do nothing (with reasoning) | Always available |

### Cost Tracking

Every LLM invocation is metered:
- **Token costs**: Input and output tokens are priced per the model's rate
- **Deducted from credit balance**: Same pool as data source credits (`dataSourceCreditUsd`)
- **Pre-flight check**: If insufficient credit, the invocation is skipped and logged
- **Data source costs**: Brave Search (~$0.005/call) and Twitter (~$0.005-$0.01/call) are also metered. Finnhub newswire calls are free (no credit deduction)

Typical LLM invocation cost: $0.05–$0.30 depending on context size.

---

## Part 2: Standalone Trade Rules

Trade rules execute automatically when price conditions are met — no LLM involved. These are the same stop-loss, take-profit, and trailing stop rules from the original Trade Manager, now unified under the Trading Engine namespace.

### Check Worker Status

```bash
npx @vincentai/cli@latest trading-engine status --key-id <KEY_ID>
# Returns: worker status, active rules count, last sync time, circuit breaker state
```

### Create a Stop-Loss Rule

Automatically sell a position if price drops below a threshold:

```bash
npx @vincentai/cli@latest trading-engine create-rule --key-id <KEY_ID> \
  --market-id 0x123... --token-id 456789 \
  --rule-type STOP_LOSS --trigger-price 0.40
```

**Parameters:**
- `--market-id`: The Polymarket condition ID (from market data)
- `--token-id`: The outcome token ID you hold (from market data)
- `--rule-type`: `STOP_LOSS` (sells if price <= trigger), `TAKE_PROFIT` (sells if price >= trigger), or `TRAILING_STOP`
- `--trigger-price`: Price threshold between 0 and 1 (e.g., 0.40 = 40 cents)

### Create a Take-Profit Rule

Automatically sell a position if price rises above a threshold:

```bash
npx @vincentai/cli@latest trading-engine create-rule --key-id <KEY_ID> \
  --market-id 0x123... --token-id 456789 \
  --rule-type TAKE_PROFIT --trigger-price 0.75
```

### Create a Trailing Stop Rule

A trailing stop moves the stop price up as the price rises:

```bash
npx @vincentai/cli@latest trading-engine create-rule --key-id <KEY_ID> \
  --market-id 0x123... --token-id 456789 \
  --rule-type TRAILING_STOP --trigger-price 0.45 --trailing-percent 5
```

**Trailing stop behavior:**
- `--trailing-percent` is percent points (e.g. `5` = 5%)
- Computes `candidateStop = currentPrice * (1 - trailingPercent/100)`
- If `candidateStop` > current `triggerPrice`, updates `triggerPrice`
- `triggerPrice` never moves down
- Rule triggers when `currentPrice <= triggerPrice`

### List Rules

```bash
# All rules
npx @vincentai/cli@latest trading-engine list-rules --key-id <KEY_ID>

# Filter by status
npx @vincentai/cli@latest trading-engine list-rules --key-id <KEY_ID> --status ACTIVE
```

### Update a Rule

```bash
npx @vincentai/cli@latest trading-engine update-rule --key-id <KEY_ID> --rule-id <RULE_ID> --trigger-price 0.45
```

### Cancel a Rule

```bash
npx @vincentai/cli@latest trading-engine delete-rule --key-id <KEY_ID> --rule-id <RULE_ID>
```

### View Monitored Positions

```bash
npx @vincentai/cli@latest trading-engine positions --key-id <KEY_ID>
```

### View Event Log

```bash
# All events
npx @vincentai/cli@latest trading-engine events --key-id <KEY_ID>

# Events for specific rule
npx @vincentai/cli@latest trading-engine events --key-id <KEY_ID> --rule-id <RULE_ID>

# Paginated
npx @vincentai/cli@latest trading-engine events --key-id <KEY_ID> --limit 50 --offset 100
```

**Event types:**
- `RULE_CREATED` — Rule was created
- `RULE_TRAILING_UPDATED` — Trailing stop moved triggerPrice upward
- `RULE_EVALUATED` — Worker checked the rule against current price
- `RULE_TRIGGERED` — Trigger condition was met
- `ACTION_PENDING_APPROVAL` — Trade requires human approval, rule paused
- `ACTION_EXECUTED` — Trade executed successfully
- `ACTION_FAILED` — Trade execution failed
- `RULE_CANCELED` — Rule was manually canceled

### Rule Statuses

- `ACTIVE` — Rule is live and being monitored
- `TRIGGERED` — Condition was met, trade executed
- `PENDING_APPROVAL` — Trade requires human approval; rule paused
- `CANCELED` — Manually canceled before triggering
- `FAILED` — Triggered but trade execution failed

---

## Complete Workflow: Strategy + Trade Rules

### Step 1: Place a bet with the Polymarket skill

```bash
npx @vincentai/cli@latest polymarket bet --key-id <KEY_ID> --token-id 123456789 --side BUY --amount 10 --price 0.55
```

### Step 2: Create a strategy to monitor the thesis

```bash
npx @vincentai/cli@latest trading-engine create-strategy --key-id <KEY_ID> \
  --name "Bitcoin Bull Thesis" \
  --alert-prompt "Bitcoin is likely to break $100k on ETF inflows. Buy dips, sell if ETF outflows accelerate." \
  --web-keywords "bitcoin ETF inflows,bitcoin institutional" \
  --twitter-accounts "@BitcoinMagazine,@saborskycnbc" \
  --can-trade --can-set-rules --max-trade-usd 25 --poll-interval 10
```

### Step 3: Set a standalone stop-loss as immediate protection

```bash
npx @vincentai/cli@latest trading-engine create-rule --key-id <KEY_ID> \
  --market-id 0xabc... --token-id 123456789 \
  --rule-type STOP_LOSS --trigger-price 0.40
```

### Step 4: Activate the strategy

```bash
npx @vincentai/cli@latest trading-engine activate --key-id <KEY_ID> --strategy-id <STRATEGY_ID>
```

### Step 5: Monitor activity

```bash
# Check strategy invocations
npx @vincentai/cli@latest trading-engine invocations --key-id <KEY_ID> --strategy-id <STRATEGY_ID>

# Check trade rule events
npx @vincentai/cli@latest trading-engine events --key-id <KEY_ID>

# Check costs
npx @vincentai/cli@latest trading-engine costs --key-id <KEY_ID>
```

## Background Workers

The Trading Engine runs two independent background workers:

1. **Strategy Engine Worker** — Ticks every 30s, checks which strategy monitors are due, fetches new data, invokes the LLM when new data is detected. Also hooks into the Polymarket WebSocket for real-time price trigger evaluation.
2. **Trade Rule Worker** — Monitors prices in real-time via WebSocket (with polling fallback), evaluates stop-loss/take-profit/trailing stop rules, executes trades when conditions are met.

**Circuit Breaker:** Both workers use a circuit breaker pattern. If the Polymarket API fails 5+ consecutive times, the worker pauses and resumes after a cooldown. Check status with:

```bash
npx @vincentai/cli@latest trading-engine status --key-id <KEY_ID>
```

## Best Practices

1. **Start with alerts only** — Set `canTrade: false` initially to see what the LLM would do before enabling autonomous trading
2. **Use specific alert prompts** — Vague prompts lead to vague decisions. Be explicit about your thesis and action criteria
3. **Set both stop-loss and take-profit** on positions for protection
4. **Monitor invocation costs** — Check the costs command regularly
5. **Iterate with versions** — Duplicate a strategy to tweak the prompt or monitors without losing the original
6. **Don't set triggers too close** to current price — market noise can trigger prematurely

## Example User Prompts

When a user says:
- **"Create a strategy to monitor AI tokens"** → Create strategy with web search + Twitter monitors
- **"Set a stop-loss at 40 cents"** → Create STOP_LOSS rule
- **"What has my strategy been doing?"** → Show invocations for the strategy
- **"How much has the trading engine cost me?"** → Show cost summary
- **"Pause my strategy"** → Pause the strategy
- **"Make a new version with a different prompt"** → Duplicate, then update the draft
- **"Set a 5% trailing stop"** → Create TRAILING_STOP rule

## Important Notes

- **Authorization:** All endpoints require the same Polymarket API key used for the Polymarket skill
- **Local only:** The API listens on `localhost:19000` — only accessible from the same VPS
- **No private keys:** All trades use the Vincent API — your private key stays secure on Vincent's servers
- **Policy enforcement:** All trades (both LLM and standalone rules) go through Vincent's policy checks
- **Idempotency:** Rules only trigger once. LLM invocations are deduplicated by monitor state.
