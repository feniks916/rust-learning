# Day 157: Sender and Receiver — Who Sends, Who Receives

## Trading Analogy

The exchange is the Sender: it continuously sends tickers with prices. Your trading algorithm is the Receiver: it sits and waits for new data, processing each tick in turn. The channel between them is a message queue. That is exactly how `std::sync::mpsc` works in Rust: one or more senders, one receiver.

## mpsc::channel()

`mpsc` stands for **m**ulti-**p**roducer, **s**ingle-**c**onsumer. The `channel()` function creates a (Sender, Receiver) pair:

```rust
use std::sync::mpsc;

fn main() {
    // channel() returns a tuple (sender, receiver)
    let (tx, rx) = mpsc::channel::<String>();
    //   ^^  ^^
    //   tx = transmitter (sender)
    //   rx = receiver

    println!("Channel created!");
    // tx — sends messages into the channel
    // rx — receives messages from the channel
}
```

By convention the sender is called `tx` (transmit) and the receiver `rx` (receive).

## Sender<T>

`Sender<T>` is a typed sender. The `send()` method places a value into the channel:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    thread::spawn(move || {
        // Send BTC prices
        let prices = [65000.0, 65100.0, 64900.0, 65200.0];
        for price in prices {
            tx.send(price).expect("Send error");
            println!("Price sent: {:.2}", price);
        }
        // When tx goes out of scope — the channel closes
    });

    // Receiver receives in the main thread
    for price in rx {
        println!("Price received: {:.2}", price);
    }
}
```

`send()` returns `Result<(), SendError<T>>` — an error occurs only if the Receiver has been dropped.

## Receiver<T>

`Receiver<T>` is the message receiver. The primary method is `recv()`, which blocks the thread until a message arrives:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    thread::spawn(move || {
        let orders = vec!["BUY BTCUSDT 0.1", "SELL ETHUSDT 1.0", "BUY BNBUSDT 5.0"];
        for order in orders {
            tx.send(order.to_string()).unwrap();
        }
    });

    // recv() blocks until a message arrives
    loop {
        match rx.recv() {
            Ok(order) => println!("Executing order: {}", order),
            Err(_) => {
                println!("Channel closed, exiting");
                break;
            }
        }
    }
}
```

`recv()` returns `Err` when all senders are dropped and the channel is empty.

## send() and recv()

A closer look at return types:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    let handle = thread::spawn(move || {
        // send returns Result — handle the error
        match tx.send(42000.0) {
            Ok(()) => println!("Price sent"),
            Err(e) => println!("Error: receiver closed ({})", e),
        }

        // .unwrap() — when error handling is not needed
        tx.send(43000.0).unwrap();
    });

    // recv() blocks, waiting for the first message
    let price1 = rx.recv().unwrap();
    println!("Received 1: {:.2}", price1);

    let price2 = rx.recv().unwrap();
    println!("Received 2: {:.2}", price2);

    handle.join().unwrap();
}
```

## for msg in rx — Iteration

The most convenient way to receive messages until the channel closes:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<(String, f64)>();

    thread::spawn(move || {
        let ticks = vec![
            ("BTCUSDT".to_string(), 65000.0),
            ("ETHUSDT".to_string(), 3200.0),
            ("BNBUSDT".to_string(), 420.0),
        ];
        for tick in ticks {
            tx.send(tick).unwrap();
        }
        // Thread ends, tx is dropped, channel closes
    });

    // for ends automatically when the channel is closed
    for (ticker, price) in rx {
        println!("Ticker: {}, price: {:.2}", ticker, price);
    }
    println!("All tickers processed");
}
```

`for msg in rx` is syntactic sugar for `while let Ok(msg) = rx.recv()`.

## recv_timeout()

Sometimes you want to wait for a message for a limited time, not forever:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    thread::spawn(move || {
        thread::sleep(Duration::from_millis(200));
        tx.send(65500.0).unwrap();
    });

    // Wait for a message for at most 100ms
    match rx.recv_timeout(Duration::from_millis(100)) {
        Ok(price) => println!("Price received: {:.2}", price),
        Err(mpsc::RecvTimeoutError::Timeout) => {
            println!("Timeout! Exchange did not respond within 100ms");
        }
        Err(mpsc::RecvTimeoutError::Disconnected) => {
            println!("Channel closed");
        }
    }

    // Wait another 200ms — now the message will arrive
    match rx.recv_timeout(Duration::from_millis(200)) {
        Ok(price) => println!("Price received: {:.2}", price),
        Err(e) => println!("Error: {:?}", e),
    }
}
```

## try_recv() — Non-blocking

`try_recv()` does not block — it returns immediately:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    thread::spawn(move || {
        thread::sleep(Duration::from_millis(50));
        tx.send(65800.0).unwrap();
    });

    // Poll without blocking
    for i in 0..5 {
        match rx.try_recv() {
            Ok(price) => {
                println!("Iteration {}: received price {:.2}", i, price);
                break;
            }
            Err(mpsc::TryRecvError::Empty) => {
                println!("Iteration {}: no data yet, continuing", i);
                thread::sleep(Duration::from_millis(20));
            }
            Err(mpsc::TryRecvError::Disconnected) => {
                println!("Channel closed");
                break;
            }
        }
    }
}
```

## Typed Enum Messages

A channel can carry different message types via an enum:

```rust
use std::sync::mpsc;
use std::thread;

#[derive(Debug)]
enum ExchangeMessage {
    Tick { symbol: String, price: f64, volume: f64 },
    OrderFilled { order_id: u64, fill_price: f64 },
    Error(String),
    Disconnect,
}

fn main() {
    let (tx, rx) = mpsc::channel::<ExchangeMessage>();

    thread::spawn(move || {
        tx.send(ExchangeMessage::Tick {
            symbol: "BTCUSDT".to_string(),
            price: 65000.0,
            volume: 12.5,
        }).unwrap();

        tx.send(ExchangeMessage::OrderFilled {
            order_id: 12345,
            fill_price: 65010.0,
        }).unwrap();

        tx.send(ExchangeMessage::Disconnect).unwrap();
    });

    for msg in rx {
        match msg {
            ExchangeMessage::Tick { symbol, price, volume } => {
                println!("Tick {}: price={:.2}, volume={:.2}", symbol, price, volume);
            }
            ExchangeMessage::OrderFilled { order_id, fill_price } => {
                println!("Order #{} filled at {:.2}", order_id, fill_price);
            }
            ExchangeMessage::Error(msg) => {
                println!("ERROR: {}", msg);
            }
            ExchangeMessage::Disconnect => {
                println!("Connection closed");
                break;
            }
        }
    }
}
```

## Practical Example: Exchange Stream via Channel

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
use std::collections::HashMap;

#[derive(Debug)]
enum MarketEvent {
    PriceUpdate { symbol: String, price: f64, change_pct: f64 },
    VolumeSpike  { symbol: String, volume: f64 },
    Connected,
    Disconnected,
}

fn simulate_exchange(tx: mpsc::Sender<MarketEvent>) {
    tx.send(MarketEvent::Connected).unwrap();

    let data = vec![
        ("BTCUSDT", 65000.0, 0.5,  1200.0),
        ("ETHUSDT", 3200.0,  -0.3, 450.0),
        ("BNBUSDT", 420.0,   1.2,  8500.0), // large volume!
        ("BTCUSDT", 65300.0, 0.46, 980.0),
        ("ETHUSDT", 3190.0,  -0.6, 520.0),
    ];

    for (symbol, price, change, volume) in data {
        thread::sleep(Duration::from_millis(50));

        tx.send(MarketEvent::PriceUpdate {
            symbol: symbol.to_string(),
            price,
            change_pct: change,
        }).unwrap();

        if volume > 5000.0 {
            tx.send(MarketEvent::VolumeSpike {
                symbol: symbol.to_string(),
                volume,
            }).unwrap();
        }
    }

    tx.send(MarketEvent::Disconnected).unwrap();
}

fn main() {
    let (tx, rx) = mpsc::channel::<MarketEvent>();

    let producer = thread::spawn(move || {
        simulate_exchange(tx);
    });

    let mut prices: HashMap<String, f64> = HashMap::new();
    let mut event_count = 0usize;

    for event in rx {
        event_count += 1;
        match event {
            MarketEvent::Connected => {
                println!("=== Connected to exchange ===");
            }
            MarketEvent::PriceUpdate { symbol, price, change_pct } => {
                let arrow = if change_pct >= 0.0 { "↑" } else { "↓" };
                println!(
                    "{} {} {:.2} ({:+.2}%)",
                    symbol, arrow, price, change_pct
                );
                prices.insert(symbol, price);
            }
            MarketEvent::VolumeSpike { symbol, volume } => {
                println!("!!! VOLUME SPIKE: {} — {:.0} USDT", symbol, volume);
            }
            MarketEvent::Disconnected => {
                println!("=== Connection closed ===");
                break;
            }
        }
    }

    producer.join().unwrap();

    println!("\n--- Summary ---");
    println!("Events processed: {}", event_count);
    println!("Latest prices:");
    let mut sorted: Vec<_> = prices.iter().collect();
    sorted.sort_by_key(|(k, _)| k.as_str());
    for (sym, price) in sorted {
        println!("  {}: ${:.2}", sym, price);
    }
}
```

## What We Learned

| Syntax | Description | Use Case |
|--------|-------------|----------|
| `mpsc::channel::<T>()` | Create a channel of type T | Create a message queue |
| `tx.send(value)` | Send a value | Publish price, order |
| `rx.recv()` | Blocking receive | Wait for the next tick |
| `for msg in rx` | Iterate until channel closes | Process entire stream |
| `rx.recv_timeout(dur)` | Wait with timeout | Don't wait forever for exchange |
| `rx.try_recv()` | Non-blocking check | Poll while doing other work |
| `enum` messages | Different types in one channel | Different exchange events |

## Homework

1. Create an enum `TradingCommand` with variants:
   - `Buy { symbol: String, quantity: f64, max_price: f64 }`
   - `Sell { symbol: String, quantity: f64, min_price: f64 }`
   - `Cancel { order_id: u64 }`
   - `Shutdown`

2. Write an executor thread that receives commands via `Receiver<TradingCommand>`:
   - For `Buy` — print "Buying {qty} {symbol} at max {max_price}"
   - For `Sell` — print "Selling {qty} {symbol} at min {min_price}"
   - For `Cancel` — "Cancelling order #{id}"
   - For `Shutdown` — break the loop

3. In `main` send 4-5 different commands via `Sender<TradingCommand>`, including `Shutdown` last

4. Add `recv_timeout(Duration::from_millis(500))` in the executor:
   - If no command arrives in 500ms — print "Waiting for commands..."

5. Count how many commands of each type were executed and print statistics at the end

## Navigation

[← Previous day](../156-channels-mpsc-order-queue/en.md) | [Next day →](../158-multiple-senders/en.md)
