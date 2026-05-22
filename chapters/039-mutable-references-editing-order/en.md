# Day 39: Mutable References — Editing Someone's Order

## Trading Analogy

Sometimes a broker needs not just to view an order, but to change it — like adjusting the price. A mutable reference `&mut T` provides that access: modify data without becoming the owner.

## &mut T: Mutable Reference

```rust
fn apply_slippage(price: &mut f64, slippage: f64) {
    *price *= 1.0 + slippage;
}

fn main() {
    let mut entry_price = 50000.0;
    println!("Before: ${}", entry_price);

    apply_slippage(&mut entry_price, 0.001);
    println!("After slippage: ${:.2}", entry_price);
}
```

## Modifying Vec via &mut

```rust
fn add_fee_to_all(prices: &mut Vec<f64>, fee: f64) {
    for price in prices.iter_mut() {
        *price += fee;
    }
}

fn main() {
    let mut prices = vec![50000.0, 51000.0, 52000.0];
    println!("Before: {:?}", prices);
    add_fee_to_all(&mut prices, 50.0);
    println!("After: {:?}", prices);
}
```

## Only One &mut at a Time

```rust
fn main() {
    let mut order = String::from("BUY BTC");

    let r1 = &mut order;
    // let r2 = &mut order; // ERROR: second &mut not allowed

    r1.push_str(" 0.1");
    println!("{}", r1);
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `&mut T` | Mutable reference |
| `&mut variable` | Create a mutable reference |
| `*ref = value` | Modify through dereference |
| Only one &mut | Exclusive access rule |

## Homework

1. Write `reset_balance(b: &mut f64)` that sets balance to 10000.0
2. Write `mark_winning(trades: &mut Vec<f64>)` that doubles profitable trades
3. Try creating two &mut simultaneously — study the compiler error
4. Write `scale_prices(prices: &mut [f64], factor: f64)` multiplying all prices

## Navigation

[← Previous day](../038-borrowing-temporary-access/en.md) | [Next day →](../040-one-mutable-reference-rule/en.md)
