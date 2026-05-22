# Day 31: Project — Position Size Calculator

## About the Project

Month one is done! Today we combine everything learned into one complete project: an interactive position size calculator accounting for risk, stop-loss, and deposit size.

## Formula

```
Position size = (Balance × Risk%) / Stop-loss%
```

## Project Code

```rust
use std::io;

fn read_f64(prompt: &str) -> f64 {
    println!("{}", prompt);
    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("Read error");
    input.trim().parse().expect("Enter a number")
}

fn position_size(balance: f64, risk_pct: f64, stop_pct: f64) -> f64 {
    (balance * risk_pct / 100.0) / (stop_pct / 100.0)
}

fn trade_levels(entry: f64, stop_pct: f64, target_pct: f64) -> (f64, f64) {
    let stop   = entry * (1.0 - stop_pct / 100.0);
    let target = entry * (1.0 + target_pct / 100.0);
    (stop, target)
}

fn main() {
    println!("╔══════════════════════════════════╗");
    println!("║  Position Size Calculator        ║");
    println!("╚══════════════════════════════════╝");

    let balance    = read_f64("Balance ($):");
    let risk_pct   = read_f64("Risk per trade (%):");
    let entry      = read_f64("Entry price ($):");
    let stop_pct   = read_f64("Stop-loss (%):");
    let target_pct = read_f64("Take-profit (%):");

    let size = position_size(balance, risk_pct, stop_pct);
    let (stop_price, target_price) = trade_levels(entry, stop_pct, target_pct);
    let qty = size / entry;
    let risk_amount = balance * risk_pct / 100.0;
    let reward = qty * (target_price - entry);
    let rr_ratio = reward / risk_amount;

    println!("\n━━━━━━━━ Result ━━━━━━━━");
    println!("Position size: ${:.2}", size);
    println!("Quantity:      {:.6}", qty);
    println!("Stop-loss:     ${:.2}", stop_price);
    println!("Take-profit:   ${:.2}", target_price);
    println!("Risk:          ${:.2}", risk_amount);
    println!("Potential:     ${:.2}", reward);
    println!("R/R ratio:     {:.2}", rr_ratio);
}
```

## What We Learned This Month

| Topic | Application |
|-------|-------------|
| Variables and types | Storing prices, volumes |
| Functions | Calculating PnL, position size |
| Conditions | Trading signals |
| Loops | Processing trade series |
| match | Order types |
| User input | Interactive interface |

## Homework

1. Add Long/Short selection and compute levels correctly for Short
2. Add max loss calculation as % of balance
3. Add warning if R/R < 1.5 (poor ratio)
4. Save calculation results to a text file

## Navigation

[← Previous day](../030-simple-profit-calculator/en.md) | [Next day →](../032-stack-and-heap/en.md)
