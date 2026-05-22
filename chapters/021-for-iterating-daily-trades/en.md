# Day 21: for — Iterating Through Daily Trades

## Trading Analogy

At the end of the trading day, an analyst reviews all executed trades. The `for` loop in Rust is like going through each entry in a trading journal: convenient, safe, and no chance of going out of bounds.

## for Over an Array

```rust
fn main() {
    let trades = [100.0, -50.0, 200.0, -30.0, 150.0];
    let mut total_pnl = 0.0;

    for trade in trades {
        total_pnl += trade;
        println!("Trade: {:.2}, Total: {:.2}", trade, total_pnl);
    }

    println!("Daily PnL: {:.2}", total_pnl);
}
```

## for Over a Vec

```rust
fn main() {
    let tickers = vec!["BTC", "ETH", "SOL", "BNB"];

    for ticker in &tickers {
        println!("Analyzing {}", ticker);
    }

    // tickers still available here
    println!("Total tickers: {}", tickers.len());
}
```

## for with Index via enumerate

```rust
fn main() {
    let prices = vec![48000.0, 49000.0, 50000.0, 51000.0];

    for (i, price) in prices.iter().enumerate() {
        println!("Candle {}: ${:.2}", i + 1, price);
    }
}
```

## for Over a Range

```rust
fn main() {
    println!("Trading hours:");
    for hour in 9..18 {
        println!("  {:02}:00 — market open", hour);
    }

    println!("\nCountdown to close:");
    for i in (1..=5).rev() {
        println!("  {} seconds", i);
    }
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `for item in collection` | Iterate over collection |
| `for item in &collection` | Iterate by reference (collection remains) |
| `for (i, item) in collection.iter().enumerate()` | Iterate with index |
| `for i in 0..10` | Iterate over range |

## Homework

1. Create a vector of 7 daily PnL values and calculate total weekly income with for
2. Use enumerate to print each trade with its sequence number
3. Write for over range 1..=365 and print every 30 iterations (month)
4. Write for with &prices so prices remains accessible after the loop

## Navigation

[← Previous day](../020-while-waiting-for-target-price/en.md) | [Next day →](../022-range-prices/en.md)
