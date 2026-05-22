# Day 29: String Parsing — Converting Input to Number

## Trading Analogy

Exchange APIs return prices as strings: `"49500.50"`. To compute with them, you need to convert the string to a number. This is called parsing — and Rust does it safely via `parse()`.

## parse(): Converting String to Number

```rust
fn main() {
    let price_str = "49500.50";
    let price: f64 = price_str.parse().expect("Invalid price format");
    println!("Price: ${:.2}", price);

    let qty_str = "10";
    let qty: u32 = qty_str.parse().expect("Invalid quantity");
    println!("Quantity: {}", qty);

    let total = price * qty as f64;
    println!("Total: ${:.2}", total);
}
```

## Safe Parsing via Result

```rust
fn parse_price(s: &str) -> Result<f64, String> {
    s.trim()
     .parse::<f64>()
     .map_err(|e| format!("Parse error '{}': {}", s, e))
}

fn main() {
    let inputs = vec!["50000.0", "invalid", "48500.5", ""];

    for input in inputs {
        match parse_price(input) {
            Ok(price) => println!("Price: ${:.2}", price),
            Err(e)    => println!("Error: {}", e),
        }
    }
}
```

## Parsing with Range Check

```rust
fn parse_quantity(s: &str) -> Result<f64, String> {
    let qty: f64 = s.trim()
        .parse()
        .map_err(|_| "Not a number".to_string())?;

    if qty <= 0.0 {
        return Err("Quantity must be positive".to_string());
    }
    if qty > 1000.0 {
        return Err("Volume too large".to_string());
    }
    Ok(qty)
}

fn main() {
    for s in &["5.5", "-1", "2000", "abc"] {
        println!("{:?} -> {:?}", s, parse_quantity(s));
    }
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `s.parse::<f64>()` | Parse string to specific type |
| `s.parse().expect("msg")` | Parse with panic on error |
| `s.parse().map_err(\|e\| ...)` | Transform error |
| `.trim()` before parse | Remove whitespace and newlines |

## Homework

1. Write `parse_price(s: &str) -> Result<f64, String>` checking price > 0
2. Parse a vector of strings `["100.5", "bad", "200.0"]` and print Ok/Err for each
3. Write `parse_percent(s: &str) -> Result<f64, String>` — value 0.0 to 100.0
4. Combine price and quantity parsing into a function computing total position value

## Navigation

[← Previous day](../028-user-input-position-size/en.md) | [Next day →](../030-simple-profit-calculator/en.md)
