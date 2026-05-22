# Day 38: Borrowing — Temporary Data Access

## Trading Analogy

You give a broker temporary access to your account to execute an order. The broker can view and act, but the account is still yours. In Rust this is called borrowing.

## Rules of Borrowing

1. Any number of immutable references (`&T`) simultaneously
2. OR one mutable reference (`&mut T`)
3. But **not both** at the same time

```rust
fn main() {
    let portfolio = vec![50000.0, 3000.0];

    let r1 = &portfolio; // OK: first immutable reference
    let r2 = &portfolio; // OK: second immutable reference
    println!("{:?} {:?}", r1, r2);

    // let r3 = &mut portfolio; // ERROR: cannot mix
}
```

## Borrowing Through Functions

```rust
fn analyze(prices: &Vec<f64>) -> f64 {
    let sum: f64 = prices.iter().sum();
    sum / prices.len() as f64
}

fn main() {
    let prices = vec![48000.0, 50000.0, 52000.0];
    let avg = analyze(&prices); // borrow
    println!("Average: ${:.2}", avg);
    println!("prices untouched: {:?}", prices); // we're still owner
}
```

## What We Learned

| Rule | Description |
|------|-------------|
| Many `&T` simultaneously | OK — read only |
| One `&mut T` | OK — mutation |
| `&T` and `&mut T` together | ERROR |
| Borrow is time-limited | Reference scope |

## Homework

1. Create a Vec, make two immutable references simultaneously and use both
2. Try creating &mut and & at the same time — read the compiler error
3. Write `report(trades: &[f64])` that borrows and prints statistics
4. Show that after the reference scope ends, you can mutate data again

## Navigation

[← Previous day](../037-references-looking-at-portfolio/en.md) | [Next day →](../039-mutable-references-editing-order/en.md)
