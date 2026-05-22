# Day 34: Move — Selling the Asset

## Trading Analogy

When you sell a stock on the exchange — it leaves your portfolio forever. You can no longer trade it, receive dividends from it, or sell it again. The buyer is the new full owner. In Rust, the assignment operation for heap types works exactly the same way: `let b = a` moves ownership from `a` to `b`, and `a` becomes invalid.

---

## What is Move

Move (transfer of ownership) is the transfer of ownership from one variable to another. After a Move, the original variable is **invalid**: the compiler won't let you use it.

Important: Move is **not copying data**! The data stays in the same place on the heap. Only the responsibility for it changes.

```rust
fn main() {
    // s1 owns the string "BTCUSDT" on the heap
    let s1 = String::from("BTCUSDT");
    //      s1 on stack:
    //      [ ptr → heap, len=7, cap=7 ]
    //                ↓
    //          heap: "BTCUSDT"

    // Move: s2 takes ownership. s1 becomes an invalid pointer
    let s2 = s1;
    //      s1 on stack: [ INVALID ]
    //      s2 on stack: [ ptr → heap, len=7, cap=7 ]
    //                               ↓
    //                        heap: "BTCUSDT"  (SAME data, not a copy!)

    // println!("{}", s1); // ERROR: use of moved value: `s1`
    println!("s2 = {}", s2); // OK
}
```

---

## Move on Assignment

Move happens whenever you assign types that don't implement `Copy`:

```rust
fn main() {
    // Vec<f64> — not Copy, so Move
    let prices = vec![42000.0, 43000.0, 44000.0];
    let prices2 = prices; // Move!

    // println!("{:?}", prices); // ERROR
    println!("prices2: {:?}", prices2);

    // Strings are also Move
    let order1 = String::from("BUY BTCUSDT 0.1");
    let order2 = order1; // Move!
    // println!("{}", order1); // ERROR
    println!("order2: {}", order2);

    // f64 — Copy, so copying
    let price1 = 42000.0_f64;
    let price2 = price1; // Copy!
    println!("price1={}, price2={}", price1, price2); // BOTH work
}
```

---

## Move into a Function

Passing an argument to a function is also a Move for non-`Copy` types:

```rust
fn print_portfolio(p: Vec<String>) {
    // p receives ownership
    println!("Portfolio ({} positions):", p.len());
    for ticker in &p {
        println!("  - {}", ticker);
    }
} // p is destroyed here

fn calculate_total(prices: Vec<f64>) -> f64 {
    // prices is moved here
    prices.iter().sum()
} // prices is destroyed here

fn main() {
    let my_portfolio = vec![
        String::from("BTCUSDT"),
        String::from("ETHUSDT"),
        String::from("SOLUSDT"),
    ];

    print_portfolio(my_portfolio); // Move into function
    // println!("{:?}", my_portfolio); // ERROR: moved

    let price_list = vec![42000.0, 3000.0, 150.0];
    let total = calculate_total(price_list); // Move
    // println!("{:?}", price_list); // ERROR: moved
    println!("Total value: ${}", total);
}
```

---

## Returning Back — Move from a Function

A function can return ownership back, "moving" the value through return:

```rust
fn add_timestamp(order: String) -> String {
    // order received via Move
    format!("[2024-01-15 10:30:00] {}", order)
    // New string moves to the caller
}

fn filter_profitable(trades: Vec<f64>) -> Vec<f64> {
    // trades received via Move, return a new Vec
    trades.into_iter().filter(|&t| t > 0.0).collect()
    // Vec moves to the caller
}

fn main() {
    let raw_order = String::from("BUY BTCUSDT 0.5 @ 42000");
    let stamped_order = add_timestamp(raw_order); // raw_order moved
    println!("Timestamped order: {}", stamped_order);

    let all_trades = vec![150.0, -50.0, 300.0, -100.0, 200.0];
    let profitable = filter_profitable(all_trades); // all_trades moved
    println!("Profitable trades: {:?}", profitable);
    println!("Count: {}", profitable.len());
}
```

---

## Partial Move of a Struct

You can move an individual field from a struct. After that, the entire struct is "partially moved" — some fields are accessible, some are not:

```rust
#[derive(Debug)]
struct Trade {
    ticker: String,    // String — not Copy
    price: f64,        // f64 — Copy
    quantity: f64,     // f64 — Copy
    notes: String,     // String — not Copy
}

fn main() {
    let trade = Trade {
        ticker: String::from("BTCUSDT"),
        price: 42000.0,
        quantity: 0.1,
        notes: String::from("Resistance breakout"),
    };

    // Move only ticker
    let ticker = trade.ticker; // Move of field ticker

    // trade.ticker is now inaccessible, but other fields are OK
    println!("Ticker: {}", ticker);
    println!("Price: {}", trade.price);        // f64 — Copy, OK
    println!("Quantity: {}", trade.quantity);  // f64 — Copy, OK

    // println!("{}", trade.ticker); // ERROR: moved
    // println!("{:?}", trade); // ERROR: struct partially moved

    // Move notes as well
    let notes = trade.notes; // Move
    println!("Notes: {}", notes);
    // Only price and quantity (Copy) remain accessible from trade
}
```

---

## Copy vs Move — The Key Difference

```rust
fn demo_copy_vs_move() {
    // === Copy types (stack-based, small) ===
    let price: f64 = 42000.0;
    let volume: u64 = 1000;
    let is_long: bool = true;
    let side: char = 'B';

    let p2 = price;    // Copy — bitwise copy from stack
    let v2 = volume;   // Copy
    let b2 = is_long;  // Copy
    let s2 = side;     // Copy

    // ALL originals alive:
    println!("price={}, p2={}", price, p2);
    println!("volume={}, v2={}", volume, v2);

    // === Move types (heap, large, non-Copy) ===
    let ticker = String::from("BTCUSDT");
    let history = vec![1.0, 2.0, 3.0];
    let big_data: Box<[f64; 100]> = Box::new([0.0; 100]);

    let t2 = ticker;   // Move — pointer transfers from ticker to t2
    let h2 = history;  // Move
    let d2 = big_data; // Move

    // Originals INACCESSIBLE:
    // println!("{}", ticker); // ERROR
    // println!("{:?}", history); // ERROR

    println!("t2={}", t2);
    println!("h2 len={}", h2.len());
    println!("d2 len={}", d2.len());
}

fn main() {
    demo_copy_vs_move();
}
```

---

## When Move Is a Problem and How to Solve It

Move gets in the way when you need to use a value multiple times. There are three solutions:

```rust
fn process(data: Vec<f64>) -> f64 {
    data.iter().sum()
}

fn main() {
    let prices = vec![42000.0, 43000.0, 44000.0];

    // PROBLEM: after first call, prices is moved
    // let total1 = process(prices);
    // let total2 = process(prices); // ERROR!

    // SOLUTION 1: Clone — expensive (copies data)
    let total1 = process(prices.clone());
    let total2 = process(prices.clone()); // prices still alive
    println!("total1={}, total2={}", total1, total2);

    // SOLUTION 2: Pass reference &Vec instead of Vec
    fn process_ref(data: &Vec<f64>) -> f64 {
        data.iter().sum()
    }
    let ref_total1 = process_ref(&prices);
    let ref_total2 = process_ref(&prices); // prices lives!
    println!("ref totals: {} {}", ref_total1, ref_total2);

    // SOLUTION 3: Return ownership (tuple)
    fn process_and_return(data: Vec<f64>) -> (Vec<f64>, f64) {
        let sum: f64 = data.iter().sum();
        (data, sum)
    }
    let (prices_back, sum) = process_and_return(prices);
    println!("prices back: {:?}, sum={}", prices_back, sum);
}
```

---

## Practical Example: Order Processing Pipeline

```rust
#[derive(Debug)]
struct RawOrder {
    symbol: String,
    price: f64,
    quantity: f64,
    side: String,
}

#[derive(Debug)]
struct ValidatedOrder {
    symbol: String,
    price: f64,
    quantity: f64,
    side: String,
    notional: f64,
}

#[derive(Debug)]
struct ExecutedOrder {
    symbol: String,
    fill_price: f64,
    quantity: f64,
    side: String,
    notional: f64,
    order_id: u64,
}

// Each function consumes (Move) the order and returns the next stage
fn validate(order: RawOrder) -> Result<ValidatedOrder, String> {
    if order.price <= 0.0 {
        return Err(format!("Invalid price: {}", order.price));
    }
    if order.quantity <= 0.0 {
        return Err(format!("Invalid quantity: {}", order.quantity));
    }
    let notional = order.price * order.quantity;
    Ok(ValidatedOrder {
        symbol: order.symbol,    // Move field
        price: order.price,      // Copy
        quantity: order.quantity, // Copy
        side: order.side,        // Move
        notional,
    })
}

fn execute(order: ValidatedOrder) -> ExecutedOrder {
    let slippage = 0.0005;
    let fill_price = if order.side == "BUY" {
        order.price * (1.0 + slippage)
    } else {
        order.price * (1.0 - slippage)
    };

    ExecutedOrder {
        symbol: order.symbol,     // Move
        fill_price,
        quantity: order.quantity, // Copy
        side: order.side,         // Move
        notional: order.notional, // Copy
        order_id: 1001,
    }
}

fn main() {
    let raw = RawOrder {
        symbol: String::from("BTCUSDT"),
        price: 42000.0,
        quantity: 0.1,
        side: String::from("BUY"),
    };

    println!("Raw order: {:?}", raw);

    match validate(raw) { // raw is moved
        Ok(validated) => {
            println!("Validated: {:?}", validated);
            let executed = execute(validated); // validated is moved
            println!("Executed: {:?}", executed);
            println!(
                "Order #{}: {} {} {} @ ${:.2} (filled @ ${:.2})",
                executed.order_id,
                executed.side,
                executed.quantity,
                executed.symbol,
                executed.notional / executed.quantity,
                executed.fill_price
            );
        }
        Err(e) => println!("Validation error: {}", e),
    }
}
```

---

## What We Learned

| Syntax | Description | Trading Example |
|--------|-------------|-----------------|
| `let b = a` (String/Vec) | Move: a is invalid | Selling asset to buyer |
| `fn f(x: T)` | Move into function | Sending order for execution |
| `fn f() -> T` | Move out of function | Receiving executed order |
| `let (a, b) = f(x)` | Move and get back | Order + execution result |
| `trade.field` (String) | Partial field Move | Transfer one asset from struct |
| `.clone()` | Avoid Move via copy | Duplicate portfolio for testing |

---

## Homework

1. Write a function `sell_asset(portfolio: Vec<String>, symbol: &str) -> (Vec<String>, bool)`:
   - Takes ownership of the vector
   - Removes the first element matching `symbol`
   - Returns a tuple `(new_vector, was_sold)`
   - Confirm the original vector is inaccessible after the call

2. Create a Move chain:
   - `let a = String::from("ORDER")` → move to `b` → move to `c` → pass to function
   - Confirm that only `c` (then the function) can use the value

3. Implement partial move:
   - Struct `Position { symbol: String, price: f64, notes: String }`
   - Extract `symbol` and `notes` separately via Move
   - Show that `price` (Copy) is still accessible after moving the strings

4. Compare three approaches to "compute moving average and keep the data":
   - With `.clone()` before calling
   - With `&Vec` as function parameter
   - With returning `(Vec<f64>, Vec<f64>)` tuple

---

## Navigation

[← Previous day](../033-ownership-who-holds-asset/en.md) | [Next day →](../035-clone-copying-portfolio/en.md)
