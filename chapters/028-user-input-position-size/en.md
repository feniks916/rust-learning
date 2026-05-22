# Day 28: User Input — Trader Enters Position Size

## Trading Analogy

A trading terminal accepts trader input: position size, entry price, ticker. In Rust, `std::io::stdin()` is the terminal keyboard: read what the user types and use it in the program.

## Reading a String from stdin

```rust
use std::io;

fn main() {
    println!("Enter ticker (e.g., BTC):");

    let mut ticker = String::new();
    io::stdin()
        .read_line(&mut ticker)
        .expect("Read error");

    let ticker = ticker.trim(); // remove newline
    println!("You selected: {}", ticker);
}
```

## Reading a Number

```rust
use std::io;

fn main() {
    println!("Enter position size ($):");

    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("Read error");

    let size: f64 = input.trim()
        .parse()
        .expect("Enter a number");

    println!("Position size: ${:.2}", size);

    let fee = size * 0.001;
    println!("Fee: ${:.2}", fee);
}
```

## Multiple Input Fields

```rust
use std::io;

fn read_line() -> String {
    let mut s = String::new();
    io::stdin().read_line(&mut s).expect("Error");
    s.trim().to_string()
}

fn main() {
    println!("=== Position Calculator ===");

    println!("Ticker:");
    let ticker = read_line();

    println!("Entry price ($):");
    let entry: f64 = read_line().parse().expect("Number");

    println!("Quantity:");
    let qty: f64 = read_line().parse().expect("Number");

    let total = entry * qty;
    println!("\n{}: {} @ ${:.2} = ${:.2}", ticker, qty, entry, total);
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `io::stdin().read_line(&mut s)` | Read a line from keyboard |
| `.trim()` | Remove whitespace and `\n` |
| `.parse::<f64>()` | Convert string to number |
| `.expect("msg")` | Panic with message on error |

## Homework

1. Write a program that asks for balance and risk percent, outputs stop-loss size
2. Handle parse error with `match` instead of `expect`
3. Create `read_f64(prompt: &str) -> f64` that prints prompt and reads number
4. Write a mini PnL calculator: enter entry price, exit price, and quantity

## Navigation

[← Previous day](../027-shadowing-updating-price/en.md) | [Next day →](../029-string-parsing-input-to-number/en.md)
