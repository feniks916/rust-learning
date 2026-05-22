# День 158: Множественные отправители

## Аналогия из трейдинга

Твой агрегатор цен подключён сразу к нескольким биржам: Binance, Bybit, OKX. Каждая биржа — отдельный поток, каждая отправляет котировки. Но у тебя один алгоритм, который обрабатывает всё это. Именно для такой архитектуры создан `mpsc`: **m**ulti-**p**roducer (много отправителей), **s**ingle-**c**onsumer (один получатель).

## tx.clone()

`Sender<T>` можно клонировать — каждый клон отправляет в тот же канал:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    // Клонируем отправитель для второго потока
    let tx2 = tx.clone();

    // Первый поток — Binance
    thread::spawn(move || {
        tx.send("Binance: BTCUSDT 65000".to_string()).unwrap();
        tx.send("Binance: ETHUSDT 3200".to_string()).unwrap();
    });

    // Второй поток — Bybit (использует клон)
    thread::spawn(move || {
        tx2.send("Bybit: BTCUSDT 65050".to_string()).unwrap();
        tx2.send("Bybit: ETHUSDT 3205".to_string()).unwrap();
    });

    for msg in rx {
        println!("Агрегатор: {}", msg);
    }
}
```

Все клоны `tx` и оригинал — равноправны, все пишут в один буфер.

## drop(tx) оригинального

Канал закрывается только когда ВСЕ отправители уничтожены. Если ты создал клоны, не забудь уничтожить оригинал:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<i32>();

    let tx1 = tx.clone();
    let tx2 = tx.clone();

    // Уничтожаем оригинал сразу — нам он не нужен в main
    drop(tx);

    thread::spawn(move || {
        tx1.send(1).unwrap();
        // tx1 уничтожается при выходе из потока
    });

    thread::spawn(move || {
        tx2.send(2).unwrap();
        // tx2 уничтожается при выходе из потока
    });

    // Канал закрывается только после drop(tx) И уничтожения tx1 и tx2
    for val in rx {
        println!("Получено: {}", val);
    }
    println!("Канал закрыт (все отправители уничтожены)");
}
```

Частая ошибка: забыть `drop(tx)` в main — тогда `for val in rx` будет ждать вечно.

## Несколько потоков-отправителей

Создадим три потока, каждый с клоном `tx`:

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

    // drop оригинала после создания всех клонов
    let mut handles = Vec::new();
    for (exchange, price) in exchanges {
        let tx_clone = tx.clone();
        let handle = thread::spawn(move || {
            tx_clone.send((exchange.to_string(), price)).unwrap();
            println!("[{}] Цена отправлена: {:.2}", exchange, price);
        });
        handles.push(handle);
    }
    drop(tx); // Важно! Иначе for..in никогда не завершится

    for (exchange, price) in rx {
        println!("Агрегатор получил: {} @ {:.2}", exchange, price);
    }

    for h in handles { h.join().unwrap(); }
}
```

## Порядок получения

Порядок сообщений из разных потоков не гарантирован — кто первый отправил, того и получим первым:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    // Первый поток отправляет с задержкой
    let tx1 = tx.clone();
    thread::spawn(move || {
        thread::sleep(Duration::from_millis(20));
        tx1.send("Binance (медленная)".to_string()).unwrap();
    });

    // Второй поток отправляет сразу
    let tx2 = tx.clone();
    thread::spawn(move || {
        tx2.send("Bybit (быстрая)".to_string()).unwrap();
    });

    drop(tx);

    for msg in rx {
        println!("Пришло: {}", msg);
    }
    // Вывод: сначала "Bybit (быстрая)", потом "Binance (медленная)"
}
```

Если порядок важен — добавляй временную метку к каждому сообщению.

## Агрегация данных

Типичный паттерн: собрать лучшую цену с нескольких источников:

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

    println!("\nЛучший bid: {:.2} ({})", best_bid, best_bid_exchange);
    println!("Лучший ask: {:.2} ({})", best_ask, best_ask_exchange);
    println!("Спред: {:.2}", best_ask - best_bid);
}
```

## Практический пример: несколько бирж в один агрегатор

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
    println!("[{}] Фид завершён", name);
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
    drop(tx); // Закрываем оригинал

    // Агрегатор
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

    println!("\n=== АГРЕГИРОВАННЫЕ ДАННЫЕ ===");
    println!("Всего тиков получено: {}", tick_count);

    let mut symbols: Vec<_> = price_map.keys().collect();
    symbols.sort();

    for symbol in symbols {
        let prices = &price_map[symbol];
        let avg = prices.iter().map(|(_, p)| p).sum::<f64>() / prices.len() as f64;
        let min = prices.iter().map(|(_, p)| *p).fold(f64::MAX, f64::min);
        let max = prices.iter().map(|(_, p)| *p).fold(f64::MIN, f64::max);

        println!("\n{}:", symbol);
        println!("  Средняя цена: {:.2}", avg);
        println!("  Диапазон:     {:.2} — {:.2}", min, max);
        println!("  Спред:        {:.2}", max - min);
        for (exchange, price) in prices.iter().rev().take(1) {
            println!("  Последняя:    {:.2} ({})", price, exchange);
        }
    }
}
```

## Что мы узнали

| Синтаксис | Описание | Применение |
|-----------|----------|------------|
| `tx.clone()` | Создать ещё одного отправителя | Несколько потоков в один канал |
| `drop(tx)` | Уничтожить оригинальный Sender | Позволить каналу закрыться |
| `for msg in rx` | Завершается когда все tx уничтожены | Обработка до конца стрима |
| Порядок сообщений | Не гарантирован между потоками | Используй timestamp для сортировки |
| Агрегация | Один rx, много tx | Сбор данных с нескольких бирж |

## Домашнее задание

1. Создай структуру `OrderBook` с полями: `exchange: String`, `symbol: String`, `bids: Vec<f64>`, `asks: Vec<f64>`

2. Запусти 3 потока-"биржи", каждый создаёт и отправляет `OrderBook` через клон `tx`:
   - Binance: BTCUSDT bids=[64990, 64980], asks=[65000, 65010]
   - Bybit: BTCUSDT bids=[65010, 65000], asks=[65020, 65030]
   - OKX: BTCUSDT bids=[64980, 64970], asks=[64995, 65005]

3. Не забудь `drop(tx)` после создания всех клонов

4. Агрегируй полученные данные:
   - Найди лучший bid (максимальный) среди всех бирж
   - Найди лучший ask (минимальный) среди всех бирж
   - Посчитай, возможен ли арбитраж (лучший bid > лучший ask)

5. Добавь задержки между отправками (10-50 мс) и проверь, что порядок получения не совпадает с порядком создания потоков

## Навигация

[← Предыдущий день](../157-sender-and-receiver/ru.md) | [Следующий день →](../159-sync-channel-bounded-queue/ru.md)
