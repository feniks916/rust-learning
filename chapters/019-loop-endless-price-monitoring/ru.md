# День 19: Цикл loop — бесконечный мониторинг цены

## Аналогия из трейдинга

Торговый бот никогда не спит. Пока рынок открыт (а крипторынок открыт всегда — 24/7/365), бот непрерывно проверяет цены, сигналы, балансы и выставляет ордера. Это не задача "запустил и забыл" — это бесконечная петля: проверил цену, принял решение, подождал следующий тик, снова проверил. Цикл `loop` в Rust — это именно такой бесконечный мониторинговый движок. Он выполняет блок кода снова и снова до тех пор, пока ты явно не скажешь ему "СТОП" через `break`. Это как дежурный трейдер, который смотрит на монитор не отрываясь.

## Базовый loop

Самый простой `loop` — это просто бесконечное повторение. Без `break` он никогда не остановится!

```rust
fn main() {
    let mut tick = 0;
    let mut btc_price = 42000.0;

    loop {
        tick += 1;

        // Симулируем изменение цены (±$100 каждый тик)
        let price_delta = if tick % 3 == 0 { 100.0 } else { -50.0 };
        btc_price += price_delta;

        println!("Тик #{}: BTC = ${:.2}", tick, btc_price);

        // Обязательный выход! Иначе программа зависнет навсегда
        if tick >= 5 {
            println!("Мониторинг завершён после {} тиков", tick);
            break;
        }
    }
}
```

Вывод:
```
Тик #1: BTC = $41950.00
Тик #2: BTC = $41900.00
Тик #3: BTC = $42000.00
Тик #4: BTC = $41950.00
Тик #5: BTC = $41900.00
Мониторинг завершён после 5 тиков
```

**Золотое правило:** каждый `loop` должен иметь возможность выйти через `break`. Иначе это бесконечный цикл — программа зависнет!

## loop с счётчиком и статистикой

На практике в цикле мониторинга мы не просто смотрим на цену — мы собираем статистику.

```rust
fn main() {
    let mut price = 50000.0_f64;
    let mut tick = 0_u32;
    let mut sum_price = 0.0_f64;   // для расчёта средней цены
    let mut max_price = f64::MIN;  // максимальная цена за сессию
    let mut min_price = f64::MAX;  // минимальная цена за сессию

    // Симулированные тики цены BTC
    let ticks = [50100.0, 49800.0, 50500.0, 49600.0, 51000.0,
                 50200.0, 49900.0, 50800.0, 49700.0, 50300.0];

    loop {
        price = ticks[tick as usize];
        tick += 1;

        // Обновляем статистику
        sum_price += price;
        if price > max_price { max_price = price; }
        if price < min_price { min_price = price; }

        let avg_price = sum_price / tick as f64;

        println!("Тик {}: ${:.0} | Ср: ${:.0} | Макс: ${:.0} | Мин: ${:.0}",
                 tick, price, avg_price, max_price, min_price);

        if tick as usize >= ticks.len() {
            println!("\n=== Сессия завершена ===");
            println!("Тиков: {}", tick);
            println!("Средняя цена: ${:.2}", avg_price);
            println!("Диапазон: ${:.0} - ${:.0}", min_price, max_price);
            break;
        }
    }
}
```

## break: когда выходим из цикла

`break` можно вызывать по разным причинам. В торговом боте это обычно: тейк-профит сработал, стоп-лосс сработал, или достигнут лимит итераций.

```rust
fn main() {
    let entry_price = 42000.0;
    let take_profit = 44000.0;  // +4.76%
    let stop_loss = 40000.0;    // -4.76%
    let max_ticks = 100;        // лимит безопасности

    let mut current_price = entry_price;
    let mut tick = 0;
    let mut reason = "неизвестно";

    // Симулируем ценовые тики
    let price_changes = [200.0, -100.0, 300.0, -200.0, 400.0,
                         500.0, -300.0, 600.0, 700.0, 800.0];

    loop {
        // Симулируем изменение цены
        if tick < price_changes.len() {
            current_price += price_changes[tick];
        }
        tick += 1;

        println!("Тик {}: ${:.0} (вход: ${:.0})", tick, current_price, entry_price);

        // Тейк-профит
        if current_price >= take_profit {
            reason = "ТЕЙК-ПРОФИТ";
            break;
        }

        // Стоп-лосс
        if current_price <= stop_loss {
            reason = "СТОП-ЛОСС";
            break;
        }

        // Лимит итераций (защита от бесконечного цикла)
        if tick >= max_ticks {
            reason = "ЛИМИТ ИТЕРАЦИЙ";
            break;
        }
    }

    let pnl = current_price - entry_price;
    println!("\nПозиция закрыта: {}", reason);
    println!("PnL: ${:.2}", pnl);
}
```

## loop возвращает значение

В Rust `loop` может возвращать значение через `break значение`. Это очень удобно для поиска первого подходящего элемента.

```rust
fn find_entry_tick(prices: &[f64], target_price: f64) -> Option<usize> {
    let mut i = 0;
    loop {
        if i >= prices.len() {
            break None; // цена не достигла цели
        }

        if prices[i] <= target_price {
            break Some(i); // нашли! возвращаем индекс
        }

        i += 1;
    }
}

fn main() {
    // Цены BTC по часам
    let hourly_prices = vec![
        52000.0, 51500.0, 51000.0, 50500.0,
        50000.0, 49500.0, 49000.0, 48500.0,
    ];

    let target = 50000.0;

    let entry_tick = find_entry_tick(&hourly_prices, target);

    match entry_tick {
        Some(tick) => {
            println!("Цена достигла ${} на тике #{}", target, tick + 1);
            println!("Цена входа: ${}", hourly_prices[tick]);
        }
        None => {
            println!("Цена не достигла цели ${}", target);
        }
    }
}
```

Ещё пример — ищем тик с максимальным объёмом:

```rust
fn main() {
    let volumes = [1200.0, 1800.0, 950.0, 2500.0, 1100.0, 3100.0, 890.0];
    let mut i = 0;
    let mut max_vol_tick = 0;
    let mut max_vol = 0.0_f64;

    let result = loop {
        if volumes[i] > max_vol {
            max_vol = volumes[i];
            max_vol_tick = i;
        }
        i += 1;
        if i >= volumes.len() {
            break (max_vol_tick, max_vol); // возвращаем кортеж
        }
    };

    println!("Максимальный объём: {:.0} на тике #{}", result.1, result.0 + 1);
}
```

## continue в loop

`continue` пропускает текущую итерацию и сразу переходит к следующей. Полезно для фильтрации невалидных данных — например, нулевых цен или цен вне допустимого диапазона.

```rust
fn main() {
    // Данные с "битыми" тиками (нули — это пропуски в данных)
    let raw_prices = [
        42000.0, 0.0, 42100.0, 42050.0, 0.0,
        42200.0, 41900.0, 0.0, 42300.0, 42150.0,
    ];

    let mut valid_count = 0;
    let mut sum = 0.0;
    let mut i = 0;

    loop {
        if i >= raw_prices.len() { break; }

        let price = raw_prices[i];
        i += 1;

        // Пропускаем невалидные цены
        if price <= 0.0 {
            println!("Тик {}: пропущен (невалидная цена)", i);
            continue; // переходим к следующему тику
        }

        // Обрабатываем только валидные цены
        valid_count += 1;
        sum += price;
        println!("Тик {}: ${:.0} (обрабатываем)", i, price);
    }

    if valid_count > 0 {
        println!("\nВалидных тиков: {}/{}", valid_count, raw_prices.len());
        println!("Средняя цена: ${:.2}", sum / valid_count as f64);
    }
}
```

## Вложенные loop с метками

Когда нужно выйти из вложенного цикла во внешний — используем метки (labels). В трейдинге это полезно при мониторинге нескольких бирж или торговых пар.

```rust
fn main() {
    let exchanges = ["Binance", "Bybit", "OKX"];
    let pairs = ["BTC/USDT", "ETH/USDT", "SOL/USDT"];

    // Симулированные цены: [биржа][пара]
    let prices = [
        [42000.0, 2200.0, 95.0],   // Binance
        [42010.0, 2195.0, 94.8],   // Bybit
        [41990.0, 2205.0, 95.2],   // OKX
    ];

    let target_btc = 42005.0; // ищем BTC по цене <= цели
    let mut found = false;

    'exchange_loop: for ex_idx in 0..exchanges.len() {
        println!("Проверяем биржу: {}", exchanges[ex_idx]);

        let mut pair_idx = 0;
        'pair_loop: loop {
            if pair_idx >= pairs.len() { break 'pair_loop; }

            let price = prices[ex_idx][pair_idx];
            println!("  {}: ${}", pairs[pair_idx], price);

            if pairs[pair_idx] == "BTC/USDT" && price <= target_btc {
                println!("  Нашли выгодную цену BTC на {}!", exchanges[ex_idx]);
                found = true;
                break 'exchange_loop; // выходим из ОБОИХ циклов
            }

            pair_idx += 1;
        }
    }

    if !found {
        println!("Выгодная цена не найдена");
    }
}
```

## Практический пример: простой торговый монитор

```rust
/// Симулирует торговый монитор с полным циклом мониторинга
fn main() {
    println!("=== Торговый монитор BTC/USDT ===");

    // Параметры позиции
    let entry_price = 43000.0_f64;
    let take_profit = 45000.0_f64;   // TP: +4.65%
    let stop_loss = 41500.0_f64;     // SL: -3.49%
    let max_iterations = 20_u32;     // защитный лимит

    // Торговое состояние
    let mut current_price = entry_price;
    let mut tick = 0_u32;
    let mut max_price = entry_price;
    let mut min_price = entry_price;

    // Симуляция рыночных движений
    let movements = [
        150.0, -80.0, 200.0, -120.0, 300.0,
        -200.0, 450.0, -100.0, 600.0, -300.0,
        800.0, -400.0, 1000.0, -500.0, 1200.0,
        -600.0, 1500.0, -700.0, 1800.0, -200.0,
    ];

    println!("Вход: ${:.0} | TP: ${:.0} | SL: ${:.0}\n",
             entry_price, take_profit, stop_loss);

    let exit_reason = loop {
        if tick as usize < movements.len() {
            current_price += movements[tick as usize];
        }
        tick += 1;

        // Обновляем экстремумы
        if current_price > max_price { max_price = current_price; }
        if current_price < min_price { min_price = current_price; }

        let pnl = current_price - entry_price;
        let pnl_pct = (pnl / entry_price) * 100.0;

        println!("Тик {:02}: ${:.0} | PnL: {:+.0}$ ({:+.2}%)",
                 tick, current_price, pnl, pnl_pct);

        // Проверяем условия выхода
        if current_price >= take_profit {
            break "ТЕЙК-ПРОФИТ";
        }
        if current_price <= stop_loss {
            break "СТОП-ЛОСС";
        }
        if tick >= max_iterations {
            break "ЛИМИТ ИТЕРАЦИЙ";
        }
    };

    // Итоги позиции
    let final_pnl = current_price - entry_price;
    let final_pnl_pct = (final_pnl / entry_price) * 100.0;

    println!("\n=== Позиция закрыта: {} ===", exit_reason);
    println!("Цена закрытия: ${:.0}", current_price);
    println!("PnL: {:+.2}$ ({:+.2}%)", final_pnl, final_pnl_pct);
    println!("Максимум за сессию: ${:.0}", max_price);
    println!("Минимум за сессию: ${:.0}", min_price);
    println!("Тиков отработано: {}", tick);
}
```

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `loop { }` | Бесконечный цикл, когда число итераций неизвестно | Мониторинг цены в реальном времени |
| `break` | Выход из цикла по условию | TP/SL сработал |
| `break value` | Выход и возврат значения | `break entry_price` |
| `continue` | Пропустить текущую итерацию | Пропускаем невалидные тики |
| `'label: loop` | Метка для вложенных циклов | Мониторинг нескольких бирж |
| `break 'label` | Выход из именованного цикла | Выход из внешнего цикла |
| Счётчик итераций | Защита от бесконечного цикла | `if tick >= max_ticks { break; }` |

## Домашнее задание

1. Напиши монитор цены BTC, который:
   - Симулирует 10 тиков с произвольными изменениями цены
   - Выходит из цикла при достижении цели (TP) или стопа (SL)
   - После выхода выводит итоговый PnL в USD и в процентах

2. Используй `break value` для поиска первой прибыльной сделки:
   - Есть массив PnL: `[-50.0, -30.0, 80.0, -20.0, 120.0]`
   - Найди индекс первой прибыльной сделки через `loop` + `break Some(i)`
   - Выведи "Первая прибыль на сделке #N: $X"

3. Добавь `continue` для фильтрации данных:
   - Есть массив объёмов с нулями (пропущенные данные): `[1200.0, 0.0, 1500.0, 0.0, 1800.0]`
   - Используй `continue` чтобы пропускать нулевые значения
   - Посчитай среднее только по валидным данным

4. Создай вложенные циклы с метками:
   - Внешний цикл: 3 торговые пары (BTC, ETH, SOL)
   - Внутренний цикл: 5 ценовых тиков для каждой
   - Если цена BTC превысила 45000 — выходи из обоих циклов с меткой

## Навигация

[← Предыдущий день](../018-else-and-else-if-market-scenarios/ru.md) | [Следующий день →](../020-while-waiting-for-target-price/ru.md)
