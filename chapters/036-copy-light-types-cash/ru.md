# День 36: Copy — лёгкие типы как наличные

## Аналогия из трейдинга

Наличные — это самый ликвидный актив. Когда ты даёшь кому-то купюру, у тебя не убывает — ты как бы "копируешь" её мгновенно (представь, что у тебя всегда есть принтер). Типы с трейтом `Copy` в Rust работают так же: при присваивании создаётся точная битовая копия автоматически, и оригинал остаётся нетронутым. Это работает только для маленьких типов фиксированного размера — таких, где копирование происходит мгновенно.

---

## Copy трейт

Трейт `Copy` — это маркер-трейт: он не добавляет методов, а просто говорит компилятору "при присваивании этого типа — копируй побитово, не перемещай".

```rust
fn main() {
    let price: f64 = 42000.0;
    let price_copy = price; // НЕ Move, а Copy

    // Оба доступны после присваивания!
    println!("Оригинал: {}", price);      // ОК
    println!("Копия: {}", price_copy);    // ОК

    let qty: u32 = 100;
    let qty_backup = qty; // Copy
    println!("qty={}, backup={}", qty, qty_backup); // оба живы
}
```

---

## Какие типы Copy

Все типы `Copy` — это типы **фиксированного маленького размера**, которые полностью живут на стеке:

```rust
fn main() {
    // === Целочисленные ===
    let a: i8   = -5;      // 1 байт
    let b: i16  = 1000;    // 2 байта
    let c: i32  = 42000;   // 4 байта — цена
    let d: i64  = 1_000_000; // 8 байт — объём
    let e: i128 = 999;     // 16 байт

    let a2 = a; let b2 = b; let c2 = c;
    println!("a={}, a2={}", a, a2); // Copy

    // === Беззнаковые ===
    let f: u8  = 255;
    let g: u32 = 4_294_967_295;
    let h: u64 = 18_446_744_073_709_551_615;

    // === Числа с плавающей точкой ===
    let price: f32 = 42000.0;  // 4 байта
    let precise_price: f64 = 42000.001; // 8 байт — самый частый в трейдинге

    // === Специальные ===
    let is_long: bool = true;  // 1 байт
    let side: char = 'B';      // 4 байта

    // Все копируются:
    let p2 = price;
    let pp2 = precise_price;
    println!("price={}, p2={}, pp2={}", price, p2, pp2);

    // === НЕ Copy ===
    // String, Vec<T>, Box<T>, HashMap, и любой тип с данными на куче
}
```

---

## Copy требует Clone

Трейт `Copy` требует, чтобы тип также реализовывал `Clone`. Это логично: если можно быстро скопировать, то можно и явно клонировать:

```rust
fn main() {
    // Copy типы автоматически можно клонировать
    let price: f64 = 42000.0;
    let p2 = price;         // неявный Copy (при присваивании)
    let p3 = price.clone(); // явный clone (то же самое для примитивов)

    println!("price={}, p2={}, p3={}", price, p2, p3);

    // Для нашей структуры:
    // #[derive(Clone)] — добавляет только Clone
    // #[derive(Copy, Clone)] — добавляет оба (нужны вместе)
}
```

---

## #[derive(Copy, Clone)] для своих типов

Ты можешь сделать свой тип `Copy`, если он содержит только `Copy` поля:

```rust
// Tick — маленький тип с ценой и объёмом
#[derive(Debug, Copy, Clone)]
struct Tick {
    price: f64,   // Copy
    volume: f64,  // Copy
    timestamp: u64, // Copy
}

// OrderSignal — маленький сигнал для стратегии
#[derive(Debug, Copy, Clone, PartialEq)]
enum OrderSignal {
    Buy,
    Sell,
    Hold,
}

fn main() {
    let tick1 = Tick { price: 42000.0, volume: 1.5, timestamp: 1700000000 };
    let tick2 = tick1; // Copy! не Move

    // Оба доступны:
    println!("tick1: {:?}", tick1);
    println!("tick2: {:?}", tick2);

    let signal = OrderSignal::Buy;
    let signal_copy = signal; // Copy!
    println!("signal={:?}, copy={:?}", signal, signal_copy);

    // Функции с Copy типами:
    fn process_tick(t: Tick) -> f64 { t.price * t.volume }
    let notional = process_tick(tick1); // tick1 копируется
    println!("tick1 всё ещё живёт: {:?}", tick1); // ОК!
    println!("Оборот: ${:.2}", notional);
}
```

---

## Почему String не Copy

Это важный вопрос: почему `String` не может быть `Copy`? Потому что `Copy` означает "побитовое копирование", а у `String` на стеке хранится указатель на кучу. Если скопировать побитово — два указателя будут смотреть на одни данные, и при уничтожении первого — данные будут освобождены. Второй указатель станет висячим (dangling):

```rust
// Представь, что String была бы Copy (она НЕ такова — это просто объяснение):
// let s1 = String::from("BTC");
// let s2 = s1; // если бы это был побитовый Copy:
//
// Stack: s1 = [ptr:0x100, len:3, cap:3]
// Stack: s2 = [ptr:0x100, len:3, cap:3]  <- тот же ptr!
// Heap:  0x100: ['B','T','C']
//
// Когда s1 уничтожается -> освобождает 0x100
// s2 теперь указывает на освобождённую память -> CRASH!
//
// Поэтому String использует Move: только один владелец.

fn explain_no_copy() {
    // String: данные в куче, указатель на стеке
    // Побитовая копия = два указателя на одни данные = двойное освобождение
    // ПОЭТОМУ String: Move, не Copy

    // Правильные альтернативы:
    let s1 = String::from("BTCUSDT");
    let s2 = s1.clone(); // явная глубокая копия (безопасно, но дорого)
    println!("{} {}", s1, s2);

    // Или передавать по ссылке:
    let s3 = String::from("ETHUSDT");
    fn use_ticker(t: &str) { println!("Используем: {}", t); }
    use_ticker(&s3); // ссылка, не Move, не Copy
    println!("s3 жив: {}", s3);
}

fn main() {
    explain_no_copy();
}
```

---

## Tuple из Copy типов

Кортеж из `Copy` типов — тоже `Copy`:

```rust
fn main() {
    // Кортеж из Copy типов — сам Copy
    let order: (f64, u32, bool) = (42000.0, 100, true);
    //           price  qty   is_buy
    let order2 = order; // Copy!
    println!("order={:?}, order2={:?}", order, order2); // оба живы

    // Кортеж с String — НЕ Copy (потому что String не Copy)
    let named_order: (String, f64) = (String::from("BTC"), 42000.0);
    let named2 = named_order; // Move!
    // println!("{:?}", named_order); // ОШИБКА
    println!("{:?}", named2);

    // Массив из Copy типов — Copy (если фиксированный размер)
    let prices: [f64; 3] = [42000.0, 43000.0, 44000.0];
    let prices2 = prices; // Copy!
    println!("prices={:?}, prices2={:?}", prices, prices2);

    // Vec<f64> — НЕ Copy (динамический размер -> куча)
    let vec_prices = vec![42000.0, 43000.0];
    let vec2 = vec_prices; // Move!
    println!("{:?}", vec2);
}
```

---

## Практика: использование Copy в трейдинге

```rust
#[derive(Debug, Copy, Clone)]
struct PricePoint {
    price: f64,
    volume: f64,
}

#[derive(Debug, Copy, Clone, PartialEq)]
enum Side { Buy, Sell }

#[derive(Debug, Copy, Clone)]
struct OrderParams {
    price: f64,
    quantity: f64,
    side: Side,
    leverage: u8,
}

fn calculate_notional(params: OrderParams) -> f64 {
    // params копируется при передаче (Copy)
    params.price * params.quantity * params.leverage as f64
}

fn apply_slippage(params: OrderParams, slip_bps: f64) -> OrderParams {
    // params копируется, возвращаем изменённую копию
    let slip = slip_bps / 10000.0;
    OrderParams {
        price: match params.side {
            Side::Buy  => params.price * (1.0 + slip),
            Side::Sell => params.price * (1.0 - slip),
        },
        ..params // остальные поля копируются
    }
}

fn main() {
    let order = OrderParams {
        price: 42000.0,
        quantity: 0.1,
        side: Side::Buy,
        leverage: 10,
    };

    // Можно передавать в функции многократно без clone:
    let notional = calculate_notional(order);
    let with_slip = apply_slippage(order, 5.0); // 0.5 bps slippage

    // order всё ещё доступен (Copy):
    println!("Оригинальный ордер: {:?}", order);
    println!("С учётом слиппейджа: {:?}", with_slip);
    println!("Оборот с плечом: ${:.2}", notional);

    // Пример с Copy в цикле:
    let price_levels: [PricePoint; 5] = [
        PricePoint { price: 42000.0, volume: 1.5 },
        PricePoint { price: 42100.0, volume: 0.8 },
        PricePoint { price: 42200.0, volume: 2.1 },
        PricePoint { price: 42300.0, volume: 0.5 },
        PricePoint { price: 42400.0, volume: 1.2 },
    ];

    for level in price_levels { // Copy: не нужна ссылка
        println!("  ${:.0} x {:.1}", level.price, level.volume);
    }

    // price_levels всё ещё жив (Copy):
    println!("Уровней в стакане: {}", price_levels.len());
}
```

---

## Практический пример: Калькулятор риска

```rust
#[derive(Debug, Copy, Clone)]
struct RiskParams {
    stop_loss_pct: f64,    // процент stop-loss
    take_profit_pct: f64,  // процент take-profit
    risk_per_trade: f64,   // риск на сделку (% от капитала)
    max_leverage: u8,
}

#[derive(Debug, Copy, Clone)]
struct TradeSetup {
    entry_price: f64,
    capital: f64,
    is_long: bool,
}

fn calculate_position_size(setup: TradeSetup, risk: RiskParams) -> f64 {
    let risk_amount = setup.capital * risk.risk_per_trade / 100.0;
    let price_risk = setup.entry_price * risk.stop_loss_pct / 100.0;
    risk_amount / price_risk
}

fn calculate_exit_levels(setup: TradeSetup, risk: RiskParams) -> (f64, f64) {
    if setup.is_long {
        let stop = setup.entry_price * (1.0 - risk.stop_loss_pct / 100.0);
        let target = setup.entry_price * (1.0 + risk.take_profit_pct / 100.0);
        (stop, target)
    } else {
        let stop = setup.entry_price * (1.0 + risk.stop_loss_pct / 100.0);
        let target = setup.entry_price * (1.0 - risk.take_profit_pct / 100.0);
        (stop, target)
    }
}

fn risk_reward_ratio(risk: RiskParams) -> f64 {
    risk.take_profit_pct / risk.stop_loss_pct
}

fn main() {
    // Оба типа Copy — можно передавать много раз без clone
    let conservative = RiskParams {
        stop_loss_pct: 2.0,
        take_profit_pct: 4.0,
        risk_per_trade: 1.0,
        max_leverage: 5,
    };

    let aggressive = RiskParams {
        stop_loss_pct: 5.0,
        take_profit_pct: 15.0,
        risk_per_trade: 3.0,
        max_leverage: 20,
    };

    let setup = TradeSetup {
        entry_price: 42000.0,
        capital: 10000.0,
        is_long: true,
    };

    for &risk in &[conservative, aggressive] {
        let size = calculate_position_size(setup, risk); // Copy x2
        let (stop, target) = calculate_exit_levels(setup, risk); // Copy x2
        let rr = risk_reward_ratio(risk); // Copy

        println!("--- Параметры риска ---");
        println!("  Размер позиции: {:.4} BTC", size);
        println!("  Stop-loss: ${:.2}", stop);
        println!("  Take-profit: ${:.2}", target);
        println!("  Risk/Reward: 1:{:.1}", rr);
    }

    // setup и conservative и aggressive — все живы (Copy):
    println!("\nВход: ${}", setup.entry_price);
    println!("Консервативный RR: 1:{:.1}", risk_reward_ratio(conservative));
    println!("Агрессивный RR: 1:{:.1}", risk_reward_ratio(aggressive));
}
```

---

## Что мы узнали

| Синтаксис | Описание | Пример в трейдинге |
|-----------|----------|--------------------|
| `#[derive(Copy, Clone)]` | Сделать тип копируемым | Тик с ценой и объёмом |
| `let b = a` (Copy) | Побитовая копия, a жив | Цена входа скопирована |
| `fn f(x: T)` (Copy) | Копия, оригинал жив | Параметры ордера переданы |
| `i32`, `f64`, `bool`, `char` | Встроенные Copy типы | Цена, объём, флаги |
| `(f64, u32, bool)` | Кортеж из Copy = Copy | Структура сигнала |
| `[f64; N]` | Массив фикс. размера = Copy | Пять уровней стакана |
| Почему String не Copy | Указатель на кучу нельзя дублировать | Тикер — не наличные |

---

## Домашнее задание

1. Создай `#[derive(Copy, Clone, Debug)]` структуру `Candle { open: f64, high: f64, low: f64, close: f64 }`.
   - Создай массив `[Candle; 10]` с данными
   - Передай отдельные свечи в функции по значению
   - Убедись, что массив доступен после всех вызовов

2. Создай `enum TrendDirection { Up, Down, Sideways }` с `#[derive(Copy, Clone, Debug, PartialEq)]`.
   - Используй его как параметр в функциях
   - Покажи, что `Copy` позволяет использовать значение после передачи

3. Напиши функцию `analyze_tick(tick: Tick, prev_tick: Tick) -> (bool, f64)`:
   - Где `Tick { price: f64, volume: f64 }` — `Copy` тип
   - Возвращает `(is_price_up, volume_change)`
   - Оба аргумента должны быть доступны после вызова

4. Объясни в коде (через комментарии), почему каждый тип Copy или не Copy:
   - `f64` — Copy потому что...
   - `String` — не Copy потому что...
   - `Vec<f64>` — не Copy потому что...
   - `(f64, bool)` — Copy потому что...
   - `(f64, String)` — не Copy потому что...

---

## Навигация

[← Предыдущий день](../035-clone-copying-portfolio/ru.md) | [Следующий день →](../037-references-looking-at-portfolio/ru.md)
