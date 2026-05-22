# Day 65: Multiple impl Blocks

## Trading Analogy

Imagine a large trading firm. It has a department for creating trades, a department for profit calculations, a reporting department, and a risk validation department. All of them work with the same object — a Trade. That's exactly how multiple `impl` blocks work in Rust: each block handles its own group of methods, but all of them belong to the same struct.

## Basic Syntax

In Rust you can write multiple `impl` blocks for the same struct. The compiler merges them all — it's completely legal:

```rust
#[derive(Debug)]
struct Trade {
    ticker: String,
    entry_price: f64,
    exit_price: f64,
    quantity: f64,
}

// First impl block — creation
impl Trade {
    fn new(ticker: &str, entry_price: f64, exit_price: f64, quantity: f64) -> Self {
        Trade {
            ticker: ticker.to_string(),
            entry_price,
            exit_price,
            quantity,
        }
    }
}

// Second impl block — completely separate, but for the same struct
impl Trade {
    fn pnl(&self) -> f64 {
        (self.exit_price - self.entry_price) * self.quantity
    }
}

fn main() {
    let t = Trade::new("AAPL", 150.0, 165.0, 10.0);
    println!("PnL: ${:.2}", t.pnl()); // PnL: $150.00
}
```

Both `new` and `pnl` are called in exactly the same way — Rust makes no distinction between which `impl` block they are defined in.

## Why Use Multiple Blocks?

One giant `impl` block with 30 methods is hard to read. By splitting it into logical groups you get:

- **Readability**: immediately clear where creation, calculations, and output live
- **Navigation**: easy to find the method you need
- **Teamwork**: different developers can work on different blocks without conflicts

```rust
// Bad: one big block
impl Trade {
    fn new(...) -> Self { ... }
    fn pnl(&self) -> f64 { ... }
    fn summary(&self) { ... }
    fn is_valid(&self) -> bool { ... }
    fn pnl_percent(&self) -> f64 { ... }
    // ... 20 more methods mixed together
}

// Good: blocks by purpose
impl Trade { /* creation   */ }
impl Trade { /* calculations */ }
impl Trade { /* reporting  */ }
impl Trade { /* validation */ }
```

## Creation Block

The first block — constructors and factory methods:

```rust
impl Trade {
    /// Create a new trade
    fn new(ticker: &str, entry_price: f64, exit_price: f64, quantity: f64) -> Self {
        Trade {
            ticker: ticker.to_string(),
            entry_price,
            exit_price,
            quantity,
        }
    }

    /// Create a test losing trade
    fn test_loss() -> Self {
        Trade::new("TEST", 100.0, 90.0, 1.0)
    }

    /// Create with prices rounded to 2 decimal places
    fn with_rounded_prices(ticker: &str, entry: f64, exit: f64, qty: f64) -> Self {
        Trade::new(
            ticker,
            (entry * 100.0).round() / 100.0,
            (exit * 100.0).round() / 100.0,
            qty,
        )
    }
}
```

## Calculations Block

The second block — all the math:

```rust
impl Trade {
    /// Absolute PnL in dollars
    fn pnl(&self) -> f64 {
        (self.exit_price - self.entry_price) * self.quantity
    }

    /// Percentage price change
    fn pnl_percent(&self) -> f64 {
        (self.exit_price - self.entry_price) / self.entry_price * 100.0
    }

    /// Cost of position at entry
    fn position_value(&self) -> f64 {
        self.entry_price * self.quantity
    }

    /// Is the trade profitable?
    fn is_winner(&self) -> bool {
        self.pnl() > 0.0
    }

    /// Return on position
    fn return_on_position(&self) -> f64 {
        self.pnl() / self.position_value() * 100.0
    }
}
```

## Reporting Block

The third block — output and formatting:

```rust
impl Trade {
    /// Short trade summary
    fn summary(&self) {
        let status = if self.is_winner() { "PROFIT ✓" } else { "LOSS ✗" };
        println!("=== Trade {} ===", self.ticker);
        println!("  Entry:    ${:.2}", self.entry_price);
        println!("  Exit:     ${:.2}", self.exit_price);
        println!("  Quantity: {:.4}", self.quantity);
        println!("  PnL:      ${:.2} ({:+.2}%)", self.pnl(), self.pnl_percent());
        println!("  Status:   {}", status);
    }

    /// One line for a table
    fn to_csv_row(&self) -> String {
        format!(
            "{},{:.2},{:.2},{:.4},{:.2}",
            self.ticker, self.entry_price, self.exit_price,
            self.quantity, self.pnl()
        )
    }
}
```

## Validation Block

The fourth block — checks before use:

```rust
impl Trade {
    /// Are all fields correct?
    fn is_valid(&self) -> bool {
        !self.ticker.is_empty()
            && self.entry_price > 0.0
            && self.exit_price > 0.0
            && self.quantity > 0.0
    }

    /// Is the position size acceptable?
    fn check_size(&self, max_position: f64) -> Result<(), String> {
        let value = self.position_value();
        if value > max_position {
            Err(format!(
                "Position ${:.2} exceeds limit ${:.2}",
                value, max_position
            ))
        } else {
            Ok(())
        }
    }

    /// Is the price move reasonable (not more than 50%)?
    fn check_price_move(&self) -> Result<(), String> {
        let change = (self.exit_price - self.entry_price).abs() / self.entry_price;
        if change > 0.5 {
            Err(format!("Suspicious price move: {:.1}%", change * 100.0))
        } else {
            Ok(())
        }
    }
}
```

## Block Order Does Not Matter

Rust does not require a specific order of `impl` blocks. You can write validation first and creation second — it will compile either way:

```rust
// Order of blocks does not affect compilation
impl Trade { /* validation  */ }  // can be first
impl Trade { /* creation    */ }  // can be second
impl Trade { /* calculations */ } // can be third

fn main() {
    // Methods from all blocks are accessible the same way
    let t = Trade::new("ETHUSDT", 2000.0, 2200.0, 5.0);

    // From the calculations block
    println!("PnL: ${:.2}", t.pnl());

    // From the validation block
    if t.is_valid() {
        println!("Trade is valid");
    }

    // From the reporting block
    t.summary();
}
```

## Practical Example: Trade with 4 impl Blocks

```rust
#[derive(Debug, Clone)]
struct Trade {
    ticker: String,
    entry_price: f64,
    exit_price: f64,
    quantity: f64,
    commission: f64,
}

// === BLOCK 1: Creation ===
impl Trade {
    fn new(ticker: &str, entry: f64, exit: f64, qty: f64) -> Self {
        Trade {
            ticker: ticker.to_string(),
            entry_price: entry,
            exit_price: exit,
            quantity: qty,
            commission: 0.001, // 0.1% default commission
        }
    }

    fn with_commission(mut self, commission: f64) -> Self {
        self.commission = commission;
        self
    }
}

// === BLOCK 2: Calculations ===
impl Trade {
    fn gross_pnl(&self) -> f64 {
        (self.exit_price - self.entry_price) * self.quantity
    }

    fn commission_cost(&self) -> f64 {
        (self.entry_price + self.exit_price) * self.quantity * self.commission
    }

    fn net_pnl(&self) -> f64 {
        self.gross_pnl() - self.commission_cost()
    }

    fn pnl_percent(&self) -> f64 {
        self.net_pnl() / (self.entry_price * self.quantity) * 100.0
    }

    fn is_winner(&self) -> bool {
        self.net_pnl() > 0.0
    }
}

// === BLOCK 3: Reporting ===
impl Trade {
    fn summary(&self) {
        println!("╔══════════════════════════╗");
        println!("║  Trade: {:>16}  ║", self.ticker);
        println!("╠══════════════════════════╣");
        println!("║  Entry:  {:>15.2}$ ║", self.entry_price);
        println!("║  Exit:   {:>15.2}$ ║", self.exit_price);
        println!("║  Qty:    {:>15.4}  ║", self.quantity);
        println!("╠══════════════════════════╣");
        println!("║  Gross PnL:  {:>10.2}$ ║", self.gross_pnl());
        println!("║  Commission: {:>10.2}$ ║", self.commission_cost());
        println!("║  Net PnL:    {:>10.2}$ ║", self.net_pnl());
        println!("╠══════════════════════════╣");
        let result = if self.is_winner() { "PROFIT" } else { "LOSS  " };
        println!("║  Result: {:>17}  ║", result);
        println!("╚══════════════════════════╝");
    }
}

// === BLOCK 4: Validation ===
impl Trade {
    fn is_valid(&self) -> bool {
        !self.ticker.is_empty()
            && self.entry_price > 0.0
            && self.exit_price > 0.0
            && self.quantity > 0.0
            && self.commission >= 0.0
            && self.commission < 1.0
    }

    fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        if self.ticker.is_empty() {
            errors.push("Ticker cannot be empty".to_string());
        }
        if self.entry_price <= 0.0 {
            errors.push(format!("Entry price must be > 0, got: {}", self.entry_price));
        }
        if self.quantity <= 0.0 {
            errors.push(format!("Quantity must be > 0, got: {}", self.quantity));
        }
        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}

fn main() {
    let trade = Trade::new("BTCUSDT", 65000.0, 68500.0, 0.5)
        .with_commission(0.001);

    match trade.validate() {
        Ok(()) => {
            println!("Trade passed validation\n");
            trade.summary();
            println!("\nNet PnL:  ${:.2}", trade.net_pnl());
            println!("Return:   {:.2}%", trade.pnl_percent());
        }
        Err(errors) => {
            println!("Validation errors:");
            for e in &errors {
                println!("  - {}", e);
            }
        }
    }
}
```

## What We Learned

| Syntax | Description | Use Case |
|--------|-------------|----------|
| Multiple `impl Struct { }` | Several impl blocks for one struct | Organizing code into semantic groups |
| Block order | Does not matter to the compiler | Can be placed in any order |
| Methods from different blocks | Called the same way | No difference for the struct's user |
| Creation block | `new()`, `with_*()` | Constructors and builders |
| Calculations block | `pnl()`, `percent()` | Math and business logic |
| Reporting block | `summary()`, `to_csv()` | Formatting and output |
| Validation block | `is_valid()`, `validate()` | Data correctness checks |

## Homework

1. Create an `Order` struct with fields: `symbol`, `side` (Buy/Sell), `price`, `quantity`, `status`
   - Add enums `OrderSide { Buy, Sell }` and `OrderStatus { Pending, Filled, Cancelled }`

2. Write a creation block with methods:
   - `new(symbol, side, price, quantity) -> Self` — create a new order
   - `market_buy(symbol, quantity) -> Self` — market buy order (price = 0.0)

3. Write an execution block:
   - `fill(&mut self, fill_price: f64)` — fills the order at the given price
   - `cancel(&mut self)` — cancels the order
   - `is_active(&self) -> bool` — is the order neither filled nor cancelled?

4. Write a reporting block:
   - `display(&self)` — pretty-print order information
   - `to_log_string(&self) -> String` — string for writing to a log

5. Write a validation block:
   - `check_price(&self) -> Result<(), String>` — price is not negative
   - `check_quantity(&self) -> Result<(), String>` — quantity is in acceptable range (0.0001..=1000.0)

## Navigation

[← Previous day](../064-associated-functions-order-new/en.md) | [Next day →](../066-tuple-structs/en.md)
