# Day 40: One Mutable Reference Rule

## Trading Analogy

Imagine two traders simultaneously modifying the same order: one increases volume, the other changes price — the result is unpredictable. Rust prevents this scenario: only one can modify data at any moment.

## Why Only One &mut

```rust
fn main() {
    let mut price = 50000.0;

    let r1 = &mut price;
    // let r2 = &mut price; // ERROR: data race impossible in Rust

    *r1 = 51000.0;
    println!("{}", price);
}
```

## Correct Pattern: Sequential Use

```rust
fn main() {
    let mut orders = vec![100.0, 200.0, 300.0];

    {
        let first = &mut orders[0];
        *first += 50.0;
    } // first released

    {
        let second = &mut orders[1];
        *second -= 30.0;
    }

    println!("{:?}", orders);
}
```

## Non-Lexical Lifetimes (NLL)

Modern Rust is smarter — a reference ends at its last use:

```rust
fn main() {
    let mut balance = 10000.0;

    let r1 = &mut balance;
    *r1 += 500.0;
    // r1 no longer used — borrow ends

    let r2 = &mut balance; // OK! r1 is no longer active
    *r2 -= 100.0;

    println!("{}", balance);
}
```

## What We Learned

| Rule | Reason |
|------|--------|
| One &mut at a time | Prevent data races |
| NLL: reference lives until last use | Flexibility without sacrificing safety |
| split_at_mut | Safely split into mutable parts |
| Sequential blocks {} | Classic pattern |

## Homework

1. Try creating two &mut simultaneously, study the error
2. Fix using sequential {} blocks
3. Use NLL: create &mut, use it, then create another &mut after
4. Use `split_at_mut` to modify first and second halves of a Vec

## Navigation

[← Previous day](../039-mutable-references-editing-order/en.md) | [Next day →](../041-no-mixing-references/en.md)
