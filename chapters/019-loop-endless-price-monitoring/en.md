# Day 19: loop — Endless Price Monitoring

## Trading Analogy

A trading bot never sleeps. While markets are open (and crypto markets are always open — 24/7/365), the bot continuously checks prices, signals, balances, and places orders. This is not a "fire and forget" task — it is an endless loop: check the price, make a decision, wait for the next tick, check again. The `loop` construct in Rust is exactly that kind of infinite monitoring engine. It executes a block of code over and over until you explicitly say "STOP" via `break`. Think of it as a trader glued to their monitor, never blinking.

## Basic loop

The simplest `loop` just repeats forever. Without a `break` it will never stop!

```rust
fn main() {
    let mut tick = 0;
    let mut btc_price = 42000.0;

    loop {
        tick += 1;

        // Simulate price movement (±$100 per tick)
        let price_delta = if tick % 3 == 0 { 100.0 } else { -50.0 };
        btc_price += price_delta;

        println!("Tick #{}: BTC = ${:.2}", tick, btc_price);

        // Mandatory exit! Otherwise the program hangs forever
        if tick >= 5 {
            println!("Monitoring complete after {} ticks", tick);
            break;
        }
    }
}
```

Output:
```
Tick #1: BTC = $41950.00
Tick #2: BTC = $41900.00
Tick #3: BTC = $42000.00
Tick #4: BTC = $41950.00
Tick #5: BTC = $41900.00
Monitoring complete after 5 ticks
```

**Golden rule:** every `loop` must have a path to exit via `break`. Otherwise it is an infinite loop — the program will hang!

## loop with a Counter and Statistics

In practice, a monitoring loop doesn't just watch prices — it collects statistics.

```rust
fn main() {
    let mut price = 50000.0_f64;
    let mut tick = 0_u32;
    let mut sum_price = 0.0_f64;   // for average price calculation
    let mut max_price = f64::MIN;  // session high
    let mut min_price = f64::MAX;  // session low

    // Simulated BTC price ticks
    let ticks = [50100.0, 49800.0, 50500.0, 49600.0, 51000.0,
                 50200.0, 49900.0, 50800.0, 49700.0, 50300.0];

    loop {
        price = ticks[tick as usize];
        tick += 1;

        // Update statistics
        sum_price += price;
        if price > max_price { max_price = price; }
        if price < min_price { min_price = price; }

        let avg_price = sum_price / tick as f64;

        println!("Tick {}: ${:.0} | Avg: ${:.0} | High: ${:.0} | Low: ${:.0}",
                 tick, price, avg_price, max_price, min_price);

        if tick as usize >= ticks.len() {
            println!("\n=== Session complete ===");
            println!("Ticks: {}", tick);
            println!("Average price: ${:.2}", avg_price);
            println!("Range: ${:.0} - ${:.0}", min_price, max_price);
            break;
        }
    }
}
```

## break: When to Exit the Loop

`break` can be triggered for different reasons. In a trading bot these are typically: take-profit hit, stop-loss hit, or an iteration limit reached.

```rust
fn main() {
    let entry_price = 42000.0;
    let take_profit = 44000.0;  // +4.76%
    let stop_loss = 40000.0;    // -4.76%
    let max_ticks = 100;        // safety limit

    let mut current_price = entry_price;
    let mut tick = 0;
    let mut reason = "unknown";

    // Simulated price ticks
    let price_changes = [200.0, -100.0, 300.0, -200.0, 400.0,
                         500.0, -300.0, 600.0, 700.0, 800.0];

    loop {
        // Simulate price movement
        if tick < price_changes.len() {
            current_price += price_changes[tick];
        }
        tick += 1;

        println!("Tick {}: ${:.0} (entry: ${:.0})", tick, current_price, entry_price);

        // Take-profit
        if current_price >= take_profit {
            reason = "TAKE-PROFIT";
            break;
        }

        // Stop-loss
        if current_price <= stop_loss {
            reason = "STOP-LOSS";
            break;
        }

        // Iteration limit (protection against infinite loop)
        if tick >= max_ticks {
            reason = "ITERATION LIMIT";
            break;
        }
    }

    let pnl = current_price - entry_price;
    println!("\nPosition closed: {}", reason);
    println!("PnL: ${:.2}", pnl);
}
```

## loop Returns a Value

In Rust, `loop` can return a value via `break value`. This is very handy for finding the first matching element.

```rust
fn find_entry_tick(prices: &[f64], target_price: f64) -> Option<usize> {
    let mut i = 0;
    loop {
        if i >= prices.len() {
            break None; // price never reached the target
        }

        if prices[i] <= target_price {
            break Some(i); // found! return the index
        }

        i += 1;
    }
}

fn main() {
    // BTC prices by hour
    let hourly_prices = vec![
        52000.0, 51500.0, 51000.0, 50500.0,
        50000.0, 49500.0, 49000.0, 48500.0,
    ];

    let target = 50000.0;

    let entry_tick = find_entry_tick(&hourly_prices, target);

    match entry_tick {
        Some(tick) => {
            println!("Price reached ${} at tick #{}", target, tick + 1);
            println!("Entry price: ${}", hourly_prices[tick]);
        }
        None => {
            println!("Price never reached target ${}", target);
        }
    }
}
```

Another example — finding the tick with maximum volume:

```rust
fn main() {
    let volumes = [1200.0, 1800.0, 950.0, 2500.0, 1100.0, 3100.0, 890.0];
    let mut i = 0;
    let mut max_vol_tick = 0;
    let mut max_vol = 0.0_f64;

    let result = loop {
        if volumes[i] > max_vol {
            max_vol = volumes[i];
            max_vol_tick = i;
        }
        i += 1;
        if i >= volumes.len() {
            break (max_vol_tick, max_vol); // return a tuple
        }
    };

    println!("Max volume: {:.0} at tick #{}", result.1, result.0 + 1);
}
```

## continue in loop

`continue` skips the rest of the current iteration and jumps straight to the next one. It's great for filtering invalid data — for example zero prices or prices outside a valid range.

```rust
fn main() {
    // Data with "broken" ticks (zeros = missing data points)
    let raw_prices = [
        42000.0, 0.0, 42100.0, 42050.0, 0.0,
        42200.0, 41900.0, 0.0, 42300.0, 42150.0,
    ];

    let mut valid_count = 0;
    let mut sum = 0.0;
    let mut i = 0;

    loop {
        if i >= raw_prices.len() { break; }

        let price = raw_prices[i];
        i += 1;

        // Skip invalid prices
        if price <= 0.0 {
            println!("Tick {}: skipped (invalid price)", i);
            continue; // jump to the next tick
        }

        // Process only valid prices
        valid_count += 1;
        sum += price;
        println!("Tick {}: ${:.0} (processing)", i, price);
    }

    if valid_count > 0 {
        println!("\nValid ticks: {}/{}", valid_count, raw_prices.len());
        println!("Average price: ${:.2}", sum / valid_count as f64);
    }
}
```

## Nested loop with Labels

When you need to break out of a nested loop all the way to an outer one — use labels. In trading this is useful when monitoring multiple exchanges or trading pairs.

```rust
fn main() {
    let exchanges = ["Binance", "Bybit", "OKX"];
    let pairs = ["BTC/USDT", "ETH/USDT", "SOL/USDT"];

    // Simulated prices: [exchange][pair]
    let prices = [
        [42000.0, 2200.0, 95.0],   // Binance
        [42010.0, 2195.0, 94.8],   // Bybit
        [41990.0, 2205.0, 95.2],   // OKX
    ];

    let target_btc = 42005.0; // looking for BTC at or below this price
    let mut found = false;

    'exchange_loop: for ex_idx in 0..exchanges.len() {
        println!("Checking exchange: {}", exchanges[ex_idx]);

        let mut pair_idx = 0;
        'pair_loop: loop {
            if pair_idx >= pairs.len() { break 'pair_loop; }

            let price = prices[ex_idx][pair_idx];
            println!("  {}: ${}", pairs[pair_idx], price);

            if pairs[pair_idx] == "BTC/USDT" && price <= target_btc {
                println!("  Found good BTC price on {}!", exchanges[ex_idx]);
                found = true;
                break 'exchange_loop; // break out of BOTH loops
            }

            pair_idx += 1;
        }
    }

    if !found {
        println!("No favourable price found");
    }
}
```

## Practical Example: Simple Trading Monitor

```rust
/// Simulates a trading monitor with a full monitoring loop
fn main() {
    println!("=== BTC/USDT Trading Monitor ===");

    // Position parameters
    let entry_price = 43000.0_f64;
    let take_profit = 45000.0_f64;   // TP: +4.65%
    let stop_loss = 41500.0_f64;     // SL: -3.49%
    let max_iterations = 20_u32;     // safety limit

    // Trading state
    let mut current_price = entry_price;
    let mut tick = 0_u32;
    let mut max_price = entry_price;
    let mut min_price = entry_price;

    // Simulated market movements
    let movements = [
        150.0, -80.0, 200.0, -120.0, 300.0,
        -200.0, 450.0, -100.0, 600.0, -300.0,
        800.0, -400.0, 1000.0, -500.0, 1200.0,
        -600.0, 1500.0, -700.0, 1800.0, -200.0,
    ];

    println!("Entry: ${:.0} | TP: ${:.0} | SL: ${:.0}\n",
             entry_price, take_profit, stop_loss);

    let exit_reason = loop {
        if tick as usize < movements.len() {
            current_price += movements[tick as usize];
        }
        tick += 1;

        // Update extremes
        if current_price > max_price { max_price = current_price; }
        if current_price < min_price { min_price = current_price; }

        let pnl = current_price - entry_price;
        let pnl_pct = (pnl / entry_price) * 100.0;

        println!("Tick {:02}: ${:.0} | PnL: {:+.0}$ ({:+.2}%)",
                 tick, current_price, pnl, pnl_pct);

        // Check exit conditions
        if current_price >= take_profit {
            break "TAKE-PROFIT";
        }
        if current_price <= stop_loss {
            break "STOP-LOSS";
        }
        if tick >= max_iterations {
            break "ITERATION LIMIT";
        }
    };

    // Position summary
    let final_pnl = current_price - entry_price;
    let final_pnl_pct = (final_pnl / entry_price) * 100.0;

    println!("\n=== Position closed: {} ===", exit_reason);
    println!("Exit price: ${:.0}", current_price);
    println!("PnL: {:+.2}$ ({:+.2}%)", final_pnl, final_pnl_pct);
    println!("Session high: ${:.0}", max_price);
    println!("Session low: ${:.0}", min_price);
    println!("Ticks processed: {}", tick);
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `loop { }` | Infinite loop when number of iterations is unknown | Real-time price monitoring |
| `break` | Exit the loop on a condition | TP/SL triggered |
| `break value` | Exit and return a value | `break entry_price` |
| `continue` | Skip the current iteration | Skipping invalid ticks |
| `'label: loop` | Label for nested loops | Monitoring multiple exchanges |
| `break 'label` | Break out of a named loop | Breaking out of the outer loop |
| Iteration counter | Protection against infinite loops | `if tick >= max_ticks { break; }` |

## Homework

1. Write a BTC price monitor that:
   - Simulates 10 ticks with arbitrary price changes
   - Exits the loop when the take-profit or stop-loss is hit
   - After exit, prints the final PnL in USD and as a percentage

2. Use `break value` to find the first profitable trade:
   - Given a PnL array: `[-50.0, -30.0, 80.0, -20.0, 120.0]`
   - Find the index of the first profitable trade via `loop` + `break Some(i)`
   - Print "First profit on trade #N: $X"

3. Add `continue` for data filtering:
   - Given a volume array with zeros (missing data): `[1200.0, 0.0, 1500.0, 0.0, 1800.0]`
   - Use `continue` to skip zero values
   - Calculate the average only over valid data points

4. Create nested loops with labels:
   - Outer loop: 3 trading pairs (BTC, ETH, SOL)
   - Inner loop: 5 price ticks for each
   - If BTC price exceeds 45000 — break out of both loops using a label

## Navigation

[← Previous day](../018-else-and-else-if-market-scenarios/en.md) | [Next day →](../020-while-waiting-for-target-price/en.md)
