# День 204: tokio-tungstenite — WebSocket клиент

## Аналогия из трейдинга

WebSocket — это постоянное подключение к бирже, как прямая телефонная линия. Библиотека `tokio-tungstenite` — это async реализация WebSocket клиента для Rust.

## Зависимости в Cargo.toml

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = { version = "0.20", features = ["native-tls"] }
futures-util = "0.3"
url = "2"
```

## Базовое подключение

```rust
use tokio_tungstenite::connect_async;
use futures_util::{StreamExt, SinkExt};
use url::Url;

#[tokio::main]
async fn main() {
    let url = Url::parse("wss://stream.binance.com:9443/ws/btcusdt@ticker")
        .expect("Неверный URL");

    let (ws_stream, response) = connect_async(url)
        .await
        .expect("Ошибка подключения");

    println!("Подключено! Статус: {}", response.status());

    let (mut write, mut read) = ws_stream.split();

    // Читаем первые 3 сообщения
    let mut count = 0;
    while let Some(msg) = read.next().await {
        match msg {
            Ok(m) => {
                println!("Сообщение: {}", m.to_text().unwrap_or("binary"));
                count += 1;
                if count >= 3 { break; }
            }
            Err(e) => {
                eprintln!("Ошибка: {}", e);
                break;
            }
        }
    }
}
```

## Отправка сообщений (подписка)

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use serde_json::json;

#[tokio::main]
async fn main() {
    let (ws_stream, _) = connect_async("wss://stream.binance.com:9443/stream")
        .await
        .expect("Ошибка подключения");

    let (mut write, mut read) = ws_stream.split();

    // Подписываемся на стрим
    let subscribe = json!({
        "method": "SUBSCRIBE",
        "params": ["btcusdt@ticker"],
        "id": 1
    });

    write.send(Message::Text(subscribe.to_string()))
        .await
        .expect("Ошибка отправки");

    // Читаем ответы
    while let Some(Ok(msg)) = read.next().await {
        if msg.is_text() {
            println!("{}", msg.to_text().unwrap());
            break; // читаем одно сообщение
        }
    }
}
```

## Что именно даёт `tokio-tungstenite`

Это библиотека, которая соединяет два мира:

1. протокол WebSocket
2. асинхронный runtime Tokio

Благодаря этому мы можем:

1. подключаться без блокировки потока
2. ждать входящие сообщения через `.await`
3. одновременно читать и писать в сокет

Для торговых систем это особенно удобно, потому что биржевой стрим по своей природе бесконечный и событийный.

## Подключение по шагам

Когда ты пишешь:

```rust
let (ws_stream, response) = connect_async(url).await.expect("Ошибка подключения");
```

происходит следующее:

1. создаётся TCP/TLS соединение
2. выполняется WebSocket handshake
3. сервер отвечает HTTP-статусом и подтверждает апгрейд протокола
4. ты получаешь готовый объект `ws_stream`

Именно поэтому полезно иногда смотреть и на `response.status()` — это помогает понять, действительно ли сервер принял соединение.

## Зачем нужен `split()`

WebSocket-соединение двустороннее: мы можем и читать, и писать. Метод `split()` делит поток на две части:

1. `write` — отправка сообщений
2. `read` — получение сообщений

```rust
let (mut write, mut read) = ws_stream.split();
```

Это удобно, потому что дальше можно независимо обрабатывать входящий и исходящий поток.

## Как выглядит базовый цикл чтения

```rust
while let Some(msg) = read.next().await {
    match msg {
        Ok(message) => println!("Получено: {:?}", message),
        Err(error) => {
            eprintln!("Ошибка чтения: {}", error);
            break;
        }
    }
}
```

Тут важно заметить две вещи:

1. `next().await` ждёт следующее сообщение асинхронно
2. `None` означает, что поток закончился

То есть WebSocket-цикл почти всегда строится вокруг `while let Some(...) = ...`.

## Практический пример: эхо-клиент с отправкой и чтением

```rust
use futures_util::{SinkExt, StreamExt};
use tokio_tungstenite::{connect_async, tungstenite::Message};

#[tokio::main]
async fn main() {
    let (ws_stream, _) = connect_async("wss://echo.websocket.events")
        .await
        .expect("Не удалось подключиться");

    let (mut write, mut read) = ws_stream.split();

    write
        .send(Message::Text("ping from rust".into()))
        .await
        .expect("Не удалось отправить сообщение");

    if let Some(Ok(reply)) = read.next().await {
        println!("Ответ сервера: {}", reply.to_text().unwrap_or("<binary>"));
    }
}
```

Это минимальный законченный сценарий:

1. подключились
2. отправили текст
3. дождались ответа

Для понимания WebSocket-клиента этот пример намного полезнее, чем просто «подключились и что-то читаем».

## Частые ошибки

### Ошибка 1: забыть `.await` у `connect_async` или `send`

Async API не выполнится само. Без `.await` ты просто создашь future.

### Ошибка 2: пытаться читать и писать из одного объекта без разделения

Для учебных примеров это иногда возможно, но `split()` делает архитектуру понятнее.

### Ошибка 3: ожидать, что все сообщения текстовые

WebSocket может присылать текст, бинарные данные, ping/pong и close frames.

## Что мы узнали

| Компонент | Описание |
|-----------|----------|
| `connect_async(url)` | Подключение к WebSocket |
| `ws_stream.split()` | Разделение на write и read |
| `read.next().await` | Получение следующего сообщения |
| `write.send(msg)` | Отправка сообщения |
| `Message::Text(s)` | Текстовое сообщение |

## Домашнее задание

1. Подключись к тестовому WebSocket эхо-серверу `wss://echo.websocket.org` и отправь сообщение
2. Реализуй подписку на 2 символа одновременно
3. Добавь обработку ошибок подключения с выводом понятного сообщения
4. Парси JSON из WebSocket и извлекай только цену (поле "p" или "lastPrice")

## Навигация

[← Предыдущий день](../203-websocket-streaming-data/ru.md) | [Следующий день →](../205-connecting-to-price-stream/ru.md)
