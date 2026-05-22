# Day 205: Connecting to Price Stream

## Trading Analogy

A price stream is a continuous flow of quotes from the exchange, like a ticker tape in the old days. Connect once and receive real-time updates without constant requests.

## Tick Data Structure

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct BinanceTicker {
    #[serde(rename = "s")]
    symbol: String,
    #[serde(rename = "c")]
    price: String,
    #[serde(rename = "v")]
    volume: String,
    #[serde(rename = "P")]
    price_change_pct: String,
}
```

## Connect and Parse Stream

```rust
use tokio_tungstenite::connect_async;
use futures_util::StreamExt;
use serde_json;

#[tokio::main]
async fn main() {
    let url = "wss://stream.binance.com:9443/ws/btcusdt@ticker";

    let (ws_stream, _) = connect_async(url).await.expect("Connection error");
    println!("Connected to BTCUSDT stream");

    let (_write, mut read) = ws_stream.split();

    let mut tick_count = 0;
    while let Some(Ok(msg)) = read.next().await {
        if let Ok(text) = msg.to_text() {
            if let Ok(ticker) = serde_json::from_str::<BinanceTicker>(text) {
                tick_count += 1;
                let price: f64 = ticker.price.parse().unwrap_or(0.0);
                let pct: f64   = ticker.price_change_pct.parse().unwrap_or(0.0);
                println!("Tick #{}: {} = ${:.2} ({:+.2}%)", tick_count, ticker.symbol, price, pct);
            }
        }
        if tick_count >= 5 { break; }
    }
}
```

## What We Learned

| Concept | Description |
|---------|-------------|
| Stream URL | `wss://stream.binance.com:9443/ws/{symbol}@ticker` |
| JSON deserialization | `serde_json::from_str::<T>()` |
| Combined stream | Multiple symbols via `/` |
| Graceful shutdown | `tokio::select!` with Ctrl+C |

## Homework

1. Connect to ETHUSDT stream and print only price and 24h change
2. Connect to two symbols via combined stream
3. Add logic: if price rises 1% — print "ALERT: pump detected!"
4. Implement Ctrl+C handling for clean shutdown

## Navigation

[← Previous day](../204-tokio-tungstenite-websocket-client/en.md) | [Next day →](../206-ping-pong-connection-alive/en.md)
