# Day 125: std::fs — File Operations

## Trading Analogy

A trading terminal constantly reads and writes files: it saves trade history, loads strategy configuration, exports reports to CSV. The `std::fs` module is your file manager in Rust: it can create, read, copy, and delete files, just like you work with trading data on disk.

## fs::write / fs::read_to_string / fs::read

The three most basic operations — write, read as text, read as bytes:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Write text to a file (creates or overwrites)
    fs::write("trades.csv", "ticker,entry,exit,pnl\nBTCUSDT,65000,68000,1500\n")?;
    println!("File written");

    // Read file as a string
    let content = fs::read_to_string("trades.csv")?;
    println!("Contents:\n{}", content);

    // Read file as bytes (for binary data)
    let bytes = fs::read("trades.csv")?;
    println!("Size in bytes: {}", bytes.len());

    Ok(())
}
```

`fs::write` accepts a path and any value implementing `AsRef<[u8]>` — a string, `Vec<u8>`, or byte slice. If the file exists it is overwritten; if not, it is created.

## fs::copy / fs::rename / fs::remove_file

File move and delete operations:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Prepare a test file
    fs::write("btc_trades.csv", "ticker,price\nBTCUSDT,65000")?;

    // Copy file (original stays)
    let bytes_copied = fs::copy("btc_trades.csv", "btc_trades_backup.csv")?;
    println!("Bytes copied: {}", bytes_copied);

    // Rename / move file
    // fs::rename("btc_trades.csv", "archive/btc_trades_2024.csv")?;

    // Delete files
    fs::remove_file("btc_trades_backup.csv")?;
    fs::remove_file("btc_trades.csv")?;
    println!("Files deleted");

    Ok(())
}
```

`fs::rename` works as an atomic operation on Unix — the file is either moved or not. This matters when archiving trading logs.

## fs::create_dir / create_dir_all / remove_dir_all

Directory management:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create all directories in the path (like mkdir -p)
    fs::create_dir_all("data/2024/trades/btc")?;
    println!("Folder structure created");

    // Write data into the created folder
    fs::write("data/2024/trades/btc/january.csv", "date,price\n2024-01-01,42000")?;

    // Delete directory with all its contents (use with care!)
    fs::remove_dir_all("data")?;
    println!("Folder data and all contents deleted");

    Ok(())
}
```

`create_dir_all` is indispensable when building a folder structure for storing trading data by date: `data/2024/01/`, `data/2024/02/`, and so on.

## fs::read_dir — List Files

Enumerate the contents of a directory:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::create_dir_all("reports")?;
    fs::write("reports/btc.csv", "ticker\nBTCUSDT")?;
    fs::write("reports/eth.csv", "ticker\nETHUSDT")?;
    fs::write("reports/bnb.csv", "ticker\nBNBUSDT")?;

    println!("Files in reports folder:");
    for entry in fs::read_dir("reports")? {
        let entry = entry?;
        let path = entry.path();
        let file_name = entry.file_name();
        println!("  {:?}", file_name);

        // Process only CSV files
        if path.extension().map(|e| e == "csv").unwrap_or(false) {
            let content = fs::read_to_string(&path)?;
            println!("    -> {} bytes", content.len());
        }
    }

    fs::remove_dir_all("reports")?;
    Ok(())
}
```

`read_dir` returns an iterator over `DirEntry`. Each `DirEntry` contains a path, file name, and metadata.

## metadata — Size and Time

Get file information without reading it:

```rust
use std::fs;
use std::time::SystemTime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("trades.log", "lots of trade data...")?;

    let meta = fs::metadata("trades.log")?;

    println!("File size: {} bytes", meta.len());
    println!("Is file: {}", meta.is_file());
    println!("Is dir: {}", meta.is_dir());
    println!("Read-only: {}", meta.permissions().readonly());

    // Last modification time
    if let Ok(modified) = meta.modified() {
        let duration = SystemTime::now()
            .duration_since(modified)
            .unwrap_or_default();
        println!("Modified {} seconds ago", duration.as_secs());
    }

    fs::remove_file("trades.log")?;
    Ok(())
}
```

`metadata` is useful for checking: is the price cache stale, does the config file exist, is there enough disk space?

## Error Handling with ?

Errors are inevitable when working with files. The `?` operator makes handling them convenient:

```rust
use std::fs;
use std::io;

fn load_strategy_config(path: &str) -> Result<String, io::Error> {
    // If the file doesn't exist, return the error automatically
    let config = fs::read_to_string(path)?;
    Ok(config)
}

fn save_trade_report(trades: &[&str], path: &str) -> Result<(), io::Error> {
    let mut output = String::from("ticker,pnl\n");
    for trade in trades {
        output.push_str(trade);
        output.push('\n');
    }
    fs::write(path, output)?;
    Ok(())
}

fn main() {
    // Try to load the config — file may not exist
    match load_strategy_config("strategy.toml") {
        Ok(config) => println!("Config loaded: {} bytes", config.len()),
        Err(e) if e.kind() == io::ErrorKind::NotFound => {
            println!("Config not found, using defaults");
        }
        Err(e) => println!("Read error: {}", e),
    }

    // Save report
    let trades = ["BTCUSDT,1500", "ETHUSDT,-200", "BNBUSDT,300"];
    match save_trade_report(&trades, "report.csv") {
        Ok(()) => println!("Report saved"),
        Err(e) => println!("Save error: {}", e),
    }

    let _ = fs::remove_file("report.csv");
}
```

The pattern `e.kind() == io::ErrorKind::NotFound` lets you distinguish "file not found" from other I/O errors.

## Practical Example: Trade Data Manager

```rust
use std::fs;
use std::path::Path;

struct TradeDataManager {
    base_dir: String,
}

impl TradeDataManager {
    fn new(base_dir: &str) -> Result<Self, std::io::Error> {
        fs::create_dir_all(format!("{}/trades", base_dir))?;
        fs::create_dir_all(format!("{}/reports", base_dir))?;
        fs::create_dir_all(format!("{}/archive", base_dir))?;
        println!("Data manager initialized in '{}'", base_dir);
        Ok(TradeDataManager { base_dir: base_dir.to_string() })
    }

    fn save_trade(&self, ticker: &str, entry: f64, exit: f64, qty: f64)
        -> Result<(), std::io::Error>
    {
        let pnl = (exit - entry) * qty;
        let path = format!("{}/trades/{}.csv", self.base_dir, ticker.to_lowercase());

        let existing = fs::read_to_string(&path).unwrap_or_default();
        let header = if existing.is_empty() { "ticker,entry,exit,qty,pnl\n" } else { "" };
        let line = format!("{},{:.2},{:.2},{:.4},{:.2}\n", ticker, entry, exit, qty, pnl);
        fs::write(&path, format!("{}{}{}", existing, header, line))?;

        println!("Trade {} saved (PnL: {:.2})", ticker, pnl);
        Ok(())
    }

    fn list_tickers(&self) -> Result<Vec<String>, std::io::Error> {
        let trades_dir = format!("{}/trades", self.base_dir);
        let mut tickers = Vec::new();

        for entry in fs::read_dir(&trades_dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.extension().map(|e| e == "csv").unwrap_or(false) {
                if let Some(stem) = path.file_stem() {
                    tickers.push(stem.to_string_lossy().to_uppercase().to_string());
                }
            }
        }
        tickers.sort();
        Ok(tickers)
    }

    fn generate_report(&self) -> Result<String, std::io::Error> {
        let trades_dir = format!("{}/trades", self.base_dir);
        let mut total_pnl = 0.0f64;
        let mut trade_count = 0usize;
        let mut report = String::from("=== TRADING REPORT ===\n\n");

        for entry in fs::read_dir(&trades_dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.extension().map(|e| e == "csv").unwrap_or(false) {
                let content = fs::read_to_string(&path)?;
                for line in content.lines().skip(1) {
                    let parts: Vec<&str> = line.split(',').collect();
                    if parts.len() >= 5 {
                        if let Ok(pnl) = parts[4].parse::<f64>() {
                            total_pnl += pnl;
                            trade_count += 1;
                        }
                    }
                }
            }
        }

        report.push_str(&format!("Total trades: {}\n", trade_count));
        report.push_str(&format!("Total PnL: ${:.2}\n", total_pnl));
        report.push_str(&format!(
            "Average PnL: ${:.2}\n",
            if trade_count > 0 { total_pnl / trade_count as f64 } else { 0.0 }
        ));

        let report_path = format!("{}/reports/summary.txt", self.base_dir);
        fs::write(&report_path, &report)?;
        println!("Report saved to '{}'", report_path);

        Ok(report)
    }

    fn archive_all(&self) -> Result<(), std::io::Error> {
        let trades_dir = format!("{}/trades", self.base_dir);
        let archive_dir = format!("{}/archive", self.base_dir);

        for entry in fs::read_dir(&trades_dir)? {
            let entry = entry?;
            let src = entry.path();
            if let Some(name) = src.file_name() {
                let dst = Path::new(&archive_dir).join(name);
                fs::copy(&src, &dst)?;
                fs::remove_file(&src)?;
                println!("Archived: {:?}", name);
            }
        }
        Ok(())
    }

    fn cleanup(&self) -> Result<(), std::io::Error> {
        fs::remove_dir_all(&self.base_dir)?;
        println!("All data deleted");
        Ok(())
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let manager = TradeDataManager::new("trading_data")?;

    manager.save_trade("BTCUSDT", 65000.0, 68500.0, 0.5)?;
    manager.save_trade("ETHUSDT", 3200.0, 3050.0, 2.0)?;
    manager.save_trade("BTCUSDT", 68500.0, 70000.0, 0.3)?;
    manager.save_trade("BNBUSDT", 420.0, 445.0, 10.0)?;

    let tickers = manager.list_tickers()?;
    println!("\nTickers with data: {:?}", tickers);

    println!("\n{}", manager.generate_report()?);

    println!("Archiving trades...");
    manager.archive_all()?;

    manager.cleanup()?;

    Ok(())
}
```

## What We Learned

| Syntax | Description | Use Case |
|--------|-------------|----------|
| `fs::write(path, data)` | Write data to a file | Saving trade data, reports |
| `fs::read_to_string(path)` | Read file as a string | Loading configs, CSV files |
| `fs::read(path)` | Read file as bytes | Binary files, serialized data |
| `fs::copy(src, dst)` | Copy a file | Backups |
| `fs::rename(src, dst)` | Move/rename a file | Archiving files |
| `fs::remove_file(path)` | Delete a file | Cleaning up temp files |
| `fs::create_dir_all(path)` | Create all dirs in path | Initializing storage structure |
| `fs::read_dir(path)` | List directory contents | Iterating over data files |
| `fs::metadata(path)` | Get file metadata | Checking size, modification time |
| `io::ErrorKind::NotFound` | "Not found" error type | Distinguishing I/O error types |

## Homework

1. Create a function `append_trade(path: &str, ticker: &str, price: f64, qty: f64)`:
   - If the file does not exist — create it with the header `ticker,price,qty`
   - If the file exists — append a new line
   - Use `fs::read_to_string` and `fs::write`

2. Create a function `count_trades(path: &str) -> Result<usize, io::Error>`:
   - Read the file and count lines (excluding the header)
   - Return 0 if the file does not exist (handle `NotFound`)

3. Create a function `backup_file(path: &str) -> Result<String, io::Error>`:
   - Copies the file with a `.bak` suffix appended
   - Returns the path to the backup

4. Write a function `find_csv_files(dir: &str) -> Result<Vec<String>, io::Error>`:
   - Returns the names of all `.csv` files in the directory
   - Sorts the list alphabetically

5. Create a struct `TradeLogger` with methods:
   - `new(dir: &str)` — creates the folder if it does not exist
   - `log(&self, ticker: &str, msg: &str)` — appends to `{dir}/{ticker}.log`
   - `get_logs(&self, ticker: &str) -> Vec<String>` — all lines from the log

## Navigation

[← Previous day](../124-bufreader-bufwriter-efficient-io/en.md) | [Next day →](../126-path-pathbuf-file-paths/en.md)
