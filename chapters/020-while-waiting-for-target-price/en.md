# Day 20: while — Waiting for Target Price

## Trading Analogy

Imagine you opened a BTC position and set a take-profit at $50,000 and a stop-loss at $38,000. What do you do next? You wait. You hold the position **while** the price has not crossed either boundary — up (take) or down (stop). That is exactly the `while` loop — it repeats a block of code **as long as a condition is true**. The moment the condition becomes false, the loop stops. Unlike `loop` with its "spin forever until I say stop", `while` knows its exit condition from the very beginning. Think of it as instructing your bot: "keep monitoring while the price is between $38,000 and $50,000".

## Basic while

The condition is checked **BEFORE** each iteration. If the condition is false from the very start, the loop body never executes.

```rust
fn main() {
    let target_price = 50000.0;
    let mut current_price = 45000.0;

    println!("Waiting for take-profit at ${:.0}...", target_price);

    while current_price < target_price {
        // Simulate price rising $1000 per tick
        current_price += 1000.0;
        println!("Price: ${:.0} | Remaining: ${:.0}",
                 current_price, target_price - current_price);
    }

    println!("\nTake-profit reached! Closing position at ${:.0}", current_price);
}
```

```rust
fn main() {
    // Example: condition is false from the start — loop never runs
    let price = 55000.0;
    let target = 50000.0;

    println!("Starting monitoring...");

    while price < target {
        // This code will never execute
        println!("Waiting...");
    }

    println!("Price ${:.0} is already above target ${:.0} — no entry possible", price, target);
}
```

## while with Multiple Conditions (&& and ||)

A real trading bot rarely watches just one condition. Use logical operators to build complex conditions.

```rust
fn main() {
    let mut balance = 10000.0_f64;
    let mut btc_price = 42000.0_f64;
    let min_balance = 1000.0;      // margin call when balance < $1000
    let target_price = 50000.0;    // price target
    let max_days = 30;
    let mut day = 0;

    println!("Start: balance = ${:.0}, price = ${:.0}", balance, btc_price);

    // Trade while balance is sufficient AND price hasn't reached target AND days limit not hit
    while balance > min_balance && btc_price < target_price && day < max_days {
        day += 1;

        // Simulate daily change
        let daily_pnl = if day % 3 == 0 { -500.0 } else { 200.0 };
        balance += daily_pnl;
        btc_price += 300.0; // price rises $300/day

        println!("Day {:02}: balance = ${:.0} | BTC = ${:.0} | PnL = {:+.0}",
                 day, balance, btc_price, daily_pnl);
    }

    // Find out why we stopped
    if balance <= min_balance {
        println!("\nStop: margin call! Balance: ${:.0}", balance);
    } else if btc_price >= target_price {
        println!("\nStop: take-profit reached! BTC = ${:.0}", btc_price);
    } else {
        println!("\nStop: {} day limit exhausted", max_days);
    }
}
```

Example with OR — exit if AT LEAST ONE danger condition is met:

```rust
fn main() {
    let mut rsi = 45.0_f64;
    let mut tick = 0;

    // Continue while RSI is in the neutral zone (not overbought AND not oversold)
    while rsi > 30.0 && rsi < 70.0 {
        tick += 1;

        // Simulate RSI change
        let delta = if tick % 2 == 0 { 5.0 } else { 3.0 };
        rsi += delta;

        println!("Tick {}: RSI = {:.1}", tick, rsi);
    }

    if rsi >= 70.0 {
        println!("RSI overbought ({:.1}) — SELL signal", rsi);
    } else {
        println!("RSI oversold ({:.1}) — BUY signal", rsi);
    }
}
```

## while for Accumulating Data

`while` is perfect when you need to accumulate data until you have enough.

```rust
fn main() {
    let required_ticks = 5;  // need 5 ticks to calculate moving average
    let mut prices = Vec::new();
    let mut tick_num = 0;

    // Simulated incoming prices
    let incoming = [42000.0, 42200.0, 41800.0, 42500.0, 42100.0, 43000.0];

    println!("Accumulating data for SMA({})...", required_ticks);

    // Accumulate until we have the required number of ticks
    while prices.len() < required_ticks {
        if tick_num < incoming.len() {
            let price = incoming[tick_num];
            prices.push(price);
            tick_num += 1;
            println!("Added price ${:.0} | Collected: {}/{}", price, prices.len(), required_ticks);
        } else {
            break; // no more data
        }
    }

    // Now we can calculate SMA
    let sum: f64 = prices.iter().sum();
    let sma = sum / prices.len() as f64;
    println!("\nSMA({}) = ${:.2}", required_ticks, sma);
}
```

## while let: Handling Option

`while let` is an elegant way to process data that may run out (like `Option`).

```rust
fn main() {
    // Queue of pending orders
    let mut pending_orders: Vec<f64> = vec![41500.0, 42000.0, 42500.0, 43000.0];

    println!("Processing order queue:");

    // Pop and process orders while the queue is not empty
    while let Some(order_price) = pending_orders.pop() {
        let order_type = if order_price < 42000.0 { "BUY" } else { "SELL" };
        println!("  Executing {} order at ${:.0}", order_type, order_price);
    }

    println!("All orders processed. Queue is empty.");
}
```

```rust
fn main() {
    let mut signals: Vec<&str> = vec!["BUY", "HOLD", "SELL", "BUY"];

    println!("Processing trading signals:");
    let mut executed = 0;

    while let Some(signal) = signals.first().copied() {
        signals.remove(0);
        executed += 1;
        println!("Signal #{}: {} — executed", executed, signal);
    }

    println!("Signals processed: {}", executed);
}
```

## Difference Between while and loop

Understanding the difference is important — it determines which tool to pick.

```rust
fn main() {
    // --- while: when condition is known upfront ---
    // Use when: condition is checked BEFORE entering the loop
    // "Spin while price < target"
    let target = 50000.0;
    let mut price = 45000.0;
    while price < target {
        price += 1000.0;
    }
    println!("while: price reached ${:.0}", price);

    // --- loop: when exit condition is complex or a value is needed ---
    // Use when: body must run at least once,
    // or exit depends on calculations inside the body
    let prices = [51000.0, 49000.0, 48000.0, 50500.0];
    let mut i = 0;
    let entry = loop {
        if prices[i] <= 50000.0 {
            break prices[i]; // return a value
        }
        i += 1;
        if i >= prices.len() {
            break 0.0;
        }
    };
    println!("loop: entry point found at ${:.0}", entry);
}
```

**Selection rule:**
- `while` — when the condition is simple and checked at the start of each iteration
- `loop` — when you need to return a value, or the exit happens in the middle/end of the loop body

## Simulating a Trading Day

Modelling a full trading day with open and close:

```rust
fn main() {
    let open_hour = 9;     // exchange opens
    let close_hour = 18;   // exchange closes
    let mut current_hour = open_hour;
    let mut total_volume = 0.0_f64;
    let mut hours_traded = 0;

    // Hourly volumes (simulation)
    let hourly_volumes = [
        500.0, 800.0, 1200.0, 1500.0, 900.0,
        1100.0, 1300.0, 950.0, 600.0,
    ];

    println!("Trading day started ({}:00 - {}:00)", open_hour, close_hour);
    println!("{:-<50}", "");

    while current_hour < close_hour {
        let idx = (current_hour - open_hour) as usize;
        let vol = if idx < hourly_volumes.len() { hourly_volumes[idx] } else { 0.0 };

        total_volume += vol;
        hours_traded += 1;

        // During peak hours — higher activity
        let activity = if vol > 1000.0 { "high" } else { "normal" };
        println!("{:02}:00 | Volume: {:.0} BTC | Activity: {}", current_hour, vol, activity);

        current_hour += 1;
    }

    println!("{:-<50}", "");
    println!("Trading day complete");
    println!("Total volume: {:.0} BTC", total_volume);
    println!("Hours traded: {}", hours_traded);
    println!("Average volume: {:.1} BTC/h", total_volume / hours_traded as f64);
}
```

## Practical Example: Waiting for an Entry Point

```rust
/// Searches for the optimal position entry point
fn main() {
    println!("=== Waiting for Entry Point ===\n");

    // Strategy parameters
    let entry_target = 42000.0_f64;  // we want to enter at this price
    let rsi_max = 40.0_f64;          // RSI must be oversold
    let min_volume = 1000.0_f64;     // minimum confirming volume
    let max_wait = 10_u32;           // wait at most 10 ticks

    // Simulated market data (price, RSI, volume)
    let market_data: Vec<(f64, f64, f64)> = vec![
        (43200.0, 55.0, 800.0),   // price high, waiting
        (43000.0, 50.0, 900.0),   // still too high
        (42800.0, 45.0, 950.0),   // getting closer
        (42400.0, 38.0, 1100.0),  // RSI good, price still high
        (42100.0, 36.0, 1200.0),  // almost at target!
        (41900.0, 33.0, 1350.0),  // price below target, RSI OK, volume OK
    ];

    let mut tick = 0_usize;
    let mut entry_found = false;

    // Wait while NOT all entry conditions are met
    while tick < market_data.len() && tick < max_wait as usize {
        let (price, rsi, volume) = market_data[tick];
        tick += 1;

        println!("Tick {:02}: price=${:.0} | RSI={:.1} | volume={:.0}",
                 tick, price, rsi, volume);

        // Check entry conditions
        let price_ok = price <= entry_target;
        let rsi_ok = rsi <= rsi_max;
        let volume_ok = volume >= min_volume;

        if !price_ok {
            println!("  Waiting: price ${:.0} > target ${:.0}", price, entry_target);
            continue;
        }
        if !rsi_ok {
            println!("  Waiting: RSI {:.1} > {:.1}", rsi, rsi_max);
            continue;
        }
        if !volume_ok {
            println!("  Waiting: volume {:.0} < {:.0}", volume, min_volume);
            continue;
        }

        // All conditions met!
        println!("\n  *** ENTERING POSITION! ***");
        println!("  Price: ${:.0}", price);
        println!("  RSI: {:.1} (oversold)", rsi);
        println!("  Volume: {:.0} (confirmed)", volume);
        entry_found = true;
        break;
    }

    if !entry_found {
        println!("\nEntry point not found within {} ticks", tick);
        println!("Continuing to monitor...");
    }
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `while condition { }` | Condition known upfront, checked before each iteration | Wait while price < target |
| `while a && b { }` | Multiple conditions at the same time | Balance > 0 AND price in range |
| `while a \|\| b { }` | Any of the conditions | While no TP AND no SL |
| `while let Some(x) = ... { }` | Handling Option, popping from Vec | Processing an order queue |
| Difference from `loop` | `while` — condition at the top, no return value | Use `while` for simple conditions |
| Difference from `loop` | `loop` — condition in the body, can return value | Use `loop` for complex exits |

## Homework

1. Write a "holding a position" simulation:
   - Price starts at $42,000 and changes by a fixed amount each tick
   - `while` continues while price is between the stop ($40,000) and take ($46,000)
   - After exit, print: reason (TP/SL), final price, number of ticks

2. Use `while` with multiple conditions for risk management:
   - Simulate trading: balance changes each "day"
   - Continue condition: balance > $1000 AND days < 30 AND drawdown < 20%
   - Print the reason for stopping at the end

3. Implement `while let` for processing a signal stream:
   - Create a Vec of signals: `vec![Some("BUY"), Some("HOLD"), None, Some("SELL")]`
   - Use `while let Some(s) = queue.pop()` — process while not None
   - Count the number of BUY and SELL signals

4. Write a data accumulator for an indicator:
   - Need to accumulate 20 prices to calculate SMA(20)
   - `while prices.len() < 20` — add prices from an array
   - After accumulation: calculate SMA(20) and print the result

## Navigation

[← Previous day](../019-loop-endless-price-monitoring/en.md) | [Next day →](../021-for-iterating-daily-trades/en.md)
