# Day 36: Copy — Light Types Like Cash

## Trading Analogy

Cash passes from hand to hand easily — and both parties keep a copy (imagine a duplicate). Types implementing the `Copy` trait work the same: assignment creates a copy automatically.

## Copy Types

```rust
fn main() {
    // All these types implement Copy:
    let price: f64 = 50000.0;
    let qty: u32 = 10;
    let is_long: bool = true;

    let price2 = price;  // copy, not Move
    let qty2   = qty;

    println!("price={}, price2={}", price, price2); // both accessible
    println!("qty={}, qty2={}", qty, qty2);
}
```

## Which Types are Copy

```rust
// Copy: i8, i16, i32, i64, i128, u8..u128, f32, f64, bool, char
// Also: tuples of Copy types, arrays of Copy types with fixed size
// NOT Copy: String, Vec<T>, Box<T>
```

## Copy in Functions

```rust
fn double_price(p: f64) -> f64 {
    p * 2.0
}

fn main() {
    let price = 50000.0;
    let doubled = double_price(price); // price is copied, not moved
    println!("Original: {}", price);   // OK!
    println!("Doubled: {}", doubled);
}
```

## Implementing Copy for Custom Types

```rust
#[derive(Debug, Copy, Clone)] // Copy requires Clone
struct Price {
    value: f64,
    decimals: u8,
}

fn main() {
    let btc = Price { value: 50000.0, decimals: 2 };
    let btc2 = btc; // Copy, not Move
    println!("btc: {:?}", btc);
    println!("btc2: {:?}", btc2);
}
```

## What We Learned

| Type | Copy? |
|------|-------|
| i8..i128, u8..u128 | ✓ |
| f32, f64 | ✓ |
| bool, char | ✓ |
| String, Vec, Box | ✗ |
| Tuples of Copy types | ✓ |

## Homework

1. Confirm f64 is Copy: assign to another variable and use both
2. Create `Tick { price: f64, volume: f64 }` with `#[derive(Copy, Clone)]`
3. Write a function taking f64, show original is accessible after call
4. Explain why String cannot be Copy (hint: heap and memory)

## Navigation

[← Previous day](../035-clone-copying-portfolio/en.md) | [Next day →](../037-references-looking-at-portfolio/en.md)
