# Day 16: Comments — Documenting Trading Logic

## Trading Analogy

Imagine you hired a new trader for your team, gave them access to the bot, and inside there are thousands of lines of code without a single explanation. What does the variable `x` do? Why are we multiplying by `0.00075`? What's the point of that `if` with the magic number `1337`? In a month even you won't remember what you meant. In professional trading every order is logged with a reason — why it was opened, which strategy, what the risk is. Comments in code are exactly the same kind of trade journal, just for the programmer. A well-documented trading bot is an asset; a poorly-documented one is a ticking time bomb.

## Why Do We Need Comments?

Look at this code without comments. Try to figure out what it does:

```rust
fn main() {
    let x = 42000.0;
    let y = 41500.0;
    let z = 0.5;
    let r = 35.0;
    let v = 1800.0;
    let mv = 1500.0;

    if x < y && r < 40.0 && v > mv {
        let q = 10000.0 * 0.02 / 0.05;
        println!("GO: {}", q);
    }
}
```

What are `x`, `y`, `z`, `r`, `v`, `mv`? What does `GO` mean? Why `0.02` and `0.05`? Now look at the same code with comments:

```rust
fn main() {
    let btc_price = 42000.0;       // current BTC/USDT price
    let sma_20 = 41500.0;          // 20-period simple moving average
    let quantity = 0.5;            // BTC quantity (unused here)
    let rsi = 35.0;                // Relative Strength Index (RSI)
    let volume = 1800.0;           // volume over the last hour
    let min_volume = 1500.0;       // minimum volume to confirm a signal

    // Buy signal: price below SMA, RSI oversold, volume sufficient
    if btc_price < sma_20 && rsi < 40.0 && volume > min_volume {
        // Position size = deposit * risk_per_trade / stop_loss_percent
        let position_size = 10000.0 * 0.02 / 0.05; // 2% risk, 5% stop
        println!("BUY signal! Position size: ${}", position_size);
    }
}
```

Huge difference! The code now reads like a book.

## Single-Line Comments //

The most common type of comment. Put `//` and everything until the end of the line is a comment.

**Comments on variables:**

```rust
fn main() {
    let ticker = "BTCUSDT";        // trading pair on Binance
    let entry_price = 42000.0;     // position entry price (USD)
    let stop_loss = 40000.0;       // stop-loss level (-4.76%)
    let take_profit = 46000.0;     // take-profit level (+9.52%)
    let quantity = 0.25;           // position size in BTC
    let leverage = 3;              // margin leverage (x3)

    let risk_reward = (take_profit - entry_price) / (entry_price - stop_loss);
    println!("Ticker: {}", ticker);
    println!("Risk/Reward ratio: {:.2}", risk_reward); // should be >= 2.0
}
```

**Comments on formulas — explain WHY this formula:**

```rust
fn main() {
    let deposit = 10000.0;
    let risk_percent = 0.01;       // risk 1% of deposit per trade
    let stop_distance = 0.05;      // stop 5% from entry price

    // Simplified Kelly Criterion: size = risk_in_dollars / stop_distance
    // This gives us the number of units with correct capital management
    let risk_amount = deposit * risk_percent;   // $100 — max loss
    let position_size = risk_amount / stop_distance; // $100 / 5% = $2000

    println!("Max risk: ${:.2}", risk_amount);
    println!("Position size: ${:.2}", position_size);
}
```

**Comments on conditions:**

```rust
fn main() {
    let rsi = 28.0;
    let volume = 2200.0;
    let avg_volume = 1500.0;
    let spread = 0.15;             // spread in percent

    // Check conditions for entering a position
    if rsi < 30.0 {
        // RSI below 30 = classic oversold condition per Wilder
        println!("RSI signal: oversold zone");
    }

    if volume > avg_volume * 1.5 {
        // Volume 1.5x above average confirms the momentum
        println!("Volume confirmed: abnormally high");
    }

    if spread < 0.2 {
        // Spread > 0.2% makes trades unprofitable for our strategy
        println!("Spread acceptable, can trade");
    }
}
```

**Comment at end of line or on its own line:**

```rust
fn main() {
    // Initialise Bollinger Bands strategy parameters
    let period = 20;                   // period for moving average calculation
    let std_multiplier = 2.0;          // number of standard deviations
    let price = 48500.0;
    let upper_band = 50000.0;          // upper Bollinger Band
    let lower_band = 47000.0;          // lower Bollinger Band
    let middle_band = (upper_band + lower_band) / 2.0; // middle line

    println!("Period: {}, Multiplier: {}", period, std_multiplier);
    println!("Middle band: {:.2}", middle_band);

    // Price near the lower band — possible buy signal
    if price < lower_band * 1.01 {
        println!("Price at lower Bollinger Band — watching for reversal");
    }
}
```

## Multi-Line Comments /* */

Use for large text blocks — for example, documenting a strategy at the top of a file.

**Documenting a trading strategy:**

```rust
/*
    STRATEGY: RSI Reversal with Volume Confirmation
    ================================================
    Author: AlgoTrader
    Version: 2.1
    Created: 2024-01-15

    DESCRIPTION:
    The strategy looks for reversal points using the RSI indicator.
    Buy when oversold (RSI < 30), sell when overbought (RSI > 70).
    Volume is used as a filter for false signals.

    PARAMETERS:
    - Timeframe: 1H (hourly)
    - Instruments: BTC/USDT, ETH/USDT
    - Deposit: from $1000
    - Max risk per trade: 2%

    BACKTEST STATS (2020-2024):
    - Win rate: 58%
    - Profit factor: 1.85
    - Max drawdown: 12%
    - Sharpe ratio: 1.42
*/

fn rsi_strategy(rsi: f64, volume: f64, avg_volume: f64) -> &'static str {
    let volume_confirmed = volume > avg_volume; // volume above average

    if rsi < 30.0 && volume_confirmed {
        "BUY"   // oversold confirmed by volume
    } else if rsi > 70.0 && volume_confirmed {
        "SELL"  // overbought confirmed by volume
    } else {
        "HOLD"  // no clear signal
    }
}
```

**Temporarily disabling code (debugging):**

```rust
fn main() {
    let price = 45000.0;

    /* TEMPORARILY DISABLED — testing without extra filters
    let macd_signal = 150.0;
    let macd_line = 200.0;
    if macd_line < macd_signal {
        println!("MACD bearish — skipping signal");
        return;
    }
    */

    // Simplified logic for testing
    println!("BTC price: ${}", price);
    println!("Ready to trade (MACD filter disabled)");
}
```

## Documentation Comments ///

These are special Rust comments — they generate HTML documentation via `cargo doc`. Use them for all public functions.

```rust
/// Calculates position size based on risk management rules.
///
/// Uses the formula: `position = (deposit * risk_pct) / stop_pct`
/// This guarantees that when the stop-loss triggers we lose
/// no more than the specified percentage of the deposit.
///
/// # Arguments
///
/// * `deposit` - trading account balance in USD (must be > 0)
/// * `risk_pct` - fraction of deposit at risk (0.01 = 1%, typical 0.01-0.03)
/// * `stop_pct` - stop-loss distance from entry price (0.05 = 5%)
///
/// # Returns
///
/// Position size in USD. Returns 0.0 if input parameters are zero.
///
/// # Panics
///
/// Panics if `stop_pct` is zero (division by zero).
///
/// # Example
///
/// ```
/// let size = calculate_position_size(10000.0, 0.02, 0.05);
/// assert_eq!(size, 4000.0); // risking $200 with a 5% stop
/// ```
fn calculate_position_size(deposit: f64, risk_pct: f64, stop_pct: f64) -> f64 {
    if deposit <= 0.0 || risk_pct <= 0.0 {
        return 0.0;
    }
    (deposit * risk_pct) / stop_pct
}

/// Validates whether an order price is acceptable for placement.
///
/// # Arguments
///
/// * `price` - order price in USD
/// * `min_price` - minimum allowed price (usually 0.01)
/// * `max_price` - maximum allowed price for this pair
///
/// # Returns
///
/// `true` if price is within the allowed range, otherwise `false`
///
/// # Example
///
/// ```
/// assert!(is_valid_price(42000.0, 0.01, 100000.0));
/// assert!(!is_valid_price(-100.0, 0.01, 100000.0));
/// ```
fn is_valid_price(price: f64, min_price: f64, max_price: f64) -> bool {
    price >= min_price && price <= max_price
}

fn main() {
    let pos = calculate_position_size(10000.0, 0.02, 0.05);
    println!("Position size: ${:.2}", pos);

    let valid = is_valid_price(42000.0, 0.01, 200000.0);
    println!("Price valid: {}", valid);
}
```

## Good vs Bad Comments

Not all comments are equally useful. A bad comment is noise that only gets in the way.

| Bad Comment | Good Comment |
|-------------|-------------|
| `let price = 100.0; // assign price to 100` | `let price = 100.0; // minimum order price on Binance` |
| `i = i + 1; // increment i by 1` | `tick_count += 1; // count ticks for VWAP calculation` |
| `// TODO` | `// TODO: add spread check before entry (issue #42)` |
| `// multiply by 0.001` | `// 0.001 = standard Binance Maker fee` |
| `// profit function` | `// Using abs() because short trades give negative difference` |

Rule: a good comment answers **WHY**, not **WHAT**. The code already shows WHAT is happening — explain the intent and reason.

## How NOT to Comment

**Anti-pattern 1: Comment repeats the code**

```rust
// BAD: comment is useless — code is self-evident
let total = price * quantity; // multiply price by quantity

// GOOD: explain the business logic
let total = price * quantity; // position value before fees
```

**Anti-pattern 2: Commented-out dead code**

```rust
// BAD: clutters the file, nobody knows if it can be deleted
// let old_fee = 0.002;
// let fee = amount * old_fee;
// println!("old fee: {}", fee);
let fee = amount * 0.001; // current fee since 2023

// GOOD: delete dead code, git history preserves it
let fee = amount * 0.001; // current Binance Spot fee (since 2023)
```

**Anti-pattern 3: Stale comment (worse than no comment!)**

```rust
// DANGEROUS: comment is lying! Will cause a bug when reading the code
// Stop-loss = 3% from entry price
let stop_distance = entry_price * 0.05; // <- actually 5%!

// GOOD: update comment when changing code
let stop_distance = entry_price * 0.05; // stop = 5% from entry price
```

**Anti-pattern 4: Over-commenting the obvious**

```rust
// BAD: obvious without comments
fn main() {
    // start of main function
    let x = 5; // variable x equals 5
    // end of main function
}

// GOOD: comment only non-trivial things
fn main() {
    let rsi_period = 14; // standard RSI period per Wilder (1978)
}
```

## Practical Example: Documenting a Trading Bot

```rust
/*
    Module: BTC/USDT Trading Signal
    Strategy: RSI + Volume Breakout
    Version: 1.0.0
*/

/// Generates a trading signal based on technical indicators.
///
/// # Arguments
/// * `price` - current asset price
/// * `sma` - moving average value
/// * `rsi` - RSI value (0-100)
/// * `volume` - current trading volume
/// * `avg_volume` - average volume over N periods
///
/// # Returns
/// String signal: "BUY", "SELL", or "HOLD"
fn generate_signal(
    price: f64,
    sma: f64,
    rsi: f64,
    volume: f64,
    avg_volume: f64,
) -> &'static str {
    // Volume must confirm the signal (filter for false breakouts)
    let volume_ok = volume > avg_volume * 1.2; // volume 20% above average

    // Buy conditions: price above SMA, RSI not overbought, volume confirmed
    let buy_signal = price > sma && rsi < 65.0 && volume_ok;

    // Sell conditions: price below SMA, RSI not oversold, volume confirmed
    let sell_signal = price < sma && rsi > 35.0 && volume_ok;

    if buy_signal {
        "BUY"
    } else if sell_signal {
        "SELL"
    } else {
        "HOLD" // no clear signal — better not to trade
    }
}

/// Calculates stop-loss and take-profit levels.
///
/// # Arguments
/// * `entry` - position entry price
/// * `is_long` - true for long, false for short
///
/// # Returns
/// Tuple (stop_loss, take_profit)
fn calculate_levels(entry: f64, is_long: bool) -> (f64, f64) {
    let stop_pct = 0.04;    // stop 4% from entry
    let profit_pct = 0.08;  // take 8% from entry (RR = 2.0)

    if is_long {
        // For long: stop below entry, take above
        let stop = entry * (1.0 - stop_pct);
        let take = entry * (1.0 + profit_pct);
        (stop, take)
    } else {
        // For short: stop above entry, take below
        let stop = entry * (1.0 + stop_pct);
        let take = entry * (1.0 - profit_pct);
        (stop, take)
    }
}

fn main() {
    // --- Market data (in a real bot we get this from the exchange) ---
    let btc_price = 43500.0;     // BTC/USDT price
    let sma_20 = 42800.0;        // 20-hour SMA
    let rsi_14 = 52.0;           // RSI(14) hourly
    let hour_volume = 1850.0;    // volume for the last hour (BTC)
    let avg_volume_24h = 1500.0; // average hourly volume over 24h

    println!("=== BTC/USDT Analysis ===");
    println!("Price: ${:.2}", btc_price);
    println!("SMA(20): ${:.2}", sma_20);
    println!("RSI(14): {:.1}", rsi_14);
    println!();

    // Generate the signal
    let signal = generate_signal(btc_price, sma_20, rsi_14, hour_volume, avg_volume_24h);
    println!("Signal: {}", signal);

    // If there is a buy signal — calculate levels
    if signal == "BUY" {
        let (stop, take) = calculate_levels(btc_price, true);
        println!("Stop-loss: ${:.2}", stop);
        println!("Take-profit: ${:.2}", take);
        println!("Risk/Reward: {:.1}", (take - btc_price) / (btc_price - stop));
    }
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `// text` | Explaining a line or code block | `let fee = 0.001; // Maker fee` |
| `/* text */` | Large block, strategy docs, temporarily disabling code | `/* RSI Reversal Strategy v2.1 */` |
| `/// text` | Documenting public functions and types | `/// Calculates PnL for a trade` |
| `# Arguments` | Arguments section in `///` | Description of each parameter |
| `# Returns` | Return value section in `///` | What the function returns |
| `# Example` | Example section in `///` | Working code with `assert!` |
| `# Panics` | Panic conditions in `///` | When the function can crash |
| `cargo doc` | Build HTML documentation | `cargo doc --open` |

## Homework

1. Take any code from previous lessons and add comments:
   - Add a `//` comment to each variable explaining its business meaning
   - Add a `/* */` block comment at the top of the file describing the "strategy"
   - Find a non-trivial place (formula, condition) and explain WHY it is written that way

2. Write a function `calculate_pnl(entry: f64, exit: f64, qty: f64, is_long: bool) -> f64` with full `///` documentation:
   - A `# Arguments` section describing all parameters
   - A `# Returns` section explaining the result
   - A `# Example` section with a working example
   - A `# Panics` section (if applicable)

3. Find 3 bad comments in the following code and rewrite them:
   ```rust
   let a = 100.0; // a equals 100
   let b = a * 2.0; // multiply a by 2
   // function
   fn f(x: f64) -> f64 { x * 0.00075 }
   ```

4. Create documentation for a mini-system of 2 functions:
   - `fn is_market_open(hour: u32) -> bool` (exchange open 9 to 18)
   - `fn get_commission(volume: f64) -> f64` (0.1% up to $10k, 0.05% above)
   - Run `cargo doc --open` and view the result in your browser

## Navigation

[← Previous day](../015-return-values-pnl/en.md) | [Next day →](../017-if-conditions-buy-or-sell/en.md)
