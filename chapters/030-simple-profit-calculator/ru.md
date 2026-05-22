# День 30: Простой калькулятор прибыли

## Аналогия из трейдинга

После закрытия сделки трейдер считает: сколько купил, по какой цене, за сколько продал, какова комиссия, и какова итоговая прибыль. Мы напишем именно такую программу.

## Базовый расчёт PnL

```rust
fn calculate_pnl(
    entry: f64,
    exit: f64,
    quantity: f64,
    fee_rate: f64,
) -> (f64, f64, f64) {
    let gross_pnl = (exit - entry) * quantity;
    let entry_fee = entry * quantity * fee_rate;
    let exit_fee  = exit  * quantity * fee_rate;
    let net_pnl   = gross_pnl - entry_fee - exit_fee;
    (gross_pnl, entry_fee + exit_fee, net_pnl)
}

fn main() {
    let (gross, fees, net) = calculate_pnl(50000.0, 55000.0, 0.1, 0.001);
    println!("Валовая прибыль: ${:.2}", gross);
    println!("Комиссии:        ${:.2}", fees);
    println!("Чистая прибыль:  ${:.2}", net);
}
```

## Разбираем формулу по шагам

Чтобы калькулятор не выглядел как «магия в одной формуле», полезно разложить расчёт на части:

1. `gross_pnl` — сколько заработали или потеряли до комиссий
2. `entry_fee` — комиссия на входе
3. `exit_fee` — комиссия на выходе
4. `net_pnl` — что осталось после всех расходов

Для long-сделки базовая логика такая:

```text
валовой результат = (цена выхода - цена входа) * количество
```

Если цена выросла — результат положительный. Если упала — отрицательный.

## Почему количество умножается в конце

Разница между ценой входа и выхода говорит, сколько заработано на одной единице актива. Но трейдер обычно покупает не одну единицу, а некоторый объём.

```rust
fn main() {
    let entry = 50_000.0;
    let exit = 51_500.0;
    let qty = 0.2;

    let pnl_per_unit = exit - entry;
    let pnl_total = pnl_per_unit * qty;

    println!("На единицу: {}", pnl_per_unit);
    println!("Итого: {}", pnl_total);
}
```

Так проще понять формулу, чем сразу видеть всё в одной строке.

## Калькулятор с процентами

```rust
fn trade_report(entry: f64, exit: f64, qty: f64) {
    let pnl = (exit - entry) * qty;
    let pct = (exit - entry) / entry * 100.0;
    let invested = entry * qty;

    println!("━━━━━━ Отчёт по сделке ━━━━━━");
    println!("Вход:       ${:.2} × {}", entry, qty);
    println!("Выход:      ${:.2}", exit);
    println!("Инвестиции: ${:.2}", invested);
    println!("PnL:        ${:.2} ({:+.2}%)", pnl, pct);
    println!("Результат:  {}", if pnl > 0.0 { "ПРИБЫЛЬ ✓" } else { "УБЫТОК ✗" });
}

fn main() {
    trade_report(48000.0, 52000.0, 0.2);
    trade_report(50000.0, 47000.0, 0.1);
}
```

## Несколько сделок

```rust
fn main() {
    let trades: Vec<(f64, f64, f64)> = vec![
        (48000.0, 52000.0, 0.2),
        (51000.0, 49000.0, 0.1),
        (47000.0, 55000.0, 0.05),
    ];

    let mut total_pnl = 0.0;
    for (i, (entry, exit, qty)) in trades.iter().enumerate() {
        let pnl = (exit - entry) * qty;
        total_pnl += pnl;
        println!("Сделка {}: ${:+.2}", i + 1, pnl);
    }

    let wins = trades.iter().filter(|(e, x, _)| x > e).count();
    println!("Итого PnL: ${:+.2}", total_pnl);
    println!("Win rate: {}/{}", wins, trades.len());
}
```

## Long и Short — важное различие

В текущем примере формула подходит для long-позиции. Для short логика меняется на противоположную: прибыль появляется, когда цена падает.

```rust
fn pnl(entry: f64, exit: f64, qty: f64, is_long: bool) -> f64 {
    if is_long {
        (exit - entry) * qty
    } else {
        (entry - exit) * qty
    }
}

fn main() {
    println!("Long:  {}", pnl(50_000.0, 51_000.0, 0.1, true));
    println!("Short: {}", pnl(50_000.0, 49_000.0, 0.1, false));
}
```

Эту разницу важно понять сразу, иначе калькулятор будет давать правильные ответы только для половины реальных ситуаций.

## Практический пример: полный отчёт по сделке

```rust
fn trade_summary(entry: f64, exit: f64, qty: f64, fee_rate: f64) {
    let invested = entry * qty;
    let gross = (exit - entry) * qty;
    let fees = (entry * qty + exit * qty) * fee_rate;
    let net = gross - fees;
    let roi = net / invested * 100.0;

    println!("--- Trade Summary ---");
    println!("Вход:           ${:.2}", entry);
    println!("Выход:          ${:.2}", exit);
    println!("Количество:     {:.4}", qty);
    println!("Инвестировано:  ${:.2}", invested);
    println!("Валовый PnL:    ${:+.2}", gross);
    println!("Комиссии:       ${:.2}", fees);
    println!("Чистый PnL:     ${:+.2}", net);
    println!("ROI:            {:+.2}%", roi);
}

fn main() {
    trade_summary(48_000.0, 50_500.0, 0.3, 0.001);
}
```

Такой формат уже ближе к настоящему торговому отчёту, а не к одной абстрактной формуле.

## Частые ошибки

### Ошибка 1: забывать комиссии

Валовая прибыль и чистая прибыль — разные вещи. Новички часто считают только разницу цен.

### Ошибка 2: путать проценты и абсолютные значения

PnL в долларах и PnL в процентах — это не одно и то же. Оба показателя полезны, но отвечают на разные вопросы.

### Ошибка 3: использовать long-формулу для short-сделки

Это одна из самых частых логических ошибок в первых калькуляторах.

## Что мы узнали

| Концепция | Описание |
|-----------|----------|
| `(exit - entry) * qty` | Базовая формула PnL |
| Кортеж в возврате функции | Возвращаем несколько значений |
| `.filter()` | Фильтрация итератора |
| `{:+.2}` | Форматирование со знаком |

## Домашнее задание

1. Добавь в `calculate_pnl` параметр `is_long: bool` (Long: exit-entry, Short: entry-exit)
2. Напиши функцию `best_trade(trades: &[(f64, f64, f64)]) -> usize` возвращающую индекс лучшей сделки
3. Рассчитай Sharpe ratio: средний PnL / стандартное отклонение PnL
4. Выведи отчёт по всем сделкам с суммарным PnL и win rate

## Навигация

[← Предыдущий день](../029-string-parsing-input-to-number/ru.md) | [Следующий день →](../031-project-position-size-calculator/ru.md)
