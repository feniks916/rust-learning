# День 35: Clone — копируем портфель

## Аналогия из трейдинга

Ты хочешь протестировать новую стратегию, не рискуя реальным портфелем. Для этого ты делаешь копию — виртуальный портфель с теми же позициями. Изменения в копии не влияют на оригинал. В Rust метод `.clone()` делает именно это: создаёт полную независимую копию данных в куче.

---

## .clone() — полная копия

`.clone()` — это явный запрос: "я хочу независимую копию этих данных". В отличие от Move, после clone у тебя два независимых объекта.

```rust
fn main() {
    let original_ticker = String::from("BTCUSDT");

    // Move — только один владелец
    // let moved = original_ticker;
    // println!("{}", original_ticker); // ОШИБКА

    // Clone — два независимых объекта
    let cloned_ticker = original_ticker.clone();

    println!("Оригинал: {}", original_ticker); // ОК
    println!("Копия: {}", cloned_ticker);        // ОК

    // Изменение копии не затрагивает оригинал
    let mut test_ticker = original_ticker.clone();
    test_ticker.push_str("_TEST");
    println!("Оригинал: {}", original_ticker); // "BTCUSDT" — не изменился
    println!("Тест: {}", test_ticker);          // "BTCUSDT_TEST"
}
```

---

## #[derive(Clone)] для своих структур

Чтобы `clone()` работал для твоей структуры, нужно либо реализовать трейт `Clone` вручную, либо использовать `#[derive(Clone)]`:

```rust
#[derive(Debug, Clone)]
struct Portfolio {
    name: String,
    positions: Vec<String>,
    cash_balance: f64,
}

impl Portfolio {
    fn add_position(&mut self, ticker: String) {
        self.positions.push(ticker);
    }

    fn total_positions(&self) -> usize {
        self.positions.len()
    }
}

fn main() {
    let mut real_portfolio = Portfolio {
        name: String::from("Real Account"),
        positions: vec![String::from("BTCUSDT"), String::from("ETHUSDT")],
        cash_balance: 10000.0,
    };

    // Клонируем для бэктестинга
    let mut test_portfolio = real_portfolio.clone();
    test_portfolio.name = String::from("Backtest Account");
    test_portfolio.add_position(String::from("SOLUSDT")); // только в копии

    println!("Реальный: {:?}", real_portfolio);     // 2 позиции
    println!("Тест: {:?}", test_portfolio);          // 3 позиции

    // Изменяем реальный — копия не меняется
    real_portfolio.cash_balance -= 5000.0;
    println!("Реальный баланс: ${}", real_portfolio.cash_balance);
    println!("Тест баланс: ${}", test_portfolio.cash_balance); // всё ещё 10000
}
```

---

## Deep Clone vs Shallow Clone

В Rust `clone()` всегда делает **глубокое копирование** (deep clone): все вложенные данные тоже копируются. Нет понятия "shallow clone" как в Python — вы всегда получаете полную независимость:

```rust
fn main() {
    // Сложная структура с вложенными данными
    let original: Vec<Vec<f64>> = vec![
        vec![42000.0, 42100.0, 41900.0], // дни недели
        vec![43000.0, 43200.0, 42800.0],
        vec![44000.0, 44100.0, 43900.0],
    ];

    let cloned = original.clone(); // Deep clone: копируется ВСЁТ

    // Изменить cloned нельзя через оригинал и наоборот
    // (в Python shallow clone — vec[0][0] было бы общим!)

    println!("Оригинал [0][0]: {}", original[0][0]);
    println!("Клон [0][0]: {}", cloned[0][0]);

    // Строки внутри Vec тоже клонируются полностью
    let tickers = vec![String::from("BTC"), String::from("ETH")];
    let tickers_clone = tickers.clone(); // каждая строка — отдельная копия в куче

    println!("Тикеры: {:?}", tickers);
    println!("Клон тикеров: {:?}", tickers_clone);
}
```

---

## Когда нужен clone

Clone нужен, когда необходимо:
1. Сохранить оригинал после передачи в функцию по значению
2. Создать независимую копию для модификации
3. Поместить одни данные в несколько мест

```rust
fn run_backtest(prices: Vec<f64>, strategy_name: &str) -> f64 {
    // Принимает Vec по значению — потребляет
    let returns: Vec<f64> = prices.windows(2)
        .map(|w| (w[1] - w[0]) / w[0])
        .collect();
    let total_return: f64 = returns.iter().sum();
    println!("Стратегия {}: доходность = {:.2}%", strategy_name, total_return * 100.0);
    total_return
}

fn main() {
    let price_history = vec![40000.0, 42000.0, 41000.0, 44000.0, 43000.0, 45000.0];

    // Хотим запустить несколько стратегий на одних данных
    // БЕЗ clone первый вызов "съест" price_history:
    let r1 = run_backtest(price_history.clone(), "Momentum");
    let r2 = run_backtest(price_history.clone(), "Mean Reversion");
    let r3 = run_backtest(price_history.clone(), "Buy and Hold");
    // price_history всё ещё доступен!

    println!("\nИтоги:");
    println!("  Momentum: {:.2}%", r1 * 100.0);
    println!("  Mean Rev: {:.2}%", r2 * 100.0);
    println!("  B&H: {:.2}%", r3 * 100.0);
    println!("Исходный ряд: {} точек", price_history.len());
}
```

---

## Clone дорогой — пример с измерением

Clone выделяет новую память и копирует все байты. Для больших структур это медленно:

```rust
use std::time::Instant;

fn main() {
    // Большой вектор цен — как реальные тиковые данные
    let tick_data: Vec<f64> = (0..1_000_000).map(|i| 40000.0 + (i as f64) * 0.001).collect();

    // Измеряем время clone
    let start = Instant::now();
    let _cloned = tick_data.clone(); // копируем 8 МБ
    let clone_time = start.elapsed();

    // Измеряем время работы с ссылкой (без копирования)
    let start = Instant::now();
    let sum: f64 = tick_data.iter().sum(); // просто читаем
    let ref_time = start.elapsed();

    println!("Размер данных: {} байт", tick_data.len() * 8);
    println!("Clone: {:?}", clone_time);   // например 5ms
    println!("Ref:   {:?}", ref_time);     // например 0.5ms — в 10x быстрее!
    println!("Сумма: {}", sum);

    // ВЫВОД: если нужно только читать — передавай &Vec, не клонируй!
}
```

Правило: **используй `clone()` только когда действительно нужна независимая копия**. Если просто читаешь — используй `&T`.

---

## Паттерн clone-перед-move

Классический паттерн: клонируй до того, как значение будет перемещено:

```rust
fn send_order(order: String) {
    println!("[EXCHANGE] Отправлен ордер: {}", order);
    // order уничтожается здесь
}

fn log_order(order: &str) {
    println!("[LOG] Логируем: {}", order);
}

fn main() {
    let order = String::from("BUY BTCUSDT 0.1 @ 42000");

    // Сначала логируем (через ссылку — без Move)
    log_order(&order);

    // Сохраняем копию для истории
    let order_history = order.clone(); // clone перед move

    // Отправляем оригинал (Move)
    send_order(order);
    // order теперь недоступен

    // Но у нас есть копия в истории
    println!("[HISTORY] Записан ордер: {}", order_history);

    // Паттерн для Vec
    let portfolio = vec![String::from("BTC"), String::from("ETH")];
    let portfolio_backup = portfolio.clone(); // clone перед move

    let result = portfolio.into_iter()
        .map(|t| format!("{}USDT", t))
        .collect::<Vec<_>>();

    // portfolio перемещён в into_iter и уничтожен
    println!("Результат: {:?}", result);
    println!("Бэкап: {:?}", portfolio_backup); // clone всё ещё жив
}
```

---

## Практический пример: Система бэктестирования

```rust
#[derive(Debug, Clone)]
struct TradingStrategy {
    name: String,
    stop_loss: f64,    // процент от входа
    take_profit: f64,  // процент от входа
    position_size: f64, // доля от депозита
}

#[derive(Debug, Clone)]
struct BacktestConfig {
    strategy: TradingStrategy,
    initial_capital: f64,
    prices: Vec<f64>,
}

fn run_backtest(config: BacktestConfig) -> (String, f64, usize) {
    // config потребляется (Move)
    let mut capital = config.initial_capital;
    let mut trades = 0;

    let entry_price = config.prices[0];
    for &price in &config.prices[1..] {
        let change_pct = (price - entry_price) / entry_price;

        if change_pct >= config.strategy.take_profit {
            capital *= 1.0 + config.strategy.position_size * config.strategy.take_profit;
            trades += 1;
        } else if change_pct <= -config.strategy.stop_loss {
            capital *= 1.0 - config.strategy.position_size * config.strategy.stop_loss;
            trades += 1;
        }
    }

    (config.strategy.name, capital, trades)
}

fn main() {
    let prices = vec![
        40000.0, 41000.0, 39500.0, 42000.0, 44000.0,
        43000.0, 45000.0, 44500.0, 46000.0, 45000.0,
    ];

    let base_config = BacktestConfig {
        strategy: TradingStrategy {
            name: String::from("Conservative"),
            stop_loss: 0.02,
            take_profit: 0.04,
            position_size: 0.1,
        },
        initial_capital: 10000.0,
        prices: prices.clone(), // clone для конфига
    };

    // Тестируем разные варианты стратегии — все через clone базового конфига
    let mut aggressive_config = base_config.clone();
    aggressive_config.strategy.name = String::from("Aggressive");
    aggressive_config.strategy.stop_loss = 0.05;
    aggressive_config.strategy.take_profit = 0.10;
    aggressive_config.strategy.position_size = 0.3;

    let mut scalping_config = base_config.clone();
    scalping_config.strategy.name = String::from("Scalping");
    scalping_config.strategy.stop_loss = 0.005;
    scalping_config.strategy.take_profit = 0.01;
    scalping_config.strategy.position_size = 0.5;

    println!("=== Результаты бэктеста ===");
    for config in [base_config, aggressive_config, scalping_config] {
        let (name, final_capital, trade_count) = run_backtest(config);
        println!(
            "{}: ${:.2} ({} сделок, {}%)",
            name, final_capital, trade_count,
            ((final_capital / 10000.0 - 1.0) * 100.0) as i32
        );
    }
}
```

---

## Что мы узнали

| Синтаксис | Описание | Пример в трейдинге |
|-----------|----------|--------------------|
| `.clone()` | Полная независимая копия | Клонировать портфель для теста |
| `#[derive(Clone)]` | Автоматически реализовать Clone | Структура Strategy с настройками |
| Deep clone | Все вложенные данные тоже копируются | Вся история цен, вложенная в структуру |
| Clone перед Move | Сохранить копию до передачи функции | Бэкап ордера перед отправкой |
| Clone дорогой | Выделяет память и копирует байты | Не клонируй 1М тиков без причины |
| `&T` вместо clone | Для чтения не нужна копия | Анализ данных через ссылку |

---

## Домашнее задание

1. Создай структуру `PriceBar { open: f64, high: f64, low: f64, close: f64, volume: f64 }` с `#[derive(Clone, Debug)]`.
   - Создай `Vec<PriceBar>` с 5 свечами
   - Клонируй вектор и примени к копии "нормализацию" (разделить все цены на close первой свечи)
   - Убедись, что оригинальные свечи не изменились

2. Напиши функцию `optimize_strategy(base: TradingStrategy, param_sets: Vec<(f64, f64)>) -> Vec<TradingStrategy>`:
   - `TradingStrategy { name: String, stop: f64, target: f64 }`
   - Для каждой пары параметров из `param_sets` создаёт `.clone()` базовой стратегии и подставляет параметры
   - Возвращает вектор модифицированных стратегий

3. Изучи стоимость clone:
   - Создай `Vec<String>` с 10 000 тикерами через `(0..10000).map(|i| format!("TICKER_{:05}", i)).collect()`
   - Измерь время `clone()` через `std::time::Instant`
   - Сравни с временем простого итерирования через `&vec`

4. Реализуй "снапшот и откат":
   - Структура `OrderBook { bids: Vec<f64>, asks: Vec<f64> }`
   - Функция `snapshot(book: &OrderBook) -> OrderBook` возвращает `book.clone()`
   - Функция `apply_trade(book: &mut OrderBook, price: f64)` изменяет книгу
   - После нескольких `apply_trade` откатись к снапшоту

---

## Навигация

[← Предыдущий день](../034-move-selling-asset/ru.md) | [Следующий день →](../036-copy-light-types-cash/ru.md)
