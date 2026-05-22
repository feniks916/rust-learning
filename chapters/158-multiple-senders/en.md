# Day 158: Multiple Senders

## Trading Analogy

Your price aggregator is connected to several exchanges at once: Binance, Bybit, OKX. Each exchange is a separate thread, each sending quotes. But you have one algorithm processing all of it. That is exactly the architecture `mpsc` was built for: **m**ulti-**p**roducer (many senders), **s**ingle-**c**onsumer (one receiver).

## tx.clone()

`Sender<T>` can be cloned — each clone sends into the same channel:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    // Clone the sender for the second thread
    let tx2 = tx.clone();

    // First thread — Binance
    thread::spawn(move || {
        tx.send("Binance: BTCUSDT 65000".to_string()).unwrap();
        tx.send("Binance: ETHUSDT 3200".to_string()).unwrap();
    });

    // Second thread — Bybit (uses the clone)
    thread::spawn(move || {
        tx2.send("Bybit: BTCUSDT 65050".to_string()).unwrap();
        tx2.send("Bybit: ETHUSDT 3205".to_string()).unwrap();
    });

    for msg in rx {
        println!("Aggregator: {}", msg);
    }
}
```

All clones of `tx` and the original are equal — they all write into the same buffer.

## drop(tx) of the Original

The channel closes only when ALL senders are dropped. If you created clones, remember to drop the original:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<i32>();

    let tx1 = tx.clone();
    let tx2 = tx.clone();

    // Drop the original immediately — we don't need it in main
    drop(tx);

    thread::spawn(move || {
        tx1.send(1).unwrap();
        // tx1 is dropped when the thread exits
    });

    thread::spawn(move || {
        tx2.send(2).unwrap();
        // tx2 is dropped when the thread exits
    });

    // Channel closes only after drop(tx) AND tx1, tx2 are both dropped
    for val in rx {
        println!("Received: {}", val);
    }
    println!("Channel closed (all senders dropped)");
}
```

A common mistake: forgetting `drop(tx)` in main — then `for val in rx` will wait forever.

## Multiple Sender Threads

Create three threads, each with a clone of `tx`:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<(String, f64)>();

    let exchanges = vec![
        ("Binance", 65000.0f64),
        ("Bybit",   65050.0f64),
        ("OKX",     64980.0f64),
    ];

    let mut handles = Vec::new();
    for (exchange, price) in exchanges {
        let tx_clone = tx.clone();
        let handle = thread::spawn(move || {
            tx_clone.send((exchange.to_string(), price)).unwrap();
            println!("[{}] Price sent: {:.2}", exchange, price);
        });
        handles.push(handle);
    }
    drop(tx); // Important! Otherwise for..in never ends

    for (exchange, price) in rx {
        println!("Aggregator received: {} @ {:.2}", exchange, price);
    }

    for h in handles { h.join().unwrap(); }
}
```

## Message Order

The order of messages from different threads is not guaranteed — whoever sends first arrives first:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    // First thread sends with a delay
    let tx1 = tx.clone();
    thread::spawn(move || {
        thread::sleep(Duration::from_millis(20));
        tx1.send("Binance (slow)".to_string()).unwrap();
    });

    // Second thread sends immediately
    let tx2 = tx.clone();
    thread::spawn(move || {
        tx2.send("Bybit (fast)".to_string()).unwrap();
    });

    drop(tx);

    for msg in rx {
        println!("Arrived: {}", msg);
    }
    // Output: "Bybit (fast)" first, then "Binance (slow)"
}
```

If order matters — include a timestamp in each message.

## Data Aggregation

Typical pattern: collect the best price from multiple sources:

```rust
use std::sync::mpsc;
use std::thread;

#[derive(Debug)]
struct Quote {
    exchange: String,
    symbol: String,
    bid: f64,
    ask: f64,
}

fn main() {
    let (tx, rx) = mpsc::channel::<Quote>();

    let quotes_data = vec![
        Quote { exchange: "Binance".into(), symbol: "BTCUSDT".into(), bid: 64990.0, ask: 65000.0 },
        Quote { exchange: "Bybit".into(),   symbol: "BTCUSDT".into(), bid: 65010.0, ask: 65020.0 },
        Quote { exchange: "OKX".into(),     symbol: "BTCUSDT".into(), bid: 64980.0, ask: 64995.0 },
    ];

    for quote in quotes_data {
        let tx_c = tx.clone();
        thread::spawn(move || {
            tx_c.send(quote).unwrap();
        });
    }
    drop(tx);

    let mut best_bid = 0.0f64;
    let mut best_ask = f64::MAX;
    let mut best_bid_exchange = String::new();
    let mut best_ask_exchange = String::new();

    for q in rx {
        println!("{}: bid={:.2}, ask={:.2}", q.exchange, q.bid, q.ask);
        if q.bid > best_bid {
            best_bid = q.bid;
            best_bid_exchange = q.exchange.clone();
        }
        if q.ask < best_ask {
            best_ask = q.ask;
            best_ask_exchange = q.exchange.clone();
        }
    }

    println!("\nBest bid: {:.2} ({})", best_bid, best_bid_exchange);
    println!("Best ask: {:.2} ({})", best_ask, best_ask_exchange);
    println!("Spread:   {:.2}", best_ask - best_bid);
}
```

## Practical Example: Multiple Exchanges into One Aggregator

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Tick {
    exchange: String,
    symbol: String,
    price: f64,
    volume: f64,
    timestamp_ms: u64,
}

fn run_exchange_feed(
    name: &'static str,
    base_prices: Vec<(&'static str, f64)>,
    tx: mpsc::Sender<Tick>,
    delay_ms: u64,
) {
    thread::sleep(Duration::from_millis(delay_ms));

    for (symbol, base_price) in base_prices {
        for i in 0..3u64 {
            let price = base_price * (1.0 + (i as f64 * 0.001));
            let tick = Tick {
                exchange: name.to_string(),
                symbol: symbol.to_string(),
                price,
                volume: 10.0 + i as f64 * 5.0,
                timestamp_ms: delay_ms + i * 50,
            };
            tx.send(tick).unwrap();
            thread::sleep(Duration::from_millis(50));
        }
    }
    println!("[{}] Feed finished", name);
}

fn main() {
    let (tx, rx) = mpsc::channel::<Tick>();

    let feeds: Vec<(&'static str, Vec<(&'static str, f64)>, u64)> = vec![
        ("Binance", vec![("BTCUSDT", 65000.0), ("ETHUSDT", 3200.0)], 0),
        ("Bybit",   vec![("BTCUSDT", 65020.0), ("ETHUSDT", 3195.0)], 10),
        ("OKX",     vec![("BTCUSDT", 64990.0), ("ETHUSDT", 3210.0)], 20),
    ];

    let mut handles = Vec::new();
    for (name, prices, delay) in feeds {
        let tx_clone = tx.clone();
        let handle = thread::spawn(move || {
            run_exchange_feed(name, prices, tx_clone, delay);
        });
        handles.push(handle);
    }
    drop(tx); // Close the original

    // Aggregator
    let mut price_map: HashMap<String, Vec<(String, f64)>> = HashMap::new();
    let mut tick_count = 0usize;

    for tick in rx {
        tick_count += 1;
        println!(
            "[{}] {} {} = {:.2} (vol: {:.1})",
            tick.exchange, tick.symbol, tick.timestamp_ms, tick.price, tick.volume
        );

        price_map
            .entry(tick.symbol.clone())
            .or_default()
            .push((tick.exchange.clone(), tick.price));
    }

    for h in handles { h.join().unwrap(); }

    println!("\n=== AGGREGATED DATA ===");
    println!("Total ticks received: {}", tick_count);

    let mut symbols: Vec<_> = price_map.keys().collect();
    symbols.sort();

    for symbol in symbols {
        let prices = &price_map[symbol];
        let avg = prices.iter().map(|(_, p)| p).sum::<f64>() / prices.len() as f64;
        let min = prices.iter().map(|(_, p)| *p).fold(f64::MAX, f64::min);
        let max = prices.iter().map(|(_, p)| *p).fold(f64::MIN, f64::max);

        println!("\n{}:", symbol);
        println!("  Average price: {:.2}", avg);
        println!("  Range:         {:.2} — {:.2}", min, max);
        println!("  Spread:        {:.2}", max - min);
        for (exchange, price) in prices.iter().rev().take(1) {
            println!("  Latest:        {:.2} ({})", price, exchange);
        }
    }
}
```

## What We Learned

| Syntax | Description | Use Case |
|--------|-------------|----------|
| `tx.clone()` | Create another sender | Multiple threads into one channel |
| `drop(tx)` | Drop the original Sender | Allow channel to close properly |
| `for msg in rx` | Ends when all tx are dropped | Process until end of stream |
| Message order | Not guaranteed across threads | Use timestamp for sorting |
| Aggregation | One rx, many tx | Collect data from multiple exchanges |

## Homework

1. Create a struct `OrderBook` with fields: `exchange: String`, `symbol: String`, `bids: Vec<f64>`, `asks: Vec<f64>`

2. Launch 3 "exchange" threads, each creating and sending an `OrderBook` via a clone of `tx`:
   - Binance: BTCUSDT bids=[64990, 64980], asks=[65000, 65010]
   - Bybit: BTCUSDT bids=[65010, 65000], asks=[65020, 65030]
   - OKX: BTCUSDT bids=[64980, 64970], asks=[64995, 65005]

3. Do not forget `drop(tx)` after creating all clones

4. Aggregate the received data:
   - Find the best bid (maximum) across all exchanges
   - Find the best ask (minimum) across all exchanges
   - Check whether arbitrage is possible (best bid > best ask)

5. Add delays between sends (10-50 ms) and verify that the receive order does not match the thread creation order

## Navigation

[← Previous day](../157-sender-and-receiver/en.md) | [Next day →](../159-sync-channel-bounded-queue/en.md)
