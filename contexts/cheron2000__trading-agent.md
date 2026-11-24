# Repository Context: cheron2000/trading-agent

> Reviewed on 2026-11-24. The complete recursive Git tree contained 341 entries (not truncated). Review covered the seven-layer application, live runner, broker adapter, risk engine, LLM/candle/ML strategies, data providers, dashboard and Telegram control paths, analytics, CI, tests, model training script, and key documentation. This is a static source review; no tests, broker calls, market-data calls, paper-trading session, or live-trading session were executed.

## 1. Executive summary

AI Trading OS is a Python 3.11+ event-driven trading platform organized into seven layers:

1. Foundation — events, configuration, validation, logging, serialization.
2. Communication — synchronous in-process EventBus, scheduler, health monitor, rate limiter.
3. Data — market/news providers, normalization, technical features, regime classification.
4. Intelligence — rule-based, Groq/ATLAS, Ollama, candle-pattern and online ML strategies.
5. Execution — deterministic paper order manager, portfolio tracking, risk engine, optional Alpaca adapter.
6. Analytics — realized-P&L metrics, hash-chained trade journal, reports.
7. Dashboard — Flask/SSE web UI, terminal view, Telegram alerts and commands.

The project has progressed substantially beyond the early architecture-only documentation. The repository contains the full runtime, a live-data runner, optional Alpaca paper-broker integration, a large test collection, and CI.

The architecture is not yet safe to treat as production/live-money ready. The strongest findings are in broker safety and remote-control surfaces:

- Alpaca's price lookup returns 0.0 on error, which makes the notional check pass. Its account-equity failure paths also skip drawdown enforcement. These are fail-open risk controls.
- Alpaca's fill poller treats partially_filled as a completed fill but records the originally requested quantity, overstating actual execution and corrupting local portfolio state.
- Telegram inbound commands do not check the sender/chat against the configured chat_id. Anyone who can message the bot can query portfolio information and invoke /stop.
- Flask dashboard strategy-switch, manual-tick, and kill endpoints have no authentication or CSRF/origin protection. The runner currently binds to loopback, which reduces remote reachability but does not replace request authorization.
- ATLAS embeds external news and trade-reflection text into an LLM prompt, despite SECURITY.md describing a numeric-only prompt boundary for the older PromptBuilder/LLMAgent path.
- Documentation overstates or conflicts with current code: default capital, strategy fallback, symbol universe, test count, and broker risk sizing differ across files.

The core design is promising, but risk enforcement, broker state reconciliation, access control, model validation, and release evidence need to be tightened before any live execution.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | cheron2000/trading-agent |
| Visibility | Public |
| Default branch | main |
| GitHub size | 1,000 KB |
| Archived | No |
| Recursive tree | 341 entries; complete |
| Product | AI Trading OS |
| Runtime | Python 3.11+ |
| Main runner | run_hour.py |
| Primary source | src/ |
| Dashboard | Flask + SSE |
| Cloud LLM | Groq API |
| Local LLM | Ollama |
| Broker adapter | Alpaca Trading API |
| Market data | yfinance; Alpha Vantage provider also present |
| Notifications | Telegram Bot API |
| CI | GitHub Actions workflow for lint, formatting, typing, tests, architecture lint |
| Test files | 53 paths matching test_*.py: 30 under src/, 14 under legacy Trading_agent/, 9 at repository root |

No actual keys.env or other credential file appeared in the recursive tree. .gitignore excludes *.env, .key, .pem, and common secret/config artifacts. This is a useful control, but secret scanning and Git-history checks were not performed.

## 3. Repository structure

- src/foundation/ — L1: base events, config, validation, logging
- src/communication/ — L2: EventBus, scheduler, rate limiter, health
- src/data/ — L3: market/news providers, features, pipeline
- src/intelligence/ — L4: ATLAS, LLM, Ollama, candle, ML strategies
- src/execution/ — L5: risk, portfolio, paper and Alpaca execution
- src/analytics/ — L6: metrics, trade journal, reports
- src/dashboard/ — L7: Flask/SSE, Telegram, terminal view
- src/tests/ — integration and cross-layer tests
- Trading_agent/ — older parallel Foundation/plugin package + tests
- data_store/fixtures/ — offline market tick fixture
- data_store/memory/ — trade_reflections.jsonl
- models/candle_rf.pkl — serialized candle classifier
- scripts/ — model training, setup, architecture lint
- deploy/ — service unit and setup instructions
- .github/workflows/ — Python CI
- run_hour.py — primary live-data + paper-execution runner
- backtest.py — backtest entry point
- load_keys.py — credential/config file loader
- requirements*.txt — runtime and development dependencies

The legacy Trading_agent/ directory contains a second Foundation-style package and 14 test files, while the active architecture and CI focus on src/. Its maintenance status and relationship to src/foundation/ should be made explicit. Avoid making changes in both namespaces without confirming which is canonical.

## 4. Runtime and event flow

The documented pipeline is:

Market provider
  -> DataPipeline
  -> Market normalization + feature engineering
  -> FeatureVectorEvent
  -> Strategy (ATLAS / Groq LLM / Ollama / simple rule)
  -> DecisionEvent
  -> RiskEngine
  -> OrderManager or AlpacaOrderManager
  -> FillEvent
  -> PortfolioTracker + MetricsEngine + TradeJournal
  -> Flask/SSE dashboard + Telegram notifications

The in-process EventBus is synchronous and thread-safe:
- Subscribers register event-pattern handlers using fnmatch-style matching.
- The bus snapshots matching handlers under a lock and invokes them outside the lock.
- Exceptions in one subscriber are logged and do not prevent remaining handlers from receiving the event.
- Rate limiting is optional and checked at publish time.

This is a modular monolith, not a distributed event platform. Events are not durably queued, replayed, or delivered across processes. A process crash loses in-memory events and state unless a specific component persists it.

## 5. Main runner: run_hour.py

The primary entry point:
- Starts with default capital of $200 in the reviewed source and default duration of 120 minutes.
- Uses a six-symbol universe: AAPL, MSFT, GOOGL, BTC-USD, ETH-USD, TSLA.
- Tries live yfinance data first and falls back to fixture data on fetch failure.
- Processes two symbols per cycle using round-robin rotation.
- Uses daily-trend filtering, a directional ML predictor, support/resistance calculations, ADX/ATR regime context, news context, and trade reflections.
- Supports ATLAS, GROQ-LLM, SIMPLE-RULE, and OLLAMA modes.
- Supports optional Telegram notifications and optional Alpaca paper execution through command-line flags.
- Uses local in-memory portfolio tracking, realized-P&L metrics, and a persisted trade journal under data_store/live/.
- Hosts the dashboard on 127.0.0.1:5000.

### Important execution-path observations
- The default --alpaca path explicitly constructs AlpacaOrderManager(live_trading=False); this flag uses Alpaca paper trading, not live trading.
- The local RiskEngine is configured for up to 25% per-position sizing (volatility-adjusted), while AlpacaOrderManager independently caps each order at 2% of peak portfolio value. With the default $200 capital, the local sizing layer can propose roughly $50 orders while the broker adapter permits only $4. This mismatch can reject most ordinary Alpaca orders. Align the two layers and test their combined behavior.
- The local portfolio is initialized from CLI capital and updated from locally constructed FillEvents. The runner does not visibly hydrate initial portfolio/positions from Alpaca account state on startup. Existing broker holdings, buying power, fractional quantities, and fills can therefore diverge from the local tracker.
- The main loop publishes fills to the bus after the order manager returns. AlpacaOrderManager also publishes its fill internally. Verify duplicate event delivery and analytics/journal subscription behavior before changing this path.
- If a market-data fetch fails, the runner switches to fixture data for the rest of the session. That fallback should be visible as a degraded/stale-data state and should not silently be treated as live trading evidence.
- The news fetch runs in a daemon thread with a five-second join timeout; if it continues after the timeout, the loop moves on while that thread may still mutate shared per-cycle dictionaries.
- Some exception paths in the news loop are silently ignored, despite SECURITY.md claiming exceptions are consistently logged.

## 6. Broker execution and risk controls

### 6.1 Paper order manager
src/execution/engine/order_manager.py is a deterministic local paper-fill adapter:
- Rejects live_mode=True with NotImplementedError.
- Uses the injected price feed as fill price.
- Publishes a FillEvent.
- Has no slippage, spread, market impact, partial-fill, or transaction-cost simulation.

This is appropriate as a basic integration-test executor, but its reported returns should not be presented as realistic trading performance without execution-cost modeling.

### 6.2 Alpaca adapter — critical fail-open defects
src/execution/broker/alpaca_order_manager.py includes a 2% per-trade notional cap, 10% session drawdown threshold, Alpaca order submission, and fill polling.

**P0 — market-price lookup failure bypasses the trade cap.**
- _get_current_price() catches all exceptions and returns 0.0.
- _check_capital_limit() computes notional as quantity × price.
- A zero price makes notional zero, so the order passes the 2% cap.
- The code comment explicitly says the capital check is skipped on lookup failure.

Fail closed: reject the order if a reliable price cannot be obtained. Do not substitute zero or another permissive value.

**P0 — account/equity lookup failure bypasses drawdown protection.**
- execute() falls back to the previous peak if get_portfolio_value() fails.
- _check_drawdown_limit() catches account-query errors and returns without rejecting the order.
- A broker outage or account endpoint failure can therefore allow execution without a verified current drawdown state.

Fail closed on unavailable equity/risk state, or use an independently verified cached state with strict age limits and an explicit degraded mode.

**P0 — partial fills are represented as full fills.**
- _FILLED_STATUSES contains both filled and partially_filled.
- _await_fill() returns only the average fill price.
- execute() creates the FillEvent with quantity=order.quantity, even when Alpaca reports a partial fill.
- The local portfolio and P&L can record more shares than the broker actually executed.

Wait for terminal order state, read actual filled quantity and average price, model remaining quantity/cancel/replace behavior, and reconcile the local portfolio to broker positions.

**P1 — price client is stock-only.** The adapter uses StockHistoricalDataClient and StockLatestTradeRequest for every symbol. The runner's universe includes BTC-USD and ETH-USD. Verify supported asset classes and canonical symbol formatting; implement a separate crypto data path or explicitly reject unsupported symbols. Current exception fallback to zero compounds the issue.

**P1 — live-trading confirmation is a caller-supplied boolean.**
The constructor allows live mode when paper_validation_complete=True. This is not evidence that 30 days of paper validation occurred; any caller can pass the flag. A stronger release gate should validate an auditable approval/config artifact, enforce account/environment separation, and require explicit operator confirmation outside ordinary code defaults.

**P1 — local portfolio and broker account are not reconciled.**
The broker adapter exposes account equity and positions, but the primary runner initializes a fresh local Portfolio from CLI capital and does not appear to restore broker positions. Before any broker execution, reconcile cash, positions, open orders, and average entry prices from the broker.

## 7. Risk engine

src/execution/risk/risk_engine.py enforces:
- Reject HOLD.
- Reject confidence below a configured threshold.
- Reject unknown symbols or non-positive prices.
- Size orders from portfolio cash and max position percentage.
- Enforce a minimum quantity.
- Cap total portfolio exposure based on marked equity.

This is a useful deterministic layer. However:
- It trusts the DecisionEvent's action/confidence and does not validate the originating strategy or independently verify the model's stated reward/risk calculation.
- Symbol allowlisting is identified as a gap in SECURITY.md. Validate instruments against an explicit approved universe, asset class, exchange, and trading-hours policy.
- The runner mutates the private risk_engine._max_position_pct attribute directly each cycle. Provide a validated public sizing-policy method or construct a new immutable risk policy instead.
- Risk limits are split between runner-level daily loss checks, RiskEngine exposure/position sizing, and Alpaca adapter caps. These must be composed and tested as one coherent risk policy.

## 8. Strategy and model layer

### 8.1 ATLAS
ATLAS builds a long prompt containing engineered market features, regime data, daily trend, ML probability, support/resistance, news context, and prior trade reflections. It rotates Groq API keys, requests JSON, parses action/confidence, and returns HOLD on call/parse errors.

Important gaps:
- The README and deployment guide describe Groq-to-Ollama fallback and claim the system can run without cloud keys. The reviewed AtlasStrategy requires at least one Groq key and only calls Groq; its error path returns HOLD. In run_hour.py, ATLAS is the default strategy and is initialized before Ollama is started. With no Groq key, default startup can fail instead of falling back to Ollama.
- The ATLAS prompt includes news_context and trade_reflections, which are external or persisted free text. These can contain prompt-injection content. SECURITY.md's numeric-only prompt statement describes the older PromptBuilder/LLMAgent path, not this ATLAS path. Delimit and treat external text as untrusted data, or exclude it from the action-generating prompt.
- The prompt requests confluence, stop-loss, take-profit, and reward/risk fields, but _parse_atlas_response() only consumes action, confidence, regime label, and reasoning. The model-provided risk fields and confluence claims are not validated or enforced. Critical controls must be computed and enforced in deterministic code.
- Confidence is model-reported, not statistically calibrated. The quarter-Kelly language in documentation should not be interpreted as an empirically validated probability estimate unless calibration data and methodology are supplied.
- Key status exposes masked key metadata to the dashboard. Keep it masked in all cases and never log or stream raw credentials.

### 8.2 Groq and Ollama strategies
- LLMStrategy uses the GroqClient adapter and strict JSON parsing with action/confidence/rationale validation.
- OllamaStrategy uses the local Ollama API and falls back to HOLD on malformed responses/errors.
- GroqClient rotates keys on HTTP/rate-limit failures. Key rotation is not a substitute for respecting provider quotas and terms.
- Ollama host is configurable through keys.env; if remote hosts are permitted, restrict destination/credentials and avoid exposing an unauthenticated Ollama server on a public interface.

### 8.3 Candle Intelligence Layer
- CandleStrategy fetches 5-minute candles, extracts candle features, uses a committed RandomForest model, and optionally asks Ollama to review the model signal.
- If Ollama is unavailable, the implementation returns the pretrained model's BUY/SELL signal directly when its model confidence clears the threshold. This means the LLM reviewer is not a mandatory safety gate.
- The training script can fall back to synthetic Geometric Brownian Motion candles. The tracked model's exact training provenance, data snapshot, validation split, and calibration are not independently established by repository metadata. Do not assume the committed model is trained exclusively on real market data merely from the models README.
- No evidence of a robust walk-forward or out-of-sample validation report was found in the reviewed paths.

### 8.4 Directional ML predictor
DirectionalPredictor trains a GradientBoostingClassifier on-the-fly using roughly the most recent 100 price/volume observations per symbol and retrains every 30 minutes.
- This is a very small, highly local training sample; there is no visible walk-forward evaluation or calibration.
- RSI, MACD, Bollinger, and ATR features are approximated from close-price sequences rather than full OHLC bars.
- Its output is one input to ATLAS context, not an independently validated forecast.
- Treat P(price_up_next_interval) as an unvalidated model score until measured against a time-ordered holdout and baseline.

### 8.5 Regime classifier
regime_classifier.py approximates ADX/ATR using a single price series rather than full high/low/close OHLC candles. It is useful as a rough feature, but the labels should not be described as standard Wilder ADX/ATR calculations without qualification.

## 9. Data and market context

- YFinanceProvider batches and caches data, retries with backoff, and can route requests through Tor.
- The runner calls yfinance for current prices and daily trend; yfinance data is not a guaranteed exchange-grade real-time feed.
- Alpha Vantage and multiple news sources (Finnhub, Alpha Vantage, Yahoo) are implemented as optional providers.
- The runner's fallback to fixture data can make a run continue during data-source failure. Record source, timestamp, staleness, and fallback status on every decision and block execution when data is too stale.
- Tor IP rotation is described as a way to bypass Yahoo Finance rate limiting. This is operationally fragile and may conflict with provider terms or anti-abuse expectations. Prefer an authorized market-data plan with documented rate limits.
- The current universe mixes US equities and crypto. Market-hours checks, asset-specific data APIs, precision, fractional sizing, order type, and fee handling must be asset-class aware.

## 10. Dashboard and remote-control security

### 10.1 Flask web dashboard
src/dashboard/web/app.py exposes:
- GET /, GET /api/snapshot, GET /api/state, GET /stream
- POST /api/control/tick
- POST /api/control/strategy
- POST /api/control/kill

No authentication, authorization, CSRF token, or Origin/Host validation is visible on the control endpoints or private portfolio snapshot endpoints.

The main runner currently binds Flask to 127.0.0.1, which limits direct network exposure. This is still not a complete security boundary: local malicious web pages can attempt requests to loopback services, and reverse-proxy/container deployment could expose routes more broadly. Require local authentication or a per-run secret, enforce Origin/Host/CSRF protections for state-changing routes, and keep dashboard binding explicit.

The optional create_app(state) injection path is not consistently used by all routes: the SSE and control endpoints reference module-level _ds rather than the injected state. This can make isolated app instances/tests behave differently from expected.

### 10.2 Telegram bot — missing inbound authorization
TelegramNotifier registers /status, /positions, /pnl, and /stop. The configured chat_id is used for outbound notifications, but the inbound command handlers do not compare the incoming sender/chat ID to an allowlist.

Consequences:
- Anyone who can find and message the bot may query portfolio/position/P&L information.
- Anyone who can message the bot may publish system.shutdown_requested via /stop.

Add an authorization check to every inbound handler using an explicit allowlist of Telegram user IDs and/or chat IDs. Reject all commands by default and test unauthorized senders for all four commands.

## 11. Analytics and audit trail

### TradeJournal
- Stores append-only JSONL records with a SHA-256 hash chain.
- Supports integrity verification and loading from a persisted file.
- Hash chaining detects modification only when the trusted chain head is anchored elsewhere. An attacker able to rewrite the entire file can recompute the hashes. It is tamper-evident, not independently immutable.
- File persistence uses local storage; access control, backup, rotation, and external chain-head anchoring are not shown.

The tracked data_store/memory/trade_reflections.jsonl should be reviewed for sensitive trading history or personally identifying notes before sharing/forking. No individual record values are reproduced in this report.

### MetricsEngine
- Computes realized P&L from SELL fills and entry prices.
- Total return is realized P&L divided by initial capital.
- The equity curve used for max drawdown advances only on realized SELL P&L; it does not represent mark-to-market portfolio equity.
- Sharpe is computed from per-trade P&L amounts and multiplied by sqrt(252). These are not periodic portfolio returns, and the annualization factor is not justified for variable-duration trades.
- Therefore reported Sharpe and max drawdown are not standard portfolio-performance statistics. Use time-indexed equity returns, include unrealized P&L and costs, and annualize according to the sampling frequency.

## 12. CI, tests, and code quality

### CI workflow
.github/workflows/python-ci.yml runs on push and pull request:
1. Install dev dependencies on Ubuntu / Python 3.12.
2. Ruff lint on src/.
3. Black check on src/.
4. MyPy on src/ with continue-on-error: true.
5. Pytest on src/ with coverage threshold 60%.
6. Architecture lint.

Important distinctions:
- MyPy is informational/non-blocking because the workflow explicitly sets continue-on-error: true.
- The 60% coverage threshold reflects documented test debt; it is not evidence of high coverage.
- 30 test files are under src/ and are within the CI's pytest src/ discovery path.
- 14 tests in Trading_agent/ and 9 root-level test_*.py scripts are outside that explicit CI target. Some root scripts appear to be manual API/key/network checks rather than unit tests.
- pytest.ini sets testpaths = src/tests, but CI passes src/ explicitly, so nested tests under src/*/tests are also discovered by that command.
- README claims 562 automated tests passing and deployment docs state all CI checks pass as of 2026-08-06. Those are repository documentation claims; current CI was not executed during this review.

### Test debt
TEST_DEBT.md records reduced coverage from 80% to 60% after new ATLAS, Telegram, Alpaca, and credential-loader features. It explicitly lists missing property/unit tests for critical broker gates, Telegram lifecycle and commands, ATLAS fallback, and credential parsing.

No tests were run in this review. The current GitHub workflow configuration and source files were inspected only.

## 13. Documentation inconsistencies

The repository contains many freeze manifests, build reports, deployment notes, bug lists, and status summaries from different dates. They are not all synchronized with the active code.

Examples:
- README describes default capital as $100,000, while run_hour.py defaults to $200.
- README describes ATLAS as Groq with automatic local Ollama fallback and says the system can run without keys; the current ATLAS constructor requires Groq keys and its failure path returns HOLD.
- DEPLOYMENT_COMPLETE.md describes a crypto-only universe (BTC, ETH, SOL, AVAX, MATIC, LINK), while the current runner has AAPL, MSFT, GOOGL, BTC-USD, ETH-USD, TSLA.
- Deployment documentation says the per-trade risk cap is 2%, while the runner's local RiskEngine is configured to size up to 25% per position and the broker adapter applies the 2% cap independently.
- Some older architecture documentation says live execution is not implemented, while a later Alpaca adapter exists. The current runner's --alpaca mode is paper-only.
- The README test count (562) and older test-debt reports describe different snapshots. Current tree inventory gives 30 test modules under src/, plus 23 tests/scripts outside that CI target; the number of test cases was not executed or recounted.

Create one generated release manifest from code/config and date-stamp historical documents. Keep one canonical current architecture, operational guide, and risk policy.

## 14. Prioritized roadmap

### P0 — before enabling any broker order path
1. Make Alpaca price lookup fail closed; never permit a trade when price is unknown.
2. Make account/equity/drawdown lookup fail closed; do not skip drawdown checks on API failure.
3. Correct partial-fill handling using actual filled quantity, average price, and terminal order status.
4. Reconcile broker account, open positions, open orders, and buying power before/after each run.
5. Align local RiskEngine sizing and broker per-trade caps; test combined configuration.
6. Add Telegram inbound user/chat allowlists to every command; unauthorized /stop must be impossible.
7. Protect dashboard state/control routes with authentication and CSRF/Origin protections.

### P1 — strategy/model safety
1. Implement and test the documented ATLAS Groq-to-Ollama fallback, or correct the docs/default strategy behavior.
2. Treat news and trade reflections as untrusted prompt content; delimit/strip/validate or remove free text from action prompts.
3. Enforce stop-loss, take-profit, confluence, regime, and symbol constraints deterministically outside the LLM.
4. Establish walk-forward validation, calibration, and baseline comparisons for the directional predictor and candle classifier.
5. Record the committed model's training data, source, label definition, version, metrics, and holdout performance.
6. Use OHLCV-based regime calculations or clearly label the current price-only approximations.

### P1 — execution and analytics correctness
1. Add slippage, spread, fees, partial fills, and rejected/canceled orders to paper simulation.
2. Use broker-native positions/equity as source of truth in Alpaca mode.
3. Rework metrics using time-indexed marked-to-market equity returns and valid Sharpe/drawdown calculations.
4. Anchor journal chain heads externally and implement backup/retention/access controls.
5. Ensure stale/fixture fallback data is not silently eligible for broker execution.

### P2 — maintainability and operations
1. Declare src/ as the only active package; archive/remove or explicitly maintain Trading_agent/.
2. Move secrets from local flat files to a secrets manager for hosted deployments; preserve local-only development support.
3. Split run_hour.py into configuration, lifecycle, strategy orchestration, execution, and shutdown modules.
4. Make symbol universe, asset class, market hours, and data-provider settings configuration-driven.
5. Add structured run manifests, data-source quality indicators, and reproducible backtest artifacts.
6. Make MyPy blocking after resolving the current errors; raise coverage threshold only after adding tests.
7. Add dependency pinning/lockfile, vulnerability scanning, and release artifact provenance.
8. Update README, deployment guide, architecture docs, and freeze manifests from one canonical configuration source.

## 15. Coding-agent handoff

Start with the Alpaca adapter and Telegram command handlers. Add failing tests for:
- Price lookup error must reject order.
- Account-equity/drawdown lookup error must reject order.
- Partial fill must use actual filled quantity and must not report a complete fill.
- Unsupported crypto symbol/asset class must be rejected explicitly or use a supported crypto endpoint.
- Telegram commands from non-allowlisted user/chat must not return account information or stop the runner.
- Dashboard control endpoints must reject unauthenticated and cross-origin state-changing requests.
- Alpaca paper mode must use one consistent risk budget across RiskEngine and broker adapter.

Then fix ATLAS startup/fallback and prompt trust boundaries. Do not tune ML thresholds or claim model performance before establishing time-ordered, leakage-resistant evaluation. Finally reconcile the documentation to the actual current runner and configuration.

## 16. Validation status and limitations

- Repository metadata and recursive tree: inspected (341 entries).
- README, architecture, security policy, prioritized bugs, test debt, deployment docs, CI: inspected.
- Runner, EventBus, market/news providers, strategies, risk engine, paper and Alpaca execution, analytics, dashboard, Telegram, model training script: inspected.
- Test inventory: counted by file path (30 under src/, 14 under Trading_agent/, 9 at repository root).
- Tests, lint, type checking, architecture lint, broker calls, market-data calls, Telegram polling, backtests, and trading sessions: not executed.
- No performance, profitability, or model-accuracy claim was independently validated.
- No credentials or individual trade-reflection contents are reproduced here.
