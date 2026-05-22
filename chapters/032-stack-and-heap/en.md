# Day 32: Stack and Heap — Cash and Bank Account

## Trading Analogy

Imagine you're a trader. You have cash in your wallet — you can instantly pay at the exchange, grab a bill and put it back. That's the **stack**. But you also have a bank account where large sums are stored — access is slightly slower, but you can put as much as you want there. That's the **heap**. Rust decides on its own where to store each type of data and frees memory automatically — no leaks.

---

## What is the Stack

The stack is a memory region that works on the **LIFO** principle (Last In, First Out). Imagine a stack of trading orders: you put a new order on top, and you process that one first.

Key properties of the stack:
- Data size is **known at compile time**
- Allocation and deallocation are **very fast** (just a pointer shift)
- Data is stored **sequentially** in memory
- When a function returns, its stack frame is **automatically destroyed**

```rust
fn calculate_pnl(entry_price: f64, exit_price: f64, qty: u32) -> f64 {
    // All these variables live on the stack
    let price_diff: f64 = exit_price - entry_price;  // 8 bytes
    let quantity: f64 = qty as f64;                   // 8 bytes (cast)
    let pnl: f64 = price_diff * quantity;             // 8 bytes
    let commission: f64 = 0.001 * exit_price * quantity; // 8 bytes
    let net_pnl: f64 = pnl - commission;              // 8 bytes

    net_pnl
    // When the function returns — price_diff, quantity, pnl,
    // commission are automatically destroyed
}

fn main() {
    let entry: f64 = 42000.0;  // On the stack
    let exit: f64 = 45000.0;   // On the stack
    let qty: u32 = 2;           // On the stack

    let result = calculate_pnl(entry, exit, qty);
    println!("Net PnL: ${:.2}", result);
}
```

Numeric types (`i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64`, `bool`, `char`) always live on the stack — their size is fixed.

---

## What is the Heap

The heap is a large memory region for data whose **size is unknown in advance or changes over time**. Like a bank account: you can put as much as you want, but each operation takes a bit more effort.

Heap characteristics:
- Data size can be **anything** and change at runtime
- Memory allocation via allocator (slightly slower)
- Data is **not necessarily adjacent** in memory
- In Rust, deallocation happens **automatically through the ownership system** (no manual `free()`)

```rust
fn main() {
    // String lives on the heap — we don't know the length upfront
    let ticker = String::from("BTCUSDT");

    // Vec<f64> lives on the heap — the price list can grow
    let mut prices: Vec<f64> = Vec::new();
    prices.push(42000.0);
    prices.push(42500.0);
    prices.push(41800.0);
    prices.push(43000.0);

    println!("Ticker: {}", ticker);
    println!("Last price: ${}", prices.last().unwrap());
    println!("Total candles: {}", prices.len());
    // When main() finishes — ticker and prices are automatically
    // freed, memory is returned to the operating system
}
```

---

## Which Types Live Where

The rule is simple: if the compiler knows the size of a type — stack; if not — heap.

```rust
fn main() {
    // === STACK ===
    let price: f64 = 50000.0;        // 8 bytes — stack
    let volume: u64 = 1_000_000;     // 8 bytes — stack
    let is_long: bool = true;         // 1 byte — stack
    let side: char = 'B';             // 4 bytes — stack
    let order_id: i32 = 12345;        // 4 bytes — stack
    let coords: (f64, f64) = (1.0, 2.0); // 16 bytes — stack (known-size tuple)
    let last_three: [f64; 3] = [100.0, 101.0, 99.0]; // 24 bytes — stack (fixed array)

    // === HEAP ===
    let symbol = String::from("ETHUSDT");     // symbol data on heap
    let history: Vec<f64> = vec![1.0, 2.0];  // vector data on heap
    let nested: Vec<Vec<f64>> = vec![         // both outer and inner — on heap
        vec![1.0, 2.0],
        vec![3.0, 4.0],
    ];

    println!("Price: {}, Symbol: {}", price, symbol);
    println!("History: {:?}", history);
    drop(nested); // explicitly free, though Rust would do it anyway
}
```

---

## Box\<T\> — Placing Stack Data on the Heap

`Box<T>` is a smart pointer that forcibly places data on the heap. It's like depositing cash (stack data) in a bank (heap).

When you need it:
- For recursive types (tree, linked list) where the size depends on the type itself
- For passing large structures without copying
- For working with traits via `dyn Trait`

```rust
// Without Box — recursive type won't compile!
// enum OrderBook { Node(f64, OrderBook) } // ERROR: infinite size

// With Box — compiler knows the size of Box (just a pointer)
#[derive(Debug)]
enum PriceLevel {
    Empty,
    Level {
        price: f64,
        volume: f64,
        next: Box<PriceLevel>, // Box has a fixed size (a pointer)
    },
}

fn main() {
    let order_book = PriceLevel::Level {
        price: 50000.0,
        volume: 1.5,
        next: Box::new(PriceLevel::Level {
            price: 49999.0,
            volume: 3.0,
            next: Box::new(PriceLevel::Empty),
        }),
    };

    // Just a large structure on the heap via Box
    let big_price_array: Box<[f64; 1000]> = Box::new([0.0; 1000]);
    println!("Box size: {} bytes", std::mem::size_of_val(&big_price_array));
    // Box itself is tiny (8-byte pointer), data is on the heap
}
```

---

## Practical Significance for Rust

Understanding stack and heap explains many language decisions:

```rust
fn main() {
    // i32 is copied on assignment (Copy trait)
    // Because copying 4 bytes from stack is instantaneous
    let price1: i32 = 42000;
    let price2 = price1;       // copy, not move
    println!("price1={}, price2={}", price1, price2); // BOTH work

    // String is moved on assignment (Move semantics)
    // Because copying heap data is expensive (requires new allocation)
    // Rust MOVES by default, doesn't copy
    let ticker1 = String::from("BTCUSDT");
    let ticker2 = ticker1;     // ownership moved
    // println!("{}", ticker1); // ERROR: ticker1 no longer owns data
    println!("ticker2={}", ticker2); // only ticker2 is valid

    // To copy a String — explicitly call .clone()
    let s1 = String::from("ETHUSDT");
    let s2 = s1.clone();       // explicit expensive copy
    println!("s1={}, s2={}", s1, s2); // both work
}
```

---

## Memory Diagram in Code

Let's see exactly how data is laid out in memory:

```rust
fn show_memory_layout() {
    // Stack frame of show_memory_layout function:
    // [  price: f64  ][  qty: u32  ][  ticker: (ptr|len|cap)  ]
    //  0x7fff0000      0x7fff0008    0x7fff000C
    //      ↓                              ↓
    //   50000.0           10         pointer to heap
    //                                     ↓
    //                              Heap: ['B','T','C','U','S','D','T']

    let price: f64 = 50000.0;
    let qty: u32 = 10;
    let ticker = String::from("BTCUSDT");

    // String stores THREE things on the stack:
    // 1. ptr — pointer to data on the heap
    // 2. len — current length (7)
    // 3. cap — capacity (usually >= len)
    println!("Size of String struct on stack: {} bytes",
             std::mem::size_of::<String>()); // 24 bytes (3 × 8)
    println!("Size of heap data: {} bytes", ticker.len()); // 7 bytes

    // Vec is similar:
    let prices: Vec<f64> = vec![1.0, 2.0, 3.0];
    println!("Size of Vec struct on stack: {} bytes",
             std::mem::size_of::<Vec<f64>>()); // 24 bytes
    println!("Price: {}, Qty: {}", price, qty);
}

fn main() {
    show_memory_layout();
}
```

---

## Practical Example: Trading Data Analyzer

Let's bring it all together — building a structure that correctly uses stack and heap:

```rust
#[derive(Debug)]
struct TradeSession {
    // Fixed-size fields — will be on the stack inside the struct
    session_id: u32,
    open_price: f64,
    close_price: f64,
    is_profitable: bool,

    // Dynamic-size fields — data on heap, pointers on stack
    symbol: String,
    trade_prices: Vec<f64>,
    notes: String,
}

impl TradeSession {
    fn new(symbol: &str, open_price: f64) -> TradeSession {
        TradeSession {
            session_id: 1,
            open_price,
            close_price: 0.0,
            is_profitable: false,
            symbol: String::from(symbol),  // allocates memory on heap
            trade_prices: Vec::new(),       // empty vector on heap
            notes: String::new(),
        }
    }

    fn add_trade(&mut self, price: f64) {
        self.trade_prices.push(price);  // may expand heap
        self.close_price = price;
        self.is_profitable = price > self.open_price;
    }

    fn summary(&self) -> String {  // returns a new String (heap)
        format!(
            "Session {}: {} | Open: ${:.2} | Close: ${:.2} | {} | Trades: {}",
            self.session_id,
            self.symbol,
            self.open_price,
            self.close_price,
            if self.is_profitable { "PROFIT" } else { "LOSS" },
            self.trade_prices.len()
        )
    }

    fn average_price(&self) -> f64 {  // returns f64 (stack)
        if self.trade_prices.is_empty() {
            return self.open_price;
        }
        let sum: f64 = self.trade_prices.iter().sum();
        sum / self.trade_prices.len() as f64
    }
}

fn main() {
    // session lives on stack (struct), String/Vec data — on heap
    let mut session = TradeSession::new("BTCUSDT", 42000.0);

    session.add_trade(42500.0);
    session.add_trade(43000.0);
    session.add_trade(42800.0);
    session.add_trade(44000.0);

    println!("{}", session.summary());
    println!("Average price: ${:.2}", session.average_price());

    // Box<TradeSession> — entire struct on heap (for large structures)
    let boxed_session: Box<TradeSession> = Box::new(
        TradeSession::new("ETHUSDT", 3000.0)
    );
    println!("Box pointer size: {} bytes", std::mem::size_of_val(&boxed_session));
    // When main() finishes — everything is freed automatically:
    // boxed_session -> Drop for Box -> Drop for TradeSession -> heap freed
}
```

---

## What We Learned

| Syntax | Description | Trading Example |
|--------|-------------|-----------------|
| `let x: f64 = 1.0` | Stack variable | Order price — known size |
| `String::from("X")` | String on heap | Instrument ticker |
| `Vec::new()` | Vector on heap | Price history |
| `Box::new(x)` | Place data on heap | Large order book struct |
| `std::mem::size_of::<T>()` | Type size in bytes | Memory usage analysis |
| `drop(x)` | Explicit deallocation | Close position early |
| `Vec::push(v)` | Add to heap | New trade in history |

---

## Homework

1. Declare an `Order` struct with stack fields (`id: u32`, `price: f64`, `qty: u32`, `is_buy: bool`) and heap fields (`symbol: String`, `tags: Vec<String>`).
   - Create 3 `Order` instances
   - Print the struct size via `std::mem::size_of::<Order>()`

2. Write a `stack_demo()` function that:
   - Creates 5 stack variables (different numeric types)
   - Shows their sizes via `std::mem::size_of_val(&x)`
   - Packs them into a tuple and prints it

3. Implement a `Box` wrapper:
   - Create a large `[f64; 10000]` array first directly (stack), then via `Box` (heap)
   - Fill it with prices from 40000.0 to 50000.0
   - Find minimum and maximum

4. Explore the difference between `String` and `&str`:
   - `&str` is just a reference to a string slice (often stack or static memory)
   - `String` is an owning string on the heap
   - Print sizes of both types via `size_of`

---

## Navigation

[← Previous day](../031-project-position-size-calculator/en.md) | [Next day →](../033-ownership-who-holds-asset/en.md)
