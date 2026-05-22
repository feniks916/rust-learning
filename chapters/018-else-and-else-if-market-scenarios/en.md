# Day 18: else and else if — Three Market Scenarios

## Trading Analogy

Every experienced trader always thinks in scenarios: what do I do IF the market goes up? What IF it falls? What IF it moves sideways? A good trading plan is not "buy and pray" — it is a clear set of branches: **bull market** — buy on pullbacks, **bear market** — short on bounces, **sideways** — trade from levels or sit on your hands. In Rust, `else` and `else if` are exactly that trading plan: we precisely describe what to do in every scenario, and no situation goes unanswered.

## else: When the Condition Is Not Met

`else` means "otherwise". If the `if` block did not fire, we execute the `else` block. This is a binary decision: either one thing or the other.

```rust
fn main() {
    let btc_price = 48000.0;
    let target_price = 50000.0;

    if btc_price >= target_price {
        println!("Price reached target — SELL!");
        println!("Taking profit at ${}", target_price);
    } else {
        println!("Price below target — HOLD position");
        println!("Waiting for ${:.0}", target_price);
    }
}
```

**Important:** `else` has no condition — it runs whenever the `if` condition was false. This guarantees we always have a response for every market situation.

```rust
fn main() {
    let pnl = -350.0; // current profit/loss in USD

    let result = if pnl >= 0.0 {
        "Profitable position"
    } else {
        "Losing position"
    };

    println!("Status: {} (${:.2})", result, pnl);
}
```

When to use `else`:
- Exactly two possible outcomes
- Binary states: open/closed, long/short, profit/loss
- A default value when the condition is not met

## else if: Multiple RSI Scenarios

The market is rarely just "good" or "bad" — it is nuanced. `else if` lets you describe as many scenarios as you need.

```rust
fn classify_rsi(rsi: f64) -> &'static str {
    if rsi < 20.0 {
        "Extreme oversold — strong BUY signal"
    } else if rsi < 30.0 {
        "Oversold — moderate BUY signal"
    } else if rsi < 45.0 {
        "Weakness — cautious buy"
    } else if rsi < 55.0 {
        "Neutral zone — HOLD"
    } else if rsi < 70.0 {
        "Strength — cautious sell"
    } else if rsi < 80.0 {
        "Overbought — moderate SELL signal"
    } else {
        "Extreme overbought — strong SELL signal"
    }
}

fn main() {
    println!("RSI 15: {}", classify_rsi(15.0));
    println!("RSI 28: {}", classify_rsi(28.0));
    println!("RSI 50: {}", classify_rsi(50.0));
    println!("RSI 72: {}", classify_rsi(72.0));
    println!("RSI 85: {}", classify_rsi(85.0));
}
```

Another example — classifying daily price change:

```rust
fn classify_price_change(change_pct: f64) -> &'static str {
    if change_pct > 10.0 {
        "Pump — suspicious spike, be careful!"
    } else if change_pct > 5.0 {
        "Strong bull day — long trades are good"
    } else if change_pct > 2.0 {
        "Moderate rally — neutral-bullish"
    } else if change_pct > -2.0 {
        "Sideways — trade from key levels"
    } else if change_pct > -5.0 {
        "Moderate decline — neutral-bearish"
    } else if change_pct > -10.0 {
        "Strong bear day — short trades are good"
    } else {
        "Dump — suspicious drop, be careful!"
    }
}

fn main() {
    println!("{}", classify_price_change(12.5));
    println!("{}", classify_price_change(3.2));
    println!("{}", classify_price_change(-1.1));
    println!("{}", classify_price_change(-7.8));
}
```

## Check Priority

This is CRITICALLY important to understand: checks run top to bottom, and **only the first matching block executes**. The rest are skipped.

```rust
fn main() {
    let rsi = 25.0;

    // Checks run in order:
    if rsi < 30.0 {
        println!("Oversold");   // <- THIS runs (rsi=25 < 30)
    } else if rsi < 20.0 {
        // We NEVER reach here when rsi=25,
        // because the block above already matched!
        println!("Extreme oversold");
    } else {
        println!("Neutral");
    }
}
```

**Correct order — from specific to general (smallest threshold first):**

```rust
fn check_rsi_correct(rsi: f64) {
    // Correct: most restrictive condition first
    if rsi < 20.0 {
        println!("Extreme oversold");    // rsi < 20
    } else if rsi < 30.0 {
        println!("Oversold");            // 20 <= rsi < 30
    } else if rsi < 70.0 {
        println!("Neutral");             // 30 <= rsi < 70
    } else if rsi < 80.0 {
        println!("Overbought");          // 70 <= rsi < 80
    } else {
        println!("Extreme overbought");  // rsi >= 80
    }
}

fn main() {
    check_rsi_correct(15.0); // Extreme oversold
    check_rsi_correct(25.0); // Oversold
    check_rsi_correct(55.0); // Neutral
    check_rsi_correct(75.0); // Overbought
    check_rsi_correct(90.0); // Extreme overbought
}
```

## Nested if/else: Analysing by Price AND Volume

Sometimes you need to check one parameter first and only then check another. That's nested conditions.

```rust
fn analyze_market(price: f64, sma: f64, volume: f64, avg_volume: f64) -> &'static str {
    if price > sma {
        // Price above SMA — bullish zone
        if volume > avg_volume * 1.5 {
            "Strong bull trend (high volume confirms)"
        } else if volume > avg_volume {
            "Moderate bull trend (volume normal)"
        } else {
            "Weak bullish signal (low volume — be cautious)"
        }
    } else {
        // Price below SMA — bearish zone
        if volume > avg_volume * 1.5 {
            "Strong bear trend (volume confirms selling)"
        } else if volume > avg_volume {
            "Moderate bear trend"
        } else {
            "Weak bearish signal (low volume — possible bounce)"
        }
    }
}

fn main() {
    println!("{}", analyze_market(51000.0, 49000.0, 2500.0, 1500.0));
    println!("{}", analyze_market(51000.0, 49000.0, 800.0, 1500.0));
    println!("{}", analyze_market(47000.0, 49000.0, 3000.0, 1500.0));
    println!("{}", analyze_market(47000.0, 49000.0, 900.0, 1500.0));
}
```

Don't over-nest — more than 2–3 levels makes code unreadable. In those cases, extract logic into separate functions.

## if/else as an Expression: Returning a Value

In Rust, `if/else` is an **expression that returns a value**. You can assign the result of an `if` directly to a variable.

```rust
fn main() {
    let price_change = -3.5; // percent change for the day

    // if/else as an expression
    let signal = if price_change > 5.0 {
        "STRONG_BUY"
    } else if price_change > 2.0 {
        "BUY"
    } else if price_change > -2.0 {
        "HOLD"
    } else if price_change > -5.0 {
        "SELL"
    } else {
        "STRONG_SELL"
    };

    println!("Price change: {:.1}%", price_change);
    println!("Signal: {}", signal);
}
```

**Important:** all branches must return the same type!

```rust
fn main() {
    let balance = 15000.0;
    let trade_value = 5000.0;

    // Calculate commission: VIP if volume > $10k
    let commission_rate = if trade_value > 10000.0 {
        0.05_f64  // VIP: 0.05%
    } else {
        0.10_f64  // Standard: 0.10%
    };

    let commission = trade_value * commission_rate / 100.0;
    let after_commission = trade_value - commission;

    println!("Trade volume: ${:.2}", trade_value);
    println!("Commission: {:.2}% = ${:.2}", commission_rate, commission);
    println!("Net amount: ${:.2}", after_commission);

    // Is the balance sufficient?
    let can_trade = if balance >= trade_value {
        "Yes, balance is sufficient"
    } else {
        "No, insufficient funds"
    };
    println!("Can trade: {}", can_trade);
}
```

## Market Classification: Function with 5 RSI Zones

Let's write a professional function that analyses the market using multiple factors at once:

```rust
#[derive(Debug)]
struct MarketAnalysis {
    rsi_zone: &'static str,
    trend: &'static str,
    action: &'static str,
    confidence: &'static str,
}

fn analyze_rsi(rsi: f64, price: f64, sma: f64) -> MarketAnalysis {
    let trend = if price > sma { "BULL" } else { "BEAR" };

    let (rsi_zone, action, confidence) = if rsi < 20.0 {
        ("Extreme oversold", "STRONG BUY", "High")
    } else if rsi < 30.0 {
        ("Oversold", "BUY", "Medium")
    } else if rsi < 45.0 {
        ("Market weakness", "Cautious buy", "Low")
    } else if rsi < 55.0 {
        ("Neutral zone", "HOLD", "No signal")
    } else if rsi < 70.0 {
        ("Market strength", "Cautious sell", "Low")
    } else if rsi < 80.0 {
        ("Overbought", "SELL", "Medium")
    } else {
        ("Extreme overbought", "STRONG SELL", "High")
    };

    MarketAnalysis { rsi_zone, trend, action, confidence }
}

fn main() {
    let analysis = analyze_rsi(25.0, 42000.0, 44000.0);
    println!("RSI zone: {}", analysis.rsi_zone);
    println!("Trend: {}", analysis.trend);
    println!("Action: {}", analysis.action);
    println!("Confidence: {}", analysis.confidence);
}
```

## Practical Example: Trading Signal System

```rust
/// Determines a trading signal from a combination of indicators
fn get_trading_signal(
    price: f64,
    sma_20: f64,
    sma_50: f64,
    rsi: f64,
    volume: f64,
    avg_volume: f64,
) -> &'static str {
    // Determine trend direction using SMAs
    let trend = if price > sma_20 && sma_20 > sma_50 {
        "strong_bull"
    } else if price > sma_20 {
        "weak_bull"
    } else if price < sma_20 && sma_20 < sma_50 {
        "strong_bear"
    } else {
        "weak_bear"
    };

    // Does volume confirm the move?
    let volume_ok = volume > avg_volume;

    // Final decision based on trend + RSI + volume
    if trend == "strong_bull" {
        if rsi < 70.0 && volume_ok {
            "STRONG BUY — trend and volume confirm"
        } else if rsi >= 70.0 {
            "WAIT — bull trend but RSI overbought"
        } else {
            "BUY — bull trend (volume weak)"
        }
    } else if trend == "weak_bull" {
        if rsi < 50.0 && volume_ok {
            "BUY — weak bull + RSI not overheated"
        } else {
            "HOLD — uncertainty"
        }
    } else if trend == "strong_bear" {
        if rsi > 30.0 && volume_ok {
            "STRONG SELL — bear trend confirmed"
        } else if rsi <= 30.0 {
            "WAIT — bear trend but RSI oversold"
        } else {
            "SELL — bear trend (volume weak)"
        }
    } else {
        "HOLD — no clear direction"
    }
}

fn main() {
    println!("=== Trading Signal System ===\n");

    // Scenario 1: Strong bull market
    let signal1 = get_trading_signal(
        52000.0, // price
        50000.0, // sma_20
        48000.0, // sma_50
        55.0,    // rsi
        2000.0,  // volume
        1500.0,  // avg_volume
    );
    println!("Scenario 1 (bull market): {}", signal1);

    // Scenario 2: Overbought
    let signal2 = get_trading_signal(
        55000.0,
        50000.0,
        48000.0,
        78.0,    // RSI overbought
        2200.0,
        1500.0,
    );
    println!("Scenario 2 (overbought): {}", signal2);

    // Scenario 3: Bear market with confirmation
    let signal3 = get_trading_signal(
        44000.0,
        46000.0,
        48000.0,
        42.0,
        2500.0,
        1500.0,
    );
    println!("Scenario 3 (bear market): {}", signal3);

    // Scenario 4: Oversold in a bear trend
    let signal4 = get_trading_signal(
        43000.0,
        46000.0,
        48000.0,
        22.0,    // RSI oversold — potential bounce
        1800.0,
        1500.0,
    );
    println!("Scenario 4 (oversold): {}", signal4);

    println!("\n=== Analysis complete ===");
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `if { } else { }` | Two options: yes/no | Profit or loss |
| `else if condition { }` | 3+ scenarios | RSI zones: 5–7 levels |
| Check order | Top to bottom, first match wins | `< 20` before `< 30` |
| Nested `if/else` | When conditions depend on each other | Price AND volume |
| `let x = if { } else { }` | if as expression for assignment | `let signal = if ... { "BUY" } else { "SELL" }` |
| Types must match | All branches of an if-expression | Can't return `&str` in one branch and `i32` in another |

## Homework

1. Write a function `classify_volatility(atr_pct: f64) -> &'static str` with 5 volatility levels:
   - Below 1% — "Very low"
   - 1–2% — "Low"
   - 2–4% — "Medium"
   - 4–7% — "High"
   - Above 7% — "Extreme"

2. Write a function `get_position_action(pnl_pct: f64) -> &'static str`:
   - PnL > 10% — take profit (TAKE PROFIT)
   - PnL 5–10% — hold and trail stop to break-even (TRAIL STOP)
   - PnL -2% to 5% — hold (HOLD)
   - PnL -5% to -2% — watch closely (WATCH)
   - PnL < -5% — stop loss (STOP LOSS)

3. Build a nested-condition system `analyze_order(price: f64, balance: f64, market_open: bool) -> &'static str`:
   - First check `market_open`
   - Inside check balance
   - Inside check price

4. Use `if/else` as an expression to calculate a commission:
   - Volume < $1000: commission 0.2%
   - Volume $1000–$10000: 0.1%
   - Volume > $10000: 0.05%
   - Store the result in `let commission_rate = if ...`

## Navigation

[← Previous day](../017-if-conditions-buy-or-sell/en.md) | [Next day →](../019-loop-endless-price-monitoring/en.md)
