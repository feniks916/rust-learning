# Day 155: move Closures — Passing Data to a Thread

## Trading Analogy

Imagine you are launching a trading bot into a separate office. It needs to take a list of tickers, the current price, and the strategy config with it. Once the bot leaves — you can no longer change the data it took. A `move` closure does exactly this: it takes variables from the current context and transfers them into a new thread, where those data now belong exclusively to the thread.

## What Is a Closure

A closure is an anonymous function that can "capture" variables from the surrounding code:

```rust
fn main() {
    let ticker = String::from("BTCUSDT");
    let price = 65000.0f64;

    // Closure captures ticker and price by reference
    let print_info = || {
        println!("Ticker: {}, price: {:.2}", ticker, price);
    };

    print_info(); // Ticker: BTCUSDT, price: 65000.00

    // ticker and price are still accessible here
    println!("Ticker still accessible: {}", ticker);
}
```

A regular closure captures variables by reference — so the original remains accessible.

## The Problem Without move

When passing a closure to a thread, Rust requires all captured variables to live long enough. Without `move` it does not compile:

```rust
use std::thread;

fn main() {
    let ticker = String::from("ETHUSDT");

    // ERROR: ticker may be dropped before the thread finishes
    let handle = thread::spawn(|| {
        // borrowed value does not live long enough
        println!("Monitoring: {}", ticker);
    });

    handle.join().unwrap();
}
```

The problem: `ticker` lives on `main`'s stack, but the thread may continue running after `main` returns. Rust protects us from this.

## The move Syntax

The `move` keyword before `||` makes the closure move (rather than borrow) all captured variables:

```rust
use std::thread;

fn main() {
    let ticker = String::from("ETHUSDT");
    let price = 3200.0f64;

    // move transfers ticker and price into the closure
    let handle = thread::spawn(move || {
        println!("Thread monitoring {} @ ${:.2}", ticker, price);
        // ticker and price now belong to this thread
    });

    // Cannot use ticker here — it was moved!
    // println!("{}", ticker); // ERROR: value used after move

    handle.join().unwrap();
}
```

## What Gets Captured

`move` captures ALL variables the closure references:

```rust
use std::thread;

fn main() {
    let symbol = String::from("BNBUSDT");
    let entry_price = 420.0f64;
    let quantity = 10.0f64;
    let stop_loss = 400.0f64;

    let handle = thread::spawn(move || {
        // All 4 variables captured and moved
        let risk = (entry_price - stop_loss) * quantity;
        println!(
            "Bot for {}: entry={:.2}, stop={:.2}, risk=${:.2}",
            symbol, entry_price, stop_loss, risk
        );
    });

    handle.join().unwrap();
    // symbol, entry_price, quantity, stop_loss — all moved
}
```

## Copy Types in move

Numbers and booleans implement `Copy` — they are copied, not moved. The original remains accessible:

```rust
use std::thread;

fn main() {
    let price = 65000.0f64;    // f64 implements Copy
    let quantity = 0.5f64;     // f64 implements Copy
    let is_active = true;       // bool implements Copy

    let handle = thread::spawn(move || {
        // price, quantity, is_active — copied, not moved
        println!("Price: {:.2}, qty: {:.2}, active: {}", price, quantity, is_active);
    });

    // Copy types are still accessible in main!
    println!("In main: price = {:.2}", price);
    println!("In main: active = {}", is_active);

    handle.join().unwrap();
}
```

## Clone Before move

If you need a variable in both the main thread and a child thread — clone before moving:

```rust
use std::thread;

fn main() {
    let tickers = vec!["BTCUSDT", "ETHUSDT", "BNBUSDT"];

    // Clone for the thread
    let tickers_for_thread = tickers.clone();

    let handle = thread::spawn(move || {
        for ticker in &tickers_for_thread {
            println!("Thread processing: {}", ticker);
        }
    });

    // Original is still accessible
    println!("In main, ticker count: {}", tickers.len());

    handle.join().unwrap();
}
```

## Multiple Threads with Different Data

Pass different data to each thread via `move`:

```rust
use std::thread;

fn main() {
    let symbols = vec![
        ("BTCUSDT", 65000.0),
        ("ETHUSDT", 3200.0),
        ("BNBUSDT", 420.0),
    ];

    let mut handles = Vec::new();

    for (symbol, price) in symbols {
        // Each iteration creates a new closure with move
        // symbol and price are small values — copied
        let handle = thread::spawn(move || {
            let alert_price = price * 1.05;
            println!(
                "Bot {}: current={:.2}, target={:.2}",
                symbol, price, alert_price
            );
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

## Practical Example: Monitoring Multiple Symbols

```rust
use std::thread;
use std::time::Duration;

#[derive(Clone)]
struct MonitorConfig {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    check_interval_ms: u64,
}

impl MonitorConfig {
    fn new(symbol: &str, entry: f64, stop: f64, target: f64) -> Self {
        MonitorConfig {
            symbol: symbol.to_string(),
            entry_price: entry,
            stop_loss: stop,
            take_profit: target,
            check_interval_ms: 100,
        }
    }
}

fn simulate_price(base: f64, tick: u32) -> f64 {
    let change = (tick as f64 * 0.7).sin() * base * 0.02;
    base + change
}

fn monitor_symbol(config: MonitorConfig) {
    println!(
        "[{}] Starting: entry={:.2}, stop={:.2}, target={:.2}",
        config.symbol, config.entry_price, config.stop_loss, config.take_profit
    );

    for tick in 0..5u32 {
        let current_price = simulate_price(config.entry_price, tick);
        let pnl = current_price - config.entry_price;

        let status = if current_price <= config.stop_loss {
            "STOP-LOSS!"
        } else if current_price >= config.take_profit {
            "TAKE-PROFIT!"
        } else {
            "in position"
        };

        println!(
            "[{}] tick {}: price={:.2}, PnL={:+.2} — {}",
            config.symbol, tick, current_price, pnl, status
        );

        thread::sleep(Duration::from_millis(config.check_interval_ms));
    }

    println!("[{}] Monitoring complete", config.symbol);
}

fn main() {
    let configs = vec![
        MonitorConfig::new("BTCUSDT", 65000.0, 63000.0, 68000.0),
        MonitorConfig::new("ETHUSDT", 3200.0, 3000.0, 3500.0),
        MonitorConfig::new("BNBUSDT", 420.0, 400.0, 450.0),
    ];

    let mut handles = Vec::new();

    for config in configs {
        // move captures config (it is moved, not cloned)
        let handle = thread::spawn(move || {
            monitor_symbol(config);
        });
        handles.push(handle);
    }

    println!("All bots launched, waiting for completion...\n");

    for handle in handles {
        handle.join().unwrap();
    }

    println!("\nAll bots finished");
}
```

## What We Learned

| Syntax | Description | Use Case |
|--------|-------------|----------|
| `\|\| { ... }` | Regular closure | Captures by reference |
| `move \|\| { ... }` | Move closure | Moves variables inside |
| `Copy` types in `move` | Copied, not moved | `f64`, `i32`, `bool` remain accessible after |
| `clone()` before `move` | Clone for thread, original stays | `String`, `Vec` need cloning |
| `thread::spawn(move \|\| ...)` | Launch thread with data capture | Pass config to worker thread |
| Multiple threads | Each gets its own data instance | Parallel ticker monitoring |

## Homework

1. Create a struct `PriceAlert` with fields `symbol: String`, `threshold: f64`, `above: bool`
   - `above = true` — alert when price is above threshold
   - `above = false` — alert when price is below threshold

2. Write a function `watch_price(alert: PriceAlert, prices: Vec<f64>)`:
   - Iterates over prices
   - Prints a message when the condition is met
   - Prints a summary at the end

3. Launch 3 threads via `thread::spawn(move || ...)`, each with its own `PriceAlert`:
   - BTC: alert above 70000
   - ETH: alert below 3000
   - BNB: alert above 450

4. Make sure `PriceAlert` implements `Clone` (add `#[derive(Clone)]`)
   - Clone the alert before passing it to a thread
   - Verify the original is accessible in `main` after launching the thread

5. Add a field `id: u32` to `PriceAlert` (a Copy type):
   - Verify that `id` is accessible in `main` after `move` into the thread

## Navigation

[← Previous day](../154-join-thread-completion/en.md) | [Next day →](../156-channels-mpsc-order-queue/en.md)
