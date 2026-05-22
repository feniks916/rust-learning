# День 211: Отладка асинхронного кода

## Аналогия из трейдинга

Баг в торговом роботе может стоить денег. Отладка async кода — сложнее синхронного: задачи запускаются параллельно, стек вызовов нелинейный. Нужны специальные инструменты.

## Почему async-код отлаживать сложнее

В синхронной программе поток выполнения обычно очевиден: одна функция вызвала другую, потом третью. В async-коде картина менее линейная:

1. задачи приостанавливаются на `.await`
2. scheduler Tokio возобновляет их позже
3. несколько операций могут чередоваться во времени
4. ошибка проявляется не там, где была допущена причина

Поэтому для async-кода особенно важны хорошие логи, таймауты и наблюдаемость.

## console-subscriber: визуализация задач

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
console-subscriber = "0.2"
```

```rust
fn main() {
    console_subscriber::init(); // запускаем tokio-console сервер

    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async_main());
}

async fn async_main() {
    // Запускаем tokio-console: cargo install tokio-console
    // Затем в другом терминале: tokio-console
    tokio::time::sleep(tokio::time::Duration::from_secs(10)).await;
}
```

## Логирование в async коде

```rust
use tracing::{info, warn, error, instrument};
use tokio::time::{sleep, Duration};

#[instrument]
async fn fetch_price(symbol: &str) -> Result<f64, String> {
    info!("Запрашиваем цену для {}", symbol);
    sleep(Duration::from_millis(100)).await;

    if symbol == "INVALID" {
        warn!("Неизвестный символ: {}", symbol);
        return Err(format!("Неизвестный символ: {}", symbol));
    }

    let price = 50000.0;
    info!("Получена цена: ${}", price);
    Ok(price)
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    match fetch_price("BTCUSDT").await {
        Ok(p)  => info!("Успех: ${}", p),
        Err(e) => error!("Ошибка: {}", e),
    }
}
```

## Дедлоки в async коде

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let mutex = Arc::new(Mutex::new(0_u64));

    let m1 = mutex.clone();
    let task = tokio::spawn(async move {
        let mut guard = m1.lock().await;
        *guard += 1;
        // ВАЖНО: не держи lock через await!
        // sleep(Duration::from_secs(1)).await; // ДЕДЛОК если другая задача ждёт
        println!("Значение: {}", *guard);
    });

    task.await.unwrap();
}
```

## Таймауты для обнаружения зависаний

```rust
use tokio::time::{timeout, Duration};

async fn slow_operation() -> f64 {
    tokio::time::sleep(Duration::from_secs(10)).await;
    42.0
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_secs(2), slow_operation()).await {
        Ok(result)  => println!("Результат: {}", result),
        Err(_)      => println!("Таймаут! Операция зависла"),
    }
}
```

## Логирование через `tracing` — первый обязательный инструмент

Обычный `println!` иногда помогает, но для асинхронных задач быстро становится недостаточен. `tracing` полезен тем, что позволяет видеть контекст:

1. какая именно задача выполняется
2. какая функция была вызвана
3. какой symbol, order_id или request_id участвовал в операции

```rust
use tracing::info;

async fn process_tick(symbol: &str, price: f64) {
    info!(symbol = symbol, price = price, "обрабатываем тик");
}
```

Даже такой простой structured log часто экономит часы поиска ошибок.

## Почему опасно держать lock через `.await`

Это одна из самых типичных проблем в async-коде. Если задача захватила `Mutex` и потом ушла в `.await`, другие задачи могут надолго зависнуть в ожидании этого lock.

Полезное правило:

1. взять lock
2. быстро прочитать или изменить данные
3. отпустить lock
4. только потом делать долгую async-операцию

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

async fn safe_increment(shared: Arc<Mutex<u64>>) {
    {
        let mut guard = shared.lock().await;
        *guard += 1;
    }

    tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
}
```

## Практический чек-лист отладки

Когда async-код ведёт себя странно, удобно идти в таком порядке:

1. добавить `tracing`-логи на входе и выходе ключевых async-функций
2. проверить, нет ли `await` внутри захваченного lock
3. поставить `timeout` на потенциально зависающие операции
4. посмотреть задачи через `tokio-console`
5. упростить сценарий до минимального воспроизводимого примера

Этот порядок почти всегда полезнее, чем хаотично ставить `println!` в десять мест.

## Практический пример: таймаут и логирование запроса

```rust
use tokio::time::{timeout, Duration, sleep};
use tracing::{error, info};

async fn fetch_snapshot() -> Result<&'static str, &'static str> {
    sleep(Duration::from_millis(300)).await;
    Ok("snapshot-ready")
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    info!("стартуем запрос снапшота");

    match timeout(Duration::from_millis(150), fetch_snapshot()).await {
        Ok(Ok(value)) => info!(result = value, "запрос завершён"),
        Ok(Err(err)) => error!(error = err, "внутренняя ошибка запроса"),
        Err(_) => error!("таймаут при получении снапшота"),
    }
}
```

Даже такой маленький пример показывает полезную связку:

1. логируем начало операции
2. ограничиваем её по времени
3. различаем внутреннюю ошибку и таймаут

## Частые ошибки

### Ошибка 1: полагаться только на `println!`

Для маленького примера это нормально, но реальный async-код лучше сразу переводить на `tracing`.

### Ошибка 2: не ставить таймауты на сетевые операции

Если операция молча зависает, отладка становится намного тяжелее.

### Ошибка 3: пытаться диагностировать сложный баг сразу во всей системе

Намного эффективнее вырезать минимальный воспроизводимый async-сценарий и проверять его отдельно.

## Что мы узнали

| Инструмент | Описание |
|------------|----------|
| `tokio-console` | Визуальный мониторинг задач |
| `#[instrument]` | Автоматические span'ы в tracing |
| Не держи lock через await | Предотвращение дедлоков |
| `timeout()` | Защита от зависающих операций |

## Домашнее задание

1. Добавь `tracing` в проект, используй `info!`, `warn!`, `error!` в async функциях
2. Используй `#[instrument]` на async функции и посмотри на вывод
3. Реализуй обнаружение медленных операций через `timeout` с логированием
4. Создай намеренный дедлок и попробуй его диагностировать

## Навигация

[← Предыдущий день](../210-graceful-shutdown-async-tasks/ru.md) | [Следующий день →](../212-realtime-price-monitor/ru.md)
