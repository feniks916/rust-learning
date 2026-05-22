# Day 204: tokio-tungstenite — WebSocket Client

## Trading Analogy

WebSocket is a persistent connection to the exchange, like a direct phone line. The `tokio-tungstenite` library is the async WebSocket client implementation for Rust.

## Cargo.toml Dependencies

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = { version = "0.20", features = ["native-tls"] }
futures-util = "0.3"
url = "2"
```

## Basic Connection

```rust
use tokio_tungstenite::connect_async;
use futures_util::StreamExt;
use url::Url;

#[tokio::main]
async fn main() {
    let url = Url::parse("wss://stream.binance.com:9443/ws/btcusdt@ticker")
        .expect("Invalid URL");

    let (ws_stream, response) = connect_async(url)
        .await
        .expect("Connection error");

    println!("Connected! Status: {}", response.status());

    let (_write, mut read) = ws_stream.split();

    let mut count = 0;
    while let Some(msg) = read.next().await {
        match msg {
            Ok(m) => {
                println!("Message: {}", m.to_text().unwrap_or("binary"));
                count += 1;
                if count >= 3 { break; }
            }
            Err(e) => { eprintln!("Error: {}", e); break; }
        }
    }
}
```

## What We Learned

| Component | Description |
|-----------|-------------|
| `connect_async(url)` | Connect to WebSocket |
| `ws_stream.split()` | Split into write and read |
| `read.next().await` | Receive next message |
| `write.send(msg)` | Send a message |
| `Message::Text(s)` | Text message |

## Homework

1. Connect to echo WebSocket server `wss://echo.websocket.org` and send a message
2. Implement subscription to 2 symbols simultaneously
3. Add connection error handling with user-friendly messages
4. Parse JSON from WebSocket and extract only the price field

## Navigation

[← Previous day](../203-websocket-streaming-data/en.md) | [Next day →](../205-connecting-to-price-stream/en.md)
