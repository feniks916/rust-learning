# День 26: Константы — фиксированная комиссия биржи

## Аналогия из трейдинга

Комиссия биржи — величина постоянная и публичная. Binance берёт 0.1% со сделки, это не меняется от часа к часу. Размер лота, минимальный ордер, торговые пары — всё это тоже фиксированные параметры. В Rust такие "не меняющиеся никогда" значения называются константами (`const`). Они хранятся прямо в коде программы, вычисляются во время компиляции и гарантированно никогда не будут изменены — компилятор физически не позволит это сделать. Это делает код безопаснее и быстрее: нет переменной — нет ошибки случайного изменения.

## const vs let: в чём разница

Сравним `const` и `let` напрямую, чтобы понять когда что использовать:

```rust
// КОНСТАНТЫ — известны при компиляции, доступны глобально
const BINANCE_FEE:       f64 = 0.001;    // 0.1%
const MAX_POSITION_SIZE: f64 = 10_000.0; // $10 000
const MIN_ORDER_USDT:    f64 = 5.0;      // минимум $5
const LEVERAGE_MAX:      u8  = 20;       // плечо до 20x

fn main() {
    // ПЕРЕМЕННЫЕ — вычисляются во время выполнения, локальны
    let balance       = 5_000.0; // может зависеть от ввода пользователя
    let trade_amount  = balance * 0.1; // 10% от баланса

    println!("Баланс: ${:.2}", balance);
    println!("Размер сделки: ${:.2}", trade_amount);
    println!("Комиссия: ${:.4}", trade_amount * BINANCE_FEE);
    println!("Максимум позиции: ${:.0}", MAX_POSITION_SIZE);

    // const нельзя переопределить:
    // BINANCE_FEE = 0.002; // ОШИБКА компиляции!

    // const нельзя использовать с mut:
    // const mut X: f64 = 1.0; // ОШИБКА компиляции!
}
```

Ключевые отличия: `const` требует явного типа, вычисляется при компиляции, доступна в любом месте кода, не может быть `mut`.

## SCREAMING_SNAKE_CASE: соглашение об именовании

В Rust константы именуются в `SCREAMING_SNAKE_CASE` — все буквы заглавные, слова разделяются подчёркиванием. Это соглашение повсеместное:

```rust
// Торговые константы
const COMMISSION_RATE_SPOT:    f64  = 0.001;   // спот
const COMMISSION_RATE_FUTURES: f64  = 0.0004;  // фьючерсы
const COMMISSION_RATE_MAKER:   f64  = 0.0002;  // мейкер
const DEFAULT_SLIPPAGE_BPS:    u32  = 5;       // 0.05% в базисных пунктах
const MAX_DAILY_LOSS_PCT:       f64  = 0.05;   // 5% максимальный дневной убыток
const RISK_PER_TRADE_PCT:       f64  = 0.01;   // 1% риска на сделку
const PAIRS_SUPPORTED:          u32  = 500;

fn main() {
    let balance = 10_000.0;
    let max_loss  = balance * MAX_DAILY_LOSS_PCT;
    let risk_size = balance * RISK_PER_TRADE_PCT;

    println!("Максимальный дневной убыток: -${:.2}", max_loss);
    println!("Риск на сделку: ${:.2}", risk_size);
    println!("Комиссия спот: {:.3}%", COMMISSION_RATE_SPOT * 100.0);
    println!("Комиссия мейкер: {:.3}%", COMMISSION_RATE_MAKER * 100.0);
    println!("Поддерживаемых пар: {}", PAIRS_SUPPORTED);
}
```

Заглавный регистр — визуальный сигнал: "это константа, не трогай!". Другой программист сразу видит разницу между `price` (переменная) и `MAX_PRICE` (константа).

## const в глобальной области видимости

Константы можно объявить на уровне модуля — они будут доступны из любой функции файла:

```rust
// Глобальные торговые параметры
const API_VERSION:        &str = "v3";
const BASE_URL:           &str = "https://api.binance.com";
const DEFAULT_TIMEOUT_MS: u64  = 5_000;
const MAX_RETRIES:        u32  = 3;
const CANDLE_INTERVALS:   &str = "1m,5m,15m,1h,4h,1d";

fn build_url(endpoint: &str) -> String {
    format!("{}/api/{}/{}", BASE_URL, API_VERSION, endpoint)
}

fn get_retry_delay(attempt: u32) -> u64 {
    // Экспоненциальная задержка: 1с, 2с, 4с
    DEFAULT_TIMEOUT_MS * (2u64.pow(attempt.saturating_sub(1)))
}

fn main() {
    println!("API URL: {}", build_url("order"));
    println!("Интервалы: {}", CANDLE_INTERVALS);

    for attempt in 1..=MAX_RETRIES {
        let delay = get_retry_delay(attempt);
        println!("Попытка {}/{}: задержка {} мс", attempt, MAX_RETRIES, delay);
    }
}
```

## const vs static: тонкое различие

В Rust есть и `static` — они похожи на `const`, но имеют важные отличия:

```rust
// const — встраивается в код напрямую (inlined), нет фиксированного адреса
const MAX_ORDERS: u32 = 100;

// static — имеет фиксированный адрес в памяти, живёт всё время программы
static EXCHANGE_NAME: &str  = "Binance";
static SUPPORTED_COINS: [&str; 5] = ["BTC", "ETH", "SOL", "BNB", "ADA"];

// static mut — изменяемая статическая переменная (требует unsafe!)
static mut GLOBAL_TRADE_COUNT: u64 = 0;

fn increment_trade_counter() {
    unsafe {
        GLOBAL_TRADE_COUNT += 1;
    }
}

fn get_trade_count() -> u64 {
    unsafe { GLOBAL_TRADE_COUNT }
}

fn main() {
    println!("Биржа: {}", EXCHANGE_NAME);
    println!("Максимум ордеров: {}", MAX_ORDERS);
    println!("Поддерживаемые монеты: {:?}", SUPPORTED_COINS);

    increment_trade_counter();
    increment_trade_counter();
    increment_trade_counter();
    println!("Количество сделок: {}", get_trade_count());
}
```

`static mut` требует `unsafe` — потому что изменение глобальной переменной небезопасно в многопоточной среде. В реальных программах для счётчиков лучше использовать `std::sync::atomic::AtomicU64`.

## const в impl блоке: константы структуры

Часто удобно хранить константы внутри структуры, к которой они относятся:

```rust
struct BinanceExchange;

impl BinanceExchange {
    const FEE_SPOT:     f64 = 0.001;
    const FEE_FUTURES:  f64 = 0.0004;
    const FEE_MAKER:    f64 = 0.0002;
    const MIN_ORDER:    f64 = 5.0;
    const MAX_LEVERAGE: u8  = 125;
    const NAME:         &'static str = "Binance";

    fn calculate_fee(amount: f64, order_type: &str) -> f64 {
        let rate = match order_type {
            "spot"    => Self::FEE_SPOT,
            "futures" => Self::FEE_FUTURES,
            "maker"   => Self::FEE_MAKER,
            _         => Self::FEE_SPOT,
        };
        amount * rate
    }

    fn is_valid_order(amount: f64) -> bool {
        amount >= Self::MIN_ORDER
    }
}

struct BybitExchange;

impl BybitExchange {
    const FEE_SPOT:    f64 = 0.001;
    const FEE_FUTURES: f64 = 0.0006;
    const MIN_ORDER:   f64 = 1.0;
    const NAME:        &'static str = "Bybit";
}

fn main() {
    let amount = 1000.0;

    println!("=== {} ===", BinanceExchange::NAME);
    println!("Комиссия спот:     ${:.4}", BinanceExchange::calculate_fee(amount, "spot"));
    println!("Комиссия фьючерс:  ${:.4}", BinanceExchange::calculate_fee(amount, "futures"));
    println!("Комиссия мейкер:   ${:.4}", BinanceExchange::calculate_fee(amount, "maker"));
    println!("Мин. ордер:        ${}", BinanceExchange::MIN_ORDER);
    println!("Макс. плечо:       {}x", BinanceExchange::MAX_LEVERAGE);
    println!("Ордер $3 валиден:  {}", BinanceExchange::is_valid_order(3.0));
    println!("Ордер $10 валиден: {}", BinanceExchange::is_valid_order(10.0));

    println!("\n=== {} ===", BybitExchange::NAME);
    println!("Мин. ордер: ${}", BybitExchange::MIN_ORDER);
}
```

## const expressions: вычисления при компиляции

Константы могут быть вычислены из других констант прямо во время компиляции:

```rust
const USDT_PRECISION:      u32 = 2;        // 2 знака после запятой
const BTC_PRECISION:       u32 = 8;        // 8 знаков после запятой
const SATS_PER_BTC:        u64 = 100_000_000; // 1 BTC = 100M сатоши
const DEFAULT_LEVERAGE:    u8  = 1;
const MAX_LEVERAGE:        u8  = 20;
const SECONDS_IN_MINUTE:   u64 = 60;
const SECONDS_IN_HOUR:     u64 = SECONDS_IN_MINUTE * 60;   // вычисление из другой const
const SECONDS_IN_DAY:      u64 = SECONDS_IN_HOUR * 24;
const SECONDS_IN_WEEK:     u64 = SECONDS_IN_DAY * 7;
const CANDLE_1M_IN_DAY:    u64 = 24 * 60;   // 1440 минутных свечей в дне
const CANDLE_1H_IN_WEEK:   u64 = SECONDS_IN_WEEK / SECONDS_IN_HOUR; // 168 часовых в неделе

fn sats_to_btc(sats: u64) -> f64 {
    sats as f64 / SATS_PER_BTC as f64
}

fn btc_to_sats(btc: f64) -> u64 {
    (btc * SATS_PER_BTC as f64) as u64
}

fn main() {
    println!("Секунд в часе:        {}", SECONDS_IN_HOUR);
    println!("Секунд в дне:         {}", SECONDS_IN_DAY);
    println!("Минутных свечей/день: {}", CANDLE_1M_IN_DAY);
    println!("Часовых свечей/нед.:  {}", CANDLE_1H_IN_WEEK);
    println!("0.1 BTC в сатоши:     {}", btc_to_sats(0.1));
    println!("5000000 сатоши в BTC: {}", sats_to_btc(5_000_000));
    println!("Плечо по умолчанию:   {}x", DEFAULT_LEVERAGE);
    println!("Макс. плечо:          {}x", MAX_LEVERAGE);
}
```

## Практика: конфигурация биржи через константы

Собираем всё вместе — конфигурация торгового бота через константы:

```rust
// ===== Конфигурация биржи =====
const EXCHANGE_NAME:     &str = "Binance";
const FEE_RATE:          f64  = 0.001;
const MIN_TRADE_USDT:    f64  = 5.0;
const MAX_TRADE_USDT:    f64  = 50_000.0;

// ===== Риск-менеджмент =====
const MAX_RISK_PER_TRADE: f64 = 0.01;   // 1% от баланса
const MAX_DAILY_DRAWDOWN: f64 = 0.05;   // 5% максимальный дневной убыток
const DEFAULT_SL_PCT:     f64 = 0.02;   // 2% стоп-лосс
const DEFAULT_TP_PCT:     f64 = 0.04;   // 4% тейк-профит (RR = 1:2)

// ===== Параметры стратегии =====
const SMA_FAST_PERIOD:   u32  = 9;
const SMA_SLOW_PERIOD:   u32  = 21;
const RSI_PERIOD:        u32  = 14;
const RSI_OVERSOLD:      f64  = 30.0;
const RSI_OVERBOUGHT:    f64  = 70.0;

fn calculate_position(balance: f64, entry: f64, stop_loss: f64) -> f64 {
    let risk_amount = balance * MAX_RISK_PER_TRADE;
    let points_risk = (entry - stop_loss).abs();
    if points_risk == 0.0 { return 0.0; }
    let raw_size = risk_amount / points_risk;
    // Ограничиваем размер позиции
    let max_size = MAX_TRADE_USDT / entry;
    raw_size.min(max_size)
}

fn trade_levels(entry: f64) -> (f64, f64) {
    let stop_loss   = entry * (1.0 - DEFAULT_SL_PCT);
    let take_profit = entry * (1.0 + DEFAULT_TP_PCT);
    (stop_loss, take_profit)
}

fn main() {
    let balance = 10_000.0;
    let entry_price = 50_000.0;

    let (sl, tp) = trade_levels(entry_price);
    let size = calculate_position(balance, entry_price, sl);
    let fee  = entry_price * size * FEE_RATE;

    println!("=== {} Торговый Бот ===", EXCHANGE_NAME);
    println!();
    println!("Баланс: ${:.2}", balance);
    println!("Цена входа: ${:.2}", entry_price);
    println!();
    println!("Стоп-лосс:    ${:.2}  (-{:.0}%)", sl, DEFAULT_SL_PCT * 100.0);
    println!("Тейк-профит:  ${:.2}  (+{:.0}%)", tp, DEFAULT_TP_PCT * 100.0);
    println!("Размер позиции: {:.6} BTC", size);
    println!("Стоимость: ${:.2}", size * entry_price);
    println!("Комиссия: ${:.4} ({:.1}%)", fee, FEE_RATE * 100.0);
    println!();
    println!("Параметры индикаторов: SMA({}/{}), RSI({})", SMA_FAST_PERIOD, SMA_SLOW_PERIOD, RSI_PERIOD);
    println!("Зоны RSI: перепродан < {}, перекуплен > {}", RSI_OVERSOLD, RSI_OVERBOUGHT);

    // Проверка ограничений
    let trade_value = size * entry_price;
    if trade_value < MIN_TRADE_USDT {
        println!("ПРЕДУПРЕЖДЕНИЕ: сделка ${:.2} ниже минимума ${}", trade_value, MIN_TRADE_USDT);
    } else if trade_value > MAX_TRADE_USDT {
        println!("ПРЕДУПРЕЖДЕНИЕ: сделка ${:.2} выше максимума ${}", trade_value, MAX_TRADE_USDT);
    } else {
        println!("Ограничения: OK (${:.2} в диапазоне ${}-${})", trade_value, MIN_TRADE_USDT, MAX_TRADE_USDT);
    }
}
```

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `const NAME: Type = val;` | Значение известно при компиляции и никогда не меняется | Комиссия биржи, лимиты ордеров |
| `SCREAMING_SNAKE_CASE` | Соглашение об именовании всех констант | `MAX_POSITION_SIZE`, `FEE_RATE` |
| `const` в глобальной области | Параметры, нужные во всём файле/модуле | API URL, таймауты, периоды индикаторов |
| `static NAME: &str` | Строковые константы с фиксированным адресом | Названия бирж, версии API |
| `static mut` + `unsafe` | Изменяемый глобальный счётчик (избегать в продакшне) | Счётчик сделок (лучше `AtomicU64`) |
| `impl Struct { const X }` | Константы, логически связанные со структурой | `BinanceExchange::FEE_RATE` |
| Вычисляемые `const` | Производные значения из других констант | `SECONDS_IN_DAY = SECONDS_IN_HOUR * 24` |

## Домашнее задание

1. Объяви файл конфигурации торгового бота через константы:
   - `EXCHANGE_NAME`, `API_KEY` (mock), `BASE_URL`
   - `FEE_MAKER`, `FEE_TAKER`, `MIN_ORDER`, `MAX_ORDER`
   - Напиши функцию `validate_order(amount: f64) -> bool`, использующую константы

2. Создай структуру `RiskConfig` с константами в `impl` блоке:
   - `MAX_RISK_PER_TRADE: f64 = 0.01`
   - `MAX_DAILY_LOSS: f64 = 0.05`
   - `MAX_OPEN_POSITIONS: u8 = 5`
   - Метод `is_trade_allowed(current_loss: f64, open_positions: u8) -> bool`

3. Вычисли производные константы:
   - `CANDLE_COUNT_PER_DAY_1M: u32`
   - `CANDLE_COUNT_PER_WEEK_1H: u32`
   - Используй их в функции, строящей список временных меток

4. Сравни `const` и `static` на практике:
   - Объяви `const TICKERS: [&str; 3]` и `static TICKERS_STATIC: [&str; 3]`
   - Выведи обе в цикле
   - Добавь комментарии: когда использовать `const`, когда `static`

## Навигация

[← Предыдущий день](../025-match-determining-order-type/ru.md) | [Следующий день →](../027-shadowing-updating-price/ru.md)
