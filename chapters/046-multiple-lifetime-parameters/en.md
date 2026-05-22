# Day 46: Multiple Lifetime Parameters

## Trading Analogy

A complex function might work with multiple data sources: historical prices (long-lived) and current snapshot (short-lived). Multiple lifetime parameters precisely describe these dependencies.

## Two Lifetime Parameters

```rust
// Return reference bound to 'a, not 'b
fn select_price<'a, 'b>(historical: &'a f64, current: &'b f64, use_hist: bool) -> &'a f64 {
    if use_hist { historical } else { historical } // only return 'a
}

fn main() {
    let hist_price = 48000.0;
    let result;
    {
        let curr_price = 50000.0;
        result = select_price(&hist_price, &curr_price, true);
    } // curr_price destroyed, but result is bound to hist_price — OK
    println!("Price: {}", result);
}
```

## Different Lifetimes for Different Data

```rust
struct PriceAnalyzer<'a, 'b> {
    history: &'a [f64],   // long-lived data
    snapshot: &'b [f64],  // short-lived data
}

impl<'a, 'b> PriceAnalyzer<'a, 'b> {
    fn avg_history(&self) -> f64 {
        self.history.iter().sum::<f64>() / self.history.len() as f64
    }

    fn avg_snapshot(&self) -> f64 {
        self.snapshot.iter().sum::<f64>() / self.snapshot.len() as f64
    }
}

fn main() {
    let history = vec![48000.0, 49000.0, 50000.0];
    let snapshot = vec![50500.0, 51000.0];

    let analyzer = PriceAnalyzer { history: &history, snapshot: &snapshot };
    println!("Avg history: ${:.2}", analyzer.avg_history());
    println!("Avg snapshot: ${:.2}", analyzer.avg_snapshot());
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `<'a, 'b>` | Two lifetime parameters |
| `&'a T, &'b U` | Independent lifetimes |
| Structs with multiple lifetimes | Different fields, different guarantees |
| Return type | Must be bound to one of the inputs |

## Homework

1. Write `merge_signals<'a, 'b>(fast: &'a str, slow: &'b str) -> &'a str`
2. Create `Strategy<'a, 'b>` struct with two reference fields of different lifetimes
3. Implement a method returning a reference from the 'a field
4. Explain why different lifetime parameters are needed instead of one

## Navigation

[← Previous day](../045-lifetime-elision/en.md) | [Next day →](../047-static-lifetime/en.md)
