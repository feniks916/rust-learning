# День 25: match — определяем тип ордера

## Аналогия из трейдинга

На бирже ордера бывают разных типов: рыночный, лимитный, стоп, стоп-лимит, трейлинг-стоп. Каждый тип обрабатывается по-своему: рыночный — немедленно по текущей цене, лимитный — ждёт нужного уровня, стоп — активируется при пробое. Оператор `match` — это умный диспетчер на бирже: смотрит на тип ордера и направляет его к нужному обработчику, гарантируя что все случаи обработаны. Если ты забыл описать какой-то тип — компилятор Rust скажет тебе об этом ещё до запуска программы.

## Базовый match: диспетчер типов ордера

Самый простой `match` — сопоставление строки с паттернами:

```rust
fn describe_order(order_type: &str) {
    match order_type {
        "market"     => println!("Рыночный: исполнить немедленно по лучшей цене"),
        "limit"      => println!("Лимитный: исполнить только по указанной цене или лучше"),
        "stop"       => println!("Стоп: превратить в рыночный при достижении уровня"),
        "stop_limit" => println!("Стоп-лимит: превратить в лимитный при достижении уровня"),
        "trailing"   => println!("Трейлинг: следовать за ценой с фиксированным шагом"),
        _            => println!("Неизвестный тип: '{}' — отклоняем", order_type),
    }
}

fn main() {
    describe_order("market");
    describe_order("limit");
    describe_order("stop");
    describe_order("foo");
}
```

Символ `_` — это паттерн "всё остальное", обязательный если не перечислены все возможные значения. Без него Rust не скомпилирует код.

## match с числами и диапазонами

`match` умеет сопоставлять числа, включая диапазоны с `..=`:

```rust
fn classify_pnl(pnl: i32) -> &'static str {
    match pnl {
        i32::MIN..=-1000 => "Катастрофический убыток (> -$1000)",
        -999..=-100      => "Значительный убыток",
        -99..=-1         => "Малый убыток",
        0                => "Безубыток (breakeven)",
        1..=99           => "Малая прибыль",
        100..=999        => "Хорошая прибыль",
        1000..=i32::MAX  => "Отличная прибыль (> +$1000)",
    }
}

fn classify_volume(volume: u64) -> &'static str {
    match volume {
        0              => "Нет объёма",
        1..=100        => "Розничный трейдер",
        101..=1_000    => "Средний трейдер",
        1_001..=10_000 => "Крупный трейдер",
        _              => "Институциональный трейдер",
    }
}

fn main() {
    let test_pnls = [-2000, -500, -50, 0, 50, 500, 2000];
    println!("Классификация P&L:");
    for pnl in test_pnls {
        println!("  {:>6}: {}", pnl, classify_pnl(pnl));
    }

    println!("\nКлассификация объёма:");
    for vol in [0u64, 50, 500, 5000, 50000] {
        println!("  {:>6}: {}", vol, classify_volume(vol));
    }
}
```

Диапазоны в `match` — это включающие (`..=`). Важно: диапазоны не должны пересекаться.

## match возвращает значение

`match` — это выражение, оно может возвращать значение. Все ветки должны возвращать один и тот же тип:

```rust
fn get_commission_rate(order_type: &str, is_maker: bool) -> f64 {
    match (order_type, is_maker) {
        ("market", _)     => 0.001,   // 0.10% всегда
        ("limit", true)   => 0.0002,  // 0.02% мейкер
        ("limit", false)  => 0.0006,  // 0.06% тейкер
        ("stop", _)       => 0.001,
        _                 => 0.001,
    }
}

fn get_order_priority(order_type: &str) -> u8 {
    let priority = match order_type {
        "market" => 1,  // высший приоритет
        "stop"   => 2,
        "limit"  => 3,
        _        => 99, // неизвестные — низкий приоритет
    };
    priority
}

fn main() {
    let amount = 10000.0;
    let types = [("market", false), ("limit", true), ("limit", false), ("stop", false)];

    println!("Комиссии за сделку ${:.0}:", amount);
    for (order_type, is_maker) in types {
        let rate = get_commission_rate(order_type, is_maker);
        let fee  = amount * rate;
        let label = if is_maker { "maker" } else { "taker" };
        println!("  {:<8} ({:<5}): {:.4}% = ${:.4}", order_type, label, rate * 100.0, fee);
    }
}
```

## Несколько паттернов через |

Оператор `|` объединяет несколько паттернов в одну ветку — как логическое "или":

```rust
fn trading_session(hour: u8) -> &'static str {
    match hour {
        0..=6             => "Азиатская ночная (низкая волатильность)",
        7 | 8             => "Открытие Лондона (рост волатильности)",
        9 | 10 | 11       => "Активная Лондонская сессия",
        12 | 13           => "Обеденный перерыв Лондона",
        14 | 15           => "Открытие Нью-Йорка + перекрытие",
        16 | 17 | 18 | 19 => "Активная Нью-Йоркская сессия",
        20 | 21           => "Закрытие Нью-Йорка",
        22 | 23           => "Азиатская вечерняя",
        _                 => "Неизвестный час",
    }
}

fn is_high_volatility_hour(hour: u8) -> bool {
    match hour {
        7 | 8 | 14 | 15 | 20 | 21 => true,  // открытие/закрытие
        _                          => false,
    }
}

fn main() {
    println!("Расписание торговых сессий:");
    for hour in [0u8, 7, 9, 12, 14, 16, 20, 22] {
        let session   = trading_session(hour);
        let volatile  = if is_high_volatility_hour(hour) { "⚡ волатильно" } else { "спокойно" };
        println!("  {:02}:00 — {} ({})", hour, session, volatile);
    }
}
```

## Guards: условие if внутри паттерна

Guard — дополнительное условие `if` прямо в паттерне, которое уточняет когда ветка должна сработать:

```rust
fn analyze_trade(ticker: &str, pnl: f64) -> &'static str {
    match (ticker, pnl as i64) {
        ("BTC", p) if p > 500  => "BTC крупная победа",
        ("BTC", p) if p > 0    => "BTC небольшая победа",
        ("BTC", _)             => "BTC убыток",
        (t, p) if p > 100 && t.starts_with('E') => "ETH-family хорошая прибыль",
        (_, p) if p > 0        => "Прибыльная сделка",
        (_, p) if p == 0       => "Безубыток",
        _                      => "Убыточная сделка",
    }
}

fn main() {
    let trades = vec![
        ("BTC", 800.0),
        ("BTC", 200.0),
        ("BTC", -150.0),
        ("ETH", 250.0),
        ("SOL", 50.0),
        ("ADA", 0.0),
        ("DOT", -30.0),
    ];

    for (ticker, pnl) in trades {
        println!("  {:>6} {:>+8.2}: {}", ticker, pnl, analyze_trade(ticker, pnl));
    }
}
```

Guards позволяют выражать сложные условия, которые не описываются паттернами диапазонов. Порядок веток важен: первое совпадение побеждает.

## match на кортеже: комбинирование условий

`match` на кортеже позволяет проверять сразу несколько значений:

```rust
fn order_action(side: &str, price_vs_market: &str) -> &'static str {
    // side: "buy" или "sell"
    // price_vs_market: "above", "at", "below"
    match (side, price_vs_market) {
        ("buy",  "below") => "Лимитный бай — хорошая цена, ждём",
        ("buy",  "at")    => "Бай по рынку — немедленно",
        ("buy",  "above") => "Покупка выше рынка — возможно ловим пробой",
        ("sell", "above") => "Лимитный сел — хорошая цена, ждём",
        ("sell", "at")    => "Сел по рынку — немедленно",
        ("sell", "below") => "Продажа ниже рынка — возможно стоп",
        _                 => "Неизвестная комбинация",
    }
}

fn main() {
    let scenarios = [
        ("buy",  "below"),
        ("buy",  "at"),
        ("sell", "above"),
        ("sell", "below"),
    ];

    for (side, price_pos) in scenarios {
        println!("  {} / {}: {}", side, price_pos, order_action(side, price_pos));
    }
}
```

## Практический диспетчер ордеров

Собираем всё вместе — полноценный диспетчер, обрабатывающий ордера разных типов:

```rust
#[derive(Debug)]
enum OrderSide { Buy, Sell }

#[derive(Debug)]
struct Order {
    id:         u32,
    ticker:     String,
    side:       OrderSide,
    order_type: String,
    price:      f64,
    quantity:   f64,
}

fn process_order(order: &Order, market_price: f64) -> String {
    let side_str = match order.side {
        OrderSide::Buy  => "BUY",
        OrderSide::Sell => "SELL",
    };

    match order.order_type.as_str() {
        "market" => {
            let filled_at = market_price;
            let total = filled_at * order.quantity;
            format!("#{} {} {} {} шт. по ${:.2} = ${:.2} [ИСПОЛНЕН]",
                order.id, side_str, order.ticker, order.quantity, filled_at, total)
        }
        "limit" => {
            let can_fill = match order.side {
                OrderSide::Buy  => market_price <= order.price,
                OrderSide::Sell => market_price >= order.price,
            };
            if can_fill {
                format!("#{} {} {} {} шт. по ${:.2} [ИСПОЛНЕН]",
                    order.id, side_str, order.ticker, order.quantity, order.price)
            } else {
                format!("#{} {} {} ждёт цены ${:.2} (рынок ${:.2}) [ОЖИДАНИЕ]",
                    order.id, side_str, order.ticker, order.price, market_price)
            }
        }
        "stop" => {
            let triggered = match order.side {
                OrderSide::Buy  => market_price >= order.price,
                OrderSide::Sell => market_price <= order.price,
            };
            if triggered {
                format!("#{} {} {} СТОП сработал при ${:.2} [АКТИВИРОВАН→РЫНОЧНЫЙ]",
                    order.id, side_str, order.ticker, market_price)
            } else {
                format!("#{} {} {} стоп-уровень ${:.2} не достигнут [ОЖИДАНИЕ]",
                    order.id, side_str, order.ticker, order.price)
            }
        }
        unknown => format!("#{} Неизвестный тип '{}' [ОТКЛОНЁН]", order.id, unknown),
    }
}

fn main() {
    let market_price = 51500.0;

    let orders = vec![
        Order { id: 1, ticker: "BTC".to_string(), side: OrderSide::Buy,  order_type: "market".to_string(), price: 0.0,     quantity: 0.1 },
        Order { id: 2, ticker: "BTC".to_string(), side: OrderSide::Buy,  order_type: "limit".to_string(),  price: 50000.0, quantity: 0.2 },
        Order { id: 3, ticker: "BTC".to_string(), side: OrderSide::Sell, order_type: "limit".to_string(),  price: 52000.0, quantity: 0.1 },
        Order { id: 4, ticker: "ETH".to_string(), side: OrderSide::Sell, order_type: "stop".to_string(),   price: 52000.0, quantity: 1.0 },
        Order { id: 5, ticker: "SOL".to_string(), side: OrderSide::Buy,  order_type: "oco".to_string(),    price: 0.0,     quantity: 10.0 },
    ];

    println!("Рыночная цена BTC: ${:.2}", market_price);
    println!("{:-<65}", "");
    for order in &orders {
        println!("{}", process_order(order, market_price));
    }
}
```

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `match value { pat => expr }` | Когда нужно обработать несколько вариантов значения | Тип ордера → действие |
| `_` (wildcard) | Обязательный "любой другой" паттерн при неполном перечислении | Неизвестный тип → отклонить |
| `a..=b` в match | Диапазоны чисел | `1000..=i32::MAX => "Крупная прибыль"` |
| `pat1 \| pat2` | Несколько значений в одной ветке | `7 \| 8 \| 14 => true` (волатильные часы) |
| Guard `if cond` | Дополнительные условия внутри паттерна | `("BTC", p) if p > 500 => ...` |
| `match (a, b)` на кортеже | Комбинирование нескольких условий | `("buy", "below") => "Лимитный бай"` |
| `match` как выражение | Вернуть значение из ветки | `let rate = match type { ... };` |

## Домашнее задание

1. Напиши функцию `fee_tier(monthly_volume: u64) -> f64` — возвращает процент комиссии по объёму торгов
   - До 10 000 → 0.1%
   - 10 001..100 000 → 0.08%
   - 100 001..1 000 000 → 0.06%
   - Свыше → 0.04%
   - Используй `match` с диапазонами

2. Реализуй диспетчер рыночных событий `handle_event(event: &str, value: f64) -> String`
   - "price_alert" + value > 55000 → "Цена выше цели"
   - "price_alert" + value < 45000 → "Цена ниже стопа"
   - "volume_spike" + value > 10000 → "Аномальный объём"
   - Используй match на кортеже со guard

3. Создай `match` по направлению и состоянию позиции `(side, in_profit: bool)`:
   - (long, true) → "Держим лонг, в прибыли"
   - (long, false) → "Лонг в убытке, рассмотреть выход"
   - и т.д. для short

4. Напиши функцию `day_type(day_num: u8) -> &str` — возвращает тип дня:
   - 1, 7 → "Выходной"
   - 2..=6 → "Рабочий"
   - Используй `|` для объединения паттернов

## Навигация

[← Предыдущий день](../024-continue-skip-losing-trades/ru.md) | [Следующий день →](../026-constants-fixed-exchange-fee/ru.md)
