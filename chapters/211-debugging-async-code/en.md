# Day 211: Debugging Async Code

## Trading Analogy

A bug in a trading bot can cost money. Debugging async code is harder than sync: tasks run in parallel, call stacks are non-linear. Special tools are needed.

## Logging in Async Code

```rust
use tracing::{info, warn, error, instrument};
use tokio::time::{sleep, Duration};

#[instrument]
async fn fetch_price(symbol: &str) -> Result<f64, String> {
    info!("Requesting price for {}", symbol);
    sleep(Duration::from_millis(100)).await;

    if symbol == "INVALID" {
        warn!("Unknown symbol: {}", symbol);
        return Err(format!("Unknown symbol: {}", symbol));
    }

    let price = 50000.0;
    info!("Price received: ${}", price);
    Ok(price)
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    match fetch_price("BTCUSDT").await {
        Ok(p)  => info!("Success: ${}", p),
        Err(e) => error!("Error: {}", e),
    }
}
```

## Timeouts for Detecting Hangs

```rust
use tokio::time::{timeout, Duration};

async fn slow_operation() -> f64 {
    tokio::time::sleep(Duration::from_secs(10)).await;
    42.0
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_secs(2), slow_operation()).await {
        Ok(result) => println!("Result: {}", result),
        Err(_)     => println!("Timeout! Operation hung"),
    }
}
```

## Don't Hold Lock Across await

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let mutex = Arc::new(Mutex::new(0_u64));
    let m1 = mutex.clone();

    tokio::spawn(async move {
        let mut guard = m1.lock().await;
        *guard += 1;
        // IMPORTANT: don't hold lock across await!
        // sleep(Duration::from_secs(1)).await; // DEADLOCK risk
        println!("Value: {}", *guard);
    }).await.unwrap();
}
```

## What We Learned

| Tool | Description |
|------|-------------|
| `tokio-console` | Visual task monitor |
| `#[instrument]` | Auto spans in tracing |
| Don't hold lock across await | Prevent deadlocks |
| `timeout()` | Guard against hanging ops |

## Homework

1. Add `tracing` to a project, use `info!`, `warn!`, `error!` in async functions
2. Use `#[instrument]` on an async function and examine the output
3. Implement slow operation detection via `timeout` with logging
4. Create an intentional deadlock and try to diagnose it

## Navigation

[← Previous day](../210-graceful-shutdown-async-tasks/en.md) | [Next day →](../212-realtime-price-monitor/en.md)
