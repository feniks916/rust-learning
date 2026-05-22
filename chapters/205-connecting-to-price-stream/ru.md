# День 205: Подключение к стриму цен

## Аналогия из трейдинга

Стрим цен — это непрерывный поток котировок с биржи, как телетайп в старые времена. Подключившись один раз, ты получаешь обновления в реальном времени без постоянных запросов.

## Структура данных тика

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

## Почему поля приходят как `String`

Во многих биржевых API числовые поля передаются строками. Это выглядит странно, но делается довольно часто:

1. чтобы не терять точность при сериализации
2. чтобы клиент сам решал, во что парсить значение
3. чтобы формат был единообразным для разных языков

Поэтому в типе `BinanceTicker` цена и объём сначала читаются как строки, а потом уже превращаются в `f64`.

## Подключение и парсинг стрима

```rust
use tokio_tungstenite::connect_async;
use futures_util::StreamExt;
use serde_json;

#[tokio::main]
async fn main() {
    let url = "wss://stream.binance.com:9443/ws/btcusdt@ticker";

    let (ws_stream, _) = connect_async(url)
        .await
        .expect("Ошибка подключения к Binance");

    println!("Подключено к стриму BTCUSDT");

    let (_write, mut read) = ws_stream.split();

    let mut tick_count = 0;
    while let Some(Ok(msg)) = read.next().await {
        if let Ok(text) = msg.to_text() {
            if let Ok(ticker) = serde_json::from_str::<BinanceTicker>(text) {
                tick_count += 1;
                let price: f64 = ticker.price.parse().unwrap_or(0.0);
                let pct: f64   = ticker.price_change_pct.parse().unwrap_or(0.0);
                println!(
                    "Тик #{}: {} = ${:.2} ({:+.2}%)",
                    tick_count, ticker.symbol, price, pct
                );
            }
        }
        if tick_count >= 5 { break; }
    }
}
```

## Множественные символы через комбинированный стрим

```rust
// URL для нескольких тикеров
let url = "wss://stream.binance.com:9443/stream?streams=btcusdt@ticker/ethusdt@ticker";
```

## Обработка прерываний

```rust
use tokio::signal;
use tokio::select;

#[tokio::main]
async fn main() {
    let (ws_stream, _) = connect_async("wss://stream.binance.com:9443/ws/btcusdt@ticker")
        .await
        .unwrap();

    let (_write, mut read) = ws_stream.split();

    loop {
        select! {
            msg = read.next() => {
                if let Some(Ok(m)) = msg {
                    println!("Данные: {}", m.to_text().unwrap_or("..."));
                } else {
                    break;
                }
            }
            _ = signal::ctrl_c() => {
                println!("Прерывание — закрываем стрим");
                break;
            }
        }
    }
}
```

## Как читать `serde(rename = ...)`

Поле API может называться коротко и неудобно, например `"s"` или `"c"`. В Rust-структуре мы даём ему нормальное имя:

```rust
#[serde(rename = "c")]
price: String,
```

Это значит: в JSON поле называется `c`, а в твоём коде ты работаешь с ним как с `price`.

Так код получается заметно понятнее.

## Подключение к стриму по шагам

Типичный цикл работы выглядит так:

1. собираем URL стрима
2. подключаемся через `connect_async`
3. делим поток на чтение и запись
4. в цикле читаем `Message`
5. превращаем текст сообщения в JSON
6. извлекаем цену, объём и другие поля

```rust
let url = "wss://stream.binance.com:9443/ws/btcusdt@ticker";
let (ws_stream, _) = connect_async(url).await.expect("Ошибка подключения");
let (_write, mut read) = ws_stream.split();
```

После этого программа живёт уже не по схеме «сделал запрос и закончил», а по схеме «слушаю поток событий до остановки».

## Combined stream: что меняется

Когда ты подписываешься сразу на несколько инструментов, формат сообщения обычно становится чуть сложнее: сервер может оборачивать полезные данные в дополнительный JSON-объект.

То есть идея такая:

1. одиночный стрим часто присылает сразу событие
2. combined stream может прислать оболочку с именем стрима и полем `data`

Поэтому при работе с несколькими символами полезно сначала посмотреть на «сырой» JSON, прежде чем жёстко строить структуру десериализации.

## Практический пример: алерт на изменение цены

```rust
use futures_util::StreamExt;
use serde::Deserialize;
use tokio_tungstenite::connect_async;

#[derive(Debug, Deserialize)]
struct MiniTicker {
    #[serde(rename = "s")]
    symbol: String,
    #[serde(rename = "c")]
    price: String,
    #[serde(rename = "P")]
    change_pct: String,
}

#[tokio::main]
async fn main() {
    let (ws_stream, _) = connect_async("wss://stream.binance.com:9443/ws/btcusdt@ticker")
        .await
        .expect("Ошибка подключения");

    let (_write, mut read) = ws_stream.split();

    while let Some(Ok(message)) = read.next().await {
        if let Ok(text) = message.to_text() {
            if let Ok(ticker) = serde_json::from_str::<MiniTicker>(text) {
                let price = ticker.price.parse::<f64>().unwrap_or(0.0);
                let change = ticker.change_pct.parse::<f64>().unwrap_or(0.0);

                println!("{} => ${:.2} ({:+.2}%)", ticker.symbol, price, change);

                if change.abs() >= 1.0 {
                    println!("ALERT: сильное движение цены");
                    break;
                }
            }
        }
    }
}
```

Это уже похоже на реальную задачу мониторинга, а не просто на «распечатай JSON».

## Частые ошибки

### Ошибка 1: забыть, что цена приходит строкой

После десериализации её ещё нужно распарсить в число.

### Ошибка 2: жёстко ожидать идеальный JSON

На практике полезно писать обработку с `if let` или `match`, потому что формат события может отличаться.

### Ошибка 3: не ограничивать бесконечный цикл во время обучения

Для первых экспериментов удобно останавливать цикл после нескольких сообщений или по условию.

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| Стрим URL | `wss://stream.binance.com:9443/ws/{symbol}@ticker` |
| Десериализация JSON | `serde_json::from_str::<T>()` |
| Комбинированный стрим | Несколько символов через `/` |
| Graceful shutdown | `tokio::select!` с Ctrl+C |

## Домашнее задание

1. Подключись к стриму ETHUSDT и выводи только цену и изменение за 24ч
2. Подключись к двум символам через combined stream
3. Добавь логику: если цена выросла на 1% — выводи "ALERT: pump detected!"
4. Реализуй обработку Ctrl+C для корректного завершения

## Навигация

[← Предыдущий день](../204-tokio-tungstenite-websocket-client/ru.md) | [Следующий день →](../206-ping-pong-connection-alive/ru.md)
