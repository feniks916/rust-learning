# Day 210: Graceful Shutdown of Async Tasks

## Trading Analogy

Before shutting down a trading bot you need to: close all positions, wait for pending orders, save state. Abrupt termination risks unclosed positions. Graceful shutdown is the correct way.

## CancellationToken

```rust
use tokio_util::sync::CancellationToken;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();

    let monitor_token = token.clone();
    let monitor = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = monitor_token.cancelled() => {
                    println!("Monitor: shutdown signal received");
                    break;
                }
                _ = sleep(Duration::from_millis(500)) => {
                    println!("Monitor: checking prices...");
                }
            }
        }
        println!("Monitor stopped gracefully");
    });

    sleep(Duration::from_secs(2)).await;

    println!("Sending shutdown signal...");
    token.cancel();

    monitor.await.unwrap();
    println!("All tasks completed");
}
```

## Shutdown via Channel

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    let mut rx1 = shutdown_tx.subscribe();
    let task1 = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = rx1.recv() => { println!("Task 1: shutting down"); break; }
                _ = sleep(Duration::from_millis(300)) => { println!("Task 1: working"); }
            }
        }
    });

    sleep(Duration::from_secs(1)).await;
    shutdown_tx.send(()).unwrap();
    task1.await.unwrap();
    println!("Graceful shutdown complete");
}
```

## What We Learned

| Tool | Description |
|------|-------------|
| `CancellationToken` | Cancel via token |
| `broadcast::channel` | Signal to all tasks |
| `tokio::select!` | Await multiple events |
| `tokio::join!` | Wait for all to finish |

## Homework

1. Implement 3 async tasks with graceful shutdown via CancellationToken
2. Add "finalization" in each task (save data before exit)
3. Add shutdown timeout: if task doesn't stop in 5s — force it
4. Implement Ctrl+C handling: `tokio::signal::ctrl_c().await`

## Navigation

[← Previous day](../209-parallel-websocket-subscriptions/en.md) | [Next day →](../211-debugging-async-code/en.md)
