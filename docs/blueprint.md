# CryptoPing — personal crypto price alerts — Bot specification

**Archetype:** finance

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

A private Telegram bot that lets users track crypto prices and set alerts for price thresholds or percentage changes. Users manage watchlists via buttons or text commands, receive on-demand price checks, and optional daily summaries. The bot owner gets aggregated metrics (user counts, alert frequency) via admin commands without exposing individual data.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- individual crypto traders
- crypto holders
- bot owner/admin

## Success criteria

- users can add/remove tickers to watchlist
- alerts trigger reliably with rate-limiting
- admin receives daily usage metrics without private data exposure

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open main menu with watchlist management and settings
- **/price** (command, actor: user, command: /price) — Check current price of a specific ticker or full watchlist
- **/watchlist** (command, actor: user, command: /watchlist) — View and manage all watchlist items
- **/settings** (command, actor: user, command: /settings) — Configure quiet hours, alert types, and summary time
- **Add Bitcoin** (button, actor: user, callback: watchlist:add:BTC) — Quickly add BTC to watchlist with default rules
- **Add Ethereum** (button, actor: user, callback: watchlist:add:ETH) — Quickly add ETH to watchlist with default rules
- **Add Toncoin** (button, actor: user, callback: watchlist:add:TON) — Quickly add TON to watchlist with default rules
- **Add Custom Ticker** (button, actor: user, callback: watchlist:add:custom) — Prompt user to enter a custom ticker symbol
- **/admin** (command, actor: owner, command: /admin) — Show aggregated metrics to bot owner

## Flows

### onboarding
_Trigger:_ /start

1. Display greeting with brief explanation
2. Apply default settings (summary ON, quiet hours 23:00-07:00)
3. Show quick-start buttons for common tickers

_Data touched:_ User profile

### price_check
_Trigger:_ /price

1. Parse ticker parameter if provided
2. Fetch current price and 24h change
3. Compare to user's stored price if available
4. Display compact price summary or full watchlist

_Data touched:_ Watchlist item, User profile

### alert_generation
_Trigger:_ price_update_event

1. Check if current price meets any active threshold rules
2. Check if current price meets percent-change rules
3. Validate against quiet hours window
4. Queue alerts if during quiet hours
5. Send alert with before/after prices and cooldown tracking

_Data touched:_ Watchlist item, Alert event, User profile

### daily_summary
_Trigger:_ scheduled_morning_time

1. Fetch current prices for all user tickers
2. Check for rules near triggering (within margin)
3. Compile summary of active alerts since last check
4. Send summary respecting quiet hours

_Data touched:_ Watchlist item, Alert event, User profile

### admin_metrics
_Trigger:_ /admin

1. Verify owner identity
2. Aggregate active users and watchlist sizes
3. Compile top-triggered tickers and rule types
4. Display metrics in private message

_Data touched:_ Admin metrics

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **User profile** _(retention: persistent)_ — User-specific settings and preferences
  - fields: telegram_user_id, timezone, quiet_hours_start, quiet_hours_end, summary_time, threshold_alerts_enabled, percent_alerts_enabled, summary_enabled, mute_state
- **Watchlist item** _(retention: persistent)_ — Tracked cryptocurrency ticker with alert rules
  - fields: ticker_symbol, display_name, threshold_rules, percent_change_rules, last_notified, cooldown_expiry
- **Alert event** _(retention: persistent)_ — Record of triggered alerts for metrics tracking
  - fields: ticker, old_price, new_price, percent_change, rule_type, timestamp
- **Admin metrics** _(retention: persistent)_ — Aggregated statistics for owner
  - fields: total_active_users, watchlist_distribution, alert_counts_by_ticker, alert_counts_by_rule_type, error_counts

## Integrations

- **Telegram** (required) — User messaging, commands, and inline buttons
- **Crypto Price API** (required) — Fetch real-time price data with retry logic
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- /admin command to view metrics
- override quiet hours for critical notifications
- configure default alert cooldown duration

## Notifications

- Price threshold alerts with before/after values
- Percent change alerts with time window context
- Daily morning summary with price updates
- Queued alerts delivered after quiet hours
- Single error notification for extended price-source failures

## Permissions & privacy

- All user data is private and isolated
- Admin metrics only show aggregated counts and top lists
- No sharing of watchlists between users
- Quiet hours override suppresses alerts during specified window

## Edge cases

- Invalid ticker symbols during add operations
- Price-source API failures during alert checks
- Alerts triggering during quiet hours requiring queueing
- Multiple rules triggering simultaneously for same ticker

## Required tests

- Verify alert cooldown prevents repeated notifications
- Test quiet hours queueing and delivery logic
- Validate morning summary includes near-trigger rules
- Confirm admin metrics don't expose private data

## Assumptions

- Price-source API will be implemented with exponential backoff
- Default alert cooldown of 6 hours is sufficient for initial release
- Morning summary time will be user-configurable in settings
