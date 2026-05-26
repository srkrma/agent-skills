---
name: pine-script-v6-development
description: Develops trading indicators, strategies, and charting tools in Pine Script v6. Use when building TradingView scripts, creating custom indicators, developing algorithmic trading strategies, or integrating with charting systems. Use when working with real-time market data, backtesting requirements, or chart-based automation.
---

# Pine Script v6 Development

## Overview

Pine Script v6 is TradingView's domain-specific language for developing trading indicators, strategies, and charting tools. This skill provides a structured approach to building reliable, performant, and maintainable Pine Script code that integrates seamlessly with TradingView's ecosystem.

Pine Script v6 introduces type safety, improved performance, and modern language features. This skill covers the complete development lifecycle: specification, strategy/indicator architecture, data handling, backtesting, and publication.

## When to Use

- Creating custom trading indicators for TradingView charts
- Developing algorithmic trading strategies for backtesting or live trading
- Building alert systems or automated charting tools
- Optimizing existing Pine Script v5 code for v6
- Integrating with market data feeds or external APIs (via webhooks/alerts)
- Implementing technical analysis patterns (moving averages, oscillators, price action, etc.)
- Debugging strategy backtest discrepancies or indicator calculation errors

**When NOT to use:** Simple chart overlays that don't require custom logic, using only built-in indicators without modification.

## Core Concepts

### 1. Script Types

Pine Script has three primary types:

| Type | Purpose | Use Case |
|---|---|---|
| **Indicator** | Draws on chart, analyzes price/volume | Technical analysis, overlay studies |
| **Strategy** | Executes trades, backtests | Automated trading, performance testing |
| **Library** | Reusable functions and types | Code sharing, modular architecture |

Choose the type based on whether you need execution (`strategy`) or visualization (`indicator`).

### 2. Timeframes and Context

Pine Script processes one bar at a time. Understand:

- **Real-time bar**: Current incomplete bar (updates on every tick)
- **Closed bar**: Finalized historical bar (immutable)
- **bar_index**: Position of current bar in chart history
- **Timeframe context**: Script runs at the specified chart timeframe; use `request.security()` to fetch other timeframes

Always clarify:
1. What timeframe(s) does the strategy/indicator target?
2. Does logic depend on real-time or closed bars?
3. Are you fetching data from other timeframes?

### 3. Order Execution Model

Strategies execute orders with lookahead bias considerations:

- Orders execute at the **next bar's open** (not current real-time bar)
- Backtesting uses historical closes; live trading uses actual fills
- Commission, slippage, and spread impact backtesting accuracy

Document assumptions about order execution and slippage.

## The Development Process

### Phase 1: Specification

**Define the trading logic before writing code.**

```markdown
# Spec: [Indicator/Strategy Name]

## Objective
What market behavior are we capturing? What decision does this enable?

## Trading Logic
- Entry conditions (with exact price levels or indicators)
- Exit conditions (profit target, stop loss, time-based)
- Position sizing rules
- Risk management (max drawdown, max loss per trade)

## Technical Details
- Input parameters (thresholds, periods, sources)
- Indicator calculations (formulas, data sources)
- Timeframe target (1m, 5m, 1h, 1d)
- Data requirements (open, high, low, close, volume, etc.)

## Backtesting Strategy
- Historical period (how far back are we testing?)
- Initial capital and position size
- Commission/slippage assumptions
- Success metrics (Sharpe ratio, win rate, drawdown, profit factor)

## Boundaries
- Always: Use `var` for persistent state, validate inputs
- Ask first: Accessing data from other timeframes, using premium TradingView features
- Never: Hard-code prices, use incomplete bar data for entries (unless real-time trading intended)
```

**Red flag:** "Build me a strategy that makes money" without defining specific entry/exit logic.

### Phase 2: Architecture

Structure your Pine Script around these principles:

```pinescript
//@version=6
indicator("My Indicator", overlay=false)

// 1. INPUT SECTION — User-configurable parameters
group_ma = "Moving Averages"
period_fast = input(20, "Fast MA Period", group=group_ma)
period_slow = input(50, "Slow MA Period", group=group_ma)

group_alerts = "Alerts"
enable_alerts = input(true, "Enable Alerts", group=group_alerts)

// 2. CALCULATION SECTION — Core logic
ma_fast = ta.sma(close, period_fast)
ma_slow = ta.sma(close, period_slow)

// 3. SIGNAL SECTION — Trading signals
bullish = ma_fast > ma_slow
bearish = ma_fast < ma_slow

// 4. OUTPUT SECTION — Plotting and alerts
plot(ma_fast, "Fast MA", color.blue)
plot(ma_slow, "Slow MA", color.orange)

alertcondition(bullish, title="Bullish Cross", message="MA crossover bullish")
```

Benefits:
- **Readability**: Clear sections for parameters, logic, signals, output
- **Testability**: Isolated calculation logic can be verified independently
- **Maintenance**: Changes to one section don't cascade unpredictably

### Phase 3: Implementation

Follow these patterns for robust Pine Script:

#### Data Fetching
```pinescript
// Current chart data
c = close              // Most recent close
h = high               // Highest in bar
o = open               // Opening price

// Alternative timeframe data
btc_price = request.security("BITSTAMP:BTCUSD", "D", close)

// Handle missing data gracefully
vol = request.security(syminfo.tickerid, "1D", volume)
if na(vol)
    vol := 0           // Default to 0 if unavailable
```

**Key rule**: Always check for `na()` (not available) values when using `request.security()`.

#### State Management
```pinescript
// Persistent variables across bars
var float entry_price = na
var int trades = 0

// Reset on new day
if dayofweek != dayofweek[1]
    trades := 0

// Update conditionally
if buy_signal
    entry_price := close
    trades += 1
```

Use `var` for variables that must retain state across bars. Initialize with `na` for price values.

#### Arrays and Collections (v6 feature)
```pinescript
// Dynamic arrays for storing multiple values
var array<float> recent_highs = array.new<float>()

// Add and remove elements
array.push(recent_highs, high)
if array.size(recent_highs) > 20
    array.shift(recent_highs)

// Access elements
highest_value = array.max(recent_highs)
```

#### Strategies: Entry and Exit
```pinescript
//@version=6
strategy("MA Crossover", overlay=true, default_qty_type=strategy.percent_of_equity, default_qty_value=100)

ma_fast = ta.sma(close, 20)
ma_slow = ta.sma(close, 50)

// ENTRY
if ma_fast > ma_slow and strategy.position_size == 0
    strategy.entry("Long", strategy.long)

// EXIT: Profit Target
if strategy.position_size > 0
    strategy.exit("Take Profit", "Long", profit=100)

// EXIT: Stop Loss
if strategy.position_size > 0
    strategy.exit("Stop Loss", "Long", loss=50)
```

**Critical**: Always check `strategy.position_size == 0` before entering to avoid multiple concurrent entries.

### Phase 4: Testing and Validation

#### Backtesting Checklist
- [ ] Strategy runs without compilation errors
- [ ] Backtest covers at least 2+ years of data
- [ ] Results show positive Sharpe ratio (> 1.0 is good)
- [ ] Drawdown is acceptable (typically < 30%)
- [ ] Win rate > 40% (adjust based on risk/reward)
- [ ] No look-ahead bias (entries use only prior bar data)

#### Common Pitfalls
| Issue | Cause | Fix |
|---|---|---|
| Backtest shows 100% wins | Look-ahead bias (using current bar's close for entry) | Move logic to next bar or use `open[1]` |
| Strategy stops executing mid-backtest | Insufficient capital for position size | Reduce `default_qty_value` or add capital checks |
| Indicator repaints | Using real-time bar data for signals | Only use closed bar data (`close[1]`, not `close`) |
| Commission mismatch | Backtest commission ≠ broker commission | Set commission in strategy properties to match broker |

#### Manual Verification
1. **Chart inspection**: Enable the strategy/indicator on a live chart
2. **Signal validation**: Do signals align with price action? Do they make sense visually?
3. **Parameter sensitivity**: Test with +/- 10-20% changes to key parameters
4. **Edge cases**: What happens during gaps, low liquidity, or extreme volatility?

### Phase 5: Code Style and Best Practices

**Naming Conventions:**
```pinescript
// Variables: snake_case
entry_price = close
max_drawdown = 0.15

// Booleans: descriptive, often past tense
is_bull_market = close > ma_slow
has_crossed_above = ta.crossover(ma_fast, ma_slow)

// Constants: UPPER_SNAKE_CASE
MAX_POSITIONS = 5
RISK_PER_TRADE = 0.02

// Functions: camelCase (PineScript convention)
f_calculate_rsi() => ta.rsi(close, 14)
```

**Code Formatting:**
```pinescript
// Good: Clear logic with comments
if close > ma and volume > avg_volume
    // High conviction bullish setup
    strategy.entry("Long", strategy.long)

// Bad: Dense, unclear
if close>ma and volume>avg_volume then strategy.entry("Long",strategy.long)
```

**Comment Discipline:**
- Add comments for non-obvious logic (the "why", not the "what")
- Document assumptions about data or timeframes
- Flag performance-critical sections

```pinescript
// Fetch daily close even on minute charts (required for overnight gaps)
daily_close = request.security(syminfo.tickerid, "D", close)

// Use var to avoid recalculating on every bar
var float highest = na
if na(highest)
    highest := close
```

## Integration and Deployment

### Publishing to TradingView Community

1. **Documentation**: Write clear description of logic and parameters
2. **License**: Choose appropriate license (open source or closed source)
3. **Disclaimer**: Add risk warning for strategies
4. **Version control**: Use meaningful version numbers (v1.0, v1.1, etc.)

### Webhook Integration

For alerting or external system integration:

```pinescript
alertcondition(entry_signal, title="Long Entry", message="Symbol {{ticker}} crossed above MA")
```

Configure webhook in TradingView alert settings to POST to external API.

## Verification Checklist

Before considering a Pine Script complete:

### Specification
- [ ] Trading logic is explicitly documented
- [ ] All entry/exit conditions are defined
- [ ] Backtest period and metrics are specified

### Code Quality
- [ ] No compilation warnings or errors
- [ ] Variable names are descriptive and consistent
- [ ] State management uses `var` correctly
- [ ] All `request.security()` calls include `na()` checks

### Testing
- [ ] Backtest runs successfully with 2+ years of data
- [ ] Results meet success criteria from spec
- [ ] Manual chart validation confirms signals make sense
- [ ] Parameter sensitivity has been tested

### Documentation
- [ ] Input parameters are grouped and documented
- [ ] Strategy/indicator purpose is clear in comments
- [ ] Known limitations are noted

### Deployment
- [ ] Code has been tested on live chart (papertrading)
- [ ] Commission and slippage settings match broker
- [ ] Risk management rules are enforced

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I don't need to specify logic, I'll code it as I go" | Ad-hoc logic leads to inconsistent signals. Define entry/exit *before* coding. |
| "Backtest shows great results, ship it" | Past performance ≠ future results. Test parameter sensitivity and validate on live data. |
| "I can use real-time bar `close` for entries" | This causes repainting and look-ahead bias. Use prior bar data only. |
| "More indicators = more accurate" | Over-optimization curve-fits to historical data. Keep logic simple and testable. |
| "This works on daily bars, so it'll work on 5-minute bars too" | Timeframe changes market microstructure, commission impact, and signal quality. Test separately. |

## Red Flags

- Backtest shows unrealistic results (>30% annual return with <5% drawdown)
- Strategy uses incomplete bar data (`close` without `[1]`) for entries
- No documentation of entry/exit logic
- Backtesting results vary wildly with small parameter changes
- Strategy stops executing mid-backtest without explanation
- Using premium TradingView data without noting compatibility
- Commission and slippage settings don't match actual broker

## External Resources

- **TradingView Pine Script Documentation**: https://www.tradingview.com/pine-script-docs/
- **TradingView Community Scripts**: https://www.tradingview.com/scripts/
- **Pine Script API Reference**: Reference section in TradingView IDE

