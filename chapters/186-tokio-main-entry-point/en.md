# Day 186: #[tokio::main] — Entry Point

## Trading Analogy

A trading terminal needs to be started before work begins. `#[tokio::main]` is the start button for the async runtime: it initializes the engine and launches the main async function.

## Simplest Example

```rust
#[tokio::main]
async fn main() {
    println!("Async trading system started!");
}
```

## What #[tokio::main] Does

The macro expands to:

```rust
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            // your async main body
        });
}
```

## Working with async in main

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("Starting price monitor...");

    sleep(Duration::from_millis(100)).await;

    let price = fetch_price("BTCUSDT").await;
    println!("BTC price: ${}", price);
}

async fn fetch_price(symbol: &str) -> f64 {
    sleep(Duration::from_millis(50)).await;
    50000.0
}
```

## Runtime Configuration

```rust
// Single-threaded (simple tasks)
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("Single-threaded async");
}

// Multi-threaded (default)
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    println!("4 worker threads");
}
```

## Error Handling in async main

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let data = load_config().await?;
    println!("Config loaded: {} params", data.len());
    Ok(())
}

async fn load_config() -> Result<Vec<String>, Box<dyn std::error::Error>> {
    Ok(vec!["api_key".to_string(), "secret".to_string()])
}
```

## What We Learned

| Syntax | Description |
|--------|-------------|
| `#[tokio::main]` | Launch tokio runtime |
| `async fn main()` | Main async function |
| `await` in main | Can await async operations |
| `flavor = "current_thread"` | Single-threaded mode |
| `-> Result<(), E>` | Return error from main |

## Homework

1. Create async main that runs 3 async functions sequentially via await
2. Add `-> Result<(), Box<dyn Error>>` to main and use `?`
3. Try `current_thread` and `multi_thread` flavors — what's the difference?
4. Write `async fn ping(exchange: &str) -> u64` simulating exchange ping (sleep + latency)

## Navigation

[← Previous day](../185-tokio-runtime-async-engine/en.md) | [Next day →](../187-tokio-spawn-async-tasks/en.md)
