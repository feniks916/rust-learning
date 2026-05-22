# Day 250: MACD — Moving Average Convergence Divergence

## Trading Analogy

MACD is like two traders with different time horizons looking at the same asset. When the fast trader (EMA 12) starts outpacing the slow one (EMA 26) — that's a bullish signal.

## MACD Formula

```
MACD Line    = EMA(12) - EMA(26)
Signal Line  = EMA(9) of MACD Line
Histogram    = MACD Line - Signal Line
```

## EMA Implementation

```rust
fn ema(prices: &[f64], period: usize) -> Vec<f64> {
    if prices.len() < period { return vec![]; }

    let k = 2.0 / (period as f64 + 1.0);
    let mut result = Vec::new();

    let first_sma: f64 = prices[..period].iter().sum::<f64>() / period as f64;
    result.push(first_sma);

    for &price in &prices[period..] {
        let prev = *result.last().unwrap();
        result.push(price * k + prev * (1.0 - k));
    }

    result
}
```

## Computing MACD

```rust
fn macd(prices: &[f64]) -> (Vec<f64>, Vec<f64>, Vec<f64>) {
    let ema12 = ema(prices, 12);
    let ema26 = ema(prices, 26);

    let len = ema12.len().min(ema26.len());
    let o12 = ema12.len() - len;
    let o26 = ema26.len() - len;

    let macd_line: Vec<f64> = ema12[o12..].iter()
        .zip(&ema26[o26..])
        .map(|(f, s)| f - s)
        .collect();

    let signal = ema(&macd_line, 9);

    let off = macd_line.len() - signal.len();
    let histogram: Vec<f64> = macd_line[off..].iter()
        .zip(&signal)
        .map(|(m, s)| m - s)
        .collect();

    (macd_line, signal, histogram)
}

fn main() {
    let prices: Vec<f64> = (0..50)
        .map(|i| 50000.0 + (i as f64 * 100.0) - ((i as f64 * 0.5).sin() * 500.0))
        .collect();

    let (macd_line, signal, histogram) = macd(&prices);
    println!("MACD computed: {} values", histogram.len());

    if let (Some(m), Some(s), Some(h)) = (macd_line.last(), signal.last(), histogram.last()) {
        println!("Latest — MACD: {:.2}, Signal: {:.2}, Hist: {:.2}", m, s, h);
        println!("Signal: {}", if m > s { "BUY" } else { "SELL" });
    }
}
```

## What We Learned

| Component | Description |
|-----------|-------------|
| MACD Line | EMA(12) - EMA(26) |
| Signal Line | EMA(9) of MACD Line |
| Histogram | MACD - Signal (visual signal strength) |
| Crossover upward | Bullish signal (buy) |
| Crossover downward | Bearish signal (sell) |

## Homework

1. Implement `macd(prices: &[f64]) -> (Vec<f64>, Vec<f64>, Vec<f64>)` returning MACD, Signal, Histogram
2. Write `macd_crossover_signal(macd: &[f64], signal: &[f64]) -> Vec<&str>` — buy/sell/hold
3. Detect divergence: price rises, MACD falls — bearish divergence
4. Test on real data by loading historical price CSV

## Navigation

[← Previous day](../249-rsi-relative-strength-index/en.md) | [Next day →](../251-bollinger-bands/en.md)
