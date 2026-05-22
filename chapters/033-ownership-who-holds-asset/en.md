# Day 33: Ownership — Who Holds the Asset

## Trading Analogy

On an exchange, every asset belongs to exactly one owner at any given moment. An Apple share cannot simultaneously appear in two portfolios — it's either yours or it was sold. When you sell it, you lose the right to it. Rust applies the same principle to memory: every piece of heap data has exactly one owner, and when the owner leaves — the data is destroyed automatically. No leaks, no dangling pointers.

---

## Three Rules of Ownership

All of Rust stands on three laws. If you understand them — you understand 80% of the language:

1. **Each value has one owner** — one variable that is responsible for it
2. **Only one owner at any given time** — you cannot give one value to two variables
3. **When the owner goes out of scope — the value is destroyed** — automatically, no `free()`

```rust
fn main() {
    // Rule 1: ticker is the sole owner of "BTCUSDT"
    let ticker = String::from("BTCUSDT");

    // Rule 2: after this line, ticker no longer owns the data
    let new_ticker = ticker; // ownership moves to new_ticker

    // println!("{}", ticker); // ERROR: ticker gave up ownership!

    // Rule 3: when main() ends, new_ticker is destroyed,
    // Rust automatically frees the heap memory
    println!("Owner: {}", new_ticker);
} // ← new_ticker is destroyed here, memory freed
```

These three rules are not limitations, but safety guarantees: no memory leaks, no double frees, no null pointers.

---

## Scope

A scope is a `{}` block within which a variable lives. Outside it, the variable is dead. Like a trading session: orders from the session don't carry over to the next one.

```rust
fn main() {
    // portfolio doesn't exist yet

    {
        let portfolio = String::from("BTC: 0.5, ETH: 2.0, SOL: 10.0");
        println!("Portfolio open: {}", portfolio);
        // portfolio lives here
    } // ← portfolio is destroyed here

    // println!("{}", portfolio); // ERROR: portfolio is out of scope

    // Nested scopes
    let market = String::from("NYSE");
    {
        let session = String::from("morning");
        println!("Market {} opened, session: {}", market, session);

        {
            let order = String::from("BUY AAPL 100");
            println!("Order in session: {}", order);
        } // order destroyed

        println!("Session {} continues", session);
    } // session destroyed
    // market is still alive
    println!("Market still running: {}", market);
} // market destroyed here
```

---

## Drop Trait

When an owner goes out of scope, Rust automatically calls the `drop()` method — this is the `Drop` trait. You can implement it for your type to run code on destruction (close a file, log a trade, release a connection).

```rust
struct TradingPosition {
    symbol: String,
    entry_price: f64,
    quantity: f64,
    is_open: bool,
}

impl Drop for TradingPosition {
    fn drop(&mut self) {
        if self.is_open {
            println!(
                "Position {} auto-closed on scope exit. Entry: ${:.2}",
                self.symbol, self.entry_price
            );
        } else {
            println!("Position {} cleanly closed", self.symbol);
        }
    }
}

fn main() {
    let btc = TradingPosition {
        symbol: String::from("BTCUSDT"),
        entry_price: 42000.0,
        quantity: 0.5,
        is_open: true,
    };

    {
        let eth = TradingPosition {
            symbol: String::from("ETHUSDT"),
            entry_price: 3000.0,
            quantity: 2.0,
            is_open: true,
        };
        println!("BTC and ETH positions open");
    } // ← eth.drop() called here ("ETHUSDT closed")

    println!("ETH already closed, BTC still open");
} // ← btc.drop() called here ("BTCUSDT closed")

// Important: destruction happens in REVERSE order of creation
```

---

## drop() Explicitly

Sometimes you need to destroy a value before the end of its scope — for example, to immediately free a large memory buffer. Use the `drop()` function:

```rust
fn main() {
    // Load a million price points — 8 MB on the heap
    let large_history: Vec<f64> = (0..1_000_000).map(|i| i as f64).collect();
    let count = large_history.len();

    // Computed what we needed
    let max = large_history.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    println!("Loaded {} prices, max: {}", count, max);

    // Data no longer needed — free IMMEDIATELY
    drop(large_history);
    // large_history is now inaccessible, 8 MB freed

    // println!("{}", large_history.len()); // ERROR: already destroyed

    // Now load new data without double memory cost
    let new_data: Vec<f64> = (1_000_000..2_000_000).map(|i| i as f64).collect();
    let new_max = new_data.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    println!("New data max: {}", new_max);
}
```

You cannot call `drop()` twice on the same value — Rust forbids this at compile time.

---

## Functions and Ownership

When you pass a value to a function — ownership transfers to the function parameter. After the call, the original variable is invalid:

```rust
fn analyze_portfolio(portfolio: String) -> usize {
    // portfolio now owns the String
    let count = portfolio.split(',').count();
    println!("Analyzing portfolio: {}", portfolio);
    println!("Number of positions: {}", count);
    count
} // portfolio is destroyed here

fn log_price(price: f64) {
    // price is a Copy type — a bitwise copy is made
    println!("[LOG] Price: ${:.2}", price);
} // price is destroyed, but the original in caller is fine

fn main() {
    let btc_price: f64 = 42000.0;
    let my_portfolio = String::from("BTC,ETH,SOL,ADA,DOT");

    log_price(btc_price);                   // f64 is copied
    println!("Price still here: {}", btc_price); // OK — Copy

    let positions = analyze_portfolio(my_portfolio); // String is MOVED
    // println!("{}", my_portfolio); // ERROR: already transferred
    println!("Total positions: {}", positions);
}
```

---

## Returning Ownership from a Function

A function can return ownership back to the calling code:

```rust
fn build_ticker(base: String, quote: &str) -> String {
    // Take ownership of base, create a new string and return it
    format!("{}{}", base, quote)
}

fn load_watchlist() -> Vec<String> {
    // Create a new Vec and give ownership to the caller
    vec![
        String::from("BTCUSDT"),
        String::from("ETHUSDT"),
        String::from("SOLUSDT"),
        String::from("BNBUSDT"),
    ]
} // returned value is MOVED, not destroyed

fn main() {
    let base = String::from("BTC");
    let ticker = build_ticker(base, "USDT");
    // base is inaccessible, ticker is the new owner
    println!("Ticker: {}", ticker);

    // Get Vec from function — we are the owners
    let watchlist = load_watchlist();
    println!("Watchlist: {:?}", watchlist);
    println!("First instrument: {}", watchlist[0]);
}
```

---

## Pattern: "Lend via Return" (Tuple)

Before borrowing, a common pattern was to take a value, use it, and return it back via a tuple. This is verbose, but shows the essence of ownership:

```rust
fn calculate_stats(prices: Vec<f64>) -> (Vec<f64>, f64, f64, f64) {
    // Take ownership of prices
    let min = prices.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = prices.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    let avg = prices.iter().sum::<f64>() / prices.len() as f64;

    // Return the original AND results
    (prices, min, max, avg)
}

fn tag_expensive(tickers: Vec<String>, threshold: f64, prices: &[f64]) -> Vec<String> {
    tickers
        .into_iter()
        .enumerate()
        .map(|(i, t)| {
            if i < prices.len() && prices[i] > threshold {
                format!("{} [EXPENSIVE]", t)
            } else {
                t
            }
        })
        .collect()
}

fn main() {
    let price_data = vec![48000.0, 49500.0, 50000.0, 51200.0, 49800.0];

    // Transfer ownership and get it back via tuple
    let (prices_back, min, max, avg) = calculate_stats(price_data);
    println!("Prices: {:?}", prices_back);
    println!("Min: ${:.2}, Max: ${:.2}, Avg: ${:.2}", min, max, avg);

    let tickers = vec![
        String::from("BTCUSDT"),
        String::from("ETHUSDT"),
        String::from("SOLUSDT"),
    ];
    let tagged = tag_expensive(tickers, 49000.0, &prices_back);
    // tickers is inaccessible, tagged is the new owner
    for t in &tagged {
        println!("  {}", t);
    }
}
```

This pattern is verbose — which is exactly why Rust invented **borrowing** (`&T`), covered in chapters 37–38.

---

## Practical Example: Position Management System

```rust
#[derive(Debug)]
struct Position {
    symbol: String,
    entry_price: f64,
    quantity: f64,
    is_open: bool,
}

impl Position {
    fn new(symbol: String, entry_price: f64, quantity: f64) -> Self {
        println!("[OPEN] Opening position {} @ ${:.2}", symbol, entry_price);
        Position { symbol, entry_price, quantity, is_open: true }
    }

    // Takes self by value — "consumes" the position
    fn close(mut self, exit_price: f64) -> f64 {
        self.is_open = false;
        let pnl = (exit_price - self.entry_price) * self.quantity;
        println!(
            "[CLOSE] Closing {} @ ${:.2}, PnL: ${:.2}",
            self.symbol, exit_price, pnl
        );
        pnl
    }
}

impl Drop for Position {
    fn drop(&mut self) {
        if self.is_open {
            let unrealized = self.entry_price * 0.01 * self.quantity;
            println!(
                "[WARNING] Position {} destroyed without closing! Unrealized P/L: ~${:.2}",
                self.symbol, unrealized
            );
        }
    }
}

fn run_strategy(symbol: String, entry: f64, exit: f64, qty: f64) -> f64 {
    let position = Position::new(symbol, entry, qty);
    println!("  Holding position...");
    position.close(exit) // ownership consumed, PnL returned
}

fn main() {
    println!("=== Trading Day ===");

    let pnl1 = run_strategy(String::from("BTCUSDT"), 42000.0, 45000.0, 0.1);
    let pnl2 = run_strategy(String::from("ETHUSDT"), 3000.0, 2800.0, 1.0);

    println!("\n=== Results ===");
    println!("BTC PnL: ${:.2}", pnl1);
    println!("ETH PnL: ${:.2}", pnl2);
    println!("Total PnL: ${:.2}", pnl1 + pnl2);

    // Demonstrate warning for unclosed position
    println!("\n=== Unclosed Position ===");
    {
        let _risky = Position::new(String::from("SOLUSDT"), 150.0, 10.0);
        println!("Position open, leaving scope...");
    } // _risky.drop() called — prints WARNING
}
```

---

## What We Learned

| Syntax | Description | Trading Example |
|--------|-------------|-----------------|
| `let x = value` | x becomes owner | Buy asset into portfolio |
| `{ let x = ... }` | x destroyed at `}` | Trading session close |
| `let y = x` (heap) | Move: x is invalid | Sell asset to someone else |
| `fn f(x: T)` | f takes ownership of x | Hand order to broker |
| `fn f() -> T` | f gives ownership to caller | Broker returns executed order |
| `drop(x)` | Explicit destruction | Force-close position |
| `impl Drop for T` | Code on destruction | Log trade closing |

---

## Homework

1. Create an `Order` struct with fields `id: u32`, `symbol: String`, `price: f64`, `qty: f64`.
   - Implement `Drop` for `Order` that prints `"[LOG] Order {id} on {symbol} destroyed"`
   - Create several orders in different `{}` scopes
   - Run and observe: destruction happens in reverse order of creation

2. Write a function `process_order(order: Order) -> String`:
   - Takes ownership of the order
   - Builds the string `"Order #{id}: {symbol} x{qty} @ ${price} — EXECUTED"`
   - After the call, confirm the original variable is inaccessible

3. Implement the "lend via return" pattern:
   - Function `validate_orders(orders: Vec<Order>) -> (Vec<Order>, Vec<u32>)`
   - Returns a tuple: (original vector, list of order ids with price > 40000)
   - Use `.into_iter()` and rebuild the vector via `.collect()`

4. Study Drop order in structs:
   - Create a struct `Portfolio { name: String, positions: Vec<String> }` with `impl Drop`
   - Nest one `Portfolio` inside another via `Box`
   - Print the destruction order and explain it

---

## Navigation

[← Previous day](../032-stack-and-heap/en.md) | [Next day →](../034-move-selling-asset/en.md)
