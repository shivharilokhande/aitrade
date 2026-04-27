# AutoScalp — Complete Development Specification
## AI-Ready Build Document

**Purpose:** This document contains everything needed to build the AutoScalp algorithmic trading system from scratch. Give this entire document to an AI coding assistant (Claude Code, Cursor, Copilot) and it can produce the complete working system.

**Current System:** 27 Python files, 11,064 lines, fully functional
**Target:** Indian equity derivatives (NIFTY, BankNIFTY, FinNIFTY options)
**Broker:** Zerodha Kite Connect API
**Validated:** 56.9% win rate on 1,402 trades across 553 days of real BankNIFTY data

---

# PART 1: PRODUCT REQUIREMENTS DOCUMENT (PRD)

## 1.1 Product Vision

Build a fully automated algorithmic trading system that:
- Connects to Zerodha's live market data via WebSocket
- Detects high-probability trading signals on NIFTY, BankNIFTY, and FinNIFTY
- Buys options (CE for bullish, PE for bearish) with precise position sizing
- Manages exits with stop-loss, take-profit, and trailing stop-loss
- Stops trading after 2 stop-loss hits per day (critical rule)
- Runs autonomously during market hours (9:15 AM - 3:30 PM IST)
- Provides a web dashboard for monitoring
- Uses Claude AI for pre/post market analysis

## 1.2 User Persona

Individual retail trader in India with:
- Zerodha trading account
- Kite Connect API subscription (₹2,000/month)
- Starting capital: ₹50,000 - ₹1,00,000
- Basic terminal/command-line knowledge
- No coding knowledge required for daily operation

## 1.3 Core Features (47 total)

### Signal Detection (F-001 to F-007)
- F-001: Real-time tick ingestion via Kite WebSocket for 3 instruments simultaneously
- F-002: Build 1-minute, 5-minute, and 15-minute OHLCV candles from raw ticks
- F-003: Detect 4 trading setups: Opening Range Breakout (ORB), VWAP Pullback, EMA Momentum Crossover, Volume Breakout
- F-004: Layer 1 confluence gate — require minimum 2 of 4 setups to agree on direction within a 5-bar window
- F-005: Layer 2 confirmation — require 3 of 4 indicators to confirm (15-min Supertrend, MACD histogram, relative volume, candle structure)
- F-006: Signal buffer — when multiple instruments fire on same candle, compare and pick the highest confidence signal
- F-007: Volatility regime filter — skip signals when ATR is in the bottom 25th percentile (market too quiet)

### Order Execution (F-008 to F-013)
- F-008: Option chain selector — fetch NFO instruments, find ATM/ITM strike, check liquidity, pick correct weekly expiry (next week on expiry day)
- F-009: LIMIT order placement at LTP + 0.5% (not MARKET) to reduce slippage
- F-010: Position sizing at 30% of current capital per trade (20% after first SL hit of the day)
- F-011: Exchange-side SL-M order placed immediately after every fill (hardware stop)
- F-012: Paper trading mode with simulated fills using live prices (default mode)
- F-013: Live trading mode with real order execution via Kite API

### Exit Management (F-014 to F-019)
- F-014: Stop-loss at 25% premium drop from entry
- F-015: Take-profit at 30% premium rise from entry
- F-016: Trailing stop-loss activates at 45% profit, follows peak with 9% gap
- F-017: IV-aware TP tightening — when IV is elevated (>1.3 std above 20-day mean), tighten TP to 20%
- F-018: Max holding time enforcer (optional, currently disabled — full day hold proven better)
- F-019: End-of-day auto-close all positions at 3:25 PM

### Daily Risk Rules (F-020 to F-025)
- F-020: 2-SL daily halt — if 2 stop-losses hit in one day (not counting trailing SL exits), stop all trading for the rest of the day. THIS IS THE MOST IMPORTANT RULE.
- F-021: 20% daily profit target — if capital grows 20% in one day, stop trading and lock profits
- F-022: Reduced allocation — after first SL of the day, reduce from 30% to 20% for subsequent trades
- F-023: Gap day ORB skip — if opening gap exceeds 0.5% from previous close, disable ORB setup (other setups still work)
- F-024: Recovery trade — after 2-SL halt, if it's past 1:30 PM and a full Layer 1+2 signal fires, allow exactly 1 more trade at 10% allocation. Mark as taken, never repeat.
- F-025: Maximum 1 concurrent position across all instruments

### Safety Mechanisms (F-026 to F-047)
- F-026: Crash recovery from persisted state file (JSON, atomic write with os.replace)
- F-027: PID lockfile preventing duplicate instances
- F-028: Token health monitoring every 5 minutes (check if Kite access token expired)
- F-029: Position reconciliation with broker every 60 seconds
- F-030: Flash crash protector — halt if index moves >3% in a short window
- F-031: Rate limiter for Kite API (token bucket, max 10 req/sec)
- F-032: Heartbeat watchdog with external cron monitor
- F-033: Thread-safe capital management (RLock on all reads/writes)
- F-034: Late fill detector for cancelled orders that fill after cancellation
- F-035: NSE holiday calendar (skip trading on holidays)
- F-036: Expiry day handler — auto-select next week's expiry on expiry day or day before
- F-037: Special events calendar — Budget day, RBI MPC, election days reduce allocation to 15%
- F-038: NaN/Inf sanitization on all feature vectors before ML
- F-039: Trend exhaustion filter (skip if trend age exceeds threshold)
- F-040: Model health tracker (log warning if ML win rate drops below threshold)

### AI Integration (F-041 to F-044)
- F-041: Claude AI pre-market analyzer — runs at 8:30 AM, analyzes last 5 days of trades, generates day_plan.json with allocation multiplier, direction preference, instruments to avoid, max trades override
- F-042: Claude AI post-market reviewer — runs at 4:00 PM, reviews every trade, grades A-F, identifies patterns, suggests improvements
- F-043: Claude AI weekly strategy adapter — runs Sunday evening, reviews week's daily reviews, suggests parameter changes
- F-044: Day plan applier — reads day_plan.json at session start, applies adjustments. Explicit fallback: if file missing/stale/invalid, trade with default parameters and log warning. NEVER silently use yesterday's plan.

### Monitoring (F-045 to F-047)
- F-045: Streamlit web dashboard with start/stop controls, 6 stat cards, rule status, trade cards, P&L charts
- F-046: Setup check script (verifies Python, dependencies, files, config, .env)
- F-047: Daily token generation helper script

## 1.4 Non-Functional Requirements

- NFR-001: Signal to order latency under 500ms
- NFR-002: WebSocket reconnection without losing state
- NFR-003: Dashboard refreshes every 3 seconds without blocking trading
- NFR-004: Structured JSON logging for all trades, signals, risk events
- NFR-005: Default to paper mode (safety first)
- NFR-006: Python 3.9+ on Linux, macOS, Windows
- NFR-007: Memory under 512MB during normal operation

---

# PART 2: TECHNICAL REQUIREMENTS DOCUMENT (TRD)

## 2.1 Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Python | 3.9+ |
| Broker API | kiteconnect | 5.0+ |
| Data Feed | Kite WebSocket (KiteTicker) | via kiteconnect |
| Numerical | numpy | 1.24+ |
| Data Analysis | pandas | 2.0+ |
| ML (optional) | xgboost | 2.0+ |
| RL (optional) | stable-baselines3 | 2.1+ |
| Dashboard | streamlit | 1.28+ |
| Charts | plotly | 5.18+ |
| AI Advisor | Anthropic Claude API | claude-sonnet-4-20250514 |
| HTTP | requests | 2.31+ |
| Config | python-dotenv | 1.0+ |

## 2.2 File Structure (27 files)

```
zerodha-algo-trader/
├── config/
│   ├── __init__.py                 # Package marker
│   └── settings.py                 # ALL configuration (dataclasses, frozen)
├── logs/                           # Runtime logs (JSON Lines)
├── models/                         # ML models (auto-generated)
├── data/                           # State files, plans, historical data
├── main_v3.py                      # ENTRY POINT — orchestrator (1,100+ lines)
├── strategy_highwr.py              # High win-rate strategy engine (660+ lines)
├── strategy.py                     # Original strategy with Signal enum (460 lines)
├── strategy_patches.py             # Strategy fixes (285 lines)
├── data_engine.py                  # WebSocket → candle builders (345 lines)
├── execution.py                    # Order placement via Kite (560 lines)
├── paper_trading.py                # Simulated execution (465 lines)
├── options_sizer.py                # Position sizing (225 lines)
├── options_enhanced.py             # Black-Scholes Greeks, IV (435 lines)
├── live_trading_fixes.py           # Option chain, slippage, events (410 lines)
├── risk_manager.py                 # Position limits, daily loss (265 lines)
├── safety_layer.py                 # 29 safety mechanisms (925 lines)
├── final_fixes.py                  # Corner cases, trailing SL (625 lines)
├── claude_advisor.py               # AI integration (600 lines)
├── ai_engine.py                    # XGBoost + PPO RL (245 lines)
├── log_system.py                   # JSON structured logging (190 lines)
├── dashboard.py                    # Streamlit web UI (335 lines)
├── backtester.py                   # Walk-forward backtest (760 lines)
├── model_trainer.py                # ML training pipeline (465 lines)
├── run_pipeline.py                 # CLI pipeline runner (385 lines)
├── profit_simulator.py             # Monte Carlo simulator (290 lines)
├── yearly_projection.py            # Annual forecasts (345 lines)
├── watchdog_cron.py                # External heartbeat monitor (160 lines)
├── generate_token.py               # Daily token helper (70 lines)
├── setup_check.py                  # Setup verification (110 lines)
├── requirements.txt                # Dependencies
├── .env.example                    # Credential template
├── .gitignore                      # Excludes .venv, __pycache__, .env, logs
└── README.md                       # Quick start guide
```

## 2.3 Configuration Specification (config/settings.py)

Use Python dataclasses with `frozen=True`. Every parameter must have a comment.

### KiteConfig
```
api_key: str          # from .env KITE_API_KEY
api_secret: str       # from .env KITE_API_SECRET
access_token: str     # from .env KITE_ACCESS_TOKEN (refreshed daily)
request_token: str    # from .env KITE_REQUEST_TOKEN
```

### RiskConfig (BACKTEST-PROVEN VALUES — do not change without re-validation)
```
capital: 50000.0
capital_per_trade_pct: 0.30        # 30% of current capital per trade
option_sl_pct: 0.25                # 25% premium drop = stop loss
option_tp_pct: 0.30                # 30% premium rise = take profit
trail_activate_pct: 0.45           # trailing SL activates at 45% profit
trail_gap_pct: 0.09                # 9% gap between peak and trailing SL
max_trades_per_day: 5              # hard max (2-SL rule usually stops at 2-3)
max_sl_per_day: 2                  # STOP after 2 SL hits
daily_profit_target_pct: 0.20      # STOP if capital up 20% today
reduced_alloc_after_sl: 0.20       # reduce to 20% after 1st SL
opening_gap_skip_pct: 0.005        # skip ORB if gap > 0.5%
breakeven_sl_at_pct: 99.0          # DISABLED — hurts on real data
sl_tighten_after_minutes: 9999     # DISABLED — hurts on real data
setup_min_winrate: 0.0             # DISABLED — was filtering good trades
max_concurrent_positions: 1
```

### StrategyConfig
```
min_confluence_setups: 2           # Layer 1: need 2 of 4 setups
confluence_window_bars: 5          # setups must agree within 5 bars
min_layer2_passed: 3               # Layer 2: need 3 of 4 confirmations
rsi_period: 14
atr_period: 14
```

### OptionsConfig
```
lot_size_nifty: 65                 # NSE Jan 2026 lot sizes
lot_size_banknifty: 30
lot_size_finnifty: 60
preferred_delta: 0.5
prefer_futures: False              # always buy options
```

### Instrument Tokens
```
NIFTY_TOKEN = 256265               # NIFTY 50
BANKNIFTY_TOKEN = 260105           # NIFTY BANK
FINNIFTY_TOKEN = 257801            # NIFTY FIN SERVICE
```

### SystemConfig instruments
```
instruments: [NIFTY_TOKEN, BANKNIFTY_TOKEN, FINNIFTY_TOKEN]
trading_symbols: {
    256265: "NIFTY 50",
    260105: "NIFTY BANK",
    257801: "NIFTY FIN SERVICE",
}
exchange: "NSE"
```

---

# PART 3: ARCHITECTURE SPECIFICATION

## 3.1 System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        AUTOSCALP v3                              │
│                                                                  │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────────────┐ │
│  │ Zerodha │◄──►│ Data Engine   │───►│ Strategy Engine         │ │
│  │ Kite    │    │ (WebSocket)   │    │ (4 Setups + Layer 1+2) │ │
│  │ Connect │    │ 1m/5m/15m     │    │ Per instrument          │ │
│  │ API     │    └──────────────┘    └────────┬────────────────┘ │
│  │         │                                  │                  │
│  │         │    ┌──────────────┐    ┌────────▼────────────────┐ │
│  │         │    │ Signal Buffer│◄───│ Signals from NIFTY,     │ │
│  │         │    │ (best pick   │    │ BankNIFTY, FinNIFTY     │ │
│  │         │    │  across 3)   │    └─────────────────────────┘ │
│  │         │    └──────┬───────┘                                │
│  │         │           │                                        │
│  │         │    ┌──────▼───────┐    ┌─────────────────────────┐ │
│  │         │    │ Risk Manager │    │ Claude AI Day Plan      │ │
│  │         │    │ • 2-SL halt  │◄───│ (alloc adj, direction,  │ │
│  │         │    │ • 20% target │    │  avoid instruments)     │ │
│  │         │    │ • ATR filter │    └─────────────────────────┘ │
│  │         │    └──────┬───────┘                                │
│  │         │           │                                        │
│  │         │    ┌──────▼───────┐    ┌─────────────────────────┐ │
│  │         │◄───│ Execution    │───►│ Option Chain Selector   │ │
│  │         │    │ Engine       │    │ (ATM/ITM, expiry,       │ │
│  │         │    │ (LIMIT order)│    │  liquidity check)       │ │
│  │         │    └──────┬───────┘    └─────────────────────────┘ │
│  │         │           │                                        │
│  │         │    ┌──────▼───────┐                                │
│  │         │◄───│ Exit Monitor │                                │
│  │         │    │ • SL 25%     │                                │
│  │         │    │ • TP 30%     │                                │
│  │         │    │ • Trail 45%  │                                │
│  │         │    │ • IV filter  │                                │
│  └─────────┘    └──────────────┘                                │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Safety Layer │  │ Dashboard    │  │ Logging                │ │
│  │ (29 mechs)   │  │ (Streamlit)  │  │ (JSON Lines)           │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

## 3.2 Data Flow — Tick to Trade

```
1. Kite WebSocket tick arrives (every ~ms)
   Fields: last_price, volume_traded, depth[5], timestamp

2. Tick Handler (_on_tick):
   ├── Append to recent_prices (flash crash detection)
   ├── Update depth store (order flow analysis)
   ├── IF position open:
   │   ├── Check max holding time → force exit if exceeded
   │   ├── Check breakeven SL → move SL to entry if +10%
   │   ├── Check trailing SL → update if activated
   │   └── Check SL/TP hit → execute exit
   └── Feed to candle builders

3. 1-Minute Candle closes → _on_candle_1m:
   ├── Check 2-SL halt (with recovery after 1:30 PM)
   ├── Check 20% daily profit target
   ├── Check ATR volatility regime (skip if bottom 25%)
   ├── Check flash crash
   ├── Run strategy_highwr.evaluate():
   │   ├── Compute indicators (EMA 9/21, RSI 14, ATR 14)
   │   ├── Build VWAP from session start
   │   ├── Check 4 setups (ORB, VWAP, EMA, Volume)
   │   ├── Layer 1: 2+ setups agree? → confluence_setups
   │   ├── Layer 2: 3+ confirmations? → layer2_passed
   │   └── Return: signal, confidence, setups, details
   ├── Buffer signal (don't execute yet)
   ├── After all 3 instruments evaluated:
   │   └── _execute_best_signal():
   │       ├── Sort by confluence count, then confidence
   │       ├── Pick best → _execute_signal()
   │       ├── Apply Claude AI day plan filters
   │       ├── IV-aware entry check
   │       ├── Select option contract (ATM/ITM, correct expiry)
   │       ├── Calculate position size (30% or 20% or 10%)
   │       ├── Build LIMIT order params
   │       ├── Execute via engine
   │       ├── Confirm fill (live mode)
   │       ├── Set recovery flags if applicable
   │       ├── Place exchange SL-M
   │       └── Store trade metadata
   └── Feed to 5m/15m builders

4. 5-Minute Candle closes → update regime detection, trend age
5. 15-Minute Candle closes → update Supertrend (Layer 2A)
```

## 3.3 Threading Model

```
Main Thread:
  - Market hour loop (wait for 9:15, run until 15:30)
  - Session management (authenticate, reset daily counters)
  - EOD processing (close positions, generate reports)

WebSocket Thread (via KiteTicker):
  - Tick reception → candle building → signal detection → execution
  - This is the hot path — must be fast (<500ms)

Timer Threads:
  - TokenHealthMonitor: every 5 minutes
  - HeartbeatWatchdog: continuous
  - LateFillDetector: every 30 seconds

Thread Safety:
  - SafeCapitalManager: RLock on capital reads/writes
  - SafeTradeRegistry: RLock on active_trades dict
  - State persistence: atomic file rename (os.replace)
```

---

# PART 4: STRATEGY SPECIFICATION

## 4.1 Setup 1: Opening Range Breakout (ORB)

**Time window:** 9:15-9:30 AM (building range), 9:30-10:30 AM (breakout window)

**Logic:**
```python
# During 9:15-9:30: track high/low of each candle
orb_high = max(all candle highs during 9:15-9:30)
orb_low = min(all candle lows during 9:15-9:30)

# After 9:30: check breakout
if close > orb_high and volume > avg_volume * 1.2:
    signal = BUY_CE
    confidence = 0.72 if ema9 > ema21 and supertrend_5m == bullish else 0.58

if close < orb_low and volume > avg_volume * 1.2:
    signal = BUY_PE
    confidence = 0.72 if ema9 < ema21 and supertrend_5m == bearish else 0.58

# SKIP if opening gap > 0.5%
if abs(day_open - prev_close) / prev_close > 0.005:
    orb_disabled = True

# Boost confidence if ORB range is tight (< 1.5x ATR)
if orb_range < atr * 1.5:
    confidence += 0.05
```

## 4.2 Setup 2: VWAP Pullback

**Logic:**
```python
# Session VWAP calculated cumulatively from 9:15
vwap = cumsum(typical_price * volume) / cumsum(volume)

# Bullish: price above VWAP, pulls back to touch, bounces
if (close > vwap 
    and min(low[-3:]) <= vwap * 1.001  # touched VWAP
    and close > vwap                    # bounced back
    and ema9 > ema21                    # trend aligned
    and supertrend_5m == bullish):
    signal = BUY_CE
    confidence = 0.55
    if volume > avg_volume * 1.3: confidence += 0.08
    if 40 < rsi < 60: confidence += 0.05

# Bearish: mirror logic
```

## 4.3 Setup 3: EMA Momentum Crossover

**Logic:**
```python
# Check if 9 EMA crossed above 21 EMA in last 3 bars
prev_ema9 = ema9[3 bars ago]
prev_ema21 = ema21[3 bars ago]
curr_ema9 = ema9[now]
curr_ema21 = ema21[now]

if (prev_ema9 < prev_ema21 and curr_ema9 > curr_ema21
    and supertrend_5m == bullish):
    signal = BUY_CE
    confidence = 0.52
    if close > vwap: confidence += 0.06
```

## 4.4 Setup 4: Volume Breakout

**Logic:**
```python
# Check for tight consolidation + volume spike
recent_range = max(high[-10:]) - min(low[-10:])
if recent_range / close < 0.004:  # < 0.4% range
    if volume > avg_volume * 2.5:  # volume spike
        if close > open:  # breakout direction
            signal = BUY_CE
```

## 4.5 Layer 1: Confluence Gate

```python
# Maintain rolling 5-bar setup window per instrument
setup_window = [(bar_index, setup_name, direction, confidence), ...]

# Remove entries older than 5 bars
setup_window = [s for s in setup_window if current_bar - s.bar_index <= 5]

# Count unique setups agreeing on each direction
ce_setups = unique setup names where direction == "CE"
pe_setups = unique setup names where direction == "PE"

# REQUIRE at least 2 different setups agreeing
if len(ce_setups) >= 2:
    direction = "CE", confluent_setups = ce_setups
elif len(pe_setups) >= 2:
    direction = "PE", confluent_setups = pe_setups
else:
    HOLD — no confluence, do not trade
```

## 4.6 Layer 2: Confirmation Indicators

```python
# After Layer 1 confluence detected, check 4 confirmations:

# 2A: 15-minute Supertrend (period=10, multiplier=3.0)
l2a = (supertrend_15m == 1) if bullish else (supertrend_15m == -1)

# 2B: MACD histogram on 5-minute, accelerating
macd_hist_current = macd_5m[now]
macd_hist_prev = macd_5m[1 bar ago]
l2b = (macd_hist > 0 and macd_hist > macd_hist_prev) if bullish
      else (macd_hist < 0 and macd_hist < macd_hist_prev)

# 2C: Relative volume above 1.0 for this time-of-day
l2c = (current_volume > time_of_day_avg_volume)

# 2D: Candle structure — 2 of last 3 candles show directional control
for each of last 3 candles:
    body_pct = abs(close - open) / (high - low)
    close_position = (close - low) / (high - low)
    if close_position > 0.6 and body_pct > 0.4: buyer_control++
l2d = (buyer_control >= 2) if bullish else (seller_control >= 2)

# REQUIRE 3 of 4 to pass
if sum([l2a, l2b, l2c, l2d]) >= 3:
    TRADE APPROVED
else:
    REJECT — insufficient confirmation
```

## 4.7 Signal Buffer (Cross-Instrument Best Pick)

```python
# Per candle cycle, buffer all signals:
signal_buffer = []

# Each instrument's _on_candle_1m appends if signal != HOLD
signal_buffer.append((instrument_token, hw_result, ltp, candle))

# After all 3 instruments evaluated (same minute):
if len(signal_buffer) > 1:
    # Sort by: (confluence_count DESC, confidence DESC)
    signal_buffer.sort(key=lambda s: (len(s[1].confluence_setups), s[1].confidence), reverse=True)
    logger.info(f"[BEST PICK] {len(signal_buffer)} signals → picked {best}")

# Execute only the best one
execute_signal(signal_buffer[0])
signal_buffer.clear()
```

---

# PART 5: EXIT MANAGEMENT SPECIFICATION

## 5.1 Stop-Loss (25% premium drop)

```python
sl_points = (estimated_premium * 0.25) / delta
if direction == "CE":
    sl_price = entry_price - sl_points
else:
    sl_price = entry_price + sl_points

# Place SL-M on exchange immediately after fill
exchange_sl_order = kite.place_order(
    variety="regular",
    order_type="SL-M",
    trigger_price=sl_price,
    transaction_type="SELL",
    ...
)
```

## 5.2 Take-Profit (30% premium rise)

```python
tp_points = (estimated_premium * 0.30) / delta
if direction == "CE":
    tp_price = entry_price + tp_points
else:
    tp_price = entry_price - tp_points
```

## 5.3 Trailing Stop-Loss (45% activate, 9% gap)

```python
trail_activate_points = (estimated_premium * 0.45) / delta
trail_gap_points = (estimated_premium * 0.09) / delta

trailing_active = False
trail_sl = initial_sl
peak = entry_price

# On every tick:
if direction == "CE":
    if current_high > peak:
        peak = current_high
    if not trailing_active and peak >= entry + trail_activate_points:
        trailing_active = True
        trail_sl = peak - trail_gap_points
    if trailing_active:
        new_sl = peak - trail_gap_points
        if new_sl > trail_sl:
            trail_sl = new_sl  # only moves UP, never down

    effective_sl = trail_sl if trailing_active else initial_sl
    if current_low <= effective_sl:
        EXIT at effective_sl (reason: "TRAIL_SL" or "SL")
    if not trailing_active and current_high >= tp_price:
        EXIT at tp_price (reason: "TP")
```

## 5.4 IV-Aware TP Tightening

```python
# Before placing trade, compute IV using Black-Scholes Newton-Raphson
iv = implied_volatility(premium, underlying, strike, time_to_expiry)
iv_detector.update(iv)
elevated, info = iv_detector.is_elevated(iv)  # z-score > 1.3

if elevated:
    # Tighten TP from 30% to 20% for this trade
    tp_pct = 0.20  # instead of 0.30
    # Log warning but don't skip the trade
```

---

# PART 6: OPTION CHAIN SELECTION SPECIFICATION

## 6.1 Strike Selection

```python
# Find ATM strike
interval = {BANKNIFTY: 100, NIFTY: 50, FINNIFTY: 50}
atm_strike = round(underlying_price / interval) * interval

# Prefer slightly ITM (delta 0.6-0.7, reduces theta)
if option_type == "CE":
    selected_strike = atm_strike - interval  # 1 strike ITM
else:
    selected_strike = atm_strike + interval  # 1 strike ITM

# Fallback to ATM if ITM contract not available
```

## 6.2 Expiry Selection

```python
# Expiry weekdays
BANKNIFTY: Wednesday (weekday 2)
NIFTY: Thursday (weekday 3)
FINNIFTY: Tuesday (weekday 1)

# Find this week's expiry
days_ahead = expiry_weekday - today.weekday()
if days_ahead < 0: days_ahead += 7
this_expiry = today + timedelta(days=days_ahead)

# On expiry day OR day before → use NEXT WEEK's expiry
if today == this_expiry or today == this_expiry - 1 day:
    expiry = this_expiry + 7 days
```

## 6.3 Symbol Building

```python
# Format: BANKNIFTY26APR1748000CE
year = expiry.year % 100  # "26"
month = "JANFEBMARAPRMAYJUNJULAUGSEPOCTNOVDEC"[month*3:month*3+3]
day = f"{expiry.day:02d}"
symbol = f"{prefix}{year}{month}{day}{int(strike)}{option_type}"
```

---

# PART 7: CLAUDE AI INTEGRATION SPECIFICATION

## 7.1 Pre-Market Analyzer

**When:** 8:30 AM daily (before market)
**Input:** Last 5 days of trade summaries from logs/risk.json
**Output:** data/day_plan.json

**Claude system prompt:**
"You are a quantitative trading analyst for Indian equity markets. Respond with ONLY a JSON object."

**Expected JSON response:**
```json
{
  "market_outlook": "BULLISH|BEARISH|SIDEWAYS|VOLATILE",
  "confidence": 0.0-1.0,
  "allocation_adjustment": 0.5-1.5,
  "preferred_direction": "CE|PE|BOTH",
  "avoid_instruments": [],
  "risk_level": "LOW|NORMAL|HIGH",
  "max_trades_override": null or 1-5,
  "special_notes": "string",
  "events_today": []
}
```

**DayPlanApplier rules:**
- Missing file → trade defaults, log warning
- Invalid JSON → trade defaults, log warning
- Wrong date → IGNORE, trade defaults, log warning ("stale plan")
- Plan > 12 hours old → use with caution warning
- HIGH risk → allocation 0.7x, max 2 trades
- Preferred direction CE → skip PE signals
- avoid_instruments → skip that instrument entirely

## 7.2 Post-Market Reviewer

**When:** 4:00 PM daily (after market close)
**Input:** Today's trades from logs/risk.json
**Output:** logs/reviews/review_YYYY-MM-DD.json

## 7.3 Weekly Adapter

**When:** Sunday evening
**Input:** Last 5 daily reviews
**Output:** data/weekly_plan.json with parameter change suggestions

---

# PART 8: SAFETY MECHANISMS (29 total)

Build these in safety_layer.py:

1. RateLimiter — token bucket, 10 req/sec for Kite API
2. OrderConfirmer — wait_for_fill() with timeout, check order status
3. ExchangeStopLoss — place SL-M on exchange after every fill
4. TokenHealthMonitor — check token validity every 5 min, halt if expired
5. StateManager — persist trading state to JSON, atomic write
6. HeartbeatWatchdog — write timestamp file, external cron monitors
7. MarginTracker — refresh available margin from Kite on session start
8. TradingCalendar — NSE holidays, skip trading
9. ExpiryDayFilter — detect expiry day, flag for next-week expiry
10. InstrumentValidator — verify instrument tokens before subscribing
11. ModelHealthTracker — track ML prediction accuracy, warn if degrading
12. FlashCrashProtector — halt if price moves >3% in recent window
13. SafeCapitalManager — RLock-protected capital read/write
14. SafeTradeRegistry — RLock-protected active trades dict
15. LateFillDetector — check for fills on cancelled orders
16. GracefulShutdown — signal handler for SIGTERM/SIGINT, close positions
17. PIDLockfile — prevent duplicate instances
18. AdaptiveThreshold — adjust signal thresholds based on regime
19. TrendAgeTracker — skip signals if trend is exhausted
20. NaNSanitizer — clean feature vectors before ML
21. CrashRecovery — reload state on startup, reconcile with broker
22. PositionReconciler — compare local state with broker positions
23. OrderFlowImputer — fill missing order flow data for backtest
24. ReEntryCooldown — prevent rapid re-entry after exit
25. AdaptiveTradeLimit — adjust max trades based on daily P&L
26. TrailingStopLoss — premium-based trailing SL logic
27. PartialProfitTaker — [DISABLED due to bugs #1/#2/#4, trailing SL covers this]
28. ExchangeSLUpdater — update exchange SL-M when trailing SL moves
29. DepthAnalyzer — weighted imbalance, spoofing detection

---

# PART 9: DASHBOARD SPECIFICATION

## 9.1 Layout

```
┌─────────────────────────────────────────────────────────────┐
│ 💰 AutoScalp Trading System          [● RUNNING] badge     │
│ Wednesday, April 16, 2026 — Market Open                     │
├─────────────────────────────────────────────────────────────┤
│ [P&L]  [Trades]  [WinRate]  [W/L]  [Capital]  [SL Hits]   │
│ +7,830    4       75%       3/1    57,830      1/2          │
├──────────────┬──────────────────────────────────────────────┤
│ 🎮 Controls  │ 🛡️ Active Rules (8 rules with status)       │
│ [▶ START]    │ ● 2-SL Halt: 1/2                            │
│ Paper toggle │ ● 20% Lock: ₹+7,830                         │
│ [⏹ STOP]    │ ● Allocation: 30%                            │
│              │ ● Confluence: 2-of-4                          │
│ ⚡ Actions    │ ● Layer 2: 3-of-4                           │
│ [Simulator]  │ ● Exchange SL-M: Active                      │
│ [Setup Chk]  │ ● Trailing SL: 45%/9%                       │
│              │ ● Exit Levels: SL:25% TP:30%                 │
├──────────────┴──────────────────────────────────────────────┤
│ 📋 Today's Trades (visual cards, green=win, red=loss)       │
│ ✅ ₹+4,200 — BANKNIFTY CE · ORB+VWAP · TP · 9:42          │
│ ✅ ₹+6,780 — NIFTY CE · EMA+VWAP · Trail SL · 10:15       │
│ ❌ ₹-2,700 — BANKNIFTY PE · ORB+EMA · SL · 11:03          │
├─────────────────────────┬───────────────────────────────────┤
│ 📈 P&L Curve            │ 📊 Win/Loss Bars                  │
│ (cumulative line chart) │ (green/red bar per trade)          │
├─────────────────────────┴───────────────────────────────────┤
│ 📅 Performance History (expandable)                          │
│ ⚙️ System Configuration (expandable)                        │
│ ❓ How does this work? (expandable, plain English)          │
└─────────────────────────────────────────────────────────────┘
```

## 9.2 Data Sources

Dashboard reads from JSON Lines files in logs/ directory:
- logs/trades.json — trade entries
- logs/risk.json — exits with P&L, halt events
- logs/market.json — market state snapshots

Dashboard auto-refreshes every 3 seconds. Dashboard does NOT interfere with trading — it's read-only.

---

# PART 10: KNOWN BUGS (ALREADY FIXED)

These bugs were found and fixed. Document them so they don't get reintroduced:

1. **Partial qty floor division** — `get_partial_qty()` used `//` which returned 0 for 1-lot positions. Fixed with `ceil()`. Feature currently disabled.
2. **P&L mismatch on partial exits** — `record_exit()` used original quantity instead of remaining. Partial profit feature disabled to eliminate.
3. **Recovery flags before fill confirmation** — flags set before order confirmed, causing state corruption on failed orders. Reordered: fill confirmation first, then flags.
4. **FinNIFTY lot size wrong** — was 40, corrected to 60 (NSE Jan 2026 revision).
5. **Missing `get_last_closed_trade()`** — method called but not defined. Added to both execution.py and paper_trading.py.
6. **Bare `except:` clauses** — caught KeyboardInterrupt/SystemExit, preventing clean shutdown. Changed to `except Exception:`.
7. **Missing imports** — `timedelta`, `Dict` were used but not imported in some versions. Fixed.
8. **`CONFIG.trading_symbol` (singular)** — doesn't exist, should be `CONFIG.trading_symbols` (dict). Fixed.

---

# PART 11: TESTING CHECKLIST

After building, verify:

- [ ] `python3 setup_check.py` passes all checks
- [ ] All 27 files compile (`python3 -c "import ast; ast.parse(open(f).read())"`)
- [ ] CONFIG loads without errors
- [ ] Dashboard launches (`streamlit run dashboard.py`)
- [ ] `python3 claude_advisor.py pre-market` generates day_plan.json (needs API key)
- [ ] `python3 main_v3.py` starts in paper mode without errors
- [ ] WebSocket connects to Kite (needs API subscription)
- [ ] Candle builders produce 1m/5m/15m candles
- [ ] Strategy evaluates and produces signals
- [ ] Signal buffer picks best across instruments
- [ ] Option chain selector finds correct contract
- [ ] Exit monitoring checks SL/TP/trail on every tick
- [ ] 2-SL halt stops trading after 2 SL hits
- [ ] 20% profit lock stops trading when hit
- [ ] Recovery trade fires after 1:30 PM with 10% allocation
- [ ] Exchange SL-M placed on every trade (live mode)
- [ ] Crash recovery restores state after restart
- [ ] PID lockfile prevents duplicate instances
- [ ] Claude AI day plan applies adjustments at session start
- [ ] Stale day plan is rejected with warning (not silently used)

---

# PART 12: PERFORMANCE BENCHMARKS

The system was validated on real data. These are the numbers to match:

| Metric | Target |
|--------|--------|
| Win rate (backtest) | 56.9% on 1,402 trades |
| Win rate (live estimate) | 60.9% |
| Trades per month | ~50 |
| Profit factor | >1.3 |
| 2-SL halt days per month | ~7 |
| Monthly profit probability | ~80% |
| Conservative monthly return | 10-15% |

---

**END OF SPECIFICATION**

**To build:** Give this entire document to Claude Code, Cursor, or any AI coding assistant with the command: "Build the complete AutoScalp trading system following this specification exactly."
