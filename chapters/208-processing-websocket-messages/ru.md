# День 208: Обработка WebSocket сообщений

## Аналогия из трейдинга

Биржевой стрим — это не только цены. Приходят разные типы сообщений: тики, агрегированные сделки, обновления стакана, системные события. Нужно правильно обработать каждый тип.

## Разные типы сообщений

```rust
use tokio_tungstenite::tungstenite::Message;
use futures_util::StreamExt;

async fn process_messages(mut read: impl StreamExt<Item = Result<Message, tokio_tungstenite::tungstenite::Error>> + Unpin) {
    while let Some(result) = read.next().await {
        match result {
            Ok(Message::Text(text)) => {
                handle_text(&text).await;
            }
            Ok(Message::Binary(data)) => {
                println!("Бинарное сообщение: {} байт", data.len());
            }
            Ok(Message::Ping(data)) => {
                println!("Получен Ping");
                // Pong отправляется автоматически
            }
            Ok(Message::Pong(_)) => {
                println!("Получен Pong");
            }
            Ok(Message::Close(frame)) => {
                println!("Сервер закрыл соединение: {:?}", frame);
                break;
            }
            Err(e) => {
                eprintln!("Ошибка: {}", e);
                break;
            }
            _ => {}
        }
    }
}

async fn handle_text(text: &str) {
    // Пытаемся определить тип сообщения
    if let Ok(value) = serde_json::from_str::<serde_json::Value>(text) {
        if value.get("e").map(|e| e == "24hrTicker").unwrap_or(false) {
            let price = value["c"].as_str().unwrap_or("0");
            println!("Тикер: {} = ${}", value["s"].as_str().unwrap_or("?"), price);
        } else if value.get("e").map(|e| e == "aggTrade").unwrap_or(false) {
            println!("Агрегированная сделка: {:?}", value["p"]);
        } else {
            println!("Другое сообщение: {}", &text[..text.len().min(100)]);
        }
    }
}
```

## Почему нужен `match` по типу сообщения

WebSocket-соединение живёт долго и может прислать не только полезные рыночные данные. Через тот же канал идут служебные кадры протокола.

Поэтому правильный мысленный подход такой:

1. сначала понять тип сообщения
2. только потом решать, что с ним делать

Именно поэтому `match result` здесь важнее, чем немедленный `serde_json::from_str`.

## Буферизация и батчинг

```rust
use std::collections::VecDeque;

struct MessageBuffer {
    buffer: VecDeque<String>,
    max_size: usize,
}

impl MessageBuffer {
    fn new(max_size: usize) -> Self {
        MessageBuffer { buffer: VecDeque::new(), max_size }
    }

    fn push(&mut self, msg: String) -> Option<Vec<String>> {
        self.buffer.push_back(msg);
        if self.buffer.len() >= self.max_size {
            let batch: Vec<String> = self.buffer.drain(..).collect();
            Some(batch)
        } else {
            None
        }
    }
}
```

## Хороший паттерн обработки: transport -> parse -> route

Удобно разделять обработку на три слоя:

1. транспортный слой получает `Message`
2. слой парсинга извлекает текст или JSON
3. слой маршрутизации решает, к какому типу событий относится сообщение

Тогда код не превращается в одну длинную функцию на сто строк.

```rust
fn event_type(value: &serde_json::Value) -> &str {
    value["e"].as_str().unwrap_or("unknown")
}
```

После этого можно уже отдельно писать логику для `24hrTicker`, `aggTrade`, `depthUpdate` и других событий.

## Что делает `to_text()`

Когда WebSocket присылает `Message::Text`, внутри лежит текстовый payload. Метод `to_text()` позволяет безопасно получить `&str` из сообщения.

```rust
if let Ok(text) = msg.to_text() {
    println!("Текст: {}", text);
}
```

Это важный шаг перед JSON-парсингом. Сначала убеждаемся, что кадр текстовый, потом работаем со строкой.

## Практический пример: последние цены и среднее значение

```rust
use std::collections::VecDeque;

struct PriceWindow {
    values: VecDeque<f64>,
    limit: usize,
}

impl PriceWindow {
    fn new(limit: usize) -> Self {
        Self {
            values: VecDeque::new(),
            limit,
        }
    }

    fn push(&mut self, price: f64) {
        self.values.push_back(price);
        if self.values.len() > self.limit {
            self.values.pop_front();
        }
    }

    fn average(&self) -> f64 {
        let sum: f64 = self.values.iter().sum();
        sum / self.values.len() as f64
    }
}

fn main() {
    let mut window = PriceWindow::new(3);

    for price in [50_000.0, 50_100.0, 50_050.0, 50_200.0] {
        window.push(price);
        println!("Добавили {:.2}, среднее = {:.2}", price, window.average());
    }
}
```

Этот пример напрямую связан с WebSocket-обработкой: как только из сообщения извлечена цена, её можно положить в окно и считать метрики.

## Частые ошибки

### Ошибка 1: пытаться JSON-парсить каждое сообщение подряд

Сначала проверь, что сообщение действительно `Text`.

### Ошибка 2: смешивать транспортный код и бизнес-логику

Лучше держать отдельно чтение сообщений и отдельно обработку цен или сделок.

### Ошибка 3: бесконтрольно накапливать все сообщения в памяти

Для потока нужен ограниченный буфер или окно, иначе память будет расти бесконечно.

## Что мы узнали

| Тип сообщения | Описание |
|---------------|----------|
| `Message::Text` | JSON или текст |
| `Message::Binary` | Бинарные данные |
| `Message::Ping/Pong` | Keepalive |
| `Message::Close` | Закрытие соединения |
| Матч на `serde_json::Value` | Динамический парсинг |

## Домашнее задание

1. Напиши функцию `parse_ticker(json: &str) -> Option<(String, f64)>` возвращающую (symbol, price)
2. Реализуй счётчик сообщений каждого типа (Text/Binary/Ping/Close)
3. Добавь фильтр: обрабатывай только сообщения с ценой выше порогового значения
4. Реализуй буфер последних 10 цен с вычислением скользящего среднего

## Навигация

[← Предыдущий день](../207-reconnect-restoring-connection/ru.md) | [Следующий день →](../209-parallel-websocket-subscriptions/ru.md)
