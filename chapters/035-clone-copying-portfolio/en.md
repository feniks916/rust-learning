# Day 35: Clone — Copying the Portfolio

## Trading Analogy

You want to test a new strategy without risking your real portfolio. So you make a copy — a virtual portfolio with the same positions. Changes to the copy don't affect the original. In Rust, the `.clone()` method does exactly this: creates a full independent copy of data on the heap.

---

## .clone() — Full Copy

`.clone()` is an explicit request: "I want an independent copy of this data". Unlike Move, after clone you have two independent objects.

```rust
fn main() {
    let original_ticker = String::from("BTCUSDT");

    // Move — only one owner
    // let moved = original_ticker;
    // println!("{}", original_ticker); // ERROR

    // Clone — two independent objects
    let cloned_ticker = original_ticker.clone();

    println!("Original: {}", original_ticker); // OK
    println!("Clone: {}", cloned_ticker);       // OK

    // Changing clone doesn't affect original
    let mut test_ticker = original_ticker.clone();
    test_ticker.push_str("_TEST");
    println!("Original: {}", original_ticker); // "BTCUSDT" — unchanged
    println!("Test: {}", test_ticker);          // "BTCUSDT_TEST"
}
```

---

## #[derive(Clone)] for Custom Structs

To make `clone()` work for your struct, you either implement the `Clone` trait manually or use `#[derive(Clone)]`:

```rust
#[derive(Debug, Clone)]
struct Portfolio {
    name: String,
    positions: Vec<String>,
    cash_balance: f64,
}

impl Portfolio {
    fn add_position(&mut self, ticker: String) {
        self.positions.push(ticker);
    }

    fn total_positions(&self) -> usize {
        self.positions.len()
    }
}

fn main() {
    let mut real_portfolio = Portfolio {
        name: String::from("Real Account"),
        positions: vec![String::from("BTCUSDT"), String::from("ETHUSDT")],
        cash_balance: 10000.0,
    };

    // Clone for backtesting
    let mut test_portfolio = real_portfolio.clone();
    test_portfolio.name = String::from("Backtest Account");
    test_portfolio.add_position(String::from("SOLUSDT")); // only in copy

    println!("Real: {:?}", real_portfolio);  // 2 positions
    println!("Test: {:?}", test_portfolio);  // 3 positions

    // Changing real doesn't affect copy
    real_portfolio.cash_balance -= 5000.0;
    println!("Real balance: ${}", real_portfolio.cash_balance);
    println!("Test balance: ${}", test_portfolio.cash_balance); // still 10000
}
```

---

## Deep Clone vs Shallow Clone

In Rust, `clone()` always performs a **deep clone**: all nested data is also copied. There is no "shallow clone" concept like in Python — you always get full independence:

```rust
fn main() {
    // Complex structure with nested data
    let original: Vec<Vec<f64>> = vec![
        vec![42000.0, 42100.0, 41900.0], // weekdays
        vec![43000.0, 43200.0, 42800.0],
        vec![44000.0, 44100.0, 43900.0],
    ];

    let cloned = original.clone(); // Deep clone: EVERYTHING is copied

    // Changing cloned can't affect original and vice versa
    // (In Python shallow clone, vec[0][0] would be shared!)

    println!("Original [0][0]: {}", original[0][0]);
    println!("Clone [0][0]: {}", cloned[0][0]);

    // Strings inside Vec are also fully cloned
    let tickers = vec![String::from("BTC"), String::from("ETH")];
    let tickers_clone = tickers.clone(); // each string — separate heap copy

    println!("Tickers: {:?}", tickers);
    println!("Clone tickers: {:?}", tickers_clone);
}
```

---

## When Clone Is Needed

Clone is needed when you need to:
1. Keep the original after passing a value by value to a function
2. Create an independent copy for modification
3. Place the same data in multiple places

```rust
fn run_backtest(prices: Vec<f64>, strategy_name: &str) -> f64 {
    // Takes Vec by value — consumes it
    let returns: Vec<f64> = prices.windows(2)
        .map(|w| (w[1] - w[0]) / w[0])
        .collect();
    let total_return: f64 = returns.iter().sum();
    println!("Strategy {}: return = {:.2}%", strategy_name, total_return * 100.0);
    total_return
}

fn main() {
    let price_history = vec![40000.0, 42000.0, 41000.0, 44000.0, 43000.0, 45000.0];

    // Want to run multiple strategies on same data
    // WITHOUT clone the first call would consume price_history:
    let r1 = run_backtest(price_history.clone(), "Momentum");
    let r2 = run_backtest(price_history.clone(), "Mean Reversion");
    let r3 = run_backtest(price_history.clone(), "Buy and Hold");
    // price_history still accessible!

    println!("\nResults:");
    println!("  Momentum: {:.2}%", r1 * 100.0);
    println!("  Mean Rev: {:.2}%", r2 * 100.0);
    println!("  B&H: {:.2}%", r3 * 100.0);
    println!("Original series: {} points", price_history.len());
}
```

---

## Clone Is Expensive — Measurement Example

Clone allocates new memory and copies all bytes. For large structures this is slow:

```rust
use std::time::Instant;

fn main() {
    // Large price vector — like real tick data
    let tick_data: Vec<f64> = (0..1_000_000).map(|i| 40000.0 + (i as f64) * 0.001).collect();

    // Measure clone time
    let start = Instant::now();
    let _cloned = tick_data.clone(); // copy 8 MB
    let clone_time = start.elapsed();

    // Measure reference access time (no copying)
    let start = Instant::now();
    let sum: f64 = tick_data.iter().sum(); // just reading
    let ref_time = start.elapsed();

    println!("Data size: {} bytes", tick_data.len() * 8);
    println!("Clone: {:?}", clone_time);  // e.g. 5ms
    println!("Ref:   {:?}", ref_time);    // e.g. 0.5ms — 10x faster!
    println!("Sum: {}", sum);

    // CONCLUSION: if you only need to read — pass &Vec, don't clone!
}
```

Rule: **use `clone()` only when you genuinely need an independent copy**. If you're just reading — use `&T`.

---

## Clone-before-Move Pattern

Classic pattern: clone before the value is moved:

```rust
fn send_order(order: String) {
    println!("[EXCHANGE] Order sent: {}", order);
    // order destroyed here
}

fn log_order(order: &str) {
    println!("[LOG] Logging: {}", order);
}

fn main() {
    let order = String::from("BUY BTCUSDT 0.1 @ 42000");

    // Log first (via reference — no Move)
    log_order(&order);

    // Save a copy for history
    let order_history = order.clone(); // clone before move

    // Send original (Move)
    send_order(order);
    // order is now inaccessible

    // But we have the copy in history
    println!("[HISTORY] Recorded order: {}", order_history);

    // Pattern for Vec
    let portfolio = vec![String::from("BTC"), String::from("ETH")];
    let portfolio_backup = portfolio.clone(); // clone before move

    let result = portfolio.into_iter()
        .map(|t| format!("{}USDT", t))
        .collect::<Vec<_>>();

    // portfolio moved into into_iter and destroyed
    println!("Result: {:?}", result);
    println!("Backup: {:?}", portfolio_backup); // clone still alive
}
```

---

## Practical Example: Backtesting System

```rust
#[derive(Debug, Clone)]
struct TradingStrategy {
    name: String,
    stop_loss: f64,    // percent from entry
    take_profit: f64,  // percent from entry
    position_size: f64, // fraction of capital
}

#[derive(Debug, Clone)]
struct BacktestConfig {
    strategy: TradingStrategy,
    initial_capital: f64,
    prices: Vec<f64>,
}

fn run_backtest(config: BacktestConfig) -> (String, f64, usize) {
    // config is consumed (Move)
    let mut capital = config.initial_capital;
    let mut trades = 0;

    let entry_price = config.prices[0];
    for &price in &config.prices[1..] {
        let change_pct = (price - entry_price) / entry_price;

        if change_pct >= config.strategy.take_profit {
            capital *= 1.0 + config.strategy.position_size * config.strategy.take_profit;
            trades += 1;
        } else if change_pct <= -config.strategy.stop_loss {
            capital *= 1.0 - config.strategy.position_size * config.strategy.stop_loss;
            trades += 1;
        }
    }

    (config.strategy.name, capital, trades)
}

fn main() {
    let prices = vec![
        40000.0, 41000.0, 39500.0, 42000.0, 44000.0,
        43000.0, 45000.0, 44500.0, 46000.0, 45000.0,
    ];

    let base_config = BacktestConfig {
        strategy: TradingStrategy {
            name: String::from("Conservative"),
            stop_loss: 0.02,
            take_profit: 0.04,
            position_size: 0.1,
        },
        initial_capital: 10000.0,
        prices: prices.clone(), // clone for config
    };

    // Test different strategy variants — all via clone of base config
    let mut aggressive_config = base_config.clone();
    aggressive_config.strategy.name = String::from("Aggressive");
    aggressive_config.strategy.stop_loss = 0.05;
    aggressive_config.strategy.take_profit = 0.10;
    aggressive_config.strategy.position_size = 0.3;

    let mut scalping_config = base_config.clone();
    scalping_config.strategy.name = String::from("Scalping");
    scalping_config.strategy.stop_loss = 0.005;
    scalping_config.strategy.take_profit = 0.01;
    scalping_config.strategy.position_size = 0.5;

    println!("=== Backtest Results ===");
    for config in [base_config, aggressive_config, scalping_config] {
        let (name, final_capital, trade_count) = run_backtest(config);
        println!(
            "{}: ${:.2} ({} trades, {}%)",
            name, final_capital, trade_count,
            ((final_capital / 10000.0 - 1.0) * 100.0) as i32
        );
    }
}
```

---

## What We Learned

| Syntax | Description | Trading Example |
|--------|-------------|-----------------|
| `.clone()` | Full independent copy | Clone portfolio for testing |
| `#[derive(Clone)]` | Auto-implement Clone | Strategy struct with settings |
| Deep clone | All nested data also copied | Full price history nested in struct |
| Clone before Move | Save copy before passing to function | Backup order before sending |
| Clone is expensive | Allocates memory and copies bytes | Don't clone 1M ticks unnecessarily |
| `&T` instead of clone | Reading doesn't need a copy | Analyze data via reference |

---

## Homework

1. Create a struct `PriceBar { open: f64, high: f64, low: f64, close: f64, volume: f64 }` with `#[derive(Clone, Debug)]`.
   - Create `Vec<PriceBar>` with 5 candles
   - Clone the vector and apply "normalization" to the copy (divide all prices by the close of the first candle)
   - Confirm the original candles are unchanged

2. Write a function `optimize_strategy(base: TradingStrategy, param_sets: Vec<(f64, f64)>) -> Vec<TradingStrategy>`:
   - `TradingStrategy { name: String, stop: f64, target: f64 }`
   - For each parameter pair from `param_sets` creates a `.clone()` of the base strategy and substitutes the parameters
   - Returns a vector of modified strategies

3. Study the cost of clone:
   - Create `Vec<String>` with 10,000 tickers via `(0..10000).map(|i| format!("TICKER_{:05}", i)).collect()`
   - Measure `clone()` time via `std::time::Instant`
   - Compare with time for simple iteration via `&vec`

4. Implement "snapshot and rollback":
   - Struct `OrderBook { bids: Vec<f64>, asks: Vec<f64> }`
   - Function `snapshot(book: &OrderBook) -> OrderBook` returns `book.clone()`
   - Function `apply_trade(book: &mut OrderBook, price: f64)` modifies the book
   - After several `apply_trade` calls, roll back to the snapshot

---

## Navigation

[← Previous day](../034-move-selling-asset/en.md) | [Next day →](../036-copy-light-types-cash/en.md)
