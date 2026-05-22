# Day 41: Rule — No Mixing References

## Trading Analogy

Imagine one trader reading a position while another modifies it simultaneously. The first gets inconsistent data. Rust prevents this: you cannot have &T and &mut T at the same time.

## Cannot Mix: & and &mut

```rust
fn main() {
    let mut portfolio = vec![50000.0, 3000.0];

    let reader = &portfolio;      // immutable reference
    // let writer = &mut portfolio; // ERROR: conflicts with reader

    println!("{:?}", reader);
    // reader no longer needed after this

    let writer = &mut portfolio;  // now OK
    writer.push(1500.0);
    println!("{:?}", portfolio);
}
```

## NLL Resolves Most Issues

```rust
fn main() {
    let mut prices = vec![48000.0, 50000.0];

    let first = &prices[0]; // &T
    println!("First: {}", first);
    // first no longer used — &T ends

    prices.push(52000.0); // implicit &mut — OK!
    println!("All: {:?}", prices);
}
```

## What We Learned

| Combination | Allowed? |
|-------------|----------|
| &T + &T | ✓ Yes |
| &mut T (one) | ✓ Yes |
| &T + &mut T | ✗ No |
| &mut T + &mut T | ✗ No |

## Homework

1. Create &T, try creating &mut T simultaneously — read the error
2. Fix code: use &T first, then create &mut T
3. Write `process(data: &[f64]) -> Vec<f64>` (reads) and `update(data: &mut Vec<f64>)` (writes), call both sequentially
4. Explain in your own words why this rule matters for safety

## Navigation

[← Previous day](../040-one-mutable-reference-rule/en.md) | [Next day →](../042-dangling-references/en.md)
