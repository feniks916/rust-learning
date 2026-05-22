# Day 23: break — Exit When Take-Profit Is Hit

## Trading Analogy

Imagine you've set a take-profit at $55,000. The price rises tick by tick, and the moment it touches your target — you immediately close the position and lock in your profit. You don't wait for the next tick, you don't check additional conditions — you simply exit. That's exactly how `break` works in Rust: as soon as the condition is met, the loop execution stops immediately and the program continues from the line after the loop. This is one of the most important flow-control tools, and in trading algorithms you'll see it absolutely everywhere.

## Basic break: Exit a Loop on a Condition

The simplest case — a `for` loop over an array of prices, exiting when the price reaches the target level:

```rust
fn main() {
    let prices = [49500.0, 50100.0, 51200.0, 52800.0, 54100.0, 55300.0, 56000.0];
    let take_profit = 55000.0;

    println!("Monitoring BTC/USDT price...");

    for (tick, price) in prices.iter().enumerate() {
        println!("Tick {}: ${:.2}", tick + 1, price);

        if *price >= take_profit {
            println!("Take-profit triggered at tick {}! Price: ${:.2}", tick + 1, price);
            println!("Closing position and locking in profit.");
            break;
        }
    }

    println!("Monitoring complete.");
}
```

After `break`, Rust immediately jumps to `println!("Monitoring complete.")` — the remaining ticks are never processed. This matters for performance: why scan 1000 candles when the signal is found on the 10th?

## break in a while Loop: Waiting for a Level Breakout

`while` with `break` is common when you don't know upfront how many iterations you'll need:

```rust
fn main() {
    let resistance = 52000.0;
    let mut price = 50000.0;
    let mut ticks = 0;

    println!("Waiting for breakout above resistance ${:.0}...", resistance);

    while price < resistance * 1.1 {
        // Simulate price movement
        price += price * 0.005; // +0.5% per tick
        ticks += 1;

        println!("Tick {}: price ${:.2}", ticks, price);

        if price > resistance {
            println!("BREAKOUT! Price ${:.2} broke through ${:.0}", price, resistance);
            println!("Opening long position on breakout.");
            break;
        }

        if ticks >= 20 {
            println!("Timeout: no breakout after {} ticks", ticks);
            break;
        }
    }

    println!("Total wait: {} ticks.", ticks);
}
```

Two `break` statements: one on a successful breakout, another on timeout. This is a classic pattern: "wait for an event, but not forever".

## break in an Infinite loop

`loop` in Rust is a special construct for an infinite loop, exited only via `break`. Perfect for a trading bot that runs continuously:

```rust
fn main() {
    let mut price = 50000.0;
    let stop_loss   = 48000.0;
    let take_profit = 54000.0;
    let mut iteration = 0;

    println!("Trading bot started. SL: ${:.0}, TP: ${:.0}", stop_loss, take_profit);

    loop {
        iteration += 1;
        // Simulate price change
        let change = if iteration % 3 == 0 { -500.0 } else { 300.0 };
        price += change;

        println!("Iteration {}: price ${:.2}", iteration, price);

        if price <= stop_loss {
            println!("STOP-LOSS! Price dropped to ${:.2}. Loss locked.", price);
            break;
        }

        if price >= take_profit {
            println!("TAKE-PROFIT! Price rose to ${:.2}. Profit locked.", price);
            break;
        }
    }

    println!("Bot stopped after {} iterations.", iteration);
}
```

`loop` without an explicit condition is honest: "run until I say stop." The exit conditions are fully under our control.

## break Returning a Value from loop

A unique Rust feature — returning a value from a loop via `break`. This lets you use `loop` as an expression:

```rust
fn find_entry_price(prices: &[f64], min_price: f64) -> Option<f64> {
    let result = loop {
        for &price in prices {
            if price > min_price {
                break Some(price); // return value from loop
            }
        }
        break None; // if not found
    };
    result
}

fn calculate_first_profit(trades: &[(f64, f64)]) -> Option<f64> {
    // (entry_price, exit_price)
    let profit = loop {
        for &(entry, exit) in trades {
            let pnl = exit - entry;
            if pnl > 0.0 {
                break Some(pnl);
            }
        }
        break None;
    };
    profit
}

fn main() {
    let prices = vec![48000.0, 49500.0, 51000.0, 52500.0];
    match find_entry_price(&prices, 50000.0) {
        Some(p) => println!("First suitable entry price: ${:.2}", p),
        None    => println!("No suitable prices found"),
    }

    let trades = vec![(50000.0, 49000.0), (49000.0, 48000.0), (48500.0, 51000.0)];
    match calculate_first_profit(&trades) {
        Some(p) => println!("First profitable trade: +${:.2}", p),
        None    => println!("No profitable trades"),
    }
}
```

`break Some(price)` is not just an exit — it passes data outward. A very elegant pattern for finding the first matching value.

## Nested Loops and 'outer Labels

When loops are nested, a plain `break` only exits the innermost one. To exit an outer loop, you need labels:

```rust
fn main() {
    let days     = ["Mon", "Tue", "Wed", "Thu", "Fri"];
    let sessions = ["London", "New York", "Asia"];
    let mut trades_found = 0;
    let max_trades = 3;

    println!("Scanning trading sessions by day...");

    'day_loop: for day in &days {
        for session in &sessions {
            // Simulate finding trade signals
            let signal = format!("{}-{}", day, session).len() % 3;

            if signal == 0 {
                trades_found += 1;
                println!("Signal found: {} session {}, total signals: {}", day, session, trades_found);

                if trades_found >= max_trades {
                    println!("Signal limit {} reached. Stopping scan.", max_trades);
                    break 'day_loop; // exit the OUTER loop
                }
            } else {
                println!("  {} {}: no signals", day, session);
            }
        }
    }

    println!("Scan complete. Signals found: {}", trades_found);
}
```

The label `'day_loop:` is placed before the outer loop. `break 'day_loop` exits it, bypassing all inner loops. Rust label naming rule: always starts with an apostrophe `'`.

## Labels with while and Double Nesting

Labels work with any loop type, not just `for`:

```rust
fn main() {
    let weeks         = 3;
    let days_per_week = 5;
    let hours_per_day = 8;
    let mut total_hours = 0;

    let emergency_day = (2, 3); // week 2, day 3

    'week: for week in 1..=weeks {
        'day: for day in 1..=days_per_week {
            let mut hour = 9;
            while hour < 9 + hours_per_day {
                if (week, day) == emergency_day && hour == 11 {
                    println!("Emergency trading halt! Week {}, day {}, hour {}", week, day, hour);
                    break 'week; // exit ALL loops
                }
                total_hours += 1;
                hour += 1;
            }
            println!("Day {}/{}: complete", week, day);
        }
    }

    println!("Total trading hours before halt: {}", total_hours);
}
```

`break 'week` jumps through all three nesting levels at once. Much cleaner than a chain of `should_stop` flags.

## Practical Example: Finding the First Level Breakout

Bringing it all together — a real algorithm to find a price breakout in an array of candles:

```rust
struct Candle {
    open:   f64,
    high:   f64,
    low:    f64,
    close:  f64,
    volume: f64,
}

fn find_breakout(candles: &[Candle], resistance: f64, min_volume: f64) -> Option<(usize, f64)> {
    let found = loop {
        for (i, candle) in candles.iter().enumerate() {
            // Breakout must be confirmed by close above the level
            if candle.close > resistance && candle.volume >= min_volume {
                break Some((i, candle.close));
            }
            // False breakout — high above, but close below
            if candle.high > resistance && candle.close <= resistance {
                println!("Candle {}: false breakout (high {:.2}, close {:.2})", i + 1, candle.high, candle.close);
            }
        }
        break None;
    };
    found
}

fn main() {
    let resistance = 52000.0;
    let min_volume = 1000.0;

    let candles = vec![
        Candle { open: 50000.0, high: 51500.0, low: 49800.0, close: 51200.0, volume:  850.0 },
        Candle { open: 51200.0, high: 52300.0, low: 51000.0, close: 51800.0, volume:  920.0 },
        Candle { open: 51800.0, high: 53000.0, low: 51500.0, close: 51900.0, volume:  780.0 }, // false
        Candle { open: 51900.0, high: 53500.0, low: 51700.0, close: 52800.0, volume: 1200.0 }, // breakout!
        Candle { open: 52800.0, high: 54000.0, low: 52500.0, close: 53500.0, volume: 1500.0 },
    ];

    println!("Searching for breakout of resistance ${:.0}...", resistance);
    println!("Minimum volume for confirmation: {:.0}", min_volume);
    println!();

    match find_breakout(&candles, resistance, min_volume) {
        Some((idx, price)) => {
            println!();
            println!("BREAKOUT CONFIRMED!");
            println!("Candle: {}", idx + 1);
            println!("Close price: ${:.2}", price);
            println!("Signal: open long above ${:.2}", resistance);
        }
        None => {
            println!("No breakout of ${:.0} found", resistance);
        }
    }
}
```

`break Some((i, candle.close))` returns two values at once — the candle index and the price. This is idiomatic Rust: `loop` as a search function with early exit.

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `break` | Immediately exit the current loop when a condition is met | Take-profit hit → exit monitoring loop |
| `break` in `loop` | Infinite loop with explicit stop condition | Trading bot runs until SL or TP |
| `break value` | Return a search result from `loop` | `let price = loop { ... break Some(p); }` |
| `'label: for` | Mark an outer loop for control from inside | Scanning days and sessions |
| `break 'label` | Exit a specific named loop | Exit all nesting levels at signal limit |
| Double `break` | Handle multiple exit conditions | Breakout or timeout |

## Homework

1. Write a `find_stop_loss_hit` function that takes an array of prices and a stop-loss level and returns `Option<(usize, f64)>` — the index and price of the first bar where the stop was triggered
   - Use `loop` with `break Some(...)` to return the result
   - Add a check: the stop is considered triggered when the close price is below the level

2. Implement a `scan_multi_timeframe` function that takes a 2D array of candles (multiple timeframes) and looks for a matching signal on all timeframes
   - Use nested loops with labels `'timeframe:` and `'candle:`
   - On finding a match — `break 'timeframe` returning the candle number

3. Write a loop that searches for the best entry price across multiple exchanges
   - Outer loop — over exchanges `["Binance", "Bybit", "OKX"]`
   - Inner loop — over bid/ask prices
   - Exit both loops when a price better than the target is found

4. Create a trading day simulation: `loop` with a tick counter, exit on three conditions — SL, TP, or end of day (100 ticks)
   - Print the reason for exit
   - Count the number of ticks until exit

## Navigation

[← Previous day](../022-range-prices/en.md) | [Next day →](../024-continue-skip-losing-trades/en.md)
