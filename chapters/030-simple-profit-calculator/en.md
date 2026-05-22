# Day 30: Simple Profit Calculator

## Trading Analogy

After closing a trade, a trader calculates: how much was bought, at what price, at what exit price, what was the fee, and what was the net profit. We'll write exactly that program.

## Basic PnL Calculation

```rust
fn calculate_pnl(
    entry: f64,
    exit: f64,
    quantity: f64,
    fee_rate: f64,
) -> (f64, f64, f64) {
    let gross_pnl = (exit - entry) * quantity;
    let entry_fee = entry * quantity * fee_rate;
    let exit_fee  = exit  * quantity * fee_rate;
    let net_pnl   = gross_pnl - entry_fee - exit_fee;
    (gross_pnl, entry_fee + exit_fee, net_pnl)
}

fn main() {
    let (gross, fees, net) = calculate_pnl(50000.0, 55000.0, 0.1, 0.001);
    println!("Gross profit: ${:.2}", gross);
    println!("Fees:         ${:.2}", fees);
    println!("Net profit:   ${:.2}", net);
}
```

## Calculator with Percentages

```rust
fn trade_report(entry: f64, exit: f64, qty: f64) {
    let pnl = (exit - entry) * qty;
    let pct = (exit - entry) / entry * 100.0;
    let invested = entry * qty;

    println!("━━━━━━ Trade Report ━━━━━━");
    println!("Entry:     ${:.2} × {}", entry, qty);
    println!("Exit:      ${:.2}", exit);
    println!("Invested:  ${:.2}", invested);
    println!("PnL:       ${:.2} ({:+.2}%)", pnl, pct);
    println!("Result:    {}", if pnl > 0.0 { "PROFIT ✓" } else { "LOSS ✗" });
}

fn main() {
    trade_report(48000.0, 52000.0, 0.2);
    trade_report(50000.0, 47000.0, 0.1);
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| `(exit - entry) * qty` | Basic PnL formula |
| Tuple return | Return multiple values |
| `.filter()` | Filter an iterator |
| `{:+.2}` | Format with sign |

## Homework

1. Add `is_long: bool` to `calculate_pnl` (Long: exit-entry, Short: entry-exit)
2. Write `best_trade(trades: &[(f64, f64, f64)]) -> usize` returning best trade index
3. Calculate Sharpe ratio: mean PnL / std deviation of PnL
4. Print a report for all trades with total PnL and win rate

## Navigation

[← Previous day](../029-string-parsing-input-to-number/en.md) | [Next day →](../031-project-position-size-calculator/en.md)
