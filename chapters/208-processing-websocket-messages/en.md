# Day 208: Processing WebSocket Messages

## Trading Analogy

An exchange stream isn't just prices. Different message types arrive: ticks, aggregated trades, order book updates, system events. Each type needs proper handling.

## Different Message Types

```rust
use tokio_tungstenite::tungstenite::Message;
use futures_util::StreamExt;

async fn handle_text(text: &str) {
    if let Ok(value) = serde_json::from_str::<serde_json::Value>(text) {
        if value.get("e").map(|e| e == "24hrTicker").unwrap_or(false) {
            let price = value["c"].as_str().unwrap_or("0");
            println!("Ticker: {} = ${}", value["s"].as_str().unwrap_or("?"), price);
        } else if value.get("e").map(|e| e == "aggTrade").unwrap_or(false) {
            println!("Agg trade: {:?}", value["p"]);
        } else {
            println!("Other: {}", &text[..text.len().min(100)]);
        }
    }
}

// In the message loop:
// Ok(Message::Text(text)) => handle_text(&text).await,
// Ok(Message::Ping(_)) => {} // pong is automatic
// Ok(Message::Close(_)) => break,
// Err(e) => { eprintln!("Error: {}", e); break; }
```

## Message Buffer

```rust
use std::collections::VecDeque;

struct PriceBuffer {
    prices: VecDeque<f64>,
    capacity: usize,
}

impl PriceBuffer {
    fn new(capacity: usize) -> Self {
        PriceBuffer { prices: VecDeque::new(), capacity }
    }

    fn push(&mut self, price: f64) {
        if self.prices.len() >= self.capacity {
            self.prices.pop_front();
        }
        self.prices.push_back(price);
    }

    fn moving_average(&self) -> f64 {
        let sum: f64 = self.prices.iter().sum();
        sum / self.prices.len() as f64
    }
}
```

## What We Learned

| Message Type | Description |
|--------------|-------------|
| `Message::Text` | JSON or text |
| `Message::Binary` | Binary data |
| `Message::Ping/Pong` | Keepalive |
| `Message::Close` | Connection close |
| `serde_json::Value` | Dynamic JSON parsing |

## Homework

1. Write `parse_ticker(json: &str) -> Option<(String, f64)>` returning (symbol, price)
2. Implement a counter for each message type (Text/Binary/Ping/Close)
3. Add filter: only process messages with price above threshold
4. Implement buffer of last 10 prices with moving average calculation

## Navigation

[← Previous day](../207-reconnect-restoring-connection/en.md) | [Next day →](../209-parallel-websocket-subscriptions/en.md)
