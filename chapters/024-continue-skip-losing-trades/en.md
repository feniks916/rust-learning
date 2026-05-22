# Day 24: continue — Skip Losing Trades in Report

## Trading Analogy

Imagine you're compiling a monthly report and want to analyze only profitable trades — ignoring those that closed in the red. You scan the trade log: if a trade is a loss, you skip it and look at the next one. That's exactly what `continue` does: it skips the rest of the current iteration and immediately jumps to the next one. Unlike `break`, it doesn't stop the loop — it just skips over one entry. In trading algorithms this is indispensable for data filtering.

## Basic continue: Filtering Losing Trades

The simplest case — iterate over trades and print only profitable ones:

```rust
fn main() {
    let trades = [
        ("BTC",  150.0),
        ("ETH",  -50.0),
        ("SOL",  200.0),
        ("BNB",  -30.0),
        ("ADA",   80.0),
        ("DOT",  -10.0),
        ("AVAX", 120.0),
    ];

    println!("Profitable trades report:");
    println!("{:-<40}", "");

    let mut profit_count = 0;
    let mut total_profit = 0.0;

    for (ticker, pnl) in &trades {
        if *pnl < 0.0 {
            continue; // skip losing trades
        }

        profit_count += 1;
        total_profit += pnl;
        println!("{:>6}: +${:.2}", ticker, pnl);
    }

    println!("{:-<40}", "");
    println!("Profitable trades: {}", profit_count);
    println!("Total profit: +${:.2}", total_profit);
}
```

After `continue`, Rust skips `profit_count += 1` and `println!` for losing trades — jumping straight to the next element. The code stays flat: no nested `if { ... }` wrapping all the processing logic.

## Filtering on Multiple Conditions

Often you need to filter out several types of records — exclude both losing trades and those with small volume and wrong direction:

```rust
fn main() {
    let trades = vec![
        // (ticker, pnl, volume, side)
        ("BTC",  250.0, 5000.0, "long"),
        ("ETH",  -80.0, 3000.0, "short"),  // loss
        ("SOL",  120.0,  200.0, "long"),   // small volume
        ("BNB",  -20.0, 1500.0, "long"),   // loss
        ("ADA",   45.0, 2500.0, "short"),  // wrong side
        ("DOT",  310.0, 8000.0, "long"),
    ];

    let min_volume  = 1000.0;
    let target_side = "long";

    println!("Quality long trades:");
    let mut skipped   = 0;
    let mut processed = 0;

    for &(ticker, pnl, volume, side) in &trades {
        if pnl < 0.0 {
            println!("  Skip {} — loss ${:.2}", ticker, pnl);
            skipped += 1;
            continue; // filter 1
        }
        if volume < min_volume {
            println!("  Skip {} — volume {:.0} too small", ticker, volume);
            skipped += 1;
            continue; // filter 2
        }
        if side != target_side {
            println!("  Skip {} — side '{}'", ticker, side);
            skipped += 1;
            continue; // filter 3
        }

        // Only quality trades reach here
        processed += 1;
        println!("  ✓ {} [{}]: +${:.2} (vol {:.0})", ticker, side, pnl, volume);
    }

    println!("\nProcessed: {}, skipped: {}", processed, skipped);
}
```

Each `continue` is a separate "entry filter." This structure is called "guard clauses." The code doesn't drift rightward with deep indentation — it stays linear.

## continue in while: Skipping Non-Trading Hours

`continue` works great in a `while` loop when simulating a trading schedule:

```rust
fn is_trading_hour(hour: u32) -> bool {
    // Trade only 9-12 and 14-17, lunch break 12-14
    (hour >= 9 && hour < 12) || (hour >= 14 && hour < 17)
}

fn main() {
    let mut hour = 8;
    let end_hour = 20;
    let mut signals_checked = 0;
    let mut hours_skipped   = 0;

    while hour <= end_hour {
        if !is_trading_hour(hour) {
            println!("Hour {:02}: not a trading session — skip", hour);
            hours_skipped += 1;
            hour += 1;
            continue; // move to the next hour
        }

        // This code runs only during trading hours
        signals_checked += 1;
        println!("Hour {:02}: analyzing market (signal #{})", hour, signals_checked);

        hour += 1;
    }

    println!("\nSignals checked: {}", signals_checked);
    println!("Non-trading hours skipped: {}", hours_skipped);
}
```

Important: in a `while` loop you must update the counter (`hour += 1`) before `continue`, otherwise the loop will run forever — `continue` doesn't execute the code below it, and the `while` condition won't change on its own.

## continue with a Label for Nested Loops

Just like `break`, `continue` supports labels for controlling outer loops:

```rust
fn main() {
    let portfolio = vec![
        ("BTC", vec![ 200.0, -50.0, 300.0, -100.0,  150.0]),
        ("ETH", vec![ -30.0, -80.0, -20.0,  -60.0,  -10.0]),  // all losses
        ("SOL", vec![  50.0,  80.0, -10.0,  120.0,   90.0]),
    ];

    println!("Portfolio analysis (skip entirely-losing assets):");
    println!();

    'asset: for (ticker, trades) in &portfolio {
        let has_profit = trades.iter().any(|&t| t > 0.0);

        if !has_profit {
            println!("{}: all trades losing — skipping asset", ticker);
            continue 'asset; // move to the next asset entirely
        }

        println!("{}:", ticker);
        for (i, &pnl) in trades.iter().enumerate() {
            if pnl < 0.0 {
                continue; // skip losing trade inside the asset
            }
            println!("  Trade {}: +${:.2}", i + 1, pnl);
        }
        let total: f64 = trades.iter().filter(|&&t| t > 0.0).sum();
        println!("  Total profit: +${:.2}\n", total);
    }
}
```

`continue 'asset` skips all remaining code for the current asset and moves to the next one. Without the label, the inner `continue` would skip only the current trade.

## skip/process Statistics: Count What Was Skipped and Why

Best practice — always track skip counts by category:

```rust
use std::collections::HashMap;

fn main() {
    let trades: Vec<(&str, f64, f64, bool)> = vec![
        // (ticker, pnl, volume, is_valid)
        ("BTC",  500.0, 10000.0, true),
        ("ETH",  -80.0,  3000.0, true),   // loss
        ("SOL",  120.0,   150.0, true),   // small volume
        ("BNB",  200.0,  5000.0, false),  // invalid
        ("BTC",  500.0, 10000.0, true),   // duplicate
        ("ADA",  350.0,  8000.0, true),
        ("DOT", -100.0,  4000.0, true),   // loss
        ("AVAX", 180.0,  6000.0, true),
    ];

    let min_volume = 1000.0;
    let mut processed = 0;
    let mut skip_counts: HashMap<&str, u32> = HashMap::new();
    let mut seen = std::collections::HashSet::new();

    println!("Trade record filtering:");

    for &(ticker, pnl, volume, valid) in &trades {
        if !valid {
            *skip_counts.entry("invalid").or_insert(0) += 1;
            continue;
        }
        if pnl < 0.0 {
            *skip_counts.entry("loss").or_insert(0) += 1;
            continue;
        }
        if volume < min_volume {
            *skip_counts.entry("low_volume").or_insert(0) += 1;
            continue;
        }
        if seen.contains(ticker) {
            *skip_counts.entry("duplicate").or_insert(0) += 1;
            continue;
        }

        seen.insert(ticker);
        processed += 1;
        println!("  ✓ {}: +${:.2} (vol {:.0})", ticker, pnl, volume);
    }

    println!("\nSummary:");
    println!("  Accepted: {}", processed);
    let total_skipped: u32 = skip_counts.values().sum();
    for (reason, count) in &skip_counts {
        println!("  Skipped [{}]: {}", reason, count);
    }
    println!("  Total skipped: {}", total_skipped);
}
```

Category counters let you later build a report: "5% of data filtered for errors, 12% for low volume."

## continue for Cleaning Dirty Quotes

In real systems data can be corrupt — zeroes, NaN, outliers:

```rust
fn main() {
    let raw_prices: Vec<f64> = vec![
        50000.0, 0.0, 51000.0, f64::NAN, 52000.0,
        -100.0, 53000.0, f64::INFINITY, 54000.0,
    ];

    let mut clean = Vec::new();
    let mut bad = 0;

    for (i, &price) in raw_prices.iter().enumerate() {
        if price <= 0.0 || price.is_nan() || price.is_infinite() {
            println!("Tick {}: invalid ({:?}) — skip", i + 1, price);
            bad += 1;
            continue;
        }

        if let Some(&prev) = clean.last() {
            let pct = (price - prev).abs() / prev * 100.0;
            if pct > 20.0 {
                println!("Tick {}: outlier {:.1}% — skip", i + 1, pct);
                bad += 1;
                continue;
            }
        }

        clean.push(price);
        println!("Tick {}: ${:.2} ✓", i + 1, price);
    }

    println!("\nClean prices: {}, skipped: {}", clean.len(), bad);
    if !clean.is_empty() {
        let avg = clean.iter().sum::<f64>() / clean.len() as f64;
        println!("Average price: ${:.2}", avg);
    }
}
```

## Practical Example: Trading Session Statistics

Bringing it all together — filter session trades and build a full report:

```rust
fn main() {
    let session: Vec<(&str, f64, f64, bool)> = vec![
        // (ticker, entry, exit, valid)
        ("BTC",  50000.0, 52000.0, true),
        ("ETH",   3000.0,  2900.0, true),   // loss
        ("SOL",    100.0,     0.0, false),  // invalid
        ("BNB",    280.0,   320.0, true),
        ("ADA",      0.5,     0.0, false),  // invalid
        ("BTC",  52000.0, 54500.0, true),
        ("DOT",      8.0,     7.2, true),   // loss
        ("AVAX",    18.0,    22.0, true),
    ];

    let fee = 0.001; // 0.1% commission
    let mut wins     = 0;
    let mut losses   = 0;
    let mut total_pnl = 0.0;
    let mut skipped  = 0;

    println!("Trading session — detailed report:");
    println!("{:=<55}", "");

    for &(ticker, entry, exit, valid) in &session {
        if !valid || entry <= 0.0 || exit <= 0.0 {
            println!("  SKIP {:>6}: invalid data", ticker);
            skipped += 1;
            continue;
        }

        let gross = exit - entry;
        let comm  = (entry + exit) * fee;
        let net   = gross - comm;

        if net >= 0.0 {
            wins += 1;
            println!("  WIN  {:>6}: ${:.2} → ${:.2} | P&L +${:.4}", ticker, entry, exit, net);
        } else {
            losses += 1;
            println!("  LOSS {:>6}: ${:.2} → ${:.2} | P&L  ${:.4}", ticker, entry, exit, net);
        }
        total_pnl += net;
    }

    println!("{:=<55}", "");
    let total = wins + losses;
    let wr = if total > 0 { wins as f64 / total as f64 * 100.0 } else { 0.0 };
    println!("Trades:   {} (skipped: {})", total, skipped);
    println!("Win/Loss: {} / {}", wins, losses);
    println!("Win rate: {:.1}%", wr);
    println!("Net P&L:  ${:.4}", total_pnl);
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `continue` | Skip current iteration, move to next | Losing trade → skip in report |
| `continue` in `while` | Skip with required counter update before `continue` | Non-trading hour → `hour += 1; continue` |
| Guard clauses (chain of continue) | Flat filter structure instead of nested `if` | Loss → continue; low volume → continue |
| `continue 'label` | Skip current iteration of an outer loop | Skip entire asset if all trades are losers |
| `continue` + HashMap counter | Count skipped records by category | `skip_counts["loss"] += 1; continue` |
| `continue` for data cleaning | Skip NaN, zeros, anomalous spikes | `price.is_nan() → continue` |

## Homework

1. Write a `filter_quality_trades` function taking a slice `&[(ticker, pnl, volume)]` and returning only records that pass all filters
   - Filter 1: `pnl > 0`
   - Filter 2: `volume > 500`
   - Filter 3: `ticker` is not empty
   - Count skips per reason with a `HashMap`

2. Implement a quote cleaner: takes `Vec<f64>`, returns a clean vector
   - Skip via `continue`: zero, negative, NaN, infinite, outliers > 15%
   - Print a detailed log for each skip

3. Write a nested loop over a portfolio (assets → days) with labels
   - If an asset is overall losing — `continue 'asset`
   - If a day is losing — `continue 'day`
   - For profitable days — print details

4. Create a weekly report generator: 5 days × N trades per day
   - `continue` to skip non-trading days (weekends)
   - `continue` to skip losing trades
   - At the end — summary: total trades, accepted, skipped, total P&L

## Navigation

[← Previous day](../023-break-exit-at-take-profit/en.md) | [Next day →](../025-match-determining-order-type/en.md)
