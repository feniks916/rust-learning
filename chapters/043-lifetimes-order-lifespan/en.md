# Day 43: Lifetimes — How Long Does an Order Live

## Trading Analogy

Every order has a lifetime: from creation to execution or cancellation. In Rust, a lifetime determines how long a reference remains valid.

## What is a Lifetime

Lifetime is checked **at compile time**. There is no runtime overhead.

```rust
// Explicit annotation: 'a means "reference lives at least as long as 'a"
fn first_price<'a>(prices: &'a [f64]) -> &'a f64 {
    &prices[0]
}

fn main() {
    let daily = vec![48000.0, 50000.0, 52000.0];
    let first = first_price(&daily);
    println!("First price: {}", first);
}
```

## Compiler Tracks Lifetimes

```rust
fn longest_ticker<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}

fn main() {
    let t1 = String::from("BTCUSDT");
    {
        let t2 = String::from("ETH");
        let result = longest_ticker(&t1, &t2);
        println!("Longer: {}", result); // OK: used within scope
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Lifetime | Region where a reference is valid |
| 'a | Lifetime annotation |
| Checked at compile time | No runtime cost |
| Goal | Prevent dangling references |

## Homework

1. Write `max_price<'a>(prices: &'a [f64]) -> &'a f64` returning reference to max
2. Experiment with variable lifetimes using {} blocks
3. See what happens when reference is used after data is destroyed (compiler error)
4. Write `compare<'a>(a: &'a str, b: &'a str) -> &'a str` returning lexicographically first

## Navigation

[← Previous day](../042-dangling-references/en.md) | [Next day →](../044-lifetime-annotations/en.md)
