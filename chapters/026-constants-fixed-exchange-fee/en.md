# Day 26: Constants — Fixed Exchange Fee

## Trading Analogy

The exchange fee is a constant, public value. Binance charges 0.1% per trade — that doesn't change from hour to hour. Lot sizes, minimum orders, supported trading pairs — these are all fixed parameters too. In Rust, values that "never change" are called constants (`const`). They are embedded directly in the compiled code, evaluated at compile time, and guaranteed to never be modified — the compiler physically prevents it. This makes code safer and faster: no variable means no accidental-change bug.

## const vs let: What's the Difference

Comparing `const` and `let` directly to understand when to use each:

```rust
// CONSTANTS — known at compile time, available globally
const BINANCE_FEE:       f64 = 0.001;    // 0.1%
const MAX_POSITION_SIZE: f64 = 10_000.0; // $10,000
const MIN_ORDER_USDT:    f64 = 5.0;      // minimum $5
const LEVERAGE_MAX:      u8  = 20;       // up to 20x leverage

fn main() {
    // VARIABLES — computed at runtime, local scope
    let balance      = 5_000.0;  // may depend on user input
    let trade_amount = balance * 0.1; // 10% of balance

    println!("Balance: ${:.2}", balance);
    println!("Trade size: ${:.2}", trade_amount);
    println!("Fee: ${:.4}", trade_amount * BINANCE_FEE);
    println!("Max position: ${:.0}", MAX_POSITION_SIZE);

    // const cannot be reassigned:
    // BINANCE_FEE = 0.002; // COMPILE ERROR!

    // const cannot be mut:
    // const mut X: f64 = 1.0; // COMPILE ERROR!
}
```

Key differences: `const` requires an explicit type, is evaluated at compile time, is accessible anywhere, and cannot be `mut`.

## SCREAMING_SNAKE_CASE: Naming Convention

In Rust, constants are named in `SCREAMING_SNAKE_CASE` — all uppercase letters, words separated by underscores. This convention is universal:

```rust
// Trading constants
const COMMISSION_RATE_SPOT:    f64  = 0.001;   // spot
const COMMISSION_RATE_FUTURES: f64  = 0.0004;  // futures
const COMMISSION_RATE_MAKER:   f64  = 0.0002;  // maker
const DEFAULT_SLIPPAGE_BPS:    u32  = 5;       // 0.05% in basis points
const MAX_DAILY_LOSS_PCT:      f64  = 0.05;    // 5% max daily loss
const RISK_PER_TRADE_PCT:      f64  = 0.01;    // 1% risk per trade
const PAIRS_SUPPORTED:         u32  = 500;

fn main() {
    let balance   = 10_000.0;
    let max_loss  = balance * MAX_DAILY_LOSS_PCT;
    let risk_size = balance * RISK_PER_TRADE_PCT;

    println!("Max daily loss: -${:.2}", max_loss);
    println!("Risk per trade: ${:.2}", risk_size);
    println!("Spot fee: {:.3}%", COMMISSION_RATE_SPOT * 100.0);
    println!("Maker fee: {:.3}%", COMMISSION_RATE_MAKER * 100.0);
    println!("Supported pairs: {}", PAIRS_SUPPORTED);
}
```

Uppercase is a visual signal: "this is a constant, hands off!" Another programmer immediately sees the difference between `price` (variable) and `MAX_PRICE` (constant).

## const at Module Level

Constants can be declared at the module level — accessible from any function in the file:

```rust
// Global trading parameters
const API_VERSION:        &str = "v3";
const BASE_URL:           &str = "https://api.binance.com";
const DEFAULT_TIMEOUT_MS: u64  = 5_000;
const MAX_RETRIES:        u32  = 3;
const CANDLE_INTERVALS:   &str = "1m,5m,15m,1h,4h,1d";

fn build_url(endpoint: &str) -> String {
    format!("{}/api/{}/{}", BASE_URL, API_VERSION, endpoint)
}

fn get_retry_delay(attempt: u32) -> u64 {
    // Exponential backoff: 1s, 2s, 4s
    DEFAULT_TIMEOUT_MS * (2u64.pow(attempt.saturating_sub(1)))
}

fn main() {
    println!("API URL: {}", build_url("order"));
    println!("Intervals: {}", CANDLE_INTERVALS);

    for attempt in 1..=MAX_RETRIES {
        let delay = get_retry_delay(attempt);
        println!("Attempt {}/{}: delay {} ms", attempt, MAX_RETRIES, delay);
    }
}
```

## const vs static: The Subtle Difference

Rust also has `static` — similar to `const` but with important differences:

```rust
// const — inlined into code directly, no fixed address
const MAX_ORDERS: u32 = 100;

// static — has a fixed memory address, lives for the entire program
static EXCHANGE_NAME:    &str        = "Binance";
static SUPPORTED_COINS: [&str; 5]   = ["BTC", "ETH", "SOL", "BNB", "ADA"];

// static mut — mutable static variable (requires unsafe!)
static mut GLOBAL_TRADE_COUNT: u64 = 0;

fn increment_trade_counter() {
    unsafe {
        GLOBAL_TRADE_COUNT += 1;
    }
}

fn get_trade_count() -> u64 {
    unsafe { GLOBAL_TRADE_COUNT }
}

fn main() {
    println!("Exchange: {}", EXCHANGE_NAME);
    println!("Max orders: {}", MAX_ORDERS);
    println!("Supported coins: {:?}", SUPPORTED_COINS);

    increment_trade_counter();
    increment_trade_counter();
    increment_trade_counter();
    println!("Trade count: {}", get_trade_count());
}
```

`static mut` requires `unsafe` — because mutating a global variable is unsafe in a multi-threaded environment. In real programs use `std::sync::atomic::AtomicU64` for counters instead.

## const in impl Blocks: Struct Constants

It's often convenient to store constants inside the struct they belong to:

```rust
struct BinanceExchange;

impl BinanceExchange {
    const FEE_SPOT:     f64          = 0.001;
    const FEE_FUTURES:  f64          = 0.0004;
    const FEE_MAKER:    f64          = 0.0002;
    const MIN_ORDER:    f64          = 5.0;
    const MAX_LEVERAGE: u8           = 125;
    const NAME:         &'static str = "Binance";

    fn calculate_fee(amount: f64, order_type: &str) -> f64 {
        let rate = match order_type {
            "spot"    => Self::FEE_SPOT,
            "futures" => Self::FEE_FUTURES,
            "maker"   => Self::FEE_MAKER,
            _         => Self::FEE_SPOT,
        };
        amount * rate
    }

    fn is_valid_order(amount: f64) -> bool {
        amount >= Self::MIN_ORDER
    }
}

struct BybitExchange;

impl BybitExchange {
    const FEE_SPOT:    f64          = 0.001;
    const FEE_FUTURES: f64          = 0.0006;
    const MIN_ORDER:   f64          = 1.0;
    const NAME:        &'static str = "Bybit";
}

fn main() {
    let amount = 1000.0;

    println!("=== {} ===", BinanceExchange::NAME);
    println!("Spot fee:    ${:.4}", BinanceExchange::calculate_fee(amount, "spot"));
    println!("Futures fee: ${:.4}", BinanceExchange::calculate_fee(amount, "futures"));
    println!("Maker fee:   ${:.4}", BinanceExchange::calculate_fee(amount, "maker"));
    println!("Min order:   ${}", BinanceExchange::MIN_ORDER);
    println!("Max leverage: {}x", BinanceExchange::MAX_LEVERAGE);
    println!("$3 order valid:  {}", BinanceExchange::is_valid_order(3.0));
    println!("$10 order valid: {}", BinanceExchange::is_valid_order(10.0));

    println!("\n=== {} ===", BybitExchange::NAME);
    println!("Min order: ${}", BybitExchange::MIN_ORDER);
}
```

## const Expressions: Compile-Time Calculations

Constants can be derived from other constants, computed at compile time:

```rust
const SATS_PER_BTC:      u64 = 100_000_000; // 1 BTC = 100M satoshis
const DEFAULT_LEVERAGE:  u8  = 1;
const MAX_LEVERAGE:      u8  = 20;
const SECONDS_IN_MINUTE: u64 = 60;
const SECONDS_IN_HOUR:   u64 = SECONDS_IN_MINUTE * 60;  // derived from another const
const SECONDS_IN_DAY:    u64 = SECONDS_IN_HOUR * 24;
const SECONDS_IN_WEEK:   u64 = SECONDS_IN_DAY * 7;
const CANDLE_1M_IN_DAY:  u64 = 24 * 60;                 // 1440 1-minute candles per day
const CANDLE_1H_IN_WEEK: u64 = SECONDS_IN_WEEK / SECONDS_IN_HOUR; // 168 hourly per week

fn sats_to_btc(sats: u64) -> f64 {
    sats as f64 / SATS_PER_BTC as f64
}

fn btc_to_sats(btc: f64) -> u64 {
    (btc * SATS_PER_BTC as f64) as u64
}

fn main() {
    println!("Seconds per hour:        {}", SECONDS_IN_HOUR);
    println!("Seconds per day:         {}", SECONDS_IN_DAY);
    println!("1m candles per day:      {}", CANDLE_1M_IN_DAY);
    println!("1h candles per week:     {}", CANDLE_1H_IN_WEEK);
    println!("0.1 BTC in satoshis:     {}", btc_to_sats(0.1));
    println!("5,000,000 sats in BTC:   {}", sats_to_btc(5_000_000));
    println!("Default leverage: {}x", DEFAULT_LEVERAGE);
    println!("Max leverage: {}x", MAX_LEVERAGE);
}
```

## Practical: Exchange Configuration via Constants

Bringing it all together — a trading bot configuration through constants:

```rust
// ===== Exchange configuration =====
const EXCHANGE_NAME:   &str = "Binance";
const FEE_RATE:        f64  = 0.001;
const MIN_TRADE_USDT:  f64  = 5.0;
const MAX_TRADE_USDT:  f64  = 50_000.0;

// ===== Risk management =====
const MAX_RISK_PER_TRADE: f64 = 0.01;  // 1% of balance
const MAX_DAILY_DRAWDOWN: f64 = 0.05;  // 5% max daily loss
const DEFAULT_SL_PCT:     f64 = 0.02;  // 2% stop-loss
const DEFAULT_TP_PCT:     f64 = 0.04;  // 4% take-profit (RR = 1:2)

// ===== Strategy parameters =====
const SMA_FAST_PERIOD:  u32 = 9;
const SMA_SLOW_PERIOD:  u32 = 21;
const RSI_PERIOD:       u32 = 14;
const RSI_OVERSOLD:     f64 = 30.0;
const RSI_OVERBOUGHT:   f64 = 70.0;

fn calculate_position(balance: f64, entry: f64, stop_loss: f64) -> f64 {
    let risk_amount = balance * MAX_RISK_PER_TRADE;
    let points_risk = (entry - stop_loss).abs();
    if points_risk == 0.0 { return 0.0; }
    let raw_size = risk_amount / points_risk;
    let max_size = MAX_TRADE_USDT / entry;
    raw_size.min(max_size)
}

fn trade_levels(entry: f64) -> (f64, f64) {
    let stop_loss   = entry * (1.0 - DEFAULT_SL_PCT);
    let take_profit = entry * (1.0 + DEFAULT_TP_PCT);
    (stop_loss, take_profit)
}

fn main() {
    let balance     = 10_000.0;
    let entry_price = 50_000.0;

    let (sl, tp) = trade_levels(entry_price);
    let size = calculate_position(balance, entry_price, sl);
    let fee  = entry_price * size * FEE_RATE;

    println!("=== {} Trading Bot ===", EXCHANGE_NAME);
    println!();
    println!("Balance: ${:.2}", balance);
    println!("Entry price: ${:.2}", entry_price);
    println!();
    println!("Stop-loss:    ${:.2}  (-{:.0}%)", sl, DEFAULT_SL_PCT * 100.0);
    println!("Take-profit:  ${:.2}  (+{:.0}%)", tp, DEFAULT_TP_PCT * 100.0);
    println!("Position size: {:.6} BTC", size);
    println!("Trade value: ${:.2}", size * entry_price);
    println!("Commission: ${:.4} ({:.1}%)", fee, FEE_RATE * 100.0);
    println!();
    println!("Indicators: SMA({}/{}), RSI({})", SMA_FAST_PERIOD, SMA_SLOW_PERIOD, RSI_PERIOD);
    println!("RSI zones: oversold < {}, overbought > {}", RSI_OVERSOLD, RSI_OVERBOUGHT);

    let trade_value = size * entry_price;
    if trade_value < MIN_TRADE_USDT {
        println!("WARNING: trade ${:.2} below minimum ${}", trade_value, MIN_TRADE_USDT);
    } else if trade_value > MAX_TRADE_USDT {
        println!("WARNING: trade ${:.2} above maximum ${}", trade_value, MAX_TRADE_USDT);
    } else {
        println!("Limits: OK (${:.2} within ${}-${})", trade_value, MIN_TRADE_USDT, MAX_TRADE_USDT);
    }
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `const NAME: Type = val;` | Value known at compile time and never changes | Exchange fee, order limits |
| `SCREAMING_SNAKE_CASE` | Naming convention for all constants | `MAX_POSITION_SIZE`, `FEE_RATE` |
| `const` at module level | Parameters needed across the whole file/module | API URLs, timeouts, indicator periods |
| `static NAME: &str` | String constants with a fixed memory address | Exchange names, API versions |
| `static mut` + `unsafe` | Mutable global counter (avoid in production) | Trade counter (prefer `AtomicU64`) |
| `impl Struct { const X }` | Constants logically tied to a struct | `BinanceExchange::FEE_RATE` |
| Computed `const` | Derived values from other constants | `SECONDS_IN_DAY = SECONDS_IN_HOUR * 24` |

## Homework

1. Declare a trading bot config file using constants:
   - `EXCHANGE_NAME`, `API_KEY` (mock value), `BASE_URL`
   - `FEE_MAKER`, `FEE_TAKER`, `MIN_ORDER`, `MAX_ORDER`
   - Write a `validate_order(amount: f64) -> bool` function using those constants

2. Create a `RiskConfig` struct with constants in an `impl` block:
   - `MAX_RISK_PER_TRADE: f64 = 0.01`
   - `MAX_DAILY_LOSS: f64 = 0.05`
   - `MAX_OPEN_POSITIONS: u8 = 5`
   - Method `is_trade_allowed(current_loss: f64, open_positions: u8) -> bool`

3. Compute derived constants:
   - `CANDLE_COUNT_PER_DAY_1M: u32`
   - `CANDLE_COUNT_PER_WEEK_1H: u32`
   - Use them in a function that builds a list of timestamps

4. Compare `const` and `static` in practice:
   - Declare `const TICKERS: [&str; 3]` and `static TICKERS_STATIC: [&str; 3]`
   - Iterate and print both
   - Add comments explaining when to use `const` vs `static`

## Navigation

[← Previous day](../025-match-determining-order-type/en.md) | [Next day →](../027-shadowing-updating-price/en.md)
