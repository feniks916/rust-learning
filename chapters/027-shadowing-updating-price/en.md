# Day 27: Shadowing — Updating Price with the Same Name

## Trading Analogy

The asset price on your terminal screen updates constantly, but we still call it "the BTC price" — the name doesn't change, only the value does. Shadowing in Rust works similarly: you can declare a new variable with the same name, and it "overshadows" the old one. Unlike `mut`, the new variable can be a completely different type. This is especially handy during parsing: first `price` is a string from the API, then `price` is a number, then `price` is a number with the fee added. One name, three forms.

## What Is Shadowing

Shadowing is not mutation — it creates a new variable under the same name:

```rust
fn main() {
    let price = 50000;              // i32 — integer
    println!("Price (int): {}", price);

    let price = price as f64;       // shadowing: now f64
    println!("Price (f64): {}", price);

    let price = price * 1.05;       // shadowing: value +5%
    println!("Price +5%: {:.2}", price);

    // price is still f64 — type didn't change in last shadowing
    let price = format!("${:.0}", price); // shadowing: now a String
    println!("Price (String): {}", price);

    // Earlier "versions" of price are inaccessible — only the latest one exists
}
```

Each `let price = ...` creates a new variable. The old one is "put in shadow" — hence the term "shadowing."

## Shadowing vs mut: Changing Type

The key difference from `mut` — shadowing allows changing the variable type:

```rust
fn main() {
    // SHADOWING: can change type
    let input = "   50000.50   "; // &str — a string
    let input = input.trim();      // &str without whitespace (still a string)
    let input: f64 = input.parse().expect("Invalid format"); // now f64
    let input = input + 0.5;       // f64 + delta
    println!("Shadowing: {:.2}", input); // 50001.00

    // MUT: type cannot change
    let mut price = 49000.0_f64;
    price = 50000.0;   // OK: same type
    // price = "expensive"; // COMPILE ERROR: expected f64, found &str

    // MUT cannot do what shadowing does:
    let mut s = "hello";
    // s = s.len(); // ERROR: type must remain &str
    let s = s.len(); // OK via shadowing: s is now usize
    println!("Length: {}", s);
}
```

`mut` is changing the value of the same variable. Shadowing is creating a new variable under the old name.

## Shadowing in Blocks: Local Shadowing

Shadowing inside a `{}` block only applies within the block:

```rust
fn main() {
    let price = 50000.0;

    // Shadow price inside the block
    let discounted = {
        let price = price * 0.98;  // -2% discount (shadowing only here)
        println!("  Inside block: {:.2}", price); // 49000.0
        price  // return from block
    };

    // Outside the block — the original value is preserved
    println!("Outside block: {:.2}", price);     // 50000.0
    println!("Discounted:    {:.2}", discounted); // 49000.0

    // Another example: processing a quote in a block
    let processed_price = {
        let price = price / 1000.0;  // convert to thousands
        let price = price.round();   // round
        price * 1000.0               // convert back
    };
    println!("Rounded price: {:.2}", processed_price); // 50000.0
}
```

Useful pattern: temporary calculations are isolated in the block, not cluttering the name space.

## Practical Parsing Pattern: str → f64 → Processed Value

Shadowing is the ideal tool for a chain of transformations during parsing:

```rust
fn process_ticker_price(raw: &str) -> f64 {
    // Step 1: trim whitespace (&str → &str)
    let raw = raw.trim();

    // Step 2: parse string to number (type change!)
    let raw: f64 = match raw.parse() {
        Ok(v)  => v,
        Err(_) => {
            println!("Warning: '{}' is not a number, using 0.0", raw);
            0.0
        }
    };

    // Step 3: validation
    let raw = if raw < 0.0 { 0.0 } else { raw };

    // Step 4: normalization (e.g., price in cents → dollars)
    let raw = if raw > 1_000_000.0 { raw / 100.0 } else { raw };

    raw
}

fn main() {
    let test_cases = ["  50000.0  ", "  -100  ", "  not_a_number  ", "  5000000  ", "  48500.50  "];

    for input in &test_cases {
        let price = process_ticker_price(input);
        println!("'{}' → ${:.2}", input.trim(), price);
    }
}
```

Each step describes one transformation, and the name `raw` stays constant — it reads like a processing history.

## Shadowing in Functions: One Name, Multiple Stages

Functions with shadowing often look like a "processing pipeline":

```rust
fn normalize_price(price_str: &str) -> Option<f64> {
    // Stage 1: trim
    let price_str = price_str.trim();
    if price_str.is_empty() {
        return None;
    }

    // Stage 2: strip currency symbols if present
    let price_str = price_str.trim_start_matches('$').trim_start_matches("USD");

    // Stage 3: parse (type change)
    let price: f64 = price_str.parse().ok()?;

    // Stage 4: range check and rounding
    let price = price.abs(); // always positive
    let price = (price * 100.0).round() / 100.0; // 2 decimal places

    Some(price)
}

fn main() {
    let inputs = ["$50000.123", " 48500.5 ", "USD51000", "", "bad", "-49000.0"];

    println!("Price normalization:");
    for input in &inputs {
        match normalize_price(input) {
            Some(p) => println!("  {:>15} → ${:.2}", format!("'{}'", input), p),
            None    => println!("  {:>15} → (invalid)", format!("'{}'", input)),
        }
    }
}
```

## When Shadowing Is Good, When It's Bad

Shadowing is powerful, but it has its place:

```rust
fn main() {
    // GOOD: parsing pattern — a logical chain
    let order_qty = "100";
    let order_qty: u32 = order_qty.parse().expect("Number");
    let order_qty = order_qty.min(500); // cap at limit
    println!("Order quantity: {}", order_qty);

    // GOOD: temporary transformation in a block
    let price = 50123.456;
    let display = {
        let price = (price / 1000.0).floor(); // whole thousands
        format!("${:.0}k", price)
    };
    println!("Price: {} (exact: {:.3})", display, price);

    // BAD: shadowing to "fix a typo"
    let enty_price = 50000.0; // typo in name
    let entry_price = enty_price; // alias instead of renaming
    // Better to name it correctly from the start!

    // BAD: meaningless chain
    let x = 5;
    let x = x + 1;
    let x = x * 2;
    // Better to use distinct names: x_base, x_adjusted, x_final
    println!("{}", x);
}
```

Shadowing is good when the name genuinely stays semantically the same ("this is still the price"), but the form changes (string → number → processed number).

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `let x = ...; let x = ...;` | New value/type under the same name | `let price = "50000"; let price: f64 = price.parse()...` |
| Shadowing changes type | Unlike `mut`, type can become something else | `let s = "3"; let s = s.len();` (usize) |
| Shadowing in a `{}` block | Isolated temporary calculations | `let result = { let x = x * 2; x };` |
| Parsing pattern | Chain: str → trim → parse → validate | Processing API quotes |
| `mut` vs shadowing | `mut` — same name and type; shadowing — same name, any type | For type change, only shadowing works |
| When NOT to use | When the variables are logically different | Better: `price_raw` and `price_clean` |

## Homework

1. Implement a `parse_ticker_input(s: &str) -> Option<(String, f64)>` function using shadowing:
   - `let s = s.trim()`
   - `let parts = s.split(':')`
   - Verify exactly 2 parts
   - `let ticker = parts[0].to_uppercase()`
   - `let price: f64 = parts[1].parse().ok()?`
   - Input: `"btc: 50000"`, output: `Some(("BTC", 50000.0))`

2. Write a volume normalization pipeline: input `"  1500000 "`, result `"1.5M"`
   - `let v = v.trim()`
   - `let v: f64 = v.parse()...`
   - `let v = v / 1_000_000.0`
   - `let v = format!("{:.1}M", v)`

3. Show the difference between shadowing and `mut`: try changing the type via `mut` (show the error in a comment), then show that shadowing makes it work

4. Create a function that processes an array of price strings: takes `&[&str]`, returns `Vec<f64>`
   - For each element shadow: trim → parse → validate (> 0) → round
   - Skip invalid values with `continue`

## Navigation

[← Previous day](../026-constants-fixed-exchange-fee/en.md) | [Next day →](../028-user-input-position-size/en.md)
