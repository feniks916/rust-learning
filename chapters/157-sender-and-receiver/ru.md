# День 157: Sender и Receiver — кто отправляет, кто получает

## Аналогия из трейдинга

Биржа — это Sender: она непрерывно отправляет тикеры с ценами. Твой торговый алгоритм — это Receiver: он сидит и ждёт новых данных, обрабатывает каждый тикер по очереди. Канал между ними — это очередь сообщений. Именно так работает `std::sync::mpsc` в Rust: один или несколько отправителей, один получатель.

## mpsc::channel()

`mpsc` расшифровывается как **m**ulti-**p**roducer, **s**ingle-**c**onsumer. Функция `channel()` создаёт пару (Sender, Receiver):

```rust
use std::sync::mpsc;

fn main() {
    // channel() возвращает кортеж (отправитель, получатель)
    let (tx, rx) = mpsc::channel::<String>();
    //   ^^  ^^
    //   tx = transmitter (отправитель)
    //   rx = receiver (получатель)

    println!("Канал создан!");
    // tx — отправляет сообщения в канал
    // rx — получает сообщения из канала
}
```

По традиции отправитель называют `tx` (transmit), получатель — `rx` (receive).

## Sender<T>

`Sender<T>` — типизированный отправитель. Метод `send()` помещает значение в канал:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    thread::spawn(move || {
        // Отправляем цены BTC
        let prices = [65000.0, 65100.0, 64900.0, 65200.0];
        for price in prices {
            tx.send(price).expect("Ошибка отправки");
            println!("Отправлена цена: {:.2}", price);
        }
        // Когда tx выходит из области видимости — канал закрывается
    });

    // Receiver получает в главном потоке
    for price in rx {
        println!("Получена цена: {:.2}", price);
    }
}
```

`send()` возвращает `Result<(), SendError<T>>` — ошибка возникает только если Receiver был уничтожен.

## Receiver<T>

`Receiver<T>` — получатель сообщений. Основной метод — `recv()`, который блокирует поток до появления сообщения:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    thread::spawn(move || {
        let orders = vec!["BUY BTCUSDT 0.1", "SELL ETHUSDT 1.0", "BUY BNBUSDT 5.0"];
        for order in orders {
            tx.send(order.to_string()).unwrap();
        }
    });

    // recv() блокируется, пока не придёт сообщение
    loop {
        match rx.recv() {
            Ok(order) => println!("Исполняем ордер: {}", order),
            Err(_) => {
                println!("Канал закрыт, выходим");
                break;
            }
        }
    }
}
```

`recv()` возвращает `Err` когда все отправители уничтожены и канал пуст.

## send() и recv()

Разберём возвращаемые типы подробнее:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    let handle = thread::spawn(move || {
        // send возвращает Result — обрабатываем ошибку
        match tx.send(42000.0) {
            Ok(()) => println!("Цена отправлена"),
            Err(e) => println!("Ошибка: получатель закрыт ({})", e),
        }

        // .unwrap() — если не важна обработка ошибки
        tx.send(43000.0).unwrap();
    });

    // recv() блокируется, ждём первое сообщение
    let price1 = rx.recv().unwrap();
    println!("Получено 1: {:.2}", price1);

    // Второе сообщение
    let price2 = rx.recv().unwrap();
    println!("Получено 2: {:.2}", price2);

    handle.join().unwrap();
}
```

## for msg in rx — итерация

Самый удобный способ получать сообщения до закрытия канала:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<(String, f64)>();

    thread::spawn(move || {
        let ticks = vec![
            ("BTCUSDT".to_string(), 65000.0),
            ("ETHUSDT".to_string(), 3200.0),
            ("BNBUSDT".to_string(), 420.0),
        ];
        for tick in ticks {
            tx.send(tick).unwrap();
        }
        // Поток завершается, tx уничтожается, канал закрывается
    });

    // for автоматически заканчивается когда канал закрыт
    for (ticker, price) in rx {
        println!("Тикер: {}, цена: {:.2}", ticker, price);
    }
    println!("Все тикеры обработаны");
}
```

`for msg in rx` — это синтаксический сахар для `while let Ok(msg) = rx.recv()`.

## recv_timeout()

Иногда нужно подождать сообщение не вечно, а определённое время:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    thread::spawn(move || {
        thread::sleep(Duration::from_millis(200));
        tx.send(65500.0).unwrap();
    });

    // Ждём сообщение не более 100мс
    match rx.recv_timeout(Duration::from_millis(100)) {
        Ok(price) => println!("Получена цена: {:.2}", price),
        Err(mpsc::RecvTimeoutError::Timeout) => {
            println!("Таймаут! Биржа не ответила за 100мс");
        }
        Err(mpsc::RecvTimeoutError::Disconnected) => {
            println!("Канал закрыт");
        }
    }

    // Ждём ещё 200мс — теперь сообщение придёт
    match rx.recv_timeout(Duration::from_millis(200)) {
        Ok(price) => println!("Получена цена: {:.2}", price),
        Err(e) => println!("Ошибка: {:?}", e),
    }
}
```

## try_recv() — неблокирующий

`try_recv()` не блокирует — сразу возвращает результат:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel::<f64>();

    thread::spawn(move || {
        thread::sleep(Duration::from_millis(50));
        tx.send(65800.0).unwrap();
    });

    // Опрашиваем без блокировки
    for i in 0..5 {
        match rx.try_recv() {
            Ok(price) => {
                println!("Итерация {}: получена цена {:.2}", i, price);
                break;
            }
            Err(mpsc::TryRecvError::Empty) => {
                println!("Итерация {}: данных нет, продолжаем работу", i);
                // Выполняем другую работу пока нет данных
                thread::sleep(Duration::from_millis(20));
            }
            Err(mpsc::TryRecvError::Disconnected) => {
                println!("Канал закрыт");
                break;
            }
        }
    }
}
```

## Типизированные enum сообщения

Канал может передавать разные типы сообщений через enum:

```rust
use std::sync::mpsc;
use std::thread;

#[derive(Debug)]
enum ExchangeMessage {
    Tick { symbol: String, price: f64, volume: f64 },
    OrderFilled { order_id: u64, fill_price: f64 },
    Error(String),
    Disconnect,
}

fn main() {
    let (tx, rx) = mpsc::channel::<ExchangeMessage>();

    thread::spawn(move || {
        tx.send(ExchangeMessage::Tick {
            symbol: "BTCUSDT".to_string(),
            price: 65000.0,
            volume: 12.5,
        }).unwrap();

        tx.send(ExchangeMessage::OrderFilled {
            order_id: 12345,
            fill_price: 65010.0,
        }).unwrap();

        tx.send(ExchangeMessage::Disconnect).unwrap();
    });

    for msg in rx {
        match msg {
            ExchangeMessage::Tick { symbol, price, volume } => {
                println!("Тикер {}: цена={:.2}, объём={:.2}", symbol, price, volume);
            }
            ExchangeMessage::OrderFilled { order_id, fill_price } => {
                println!("Ордер #{} исполнен по цене {:.2}", order_id, fill_price);
            }
            ExchangeMessage::Error(msg) => {
                println!("ОШИБКА: {}", msg);
            }
            ExchangeMessage::Disconnect => {
                println!("Соединение закрыто");
                break;
            }
        }
    }
}
```

## Практический пример: биржевой стрим через канал

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
use std::collections::HashMap;

#[derive(Debug)]
enum MarketEvent {
    PriceUpdate { symbol: String, price: f64, change_pct: f64 },
    VolumeSpike  { symbol: String, volume: f64 },
    Connected,
    Disconnected,
}

fn simulate_exchange(tx: mpsc::Sender<MarketEvent>) {
    tx.send(MarketEvent::Connected).unwrap();

    let data = vec![
        ("BTCUSDT", 65000.0, 0.5,  1200.0),
        ("ETHUSDT", 3200.0,  -0.3, 450.0),
        ("BNBUSDT", 420.0,   1.2,  8500.0), // большой объём!
        ("BTCUSDT", 65300.0, 0.46, 980.0),
        ("ETHUSDT", 3190.0,  -0.6, 520.0),
    ];

    for (symbol, price, change, volume) in data {
        thread::sleep(Duration::from_millis(50));

        tx.send(MarketEvent::PriceUpdate {
            symbol: symbol.to_string(),
            price,
            change_pct: change,
        }).unwrap();

        // Объём выше 5000 — спайк
        if volume > 5000.0 {
            tx.send(MarketEvent::VolumeSpike {
                symbol: symbol.to_string(),
                volume,
            }).unwrap();
        }
    }

    tx.send(MarketEvent::Disconnected).unwrap();
}

fn main() {
    let (tx, rx) = mpsc::channel::<MarketEvent>();

    let producer = thread::spawn(move || {
        simulate_exchange(tx);
    });

    // Потребитель — наш алгоритм
    let mut prices: HashMap<String, f64> = HashMap::new();
    let mut event_count = 0usize;

    for event in rx {
        event_count += 1;
        match event {
            MarketEvent::Connected => {
                println!("=== Подключено к бирже ===");
            }
            MarketEvent::PriceUpdate { symbol, price, change_pct } => {
                let arrow = if change_pct >= 0.0 { "↑" } else { "↓" };
                println!(
                    "{} {} {:.2} ({:+.2}%)",
                    symbol, arrow, price, change_pct
                );
                prices.insert(symbol, price);
            }
            MarketEvent::VolumeSpike { symbol, volume } => {
                println!("!!! СПАЙК ОБЪЁМА: {} — {:.0} USDT", symbol, volume);
            }
            MarketEvent::Disconnected => {
                println!("=== Соединение закрыто ===");
                break;
            }
        }
    }

    producer.join().unwrap();

    println!("\n--- Итоги ---");
    println!("Событий обработано: {}", event_count);
    println!("Последние цены:");
    let mut sorted: Vec<_> = prices.iter().collect();
    sorted.sort_by_key(|(k, _)| k.as_str());
    for (sym, price) in sorted {
        println!("  {}: ${:.2}", sym, price);
    }
}
```

## Что мы узнали

| Синтаксис | Описание | Применение |
|-----------|----------|------------|
| `mpsc::channel::<T>()` | Создать канал типа T | Создание очереди сообщений |
| `tx.send(value)` | Отправить значение | Публикация цены, ордера |
| `rx.recv()` | Блокирующее получение | Ожидание следующего тикера |
| `for msg in rx` | Итерация до закрытия канала | Обработка всего стрима |
| `rx.recv_timeout(dur)` | Ожидание с таймаутом | Не ждать вечно ответа биржи |
| `rx.try_recv()` | Неблокирующая проверка | Опрос при наличии другой работы |
| `enum` сообщения | Разные типы в одном канале | Разные события биржи |

## Домашнее задание

1. Создай enum `TradingCommand` с вариантами:
   - `Buy { symbol: String, quantity: f64, max_price: f64 }`
   - `Sell { symbol: String, quantity: f64, min_price: f64 }`
   - `Cancel { order_id: u64 }`
   - `Shutdown`

2. Напиши поток-исполнитель, который получает команды через `Receiver<TradingCommand>`:
   - Для `Buy` — печатает "Покупаем {qty} {symbol} не дороже {max_price}"
   - Для `Sell` — печатает "Продаём {qty} {symbol} не дешевле {min_price}"
   - Для `Cancel` — "Отменяем ордер #{id}"
   - Для `Shutdown` — завершает цикл

3. В `main` отправь 4-5 различных команд через `Sender<TradingCommand>`, включая `Shutdown` в конце

4. Добавь `recv_timeout(Duration::from_millis(500))` в исполнителе:
   - Если команды нет 500мс — печатать "Ожидание команд..."

5. Подсчитай сколько команд каждого типа было исполнено и выведи статистику в конце

## Навигация

[← Предыдущий день](../156-channels-mpsc-order-queue/ru.md) | [Следующий день →](../158-multiple-senders/ru.md)
