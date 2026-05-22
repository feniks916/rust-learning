# День 21: Цикл for — проходим по всем сделкам дня

## Аналогия из трейдинга

В конце торгового дня аналитик открывает журнал сделок и проходится по каждой записи: вычисляет суммарный PnL, находит лучшую и худшую сделку, считает winrate, проверяет соблюдение риск-менеджмента. Он делает это МЕТОДИЧНО — по порядку, не пропуская ни одной записи и не выходя за границы журнала. Цикл `for` в Rust — это именно такой аналитик: он проходит по каждому элементу коллекции строго по порядку, безопасно (нельзя выйти за границы массива) и элегантно. `for` — самый частый цикл в реальном коде, потому что большинство задач — это именно "пройдись по всем и сделай что-то с каждым".

## for по массиву: считаем PnL

Самый базовый случай — массив фиксированного размера.

```rust
fn main() {
    // PnL по каждой сделке за день (в USD)
    let trades_pnl = [150.0, -80.0, 220.0, -45.0, 310.0, -120.0, 180.0];

    let mut total_pnl = 0.0_f64;
    let mut wins = 0;
    let mut losses = 0;

    for pnl in trades_pnl {
        total_pnl += pnl;

        if pnl > 0.0 {
            wins += 1;
            println!("Прибыль: +${:.2}", pnl);
        } else {
            losses += 1;
            println!("Убыток: ${:.2}", pnl);
        }
    }

    let winrate = (wins as f64 / trades_pnl.len() as f64) * 100.0;

    println!("\n=== Итоги дня ===");
    println!("Суммарный PnL: ${:.2}", total_pnl);
    println!("Прибыльных: {} | Убыточных: {}", wins, losses);
    println!("Winrate: {:.1}%", winrate);
}
```

Ещё пример — найти максимальную и минимальную сделку:

```rust
fn main() {
    let trades = [320.0_f64, -150.0, 480.0, -90.0, 210.0, -380.0, 150.0];

    let mut best_trade = f64::MIN;
    let mut worst_trade = f64::MAX;

    for &trade in &trades {
        if trade > best_trade { best_trade = trade; }
        if trade < worst_trade { worst_trade = trade; }
    }

    println!("Лучшая сделка дня: +${:.2}", best_trade);
    println!("Худшая сделка дня: ${:.2}", worst_trade);
}
```

## for по Vec с &

`Vec` — динамический массив. При итерации с `&` мы берём данные по ссылке, и `Vec` остаётся доступным после цикла.

```rust
fn main() {
    let tickers = vec!["BTCUSDT", "ETHUSDT", "SOLUSDT", "BNBUSDT", "ADAUSDT"];

    println!("Сканируем рынок:");

    // & — не перемещаем, а берём ссылку. tickers доступен и после цикла
    for ticker in &tickers {
        // В реальном боте здесь был бы запрос к API биржи
        println!("  Проверяем {}", ticker);
    }

    // tickers всё ещё живёт!
    println!("\nВсего пар в портфеле: {}", tickers.len());
    println!("Первая пара: {}", tickers[0]);
}
```

Пример с Vec сделок — структура данных:

```rust
fn main() {
    // Пары (тикер, PnL)
    let daily_trades = vec![
        ("BTCUSDT", 380.0_f64),
        ("ETHUSDT", -120.0),
        ("SOLUSDT", 95.0),
        ("BNBUSDT", -45.0),
        ("ADAUSDT", 210.0),
    ];

    let mut portfolio_pnl = 0.0;

    for (ticker, pnl) in &daily_trades {
        portfolio_pnl += pnl;
        let status = if *pnl >= 0.0 { "+" } else { "" };
        println!("{}: {}${:.2}", ticker, status, pnl);
    }

    println!("\nПортфельный PnL: ${:.2}", portfolio_pnl);
    println!("Всего позиций: {}", daily_trades.len()); // Vec доступен
}
```

## enumerate: индекс + значение

`enumerate()` добавляет счётчик к каждому элементу. Очень удобно для нумерации записей в журнале.

```rust
fn main() {
    let trade_log = vec![
        ("BUY",  42000.0_f64, 0.5_f64),  // (действие, цена, объём)
        ("SELL", 44500.0,     0.5),
        ("BUY",  41800.0,     0.3),
        ("BUY",  42500.0,     0.2),
        ("SELL", 45000.0,     0.5),
    ];

    println!("=== Торговый журнал ===");
    println!("{:<5} {:<6} {:<10} {:<8} {:<10}", "№", "Тип", "Цена", "Объём", "Стоимость");
    println!("{:-<45}", "");

    for (i, (action, price, volume)) in trade_log.iter().enumerate() {
        let value = price * volume;
        println!("{:<5} {:<6} ${:<9.0} {:<8.2} ${:<9.2}",
                 i + 1, action, price, volume, value);
    }

    println!("Итого сделок: {}", trade_log.len());
}
```

```rust
fn main() {
    // Ищем нужную сделку по индексу
    let prices = vec![42000.0_f64, 41500.0, 43000.0, 44000.0, 45500.0];

    println!("Свечи:");
    for (candle_num, price) in prices.iter().enumerate() {
        let arrow = if candle_num > 0 {
            if price > &prices[candle_num - 1] { "↑" } else { "↓" }
        } else {
            "→"
        };
        println!("Свеча {:02}: ${:.0} {}", candle_num + 1, price, arrow);
    }
}
```

## for по диапазону: торговые часы

Диапазоны `0..N` (исключает N) и `0..=N` (включает N) очень удобны для работы с временными интервалами.

```rust
fn main() {
    println!("=== Торговая сессия NYSE ===");

    // Часы работы: с 9:30 до 16:00 (9..=15 для часов)
    for hour in 9..=15 {
        let activity = match hour {
            9 | 10 => "высокая (открытие)",
            11 | 12 | 13 => "умеренная",
            14 => "нарастающая",
            15 => "высокая (закрытие)",
            _ => "нет активности",
        };
        println!("{:02}:00 — {}", hour, activity);
    }

    println!("\n=== Дни недели ===");
    let days = ["Пн", "Вт", "Ср", "Чт", "Пт"];
    for (day_num, day_name) in (1..=5).zip(days.iter()) {
        println!("День {}: {} — торговый день", day_num, day_name);
    }
}
```

```rust
fn main() {
    // Симулируем недельную торговлю — 5 дней по 8 часов
    let mut weekly_pnl = 0.0_f64;

    for day in 1..=5 {
        let mut daily_pnl = 0.0_f64;

        for hour in 9..17 { // 9:00 до 17:00
            // Симуляция: нечётные часы прибыльны
            let hourly = if hour % 2 != 0 { 50.0 } else { -20.0 };
            daily_pnl += hourly;
        }

        weekly_pnl += daily_pnl;
        println!("День {}: PnL = ${:.0}", day, daily_pnl);
    }

    println!("\nНедельный PnL: ${:.0}", weekly_pnl);
}
```

## for по HashMap: портфель

`HashMap` хранит пары ключ-значение. Итерируемся по всем активам портфеля:

```rust
use std::collections::HashMap;

fn main() {
    let mut portfolio: HashMap<&str, f64> = HashMap::new();

    // Формируем портфель (тикер -> объём в USD)
    portfolio.insert("BTC",  45000.0);
    portfolio.insert("ETH",  12000.0);
    portfolio.insert("SOL",   5000.0);
    portfolio.insert("BNB",   3000.0);
    portfolio.insert("USDT", 10000.0);

    let total: f64 = portfolio.values().sum();

    println!("=== Состав портфеля ===");
    println!("{:<8} {:>10} {:>8}", "Актив", "USD", "Доля");
    println!("{:-<28}", "");

    // Итерируемся по парам ключ-значение
    let mut assets: Vec<(&&str, &f64)> = portfolio.iter().collect();
    assets.sort_by(|a, b| b.1.partial_cmp(a.1).unwrap()); // сортировка по убыванию

    for (ticker, amount) in &assets {
        let share = (*amount / total) * 100.0;
        println!("{:<8} {:>9.0}$ {:>7.1}%", ticker, amount, share);
    }

    println!("{:-<28}", "");
    println!("{:<8} {:>9.0}$  100.0%", "ИТОГО", total);
}
```

## Методы итераторов: filter, map, sum

Rust позволяет делать мощные цепочки операций над коллекциями. Это превью — в будущих главах разберём подробнее.

```rust
fn main() {
    let trades_pnl = vec![
        150.0_f64, -80.0, 320.0, -150.0, 95.0,
        -45.0, 210.0, -380.0, 180.0, -30.0
    ];

    // filter: оставляем только прибыльные сделки
    let winning_trades: Vec<f64> = trades_pnl.iter()
        .copied()
        .filter(|&pnl| pnl > 0.0)
        .collect();

    // map: конвертируем PnL в проценты (предположим вход $10000)
    let pnl_pct: Vec<f64> = trades_pnl.iter()
        .map(|&pnl| (pnl / 10000.0) * 100.0)
        .collect();

    // sum: суммируем все прибыльные
    let total_wins: f64 = winning_trades.iter().sum();
    let total_losses: f64 = trades_pnl.iter()
        .filter(|&&pnl| pnl < 0.0)
        .sum();

    println!("Прибыльных сделок: {}", winning_trades.len());
    println!("Убыточных сделок: {}", trades_pnl.len() - winning_trades.len());
    println!("Суммарная прибыль: ${:.2}", total_wins);
    println!("Суммарный убыток: ${:.2}", total_losses);

    let profit_factor = total_wins / total_losses.abs();
    println!("Profit Factor: {:.2}", profit_factor);

    println!("\nPnL по сделкам (%):");
    for (i, pct) in pnl_pct.iter().enumerate() {
        println!("  Сделка {}: {:+.2}%", i + 1, pct);
    }
}
```

## Практический пример: анализ торгового журнала

```rust
use std::collections::HashMap;

fn main() {
    println!("=== Анализ торгового журнала ===\n");

    // Полный журнал сделок за неделю
    // (тикер, тип, цена входа, цена выхода, объём в USD)
    let trade_journal = vec![
        ("BTCUSDT", "LONG",  42000.0_f64, 44500.0, 5000.0_f64),
        ("ETHUSDT", "LONG",   2200.0,      2150.0, 2000.0),
        ("SOLUSDT", "SHORT",    95.0,        88.0, 1000.0),
        ("BTCUSDT", "LONG",  43500.0,     46000.0, 4000.0),
        ("BNBUSDT", "LONG",    280.0,       270.0, 1500.0),
        ("ETHUSDT", "SHORT",  2300.0,      2180.0, 3000.0),
        ("BTCUSDT", "SHORT", 45000.0,     43000.0, 6000.0),
        ("SOLUSDT", "LONG",     90.0,        98.0, 2000.0),
    ];

    let mut total_pnl = 0.0_f64;
    let mut wins = 0_u32;
    let mut losses = 0_u32;
    let mut pnl_by_ticker: HashMap<&str, f64> = HashMap::new();

    println!("{:<10} {:<6} {:<10} {:<10} {:<10} {:<12}",
             "Тикер", "Тип", "Вход", "Выход", "Объём", "PnL");
    println!("{:-<60}", "");

    for (i, &(ticker, trade_type, entry, exit, volume)) in trade_journal.iter().enumerate() {
        // Расчёт PnL в зависимости от типа сделки
        let pnl = if trade_type == "LONG" {
            (exit - entry) / entry * volume
        } else {
            (entry - exit) / entry * volume
        };

        total_pnl += pnl;
        if pnl >= 0.0 { wins += 1; } else { losses += 1; }

        // Накапливаем PnL по тикеру
        *pnl_by_ticker.entry(ticker).or_insert(0.0) += pnl;

        let marker = if pnl >= 0.0 { "+" } else { "" };
        println!("{:<10} {:<6} ${:<9.0} ${:<9.0} ${:<9.0} {}${:<9.2}",
                 ticker, trade_type, entry, exit, volume, marker, pnl);

        let _ = i; // подавляем предупреждение о неиспользуемой переменной
    }

    println!("{:-<60}", "");
    println!("\n=== Сводка ===");
    println!("Всего сделок: {}", trade_journal.len());
    println!("Прибыльных: {} | Убыточных: {}", wins, losses);
    println!("Winrate: {:.1}%", (wins as f64 / trade_journal.len() as f64) * 100.0);
    println!("Суммарный PnL: ${:.2}", total_pnl);

    println!("\n=== PnL по инструментам ===");
    let mut tickers_sorted: Vec<(&&str, &f64)> = pnl_by_ticker.iter().collect();
    tickers_sorted.sort_by(|a, b| b.1.partial_cmp(a.1).unwrap());

    for (ticker, pnl) in &tickers_sorted {
        let marker = if **pnl >= 0.0 { "+" } else { "" };
        println!("{}: {}${:.2}", ticker, marker, pnl);
    }
}
```

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `for x in array` | Итерация по массиву (владение) | `for pnl in trades { }` |
| `for x in &vec` | Итерация по Vec без перемещения | `for ticker in &tickers { }` |
| `for (i, x) in col.iter().enumerate()` | Нужен индекс + значение | Нумерация сделок в журнале |
| `for i in 0..n` | Диапазон, N не включается | Торговые часы 9..17 |
| `for i in 0..=n` | Диапазон, N включается | Дни 1..=5 |
| `for (k, v) in &hashmap` | Итерация по всем парам | Портфель: тикер -> сумма |
| `.filter()` | Оставить только подходящие | Только прибыльные сделки |
| `.map()` | Преобразовать каждый элемент | PnL в USD → PnL в % |
| `.sum()` | Суммировать все элементы | Суммарный PnL |

## Домашнее задание

1. Создай вектор из 10 дневных PnL значений и вычисли:
   - Суммарный PnL за период
   - Максимальный дневной доход
   - Максимальный дневной убыток
   - Winrate (% прибыльных дней)
   - Используй `for` с `enumerate` для нумерации дней

2. Напиши анализ портфеля через `HashMap`:
   - Создай HashMap с 5 тикерами и их весами в USD
   - Используй `for (ticker, amount) in &portfolio`
   - Выведи каждый актив с долей в % от общей суммы

3. Используй цепочку итераторов для анализа сделок:
   - Есть Vec сделок с PnL значениями
   - Найди средний PnL прибыльных сделок через `.filter().sum()` и деление
   - Найди среднее количество дней между прибыльными сделками

4. Напиши "матрицу производительности" по дням и часам:
   - Внешний `for day in 1..=5` (5 торговых дней)
   - Внутренний `for hour in 9..17` (8 торговых часов)
   - Для каждого часа симулируй случайный PnL (можно использовать формулу)
   - Выведи итоговую таблицу PnL по дням

## Навигация

[← Предыдущий день](../020-while-waiting-for-target-price/ru.md) | [Следующий день →](../022-range-prices/ru.md)
