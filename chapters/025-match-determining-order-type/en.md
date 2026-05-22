# Day 25: match — Determining Order Type

## Trading Analogy

Orders come in many types: market, limit, stop, stop-limit, trailing stop. Each is handled differently — a market order executes immediately at the best available price, a limit order waits for a specific level, a stop order activates on a breakout. The `match` operator is like a smart exchange dispatcher: it looks at the order type and routes it to the right handler, guaranteeing that every possible case is covered. If you forget to describe some type, the Rust compiler will tell you before the program even runs.

## Basic match: Order Type Dispatcher

The simplest `match` — comparing a string against patterns:

```rust
fn describe_order(order_type: &str) {
    match order_type {
        "market"     => println!("Market: execute immediately at best price"),
        "limit"      => println!("Limit: execute only at stated price or better"),
        "stop"       => println!("Stop: become market order when level is hit"),
        "stop_limit" => println!("Stop-limit: become limit order when level is hit"),
        "trailing"   => println!("Trailing: follow price with a fixed step"),
        _            => println!("Unknown type: '{}' — rejected", order_type),
    }
}

fn main() {
    describe_order("market");
    describe_order("limit");
    describe_order("stop");
    describe_order("foo");
}
```

The `_` symbol is the "catch-all" pattern, mandatory when not all possible values are listed. Without it, Rust won't compile.

## match with Numbers and Ranges

`match` can compare numbers, including inclusive ranges with `..=`:

```rust
fn classify_pnl(pnl: i32) -> &'static str {
    match pnl {
        i32::MIN..=-1000 => "Catastrophic loss (> -$1000)",
        -999..=-100      => "Significant loss",
        -99..=-1         => "Small loss",
        0                => "Breakeven",
        1..=99           => "Small profit",
        100..=999        => "Good profit",
        1000..=i32::MAX  => "Excellent profit (> +$1000)",
    }
}

fn classify_volume(volume: u64) -> &'static str {
    match volume {
        0              => "No volume",
        1..=100        => "Retail trader",
        101..=1_000    => "Mid-size trader",
        1_001..=10_000 => "Large trader",
        _              => "Institutional trader",
    }
}

fn main() {
    let test_pnls = [-2000, -500, -50, 0, 50, 500, 2000];
    println!("P&L classification:");
    for pnl in test_pnls {
        println!("  {:>6}: {}", pnl, classify_pnl(pnl));
    }

    println!("\nVolume classification:");
    for vol in [0u64, 50, 500, 5000, 50000] {
        println!("  {:>6}: {}", vol, classify_volume(vol));
    }
}
```

Ranges in `match` are inclusive (`..=`). Important: ranges must not overlap.

## match Returns a Value

`match` is an expression — it can return a value. All branches must return the same type:

```rust
fn get_commission_rate(order_type: &str, is_maker: bool) -> f64 {
    match (order_type, is_maker) {
        ("market", _)     => 0.001,   // 0.10% always
        ("limit", true)   => 0.0002,  // 0.02% maker
        ("limit", false)  => 0.0006,  // 0.06% taker
        ("stop", _)       => 0.001,
        _                 => 0.001,
    }
}

fn get_order_priority(order_type: &str) -> u8 {
    let priority = match order_type {
        "market" => 1,   // highest priority
        "stop"   => 2,
        "limit"  => 3,
        _        => 99,  // unknown — low priority
    };
    priority
}

fn main() {
    let amount = 10000.0;
    let types  = [("market", false), ("limit", true), ("limit", false), ("stop", false)];

    println!("Fees for ${:.0} trade:", amount);
    for (order_type, is_maker) in types {
        let rate  = get_commission_rate(order_type, is_maker);
        let fee   = amount * rate;
        let label = if is_maker { "maker" } else { "taker" };
        println!("  {:<8} ({:<5}): {:.4}% = ${:.4}", order_type, label, rate * 100.0, fee);
    }
}
```

## Multiple Patterns with |

The `|` operator combines multiple patterns into one branch — like a logical "or":

```rust
fn trading_session(hour: u8) -> &'static str {
    match hour {
        0..=6             => "Asian night (low volatility)",
        7 | 8             => "London open (volatility rising)",
        9 | 10 | 11       => "Active London session",
        12 | 13           => "London lunch break",
        14 | 15           => "New York open + overlap",
        16 | 17 | 18 | 19 => "Active New York session",
        20 | 21           => "New York close",
        22 | 23           => "Asian evening",
        _                 => "Unknown hour",
    }
}

fn is_high_volatility_hour(hour: u8) -> bool {
    match hour {
        7 | 8 | 14 | 15 | 20 | 21 => true,   // open/close times
        _                          => false,
    }
}

fn main() {
    println!("Trading session schedule:");
    for hour in [0u8, 7, 9, 12, 14, 16, 20, 22] {
        let session  = trading_session(hour);
        let volatile = if is_high_volatility_hour(hour) { "volatile" } else { "calm" };
        println!("  {:02}:00 — {} ({})", hour, session, volatile);
    }
}
```

## Guards: if Condition Inside a Pattern

A guard is an extra `if` condition right inside the pattern, refining when the branch should fire:

```rust
fn analyze_trade(ticker: &str, pnl: f64) -> &'static str {
    match (ticker, pnl as i64) {
        ("BTC", p) if p > 500  => "BTC big win",
        ("BTC", p) if p > 0    => "BTC small win",
        ("BTC", _)             => "BTC loss",
        (t, p) if p > 100 && t.starts_with('E') => "ETH-family good profit",
        (_, p) if p > 0        => "Profitable trade",
        (_, p) if p == 0       => "Breakeven",
        _                      => "Losing trade",
    }
}

fn main() {
    let trades = vec![
        ("BTC",  800.0),
        ("BTC",  200.0),
        ("BTC", -150.0),
        ("ETH",  250.0),
        ("SOL",   50.0),
        ("ADA",    0.0),
        ("DOT",  -30.0),
    ];

    for (ticker, pnl) in trades {
        println!("  {:>6} {:>+8.2}: {}", ticker, pnl, analyze_trade(ticker, pnl));
    }
}
```

Guards let you express complex conditions not covered by range patterns. Branch order matters: the first match wins.

## match on a Tuple: Combining Conditions

`match` on a tuple lets you check multiple values at once:

```rust
fn order_action(side: &str, price_vs_market: &str) -> &'static str {
    // side: "buy" or "sell"
    // price_vs_market: "above", "at", "below"
    match (side, price_vs_market) {
        ("buy",  "below") => "Limit buy — good price, waiting",
        ("buy",  "at")    => "Market buy — execute immediately",
        ("buy",  "above") => "Buying above market — chasing a breakout?",
        ("sell", "above") => "Limit sell — good price, waiting",
        ("sell", "at")    => "Market sell — execute immediately",
        ("sell", "below") => "Selling below market — likely a stop",
        _                 => "Unknown combination",
    }
}

fn main() {
    let scenarios = [
        ("buy",  "below"),
        ("buy",  "at"),
        ("sell", "above"),
        ("sell", "below"),
    ];

    for (side, price_pos) in scenarios {
        println!("  {} / {}: {}", side, price_pos, order_action(side, price_pos));
    }
}
```

## Practical Order Dispatcher

Bringing it all together — a full dispatcher processing orders of different types:

```rust
#[derive(Debug)]
enum OrderSide { Buy, Sell }

#[derive(Debug)]
struct Order {
    id:         u32,
    ticker:     String,
    side:       OrderSide,
    order_type: String,
    price:      f64,
    quantity:   f64,
}

fn process_order(order: &Order, market_price: f64) -> String {
    let side_str = match order.side {
        OrderSide::Buy  => "BUY",
        OrderSide::Sell => "SELL",
    };

    match order.order_type.as_str() {
        "market" => {
            let total = market_price * order.quantity;
            format!("#{} {} {} {:.2} units @ ${:.2} = ${:.2} [FILLED]",
                order.id, side_str, order.ticker, order.quantity, market_price, total)
        }
        "limit" => {
            let can_fill = match order.side {
                OrderSide::Buy  => market_price <= order.price,
                OrderSide::Sell => market_price >= order.price,
            };
            if can_fill {
                format!("#{} {} {} @ ${:.2} [FILLED]",
                    order.id, side_str, order.ticker, order.price)
            } else {
                format!("#{} {} {} waiting for ${:.2} (market ${:.2}) [PENDING]",
                    order.id, side_str, order.ticker, order.price, market_price)
            }
        }
        "stop" => {
            let triggered = match order.side {
                OrderSide::Buy  => market_price >= order.price,
                OrderSide::Sell => market_price <= order.price,
            };
            if triggered {
                format!("#{} {} {} STOP triggered at ${:.2} [ACTIVATED→MARKET]",
                    order.id, side_str, order.ticker, market_price)
            } else {
                format!("#{} {} {} stop level ${:.2} not reached [PENDING]",
                    order.id, side_str, order.ticker, order.price)
            }
        }
        unknown => format!("#{} Unknown type '{}' [REJECTED]", order.id, unknown),
    }
}

fn main() {
    let market_price = 51500.0;

    let orders = vec![
        Order { id: 1, ticker: "BTC".to_string(), side: OrderSide::Buy,  order_type: "market".to_string(), price: 0.0,     quantity: 0.1 },
        Order { id: 2, ticker: "BTC".to_string(), side: OrderSide::Buy,  order_type: "limit".to_string(),  price: 50000.0, quantity: 0.2 },
        Order { id: 3, ticker: "BTC".to_string(), side: OrderSide::Sell, order_type: "limit".to_string(),  price: 52000.0, quantity: 0.1 },
        Order { id: 4, ticker: "ETH".to_string(), side: OrderSide::Sell, order_type: "stop".to_string(),   price: 52000.0, quantity: 1.0 },
        Order { id: 5, ticker: "SOL".to_string(), side: OrderSide::Buy,  order_type: "oco".to_string(),    price: 0.0,     quantity: 10.0 },
    ];

    println!("BTC market price: ${:.2}", market_price);
    println!("{:-<65}", "");
    for order in &orders {
        println!("{}", process_order(order, market_price));
    }
}
```

## What We Learned

| Syntax | When to Use | Example |
|--------|-------------|---------|
| `match value { pat => expr }` | Handle several variants of a value | Order type → action |
| `_` (wildcard) | Required "catch-all" when not all cases are listed | Unknown type → reject |
| `a..=b` in match | Numeric ranges | `1000..=i32::MAX => "Big profit"` |
| `pat1 \| pat2` | Multiple values in one branch | `7 \| 8 \| 14 => true` (volatile hours) |
| Guard `if cond` | Extra condition inside a pattern | `("BTC", p) if p > 500 => ...` |
| `match (a, b)` on tuple | Combine multiple conditions | `("buy", "below") => "Limit buy"` |
| `match` as expression | Return a value from a branch | `let rate = match type { ... };` |

## Homework

1. Write a `fee_tier(monthly_volume: u64) -> f64` function returning the commission rate based on trading volume
   - Up to 10,000 → 0.1%
   - 10,001..100,000 → 0.08%
   - 100,001..1,000,000 → 0.06%
   - Above → 0.04%
   - Use `match` with ranges

2. Implement a market event dispatcher `handle_event(event: &str, value: f64) -> String`
   - "price_alert" + value > 55000 → "Price above target"
   - "price_alert" + value < 45000 → "Price below stop"
   - "volume_spike" + value > 10000 → "Abnormal volume"
   - Use `match` on tuple with guards

3. Create a `match` on position direction and profit state `(side, in_profit: bool)`:
   - (long, true) → "Holding long, in profit"
   - (long, false) → "Long in loss, consider exit"
   - And so on for short

4. Write a `day_type(day_num: u8) -> &str` function returning the type of day:
   - 1, 7 → "Weekend"
   - 2..=6 → "Weekday"
   - Use `|` to combine patterns

## Navigation

[← Previous day](../024-continue-skip-losing-trades/en.md) | [Next day →](../026-constants-fixed-exchange-fee/en.md)
