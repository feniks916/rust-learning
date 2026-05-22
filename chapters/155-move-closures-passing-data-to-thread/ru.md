# День 155: Move closures — передаём данные в поток

## Аналогия из трейдинга

Представь, что ты запускаешь торгового робота в отдельный офис. Ему нужно забрать с собой список тикеров, текущую цену и конфиг стратегии. После того как робот уехал — ты уже не можешь изменить те данные, что он забрал. `move` closure делает именно это: она забирает переменные из текущего контекста и переносит их в новый поток, где те данные принадлежат уже только потоку.

## Что такое closure

Closure — это анонимная функция, которая может "захватывать" переменные из окружающего кода:

```rust
fn main() {
    let ticker = String::from("BTCUSDT");
    let price = 65000.0f64;

    // Closure захватывает ticker и price по ссылке
    let print_info = || {
        println!("Тикер: {}, цена: {:.2}", ticker, price);
    };

    print_info(); // Тикер: BTCUSDT, цена: 65000.00

    // ticker и price всё ещё доступны здесь
    println!("Тикер ещё доступен: {}", ticker);
}
```

Обычная closure захватывает переменные по ссылке — поэтому оригинал остаётся доступным.

## Проблема без move

При передаче closure в поток Rust требует, чтобы все захваченные переменные жили достаточно долго. Без `move` это не компилируется:

```rust
use std::thread;

fn main() {
    let ticker = String::from("ETHUSDT");

    // ОШИБКА: ticker может умереть раньше, чем завершится поток
    let handle = thread::spawn(|| {
        // borrowed value does not live long enough
        println!("Мониторим: {}", ticker);
    });

    handle.join().unwrap();
}
```

Проблема: `ticker` живёт на стеке `main`, а поток может продолжать работу после возврата из `main`. Rust защищает нас от этого.

## move синтаксис

Ключевое слово `move` перед `||` заставляет closure перемещать (а не заимствовать) все захваченные переменные:

```rust
use std::thread;

fn main() {
    let ticker = String::from("ETHUSDT");
    let price = 3200.0f64;

    // move перемещает ticker и price внутрь closure
    let handle = thread::spawn(move || {
        println!("Поток мониторит {} @ ${:.2}", ticker, price);
        // ticker и price теперь принадлежат этому потоку
    });

    // Нельзя использовать ticker здесь — он был перемещён!
    // println!("{}", ticker); // ОШИБКА: value used after move

    handle.join().unwrap();
}
```

## Что захватывается

`move` захватывает ВСЕ переменные, на которые ссылается closure:

```rust
use std::thread;

fn main() {
    let symbol = String::from("BNBUSDT");
    let entry_price = 420.0f64;
    let quantity = 10.0f64;
    let stop_loss = 400.0f64;

    let handle = thread::spawn(move || {
        // Все 4 переменные захвачены и перемещены
        let risk = (entry_price - stop_loss) * quantity;
        println!(
            "Робот для {}: вход={:.2}, стоп={:.2}, риск=${:.2}",
            symbol, entry_price, stop_loss, risk
        );
    });

    handle.join().unwrap();
    // symbol, entry_price, quantity, stop_loss — все перемещены
}
```

## Copy типы в move

Числа и булевы значения реализуют `Copy` — они копируются, а не перемещаются. Оригинал остаётся доступным:

```rust
use std::thread;

fn main() {
    let price = 65000.0f64;    // f64 реализует Copy
    let quantity = 0.5f64;     // f64 реализует Copy
    let is_active = true;       // bool реализует Copy

    let handle = thread::spawn(move || {
        // price, quantity, is_active — скопированы, не перемещены
        println!("Цена: {:.2}, объём: {:.2}, активен: {}", price, quantity, is_active);
    });

    // Copy типы всё ещё доступны в main!
    println!("В main: цена = {:.2}", price);
    println!("В main: активен = {}", is_active);

    handle.join().unwrap();
}
```

## Clone перед move

Если нужно использовать переменную и в основном потоке, и в дочернем — клонируй перед перемещением:

```rust
use std::thread;

fn main() {
    let tickers = vec!["BTCUSDT", "ETHUSDT", "BNBUSDT"];

    // Клонируем для потока
    let tickers_for_thread = tickers.clone();

    let handle = thread::spawn(move || {
        for ticker in &tickers_for_thread {
            println!("Поток обрабатывает: {}", ticker);
        }
    });

    // Оригинал остаётся доступным
    println!("В main тикеров: {}", tickers.len());

    handle.join().unwrap();
}
```

## Несколько потоков с разными данными

Каждому потоку передаём свои данные через `move`:

```rust
use std::thread;

fn main() {
    let symbols = vec![
        ("BTCUSDT", 65000.0),
        ("ETHUSDT", 3200.0),
        ("BNBUSDT", 420.0),
    ];

    let mut handles = Vec::new();

    for (symbol, price) in symbols {
        // Каждая итерация создаёт новую closure с move
        // symbol и price — Copy/небольшие значения, копируются
        let handle = thread::spawn(move || {
            // Имитируем работу робота
            let alert_price = price * 1.05;
            println!(
                "Робот {}: текущая={:.2}, цель={:.2}",
                symbol, price, alert_price
            );
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

## Практический пример: мониторинг нескольких символов

```rust
use std::thread;
use std::time::Duration;

#[derive(Clone)]
struct MonitorConfig {
    symbol: String,
    entry_price: f64,
    stop_loss: f64,
    take_profit: f64,
    check_interval_ms: u64,
}

impl MonitorConfig {
    fn new(symbol: &str, entry: f64, stop: f64, target: f64) -> Self {
        MonitorConfig {
            symbol: symbol.to_string(),
            entry_price: entry,
            stop_loss: stop,
            take_profit: target,
            check_interval_ms: 100,
        }
    }
}

fn simulate_price(base: f64, tick: u32) -> f64 {
    // Простая симуляция движения цены
    let change = (tick as f64 * 0.7).sin() * base * 0.02;
    base + change
}

fn monitor_symbol(config: MonitorConfig) {
    println!(
        "[{}] Запуск: вход={:.2}, стоп={:.2}, цель={:.2}",
        config.symbol, config.entry_price, config.stop_loss, config.take_profit
    );

    for tick in 0..5u32 {
        let current_price = simulate_price(config.entry_price, tick);
        let pnl = current_price - config.entry_price;

        let status = if current_price <= config.stop_loss {
            "СТОП-ЛОСС!"
        } else if current_price >= config.take_profit {
            "ТЕЙК-ПРОФИТ!"
        } else {
            "в позиции"
        };

        println!(
            "[{}] тик {}: цена={:.2}, PnL={:+.2} — {}",
            config.symbol, tick, current_price, pnl, status
        );

        thread::sleep(Duration::from_millis(config.check_interval_ms));
    }

    println!("[{}] Мониторинг завершён", config.symbol);
}

fn main() {
    let configs = vec![
        MonitorConfig::new("BTCUSDT", 65000.0, 63000.0, 68000.0),
        MonitorConfig::new("ETHUSDT", 3200.0, 3000.0, 3500.0),
        MonitorConfig::new("BNBUSDT", 420.0, 400.0, 450.0),
    ];

    let mut handles = Vec::new();

    for config in configs {
        // move захватывает config (он Clone, но здесь перемещается)
        let handle = thread::spawn(move || {
            monitor_symbol(config);
        });
        handles.push(handle);
    }

    println!("Все роботы запущены, ждём завершения...\n");

    for handle in handles {
        handle.join().unwrap();
    }

    println!("\nВсе роботы завершили работу");
}
```

## Что мы узнали

| Синтаксис | Описание | Применение |
|-----------|----------|------------|
| `\|\| { ... }` | Обычная closure | Захватывает по ссылке |
| `move \|\| { ... }` | Move closure | Перемещает переменные внутрь |
| `Copy` типы в `move` | Копируются, не перемещаются | `f64`, `i32`, `bool` доступны после |
| `clone()` перед `move` | Копия для потока, оригинал остаётся | `String`, `Vec` нужно клонировать |
| `thread::spawn(move \|\| ...)` | Запуск потока с захватом данных | Передача конфига в рабочий поток |
| Несколько потоков | Каждый получает свой экземпляр данных | Параллельный мониторинг тикеров |

## Домашнее задание

1. Создай структуру `PriceAlert` с полями `symbol: String`, `threshold: f64`, `above: bool`
   - `above = true` — алерт когда цена выше threshold
   - `above = false` — алерт когда цена ниже threshold

2. Напиши функцию `watch_price(alert: PriceAlert, prices: Vec<f64>)`:
   - Итерируется по ценам
   - Печатает сообщение когда условие выполнено
   - Печатает итог в конце

3. Запусти 3 потока через `thread::spawn(move || ...)`, каждый со своим `PriceAlert`:
   - BTC: алерт выше 70000
   - ETH: алерт ниже 3000
   - BNB: алерт выше 450

4. Убедись что `PriceAlert` реализует `Clone` (добавь `#[derive(Clone)]`)
   - Клонируй алерт перед передачей в поток
   - Убедись что оригинал доступен в `main` после запуска потока

5. Добавь поле `id: u32` в `PriceAlert` (Copy тип):
   - Убедись что `id` доступен в `main` после `move` в поток

## Навигация

[← Предыдущий день](../154-join-thread-completion/ru.md) | [Следующий день →](../156-channels-mpsc-order-queue/ru.md)
