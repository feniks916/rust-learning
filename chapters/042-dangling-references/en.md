# Day 42: Dangling References

## Trading Analogy

Imagine: an order was executed and removed from the system, but you still hold a pointer to it. Using such a pointer is catastrophic. In C/C++ this is a dangling pointer. Rust makes this **impossible**.

## What is a Dangling Reference

```rust
// This code WILL NOT compile in Rust:
// fn create_dangling() -> &String {
//     let s = String::from("BTC");
//     &s  // ERROR: s is destroyed, reference becomes dangling
// }
```

## Rust Prevents This

```rust
// CORRECT: return ownership, not a reference
fn create_ticker() -> String {
    let s = String::from("BTCUSDT");
    s // return value, not &s
}

fn main() {
    let ticker = create_ticker();
    println!("{}", ticker); // safe!
}
```

## A Reference Cannot Outlive Its Data

```rust
fn main() {
    let reference;

    {
        let price = 50000.0;
        // reference = &price; // ERROR: price lives shorter than reference
    }

    // println!("{}", reference); // would have been dangling
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Dangling reference | Reference to destroyed data |
| Rust forbids it | Compile error, not runtime crash |
| Reference lifetime ≤ data lifetime | Always |
| Solution | Return ownership (String, Vec...) |

## Homework

1. Try writing a function returning `&String` from a local variable — see the error
2. Fix: return `String` instead of `&String`
3. Try creating a reference to a variable that's destroyed sooner — see the error
4. Write 3 examples of safe references and 3 unsafe ones (commented out)

## Navigation

[← Previous day](../041-no-mixing-references/en.md) | [Next day →](../043-lifetimes-order-lifespan/en.md)
