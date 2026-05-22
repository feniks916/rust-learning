# День 20: Цикл while — ждём пока цена достигнет цели

## Аналогия из трейдинга

Представь: ты открыл позицию по BTC и выставил тейк-профит на $50 000 и стоп-лосс на $38 000. Что ты делаешь дальше? Ты ждёшь. Держишь позицию **пока** цена не вышла за границы — вверх (тейк) или вниз (стоп). Это и есть цикл `while` — он повторяет блок кода **до тех пор, пока выполняется условие**. Как только условие стало ложным — цикл останавливается. В отличие от `loop` с его "крутись вечно пока не скажу стоп", `while` с самого начала знает своё условие завершения. Это как задание боту: "крути мониторинг пока цена в диапазоне от 38 000 до 50 000".

## Базовый while

Условие проверяется **ДО** каждой итерации. Если с самого начала условие ложно — цикл не выполнится ни разу.

```rust
fn main() {
    let target_price = 50000.0;
    let mut current_price = 45000.0;

    println!("Ждём достижения тейк-профита ${:.0}...", target_price);

    while current_price < target_price {
        // Симулируем рост цены на $1000 за тик
        current_price += 1000.0;
        println!("Цена: ${:.0} | До цели: ${:.0}",
                 current_price, target_price - current_price);
    }

    println!("\nТейк-профит достигнут! Закрываем позицию по ${:.0}", current_price);
}
```

```rust
fn main() {
    // Пример: условие ложно с самого начала — цикл не запустится
    let price = 55000.0;
    let target = 50000.0;

    println!("Начинаем мониторинг...");

    while price < target {
        // Этот код никогда не выполнится
        println!("Ждём...");
    }

    println!("Цена ${:.0} уже выше цели ${:.0} — вход невозможен", price, target);
}
```

## while с несколькими условиями (&& и ||)

Реальный торговый бот редко следит только за одним условием. Используй логические операторы для сложных условий.

```rust
fn main() {
    let mut balance = 10000.0_f64;
    let mut btc_price = 42000.0_f64;
    let min_balance = 1000.0;      // маржин-колл при балансе < $1000
    let target_price = 50000.0;    // цель по цене
    let max_days = 30;
    let mut day = 0;

    println!("Старт: баланс = ${:.0}, цена = ${:.0}", balance, btc_price);

    // Торгуем пока баланс достаточный И цена не достигла цели И лимит дней не исчерпан
    while balance > min_balance && btc_price < target_price && day < max_days {
        day += 1;

        // Симулируем дневное изменение
        let daily_pnl = if day % 3 == 0 { -500.0 } else { 200.0 };
        balance += daily_pnl;
        btc_price += 300.0; // цена растёт на $300/день

        println!("День {:02}: баланс = ${:.0} | BTC = ${:.0} | PnL = {:+.0}",
                 day, balance, btc_price, daily_pnl);
    }

    // Выясняем причину остановки
    if balance <= min_balance {
        println!("\nСтоп: маржин-колл! Баланс: ${:.0}", balance);
    } else if btc_price >= target_price {
        println!("\nСтоп: тейк-профит достигнут! BTC = ${:.0}", btc_price);
    } else {
        println!("\nСтоп: исчерпан лимит {} дней", max_days);
    }
}
```

Пример с OR — выходим если ХОТЯ БЫ одно условие опасности выполнено:

```rust
fn main() {
    let mut rsi = 45.0_f64;
    let mut tick = 0;

    // Продолжаем пока RSI в нейтральной зоне (не перекуплен И не перепродан)
    while rsi > 30.0 && rsi < 70.0 {
        tick += 1;

        // Симулируем изменение RSI
        let delta = if tick % 2 == 0 { 5.0 } else { 3.0 };
        rsi += delta;

        println!("Тик {}: RSI = {:.1}", tick, rsi);
    }

    if rsi >= 70.0 {
        println!("RSI перекуплен ({:.1}) — сигнал SELL", rsi);
    } else {
        println!("RSI перепродан ({:.1}) — сигнал BUY", rsi);
    }
}
```

## while для накопления данных

`while` отлично подходит когда нужно накапливать данные, пока не набралось нужное количество.

```rust
fn main() {
    let required_ticks = 5;  // нужно 5 тиков для расчёта скользящей средней
    let mut prices = Vec::new();
    let mut tick_num = 0;

    // Симулированные поступающие цены
    let incoming = [42000.0, 42200.0, 41800.0, 42500.0, 42100.0, 43000.0];

    println!("Накапливаем данные для расчёта SMA({})...", required_ticks);

    // Накапливаем пока не набрали нужное количество тиков
    while prices.len() < required_ticks {
        if tick_num < incoming.len() {
            let price = incoming[tick_num];
            prices.push(price);
            tick_num += 1;
            println!("Добавлена цена ${:.0} | Накоплено: {}/{}", price, prices.len(), required_ticks);
        } else {
            break; // нет данных
        }
    }

    // Теперь можем посчитать SMA
    let sum: f64 = prices.iter().sum();
    let sma = sum / prices.len() as f64;
    println!("\nSMA({}) = ${:.2}", required_ticks, sma);
}
```

## while let: обработка Option

`while let` — это элегантный способ обрабатывать данные, которые могут закончиться (типа `Option`).

```rust
fn main() {
    // Очередь ожидающих ордеров
    let mut pending_orders: Vec<f64> = vec![41500.0, 42000.0, 42500.0, 43000.0];

    println!("Обрабатываем очередь ордеров:");

    // Извлекаем и обрабатываем ордера пока очередь не пуста
    while let Some(order_price) = pending_orders.pop() {
        let order_type = if order_price < 42000.0 { "BUY" } else { "SELL" };
        println!("  Исполняем {} ордер по ${:.0}", order_type, order_price);
    }

    println!("Все ордера обработаны. Очередь пуста.");
}
```

```rust
fn get_next_signal(signals: &mut Vec<&str>) -> Option<&'static str> {
    if signals.is_empty() {
        None
    } else {
        Some(signals.remove(0))
    }
}

fn main() {
    let mut signals: Vec<&str> = vec!["BUY", "HOLD", "SELL", "BUY"];

    println!("Обрабатываем торговые сигналы:");
    let mut executed = 0;

    while let Some(signal) = signals.first().copied() {
        signals.remove(0);
        executed += 1;
        println!("Сигнал #{}: {} — исполнено", executed, signal);
    }

    println!("Обработано сигналов: {}", executed);
}
```

## Отличие while от loop

Понять разницу очень важно — от этого зависит, какой инструмент выбрать.

```rust
fn main() {
    // --- while: когда условие известно заранее ---
    // Используй когда: условие проверяется ДО входа в цикл
    // "Крутись пока цена < цели"
    let target = 50000.0;
    let mut price = 45000.0;
    while price < target {
        price += 1000.0;
    }
    println!("while: цена достигла ${:.0}", price);

    // --- loop: когда условие выхода сложное или нужно значение ---
    // Используй когда: нужно выполнить тело хотя бы раз
    // или когда выход зависит от вычислений внутри тела
    let prices = [51000.0, 49000.0, 48000.0, 50500.0];
    let mut i = 0;
    let entry = loop {
        if prices[i] <= 50000.0 {
            break prices[i]; // возвращаем значение
        }
        i += 1;
        if i >= prices.len() {
            break 0.0;
        }
    };
    println!("loop: точка входа найдена по ${:.0}", entry);
}
```

**Правило выбора:**
- `while` — когда условие простое и проверяется в начале каждой итерации
- `loop` — когда нужно вернуть значение, или выход происходит в середине/конце тела цикла

## Симуляция торгового дня

Моделируем полный торговый день с открытием и закрытием сессии:

```rust
fn main() {
    let open_hour = 9;     // открытие биржи
    let close_hour = 18;   // закрытие биржи
    let mut current_hour = open_hour;
    let mut total_volume = 0.0_f64;
    let mut trades = 0;

    // Объёмы по часам (симуляция)
    let hourly_volumes = [
        500.0, 800.0, 1200.0, 1500.0, 900.0,
        1100.0, 1300.0, 950.0, 600.0,
    ];

    println!("Торговый день начался ({}:00 - {}:00)", open_hour, close_hour);
    println!("{:-<50}", "");

    while current_hour < close_hour {
        let idx = (current_hour - open_hour) as usize;
        let vol = if idx < hourly_volumes.len() { hourly_volumes[idx] } else { 0.0 };

        total_volume += vol;
        trades += 1;

        // В часы пиковой активности — больше сделок
        let activity = if vol > 1000.0 { "высокая" } else { "нормальная" };
        println!("{:02}:00 | Объём: {:.0} BTC | Активность: {}", current_hour, vol, activity);

        current_hour += 1;
    }

    println!("{:-<50}", "");
    println!("Торговый день завершён");
    println!("Суммарный объём: {:.0} BTC", total_volume);
    println!("Торговых часов: {}", trades);
    println!("Средний объём: {:.1} BTC/ч", total_volume / trades as f64);
}
```

## Практический пример: ожидание точки входа

```rust
/// Ищет оптимальную точку входа в позицию
fn main() {
    println!("=== Ожидание точки входа ===\n");

    // Параметры стратегии
    let entry_target = 42000.0_f64;  // хотим войти по этой цене
    let rsi_max = 40.0_f64;          // RSI должен быть перепродан
    let min_volume = 1000.0_f64;     // минимальный подтверждающий объём
    let max_wait = 10_u32;           // максимум 10 тиков ожидания

    // Симулированные рыночные данные (цена, RSI, объём)
    let market_data: Vec<(f64, f64, f64)> = vec![
        (43200.0, 55.0, 800.0),   // цена высокая, ждём
        (43000.0, 50.0, 900.0),   // всё ещё высоко
        (42800.0, 45.0, 950.0),   // приближаемся
        (42400.0, 38.0, 1100.0),  // RSI хорошо, цена ещё высоко
        (42100.0, 36.0, 1200.0),  // почти у цели!
        (41900.0, 33.0, 1350.0),  // цена ниже цели, RSI OK, объём OK
    ];

    let mut tick = 0_usize;
    let mut entry_found = false;

    // Ждём пока НЕ выполнены все условия входа
    while tick < market_data.len() && tick < max_wait as usize {
        let (price, rsi, volume) = market_data[tick];
        tick += 1;

        println!("Тик {:02}: цена=${:.0} | RSI={:.1} | объём={:.0}",
                 tick, price, rsi, volume);

        // Проверяем условия входа
        let price_ok = price <= entry_target;
        let rsi_ok = rsi <= rsi_max;
        let volume_ok = volume >= min_volume;

        if !price_ok {
            println!("  Ждём: цена ${:.0} > цели ${:.0}", price, entry_target);
            continue;
        }
        if !rsi_ok {
            println!("  Ждём: RSI {:.1} > {:.1}", rsi, rsi_max);
            continue;
        }
        if !volume_ok {
            println!("  Ждём: объём {:.0} < {:.0}", volume, min_volume);
            continue;
        }

        // Все условия выполнены!
        println!("\n  *** ВХОД В ПОЗИЦИЮ! ***");
        println!("  Цена: ${:.0}", price);
        println!("  RSI: {:.1} (перепродан)", rsi);
        println!("  Объём: {:.0} (подтверждён)", volume);
        entry_found = true;
        break;
    }

    if !entry_found {
        println!("\nТочка входа не найдена за {} тиков", tick);
        println!("Продолжаем мониторинг...");
    }
}
```

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `while condition { }` | Условие известно заранее, проверяется до итерации | Ждём пока цена < цели |
| `while a && b { }` | Несколько условий одновременно | Баланс > 0 И цена в диапазоне |
| `while a \|\| b { }` | Любое из условий | Пока нет TP И нет SL |
| `while let Some(x) = ... { }` | Обработка Option, извлечение из Vec | Обработка очереди ордеров |
| Отличие от `loop` | `while` — условие в начале, нет возврата значения | Выбирай `while` для простых условий |
| Отличие от `loop` | `loop` — условие в теле, может возвращать значение | Выбирай `loop` для сложных выходов |

## Домашнее задание

1. Напиши симуляцию "удержания позиции":
   - Цена стартует от $42 000, меняется на случайную величину каждый тик
   - `while` продолжается пока цена между стопом ($40 000) и тейком ($46 000)
   - После выхода выведи: причину (TP/SL), финальную цену, количество тиков

2. Используй `while` с несколькими условиями для риск-менеджмента:
   - Симулируй торговлю: каждый "день" баланс меняется
   - Условие продолжения: баланс > $1000 И количество дней < 30 И просадка < 20%
   - В конце выведи причину остановки

3. Реализуй `while let` для обработки потока сигналов:
   - Создай Vec сигналов: `vec![Some("BUY"), Some("HOLD"), None, Some("SELL")]`
   - Используй `while let Some(s) = queue.pop()` — обрабатывай пока не None
   - Подсчитай количество BUY и SELL сигналов

4. Напиши накопитель данных для индикатора:
   - Нужно накопить 20 цен для расчёта SMA(20)
   - `while prices.len() < 20` — добавляй цены из массива
   - После накопления: посчитай SMA(20) и выведи результат

## Навигация

[← Предыдущий день](../019-loop-endless-price-monitoring/ru.md) | [Следующий день →](../021-for-iterating-daily-trades/ru.md)
