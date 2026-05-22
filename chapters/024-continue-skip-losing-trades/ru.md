# День 24: continue — пропускаем убыточные сделки в отчёте

## Аналогия из трейдинга

Представь, что ты составляешь отчёт по итогам месяца и хочешь проанализировать только прибыльные сделки — без учёта тех, которые закрылись в минус. Ты просматриваешь журнал: если сделка убыточная — пропускаешь её и смотришь следующую. Именно это делает `continue`: пропускает оставшуюся часть текущей итерации и сразу переходит к следующей. В отличие от `break` — не останавливает цикл, а просто перепрыгивает через одну запись. В трейдинговых алгоритмах это незаменимо при фильтрации данных.

## Базовый continue: фильтрация убыточных сделок

Самый простой пример — перебираем сделки и выводим только прибыльные:

```rust
fn main() {
    let trades = [
        ("BTC",  150.0),
        ("ETH",  -50.0),
        ("SOL",  200.0),
        ("BNB",  -30.0),
        ("ADA",   80.0),
        ("DOT",  -10.0),
        ("AVAX", 120.0),
    ];

    println!("Отчёт по прибыльным сделкам:");
    println!("{:-<40}", "");

    let mut profit_count = 0;
    let mut total_profit = 0.0;

    for (ticker, pnl) in &trades {
        if *pnl < 0.0 {
            continue; // пропускаем убыточные
        }

        profit_count += 1;
        total_profit += pnl;
        println!("{:>6}: +${:.2}", ticker, pnl);
    }

    println!("{:-<40}", "");
    println!("Прибыльных сделок: {}", profit_count);
    println!("Суммарная прибыль: +${:.2}", total_profit);
}
```

После `continue` Rust не выполняет `profit_count += 1` и `println!` для убыточных сделок — сразу прыгает к следующему элементу массива. Код остаётся плоским: нет вложенного `if { ... }` вокруг всей логики обработки.

## Фильтрация по нескольким условиям

Часто нужно отфильтровать сразу несколько типов записей — исключить и убыточные, и сделки с маленьким объёмом, и неправильное направление:

```rust
fn main() {
    let trades = vec![
        // (ticker, pnl, volume, side)
        ("BTC",  250.0, 5000.0, "long"),
        ("ETH",  -80.0, 3000.0, "short"),  // убыток
        ("SOL",  120.0,  200.0, "long"),   // маленький объём
        ("BNB",  -20.0, 1500.0, "long"),   // убыток
        ("ADA",   45.0, 2500.0, "short"),  // не тот тип
        ("DOT",  310.0, 8000.0, "long"),
    ];

    let min_volume   = 1000.0;
    let target_side  = "long";

    println!("Качественные лонг-сделки:");
    let mut skipped = 0;
    let mut processed = 0;

    for &(ticker, pnl, volume, side) in &trades {
        if pnl < 0.0 {
            println!("  Пропуск {} — убыток ${:.2}", ticker, pnl);
            skipped += 1;
            continue; // фильтр 1
        }
        if volume < min_volume {
            println!("  Пропуск {} — объём {:.0} слишком мал", ticker, volume);
            skipped += 1;
            continue; // фильтр 2
        }
        if side != target_side {
            println!("  Пропуск {} — направление '{}'", ticker, side);
            skipped += 1;
            continue; // фильтр 3
        }

        // До сюда доходят только качественные сделки
        processed += 1;
        println!("  ✓ {} [{}]: +${:.2} (объём {:.0})", ticker, side, pnl, volume);
    }

    println!("\nОбработано: {}, пропущено: {}", processed, skipped);
}
```

Каждый `continue` — отдельный "фильтр на входе". Такая структура называется "guard clauses" — охранные предложения. Код не уходит вправо глубокими отступами, а остаётся линейным.

## continue в цикле while: пропуск нерабочих часов

`continue` отлично работает в `while`-цикле при симуляции торгового расписания:

```rust
fn is_trading_hour(hour: u32) -> bool {
    // Торгуем только 9-12 и 14-17, обед 12-14
    (hour >= 9 && hour < 12) || (hour >= 14 && hour < 17)
}

fn main() {
    let mut hour = 8;
    let end_hour = 20;
    let mut signals_checked = 0;
    let mut hours_skipped = 0;

    while hour <= end_hour {
        if !is_trading_hour(hour) {
            println!("Час {:02}: не торговая сессия — пропускаем", hour);
            hours_skipped += 1;
            hour += 1;
            continue; // переходим к следующему часу
        }

        // Этот код выполняется только в торговые часы
        signals_checked += 1;
        println!("Час {:02}: анализируем рынок (сигнал #{})", hour, signals_checked);

        hour += 1;
    }

    println!("\nПроверено сигналов: {}", signals_checked);
    println!("Пропущено нерабочих часов: {}", hours_skipped);
}
```

Важно: в `while`-цикле перед `continue` нужно обновить счётчик (`hour += 1`), иначе цикл зависнет навсегда — `continue` не выполняет тело ниже, но и условие `while` не поменяется само по себе.

## continue с меткой для вложенных циклов

Как и `break`, `continue` поддерживает метки для управления внешним циклом:

```rust
fn main() {
    let portfolio = vec![
        ("BTC", vec![ 200.0, -50.0, 300.0, -100.0,  150.0]),
        ("ETH", vec![ -30.0, -80.0, -20.0,  -60.0,  -10.0]),  // все убытки
        ("SOL", vec![  50.0,  80.0, -10.0,  120.0,   90.0]),
    ];

    println!("Анализ активов (пропускаем убыточные активы целиком):");
    println!();

    'asset: for (ticker, trades) in &portfolio {
        let has_profit = trades.iter().any(|&t| t > 0.0);

        if !has_profit {
            println!("{}: все сделки убыточные — пропускаем актив", ticker);
            continue 'asset; // переходим к следующему активу целиком
        }

        println!("{}:", ticker);
        for (i, &pnl) in trades.iter().enumerate() {
            if pnl < 0.0 {
                continue; // пропускаем убыточную сделку внутри актива
            }
            println!("  Сделка {}: +${:.2}", i + 1, pnl);
        }
        let total: f64 = trades.iter().filter(|&&t| t > 0.0).sum();
        println!("  Итог прибыли: +${:.2}\n", total);
    }
}
```

`continue 'asset` — пропускает весь оставшийся код для текущего актива и переходит к следующему. Без метки внутренний `continue` пропустил бы только текущую сделку.

## Статистика skip/process: считаем что пропустили и почему

Хорошая практика — всегда вести счётчик пропусков по категориям:

```rust
use std::collections::HashMap;

fn main() {
    let trades: Vec<(&str, f64, f64, bool)> = vec![
        // (ticker, pnl, volume, is_valid)
        ("BTC",  500.0, 10000.0, true),
        ("ETH",  -80.0,  3000.0, true),   // убыток
        ("SOL",  120.0,   150.0, true),   // малый объём
        ("BNB",  200.0,  5000.0, false),  // невалидная
        ("BTC",  500.0, 10000.0, true),   // дубликат
        ("ADA",  350.0,  8000.0, true),
        ("DOT", -100.0,  4000.0, true),   // убыток
        ("AVAX", 180.0,  6000.0, true),
    ];

    let min_volume = 1000.0;
    let mut processed = 0;
    let mut skip_counts: HashMap<&str, u32> = HashMap::new();
    let mut seen = std::collections::HashSet::new();

    println!("Фильтрация торговых записей:");

    for &(ticker, pnl, volume, valid) in &trades {
        if !valid {
            *skip_counts.entry("невалидные").or_insert(0) += 1;
            continue;
        }
        if pnl < 0.0 {
            *skip_counts.entry("убыток").or_insert(0) += 1;
            continue;
        }
        if volume < min_volume {
            *skip_counts.entry("малый_объём").or_insert(0) += 1;
            continue;
        }
        if seen.contains(ticker) {
            *skip_counts.entry("дубликат").or_insert(0) += 1;
            continue;
        }

        seen.insert(ticker);
        processed += 1;
        println!("  ✓ {}: +${:.2} (vol {:.0})", ticker, pnl, volume);
    }

    println!("\nИтоги:");
    println!("  Принято: {}", processed);
    let total_skipped: u32 = skip_counts.values().sum();
    for (reason, count) in &skip_counts {
        println!("  Пропущено [{}]: {}", reason, count);
    }
    println!("  Всего пропущено: {}", total_skipped);
}
```

Счётчики в HashMap позволяют потом строить отчёт: "5% данных отфильтровано из-за ошибок, 12% — из-за малого объёма".

## continue для очистки "грязных" котировок

В реальных системах данные бывают некорректными — нули, NaN, выбросы:

```rust
fn main() {
    let raw_prices: Vec<f64> = vec![
        50000.0, 0.0, 51000.0, f64::NAN, 52000.0,
        -100.0, 53000.0, f64::INFINITY, 54000.0,
    ];

    let mut clean = Vec::new();
    let mut bad = 0;

    for (i, &price) in raw_prices.iter().enumerate() {
        if price <= 0.0 || price.is_nan() || price.is_infinite() {
            println!("Тик {}: некорректно ({:?}) — пропуск", i + 1, price);
            bad += 1;
            continue;
        }

        if let Some(&prev) = clean.last() {
            let pct = (price - prev).abs() / prev * 100.0;
            if pct > 20.0 {
                println!("Тик {}: выброс {:.1}% — пропуск", i + 1, pct);
                bad += 1;
                continue;
            }
        }

        clean.push(price);
        println!("Тик {}: ${:.2} ✓", i + 1, price);
    }

    println!("\nЧистых цен: {}, пропущено: {}", clean.len(), bad);
    if !clean.is_empty() {
        let avg = clean.iter().sum::<f64>() / clean.len() as f64;
        println!("Средняя цена: ${:.2}", avg);
    }
}
```

## Практический пример: статистика торговой сессии

Собираем всё вместе — фильтруем сделки за день и строим полный отчёт:

```rust
fn main() {
    let session: Vec<(&str, f64, f64, bool)> = vec![
        // (ticker, entry, exit, valid)
        ("BTC",  50000.0, 52000.0, true),
        ("ETH",   3000.0,  2900.0, true),   // убыток
        ("SOL",    100.0,     0.0, false),  // невалидная
        ("BNB",    280.0,   320.0, true),
        ("ADA",      0.5,     0.0, false),  // невалидная
        ("BTC",  52000.0, 54500.0, true),
        ("DOT",      8.0,     7.2, true),   // убыток
        ("AVAX",    18.0,    22.0, true),
    ];

    let fee = 0.001; // 0.1% комиссия
    let mut wins = 0;
    let mut losses = 0;
    let mut total_pnl = 0.0;
    let mut skipped = 0;

    println!("Торговая сессия — детальный отчёт:");
    println!("{:=<55}", "");

    for &(ticker, entry, exit, valid) in &session {
        if !valid || entry <= 0.0 || exit <= 0.0 {
            println!("  ПРОПУСК {}: невалидные данные", ticker);
            skipped += 1;
            continue;
        }

        let gross = exit - entry;
        let comm  = (entry + exit) * fee;
        let net   = gross - comm;

        if net >= 0.0 {
            wins += 1;
            println!("  WIN  {:>6}: ${:.2} → ${:.2} | P&L +${:.4}", ticker, entry, exit, net);
        } else {
            losses += 1;
            println!("  LOSS {:>6}: ${:.2} → ${:.2} | P&L  ${:.4}", ticker, entry, exit, net);
        }
        total_pnl += net;
    }

    println!("{:=<55}", "");
    let total = wins + losses;
    let wr = if total > 0 { wins as f64 / total as f64 * 100.0 } else { 0.0 };
    println!("Сделок:  {} (пропущено: {})", total, skipped);
    println!("Win/Loss: {} / {}", wins, losses);
    println!("Винрейт: {:.1}%", wr);
    println!("Итог P&L: ${:.4}", total_pnl);
}
```

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `continue` | Пропустить текущую итерацию, перейти к следующей | Убыточная сделка → пропустить в отчёте |
| `continue` в `while` | Пропуск с обязательным обновлением счётчика до `continue` | Нерабочий час → `hour += 1; continue` |
| Guard clauses (цепочка continue) | Плоская структура фильтров вместо вложенных `if` | Убыток → continue; малый объём → continue |
| `continue 'label` | Пропустить итерацию внешнего цикла | Пропустить весь актив при всех убыточных сделках |
| `continue` + HashMap счётчик | Подсчёт пропущенных по категориям | `skip_counts["убыток"] += 1; continue` |
| `continue` для очистки данных | Пропуск NaN, нулей, аномальных выбросов | `price.is_nan() → continue` |

## Домашнее задание

1. Напиши функцию `filter_quality_trades`, принимающую срез сделок `&[(ticker, pnl, volume)]` и возвращающую `Vec` только прошедших фильтры
   - Фильтр 1: `pnl > 0`
   - Фильтр 2: `volume > 500`
   - Фильтр 3: `ticker` не пустой
   - Считай пропуски по каждой причине через `HashMap`

2. Реализуй "чистильщик" котировок: принимает `Vec<f64>`, возвращает очищенный вектор
   - Пропускай через `continue`: нулевые, отрицательные, NaN, бесконечные, выбросы > 15%
   - Выводи подробный лог каждого пропуска

3. Напиши вложенный цикл по портфелю (активы → дни) с метками
   - Если актив в целом убыточен — `continue 'asset`
   - Если день убыточен — `continue 'day`
   - Для прибыльных дней — вывести детали

4. Создай генератор отчёта за неделю: 5 дней × N сделок в день
   - `continue` для пропуска нерабочих дней (выходных)
   - `continue` для пропуска убыточных сделок
   - В конце — сводная статистика: всего сделок, принято, пропущено, суммарный P&L

## Навигация

[← Предыдущий день](../023-break-exit-at-take-profit/ru.md) | [Следующий день →](../025-match-determining-order-type/ru.md)
