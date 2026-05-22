# Day 263: Risk Management — Position Sizing

## Trading Analogy

A professional trader never puts their entire deposit on one trade. Risk management determines: how much can be lost on a single trade, and how to calculate position size to respect that limit.

## Basic Formula

```
Position size = (Balance × Risk%) / Stop-loss $
Stop-loss $ = Entry price × Stop%
```

## Implementation

```rust
#[derive(Debug)]
struct RiskParams {
    balance: f64,
    risk_pct: f64,      // fraction of balance to risk (e.g. 0.02 = 2%)
    stop_loss_pct: f64, // stop-loss as fraction of entry price
}

fn calculate_position(entry: f64, p: &RiskParams) -> (f64, f64, f64, f64) {
    let max_loss  = p.balance * p.risk_pct;
    let pos_value = max_loss / p.stop_loss_pct;
    let quantity  = pos_value / entry;
    let stop      = entry * (1.0 - p.stop_loss_pct);
    (pos_value, quantity, stop, max_loss)
}

fn main() {
    let params = RiskParams {
        balance: 10_000.0,
        risk_pct: 0.02,
        stop_loss_pct: 0.05,
    };

    let entry = 50000.0;
    let (pos_val, qty, stop_price, max_loss) = calculate_position(entry, &params);

    println!("=== Position Sizing ===");
    println!("Balance:     ${:.2}", params.balance);
    println!("Max loss:    ${:.2}", max_loss);
    println!("Entry:       ${:.2}", entry);
    println!("Stop-loss:   ${:.2}", stop_price);
    println!("Position:    ${:.2}", pos_val);
    println!("Quantity:    {:.6}", qty);
}
```

## Kelly Criterion

```rust
fn kelly_fraction(win_rate: f64, avg_win: f64, avg_loss: f64) -> f64 {
    let b = avg_win / avg_loss;
    let p = win_rate;
    let q = 1.0 - win_rate;
    (b * p - q) / b
}

fn main() {
    let kelly = kelly_fraction(0.55, 200.0, 100.0);
    println!("Kelly fraction: {:.2}%", kelly * 100.0);
    println!("Half-Kelly (safer): {:.2}%", kelly * 50.0);
}
```

## What We Learned

| Formula | Description |
|---------|-------------|
| `balance × risk%` | Max loss per trade |
| `max_loss / stop%` | Position size |
| `position / entry` | Number of units |
| Kelly Criterion | Optimal fraction of balance |

## Homework

1. Write `position_size(balance, risk_pct, entry, stop_pct) -> (f64, f64, f64)` — (value, qty, stop)
2. Add check: if position > 20% of balance — auto-reduce risk
3. Add R/R ratio calculation including take-profit
4. Implement Kelly Criterion and compare with fixed 2% risk

## Navigation

[← Previous day](../262-strategy-breakout/en.md) | [Next day →](../264-stop-loss-limiting-losses/en.md)
