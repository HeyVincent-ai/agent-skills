---
name: Vincent - Trading Engine for agents
description: |
  Strategy-driven automated trading for Polymarket. Use this skill when users want to create
  trading strategies, set stop-loss/take-profit/trailing stop rules, or manage automated trading.
  Triggers on "trading strategy", "stop loss", "take profit", "trailing stop", "automated trading",
  "trading engine", "trade rules", "strategy monitor".
allowed-tools: Read, Write, Bash(npx:*, curl:*)
version: 1.0.0
author: HeyVincent <contact@heyvincent.ai>
license: MIT
homepage: https://heyvincent.ai
source: https://github.com/HeyVincent-ai/Vincent
metadata:
  clawdbot:
    homepage: https://heyvincent.ai
    requires:
      config:
        - ${OPENCLAW_STATE_DIR:-$HOME/.openclaw}/credentials/agentwallet
        - ./agentwallet
---

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

## Output Format

Strategy creation:

```json
{
  "strategyId": "strat-123",
  "name": "AI Token Momentum",
  "status": "DRAFT",
  "version": 1
}
```

Rule creation:

```json
{
  "ruleId": "rule-456",
  "ruleType": "STOP_LOSS",
  "triggerPrice": 0.40,
  "status": "ACTIVE"
}
```

LLM invocation log entries:

```json
{
  "invocationId": "inv-789",
  "strategyId": "strat-123",
  "trigger": "web_search",
  "actions": ["place_trade"],
  "costUsd": 0.12,
  "createdAt": "2026-02-26T12:00:00.000Z"
}
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `401 Unauthorized` | Invalid or missing API key | Check that the key-id is correct; re-link if needed |
| `403 Policy Violation` | Trade blocked by server-side policy | User must adjust policies at heyvincent.ai |
| `402 Insufficient Credit` | Not enough credit for LLM invocation | User must add credit at heyvincent.ai |
| `INVALID_STATUS_TRANSITION` | Strategy can't transition to requested state | Check current status (e.g., only DRAFT can activate) |
| `CIRCUIT_BREAKER_OPEN` | Polymarket API failures triggered circuit breaker | Wait for cooldown; check status command |
| `429 Rate Limited` | Too many requests or concurrent LLM invocations | Wait and retry with backoff |
| `Key not found` | API key was revoked or never created | Re-link with a new token from the wallet owner |

## Important Notes

- **Authorization:** All endpoints require the same Polymarket API key used for the Polymarket skill
- **Local only:** The API listens on `localhost:19000` — only accessible from the same VPS
- **No private keys:** All trades use the Vincent API — your private key stays secure on Vincent's servers
- **Policy enforcement:** All trades (both LLM and standalone rules) go through Vincent's policy checks
- **Idempotency:** Rules only trigger once. LLM invocations are deduplicated by monitor state.

---

## Part 3: V2 Multi-Venue Strategies

V2 extends the Trading Engine with multi-venue support, a driver-based monitoring system, thesis tracking, a 6-layer signal pipeline, advanced position sizing, and escalation policies. V2 strategies can trade across any supported venue (Polymarket, and more as adapters are added), not just Polymarket.

### What V2 Adds Over V1

| Feature | V1 | V2 |
|---|---|---|
| Venues | Polymarket only | Multi-venue (driver `sources` + venue adapters) |
| Monitoring | Web search, Twitter, newswire, price triggers | Driver-based: weighted drivers with entities, keywords, embedding anchors, multiple sources |
| Thesis | Alert prompt (free text) | Structured thesis: estimate, direction, confidence, reasoning |
| Signal Pipeline | Monitor → LLM | 6-layer: Ingest → Filter → Score → Escalate → LLM → Execute |
| Position Sizing | Fixed max trade USD | Edge-scaled, fixed, or Kelly criterion with portfolio-level limits |
| Trade Rules | Separate standalone rules | Integrated auto-actions (stop-loss, take-profit, trailing stop, price delta triggers) |
| Notifications | None | Webhook or Slack on trade or thesis change |

### Core Concepts

- **Instrument**: A tradeable asset on a venue. Defined by `id`, `type` (stock, perp, swap, binary, option), `venue`, and optional constraints (leverage, margin, liquidity, fees).
- **Thesis**: Your directional view — `estimate` (target price/value), `direction` (long/short/neutral), `confidence` (0–1), and `reasoning`.
- **Driver**: A named information source that feeds the signal pipeline. Each driver has a `weight`, `direction` (bullish/bearish/contextual), and `monitoring` config (entities, keywords, embedding anchor, sources, polling interval).
- **Escalation Policy**: Controls when the LLM is woken up. `signalScoreThreshold` (minimum score to batch), `highConfidenceThreshold` (score that triggers immediate wake), `maxWakeFrequency` (e.g. "1 per 15m"), `batchWindow` (e.g. "5m").
- **Trade Rules**: Entry rules (min edge, order type), exit rules (thesis invalidation triggers), auto-actions (stop-loss, take-profit, trailing stop, price delta triggers), and sizing rules (method, max position, portfolio %, max trades/day).

### Strategy Lifecycle

Same states as V1: `DRAFT` → `ACTIVE` → `PAUSED` → `ARCHIVED`

### Create a V2 Strategy

```bash
npx @vincentai/cli@latest v2 create-strategy \
  --key-id <KEY_ID> \
  --name "BTC Multi-Venue Momentum" \
  --config '{
    "instruments": [
      { "id": "btc-usd-perp", "type": "perp", "venue": "polymarket" }
    ],
    "thesis": {
      "estimate": 105000,
      "direction": "long",
      "confidence": 0.7,
      "reasoning": "ETF inflows accelerating, halving supply shock imminent"
    },
    "drivers": [
      {
        "name": "ETF Flow Monitor",
        "weight": 2.0,
        "direction": "bullish",
        "monitoring": {
          "entities": ["BlackRock", "Fidelity"],
          "keywords": ["bitcoin ETF", "BTC inflow"],
          "embeddingAnchor": "Bitcoin ETF institutional inflows",
          "sources": ["web_search", "newswire"]
        }
      }
    ],
    "escalation": {
      "signalScoreThreshold": 0.3,
      "highConfidenceThreshold": 0.8,
      "maxWakeFrequency": "1 per 15m",
      "batchWindow": "5m"
    },
    "tradeRules": {
      "entry": { "minEdge": 0.05, "orderType": "limit", "limitOffset": 0.01 },
      "autoActions": { "stopLoss": -0.10, "takeProfit": 0.25, "trailingStop": -0.05 },
      "exit": { "thesisInvalidation": ["ETF outflows exceed $500M/week"] },
      "sizing": {
        "method": "edgeScaled",
        "maxPosition": 500,
        "maxPortfolioPct": 20,
        "maxTradesPerDay": 5,
        "minTimeBetweenTrades": "30m"
      }
    },
    "notifications": {
      "onTrade": true,
      "onThesisChange": true,
      "channel": "none"
    }
  }'
```

**Parameters:**
- `--name`: Strategy name
- `--config`: Full V2StrategyConfig JSON (see Core Concepts above for structure)
- `--data-source-secret-id`: Optional DATA_SOURCES secret for driver monitoring API calls
- `--poll-interval`: Polling interval in minutes for driver monitoring (default: 15)

### V2 Strategy Management

```bash
# List all V2 strategies
npx @vincentai/cli@latest v2 list-strategies --key-id <KEY_ID>

# Get strategy details
npx @vincentai/cli@latest v2 get-strategy --key-id <KEY_ID> --strategy-id <ID>

# Update a DRAFT strategy (pass only fields to change)
npx @vincentai/cli@latest v2 update-strategy --key-id <KEY_ID> --strategy-id <ID> \
  --name "Updated Name" --config '{ "thesis": { ... } }'

# Activate (DRAFT → ACTIVE)
npx @vincentai/cli@latest v2 activate --key-id <KEY_ID> --strategy-id <ID>

# Pause (ACTIVE → PAUSED)
npx @vincentai/cli@latest v2 pause --key-id <KEY_ID> --strategy-id <ID>

# Resume (PAUSED → ACTIVE)
npx @vincentai/cli@latest v2 resume --key-id <KEY_ID> --strategy-id <ID>

# Archive (permanent)
npx @vincentai/cli@latest v2 archive --key-id <KEY_ID> --strategy-id <ID>
```

### Portfolio & Monitoring

```bash
# Portfolio overview (positions + balances across all venues)
npx @vincentai/cli@latest v2 portfolio --key-id <KEY_ID>

# Signal log — raw signals from drivers
npx @vincentai/cli@latest v2 signal-log --key-id <KEY_ID> --strategy-id <ID> --limit 50

# Decision log — LLM thesis updates and trade decisions
npx @vincentai/cli@latest v2 decision-log --key-id <KEY_ID> --strategy-id <ID> --limit 50

# Trade log — order execution results
npx @vincentai/cli@latest v2 trade-log --key-id <KEY_ID> --strategy-id <ID> --limit 50

# Performance metrics (P&L, win rate, per-instrument breakdown)
npx @vincentai/cli@latest v2 performance --key-id <KEY_ID> --strategy-id <ID>

# Filter stats — signals passed/dropped at each pipeline layer
npx @vincentai/cli@latest v2 filter-stats --key-id <KEY_ID> --strategy-id <ID>

# Escalation stats — wake frequency, batch counts, threshold breaches
npx @vincentai/cli@latest v2 escalation-stats --key-id <KEY_ID> --strategy-id <ID>
```

### Manual Overrides

```bash
# Place a manual order on any venue
npx @vincentai/cli@latest v2 place-order --key-id <KEY_ID> \
  --instrument-id <TOKEN_ID> --venue polymarket \
  --side BUY --size 10 --order-type market

# Place a limit order
npx @vincentai/cli@latest v2 place-order --key-id <KEY_ID> \
  --instrument-id <TOKEN_ID> --venue polymarket \
  --side BUY --size 10 --order-type limit --limit-price 0.45

# Cancel an order
npx @vincentai/cli@latest v2 cancel-order --key-id <KEY_ID> \
  --venue polymarket --order-id <ORDER_ID>

# Close a position (opposite-side market order)
npx @vincentai/cli@latest v2 close-position --key-id <KEY_ID> \
  --instrument-id <TOKEN_ID> --venue polymarket

# Emergency kill switch — pause all strategies + cancel all orders
npx @vincentai/cli@latest v2 kill-switch --key-id <KEY_ID>
```

### Signal Pipeline Architecture

V2 strategies process information through a 6-layer pipeline:

1. **Ingest** — Raw data from driver sources (web search, Twitter, newswire, price feeds, RSS, Reddit, on-chain, filings, options flow)
2. **Filter** — Deduplication and relevance filtering. Drops signals already seen or below quality threshold
3. **Score** — Each signal is scored (0–1) based on driver weight, embedding similarity to the anchor, and entity/keyword matches
4. **Escalate** — Scored signals are batched according to the escalation policy. Low-score signals accumulate in a batch window; high-confidence signals trigger immediate LLM wake
5. **LLM** — The LLM evaluates batched signals against the current thesis. It can update the thesis, issue trade decisions, update driver states, or take no action
6. **Execute** — Trade decisions pass through policy enforcement and are routed to the appropriate venue adapter for execution

### V2 Best Practices

1. **Start with `confidence: 0.5`** and let the LLM adjust — avoid overconfidence in the initial thesis
2. **Weight drivers by importance** — a driver with `weight: 3.0` has 3x the signal score contribution of `weight: 1.0`
3. **Use `edgeScaled` sizing** for adaptive position sizes based on thesis confidence and edge
4. **Set `maxPortfolioPct`** to limit exposure — even high-confidence strategies shouldn't risk the entire portfolio
5. **Monitor `filter-stats`** to tune the escalation policy — if too many signals are batched without action, lower `signalScoreThreshold`; if the LLM wakes too often, raise it
6. **Use `thesisInvalidation` exit rules** to define explicit conditions that should trigger position exits
7. **Use the kill switch** in emergencies — it pauses all strategies and cancels all open orders in one call

### Example User Prompts (V2)

When a user says:
- **"Create a multi-venue strategy for BTC"** → Create V2 strategy with instruments, thesis, drivers
- **"What's my portfolio across venues?"** → Call v2 portfolio
- **"Show me the signal pipeline activity"** → Call signal-log, filter-stats
- **"What has the LLM decided?"** → Call decision-log
- **"How is my strategy performing?"** → Call performance
- **"Place a manual buy order"** → Call v2 place-order
- **"Emergency stop everything"** → Call v2 kill-switch
- **"Pause my V2 strategy"** → Call v2 pause
