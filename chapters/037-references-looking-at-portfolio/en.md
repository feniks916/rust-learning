# Day 37: References — Looking at Someone's Portfolio

## Trading Analogy

An analyst can look at a client's portfolio without taking the assets. A reference in Rust is exactly that — access to data without transferring ownership.

## References: &T

```rust
fn print_portfolio(portfolio: &Vec<String>) {
    // Look at portfolio, don't own it
    for asset in portfolio {
        println!("  {}", asset);
    }
}

fn main() {
    let my_portfolio = vec![
        String::from("BTC"),
        String::from("ETH"),
        String::from("SOL"),
    ];

    print_portfolio(&my_portfolio); // pass a reference
    println!("Portfolio still mine: {} assets", my_portfolio.len());
}
```

## References Don't Transfer Ownership

```rust
fn total_value(prices: &Vec<f64>) -> f64 {
    prices.iter().sum()
}

fn main() {
    let prices = vec![50000.0, 3000.0, 150.0];
    let total = total_value(&prices); // reference, not Move
    println!("Sum: ${:.2}", total);
    println!("prices still here: {:?}", prices); // OK!
}
```

## Dereferencing: *

```rust
fn add_fee(price: &f64, fee: f64) -> f64 {
    *price + fee // * dereferences the reference
}

fn main() {
    let price = 50000.0;
    let with_fee = add_fee(&price, 50.0);
    println!("With fee: ${}", with_fee);
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `&T` | Immutable reference to T |
| `&variable` | Create a reference |
| `*reference` | Dereference |
| `&[T]` | Reference to a slice |

## Homework

1. Write `count_wins(trades: &Vec<f64>) -> usize` — count profitable trades
2. Write `average(prices: &[f64]) -> f64` — arithmetic mean
3. Show that after passing &Vec the caller retains access to Vec
4. Write a function with multiple reference arguments: `compare(a: &f64, b: &f64) -> bool`

## Navigation

[← Previous day](../036-copy-light-types-cash/en.md) | [Next day →](../038-borrowing-temporary-access/en.md)
