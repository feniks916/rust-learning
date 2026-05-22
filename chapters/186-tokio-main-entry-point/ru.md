# День 186: #[tokio::main] — входная точка async приложения

## Аналогия из трейдинга

Представь торговую платформу. Прежде чем торговать, нужно запустить сервер: инициализировать потоки, подключить базу данных, открыть WebSocket соединения. `#[tokio::main]` — это именно такой "запуск платформы": он создаёт асинхронный движок (runtime) и запускает твою главную функцию уже внутри него, где доступны все async возможности.

## Что делает макрос

`#[tokio::main]` — это процедурный макрос, который преобразует `async fn main()` в обычную `fn main()`, создающую Tokio runtime и запускающую твою async функцию:

```rust
// То, что ты пишешь:
#[tokio::main]
async fn main() {
    println!("Торговый бот запущен");
}
```

Без этого макроса `async fn main()` в Rust не существует — язык не поддерживает асинхронную main нативно.

## Во что раскрывается

Макрос разворачивается примерно в следующий код:

```rust
// То, во что разворачивается #[tokio::main]:
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            // Здесь твой async код
            println!("Торговый бот запущен");
        })
}
```

`block_on` блокирует текущий поток до завершения async блока — это безопасный мост между синхронным миром `fn main()` и асинхронным миром Tokio.

## async fn main()

В async main ты можешь использовать `.await` напрямую:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("1. Подключаемся к бирже...");
    sleep(Duration::from_millis(100)).await; // не блокирует поток
    println!("2. Загружаем исторические данные...");
    sleep(Duration::from_millis(100)).await;
    println!("3. Запускаем стратегию");
}
```

Каждый `.await` — точка, где Tokio может переключиться на другие задачи. Это позволяет тысячам WebSocket соединений работать в одном потоке.

## await в main

В async main можно напрямую вызывать async функции:

```rust
async fn fetch_price(symbol: &str) -> f64 {
    // В реальном коде здесь был бы HTTP запрос
    tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    match symbol {
        "BTCUSDT" => 65000.0,
        "ETHUSDT" => 3200.0,
        _ => 0.0,
    }
}

#[tokio::main]
async fn main() {
    let btc = fetch_price("BTCUSDT").await;
    let eth = fetch_price("ETHUSDT").await;

    println!("BTC: ${:.2}", btc);
    println!("ETH: ${:.2}", eth);
    println!("BTC/ETH ratio: {:.2}", btc / eth);
}
```

## -> Result в main

async main может возвращать `Result` для удобной обработки ошибок через `?`:

```rust
use std::error::Error;

async fn connect_to_exchange(url: &str) -> Result<String, Box<dyn Error>> {
    if url.is_empty() {
        return Err("URL биржи не может быть пустым".into());
    }
    tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    Ok(format!("Подключено к {}", url))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let connection = connect_to_exchange("wss://stream.binance.com").await?;
    println!("{}", connection);

    // Если здесь вернётся Err — программа завершится с ненулевым кодом
    let _config = std::fs::read_to_string("strategy.toml")?;

    println!("Всё инициализировано");
    Ok(())
}
```

## flavor = "current_thread"

Для лёгких приложений или тестов — однопоточный runtime:

```rust
// Использует только один поток — меньше накладных расходов
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("Однопоточный режим");

    // Полезно для скриптов, CLI инструментов, тестирования
    let result = calculate_rsi(vec![100.0, 102.0, 101.0, 103.0]).await;
    println!("RSI: {:.2}", result);
}

async fn calculate_rsi(prices: Vec<f64>) -> f64 {
    // Имитация вычисления
    tokio::time::sleep(tokio::time::Duration::from_millis(1)).await;
    prices.last().map(|p| p / 100.0 * 50.0).unwrap_or(50.0)
}
```

## flavor = "multi_thread" с worker_threads

По умолчанию Tokio создаёт по одному рабочему потоку на каждое логическое ядро CPU. Можно задать явно:

```rust
// 4 рабочих потока для параллельной обработки нескольких стримов
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    println!("Запущено с 4 рабочими потоками");

    // Параллельный мониторинг 4 инструментов
    let handles: Vec<_> = ["BTCUSDT", "ETHUSDT", "BNBUSDT", "SOLUSDT"]
        .iter()
        .map(|sym| {
            let s = sym.to_string();
            tokio::spawn(async move {
                println!("Поток обрабатывает: {}", s);
            })
        })
        .collect();

    for h in handles {
        h.await.unwrap();
    }
}
```

## Ручной Runtime::new()

Иногда нужен полный контроль над runtime:

```rust
use tokio::runtime::Runtime;

fn main() {
    // Ручное создание runtime
    let rt = Runtime::new().expect("Не удалось создать Tokio runtime");

    rt.block_on(async {
        println!("Работаем внутри runtime");
        tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
        println!("Готово");
    });

    // Runtime можно использовать повторно
    rt.block_on(async {
        println!("Снова внутри того же runtime");
    });

    // Runtime завершается при drop
    println!("Runtime завершён");
}
```

Ручной `Runtime` нужен когда: встраиваешь Tokio в синхронное приложение, или создаёшь несколько независимых runtime.

## Практический пример: async main с несколькими await

```rust
use tokio::time::{sleep, Duration, Instant};
use std::collections::HashMap;

// Имитация асинхронного получения цены
async fn fetch_price_async(symbol: &str) -> Result<f64, String> {
    sleep(Duration::from_millis(20)).await;
    let prices = [
        ("BTCUSDT", 65000.0),
        ("ETHUSDT", 3200.0),
        ("BNBUSDT", 420.0),
        ("SOLUSDT", 180.0),
    ];
    prices.iter()
        .find(|(s, _)| *s == symbol)
        .map(|(_, p)| *p)
        .ok_or_else(|| format!("Символ {} не найден", symbol))
}

// Имитация асинхронной загрузки конфига
async fn load_config() -> HashMap<String, String> {
    sleep(Duration::from_millis(10)).await;
    let mut config = HashMap::new();
    config.insert("strategy".to_string(), "momentum".to_string());
    config.insert("risk_pct".to_string(), "1.0".to_string());
    config.insert("max_positions".to_string(), "5".to_string());
    config
}

// Имитация асинхронного подключения к бирже
async fn connect_exchange(name: &str) -> Result<String, String> {
    sleep(Duration::from_millis(30)).await;
    Ok(format!("ws://{}:9443/stream", name.to_lowercase()))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let start = Instant::now();
    println!("=== Запуск торгового бота ===\n");

    // Шаг 1: Загрузка конфигурации
    println!("Загрузка конфигурации...");
    let config = load_config().await;
    println!("Стратегия: {}", config["strategy"]);
    println!("Риск: {}%", config["risk_pct"]);

    // Шаг 2: Подключение к биржам
    println!("\nПодключение к биржам...");
    let binance_url = connect_exchange("Binance").await?;
    let bybit_url = connect_exchange("Bybit").await?;
    println!("Binance: {}", binance_url);
    println!("Bybit:   {}", bybit_url);

    // Шаг 3: Получение начальных цен
    println!("\nПолучение цен...");
    let symbols = ["BTCUSDT", "ETHUSDT", "BNBUSDT", "SOLUSDT"];
    let mut prices = HashMap::new();

    for symbol in &symbols {
        match fetch_price_async(symbol).await {
            Ok(price) => {
                println!("  {}: ${:.2}", symbol, price);
                prices.insert(symbol.to_string(), price);
            }
            Err(e) => println!("  {}: ошибка — {}", symbol, e),
        }
    }

    // Шаг 4: Вычисление метрик портфеля
    println!("\nМетрики портфеля:");
    let total_value: f64 = prices.values().sum();
    println!("  Суммарная стоимость: ${:.2}", total_value);
    println!("  Количество инструментов: {}", prices.len());

    if let Some(btc) = prices.get("BTCUSDT") {
        if let Some(eth) = prices.get("ETHUSDT") {
            println!("  BTC/ETH соотношение: {:.2}", btc / eth);
        }
    }

    let elapsed = start.elapsed();
    println!("\n=== Бот готов к работе за {:.0}мс ===", elapsed.as_millis());

    Ok(())
}
```

## Что мы узнали

| Синтаксис | Описание | Применение |
|-----------|----------|------------|
| `#[tokio::main]` | Макрос входной точки | Запуск async приложения |
| `async fn main()` | Асинхронная главная функция | Использование `.await` в main |
| `.await` в main | Ожидание async операций | HTTP, WebSocket, файловый ввод-вывод |
| `-> Result<(), E>` в main | Возврат ошибки из main | Удобное использование `?` |
| `flavor = "current_thread"` | Однопоточный runtime | Скрипты, тесты, CLI |
| `flavor = "multi_thread"` | Многопоточный runtime | Серверы, боты с несколькими стримами |
| `worker_threads = N` | Число рабочих потоков | Настройка параллелизма |
| `Runtime::new()` | Ручное создание runtime | Встраивание в синхронный код |

## Домашнее задание

1. Создай async функции для инициализации торгового бота:
   - `async fn init_db() -> Result<String, String>` — имитирует подключение к БД (sleep 30мс)
   - `async fn fetch_balance() -> f64` — возвращает баланс (sleep 20мс, возвращает 10000.0)
   - `async fn subscribe_feed(symbol: &str) -> String` — подписка на стрим (sleep 15мс)

2. Напиши `#[tokio::main] async fn main() -> Result<(), Box<dyn Error>>`:
   - Последовательно вызови все три функции через `.await`
   - Используй `?` для распространения ошибок
   - Измеряй время каждого шага через `Instant::now()`

3. Попробуй `flavor = "current_thread"` — работает ли программа так же?

4. Создай вторую программу с `Runtime::new()` (без макроса):
   - Выполни те же шаги через `rt.block_on(async { ... })`
   - Сравни: одинаково ли работает?

5. Добавь инициализацию для 3 символов в цикле (BTCUSDT, ETHUSDT, BNBUSDT):
   - Для каждого вызывай `subscribe_feed` с `.await`
   - Выводи сообщение об успешной подписке

## Навигация

[← Предыдущий день](../185-tokio-runtime-async-engine/ru.md) | [Следующий день →](../187-tokio-spawn-async-tasks/ru.md)
