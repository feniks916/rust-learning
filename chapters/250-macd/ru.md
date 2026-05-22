# День 250: MACD — схождение/расхождение скользящих средних

## Аналогия из трейдинга

MACD (Moving Average Convergence Divergence) — это как два трейдера с разными горизонтами смотрят на один актив. Когда быстрый трейдер (EMA 12) начинает обгонять медленного (EMA 26) — это бычий сигнал.

## Формула MACD

```
MACD Line    = EMA(12) - EMA(26)
Signal Line  = EMA(9) от MACD Line
Histogram    = MACD Line - Signal Line
```

## Реализация EMA

```rust
fn ema(prices: &[f64], period: usize) -> Vec<f64> {
    if prices.len() < period {
        return vec![];
    }

    let k = 2.0 / (period as f64 + 1.0); // сглаживающий коэффициент
    let mut ema_values = Vec::new();

    // Первое значение EMA = SMA первых period цен
    let first_sma: f64 = prices[..period].iter().sum::<f64>() / period as f64;
    ema_values.push(first_sma);

    for &price in &prices[period..] {
        let prev_ema = *ema_values.last().unwrap();
        let new_ema = price * k + prev_ema * (1.0 - k);
        ema_values.push(new_ema);
    }

    ema_values
}
```

## Почему MACD строится именно на EMA

EMA (экспоненциальная скользящая средняя) реагирует на новые цены быстрее, чем простая SMA. Для MACD это важно, потому что индикатор должен улавливать изменение импульса, а не только средний уровень цены.

Идея такая:

1. быстрая EMA отражает недавнее движение
2. медленная EMA отражает более широкий контекст
3. разница между ними показывает ускорение или ослабление тренда

Когда быстрая EMA начинает уходить выше медленной, это сигнализирует, что рынок ускоряется вверх.

## Вычисление MACD

```rust
fn macd(prices: &[f64]) -> (Vec<f64>, Vec<f64>, Vec<f64>) {
    let ema12 = ema(prices, 12);
    let ema26 = ema(prices, 26);

    // Приводим к одной длине
    let len = ema12.len().min(ema26.len());
    let offset12 = ema12.len() - len;
    let offset26 = ema26.len() - len;

    // MACD Line
    let macd_line: Vec<f64> = ema12[offset12..]
        .iter()
        .zip(&ema26[offset26..])
        .map(|(fast, slow)| fast - slow)
        .collect();

    // Signal Line = EMA(9) от MACD Line
    let signal = ema(&macd_line, 9);

    // Histogram
    let offset = macd_line.len() - signal.len();
    let histogram: Vec<f64> = macd_line[offset..]
        .iter()
        .zip(&signal)
        .map(|(m, s)| m - s)
        .collect();

    (macd_line, signal, histogram)
}

fn generate_signal(macd_line: &[f64], signal: &[f64]) -> Vec<&'static str> {
    macd_line.iter().zip(signal.iter()).map(|(m, s)| {
        if m > s { "BUY" } else if m < s { "SELL" } else { "HOLD" }
    }).collect()
}

fn main() {
    // Симулируем 50 свечей
    let prices: Vec<f64> = (0..50)
        .map(|i| 50000.0 + (i as f64 * 100.0) - ((i as f64 * 0.5).sin() * 500.0))
        .collect();

    let (macd_line, signal, histogram) = macd(&prices);

    let n = macd_line.len().min(signal.len()).min(histogram.len());
    println!("MACD анализ (последние 5 значений):");
    for i in (n.saturating_sub(5)..n) {
        println!(
            "  MACD={:+.2}, Signal={:+.2}, Hist={:+.2}",
            macd_line[i + macd_line.len() - n],
            signal[i],
            histogram[i]
        );
    }

    let signals = generate_signal(&macd_line[macd_line.len()-n..], &signal);
    println!("Последний сигнал: {}", signals.last().unwrap_or(&"N/A"));
}
```

## Что означают три линии MACD

Важно не просто запомнить названия, а понимать роль каждой части:

1. `MACD Line` — разница между быстрой и медленной EMA
2. `Signal Line` — сглаживание самой MACD-линии
3. `Histogram` — расстояние между MACD и Signal

То есть:

1. MACD показывает направление и силу импульса
2. Signal помогает отфильтровать шум
3. Histogram показывает, усиливается ли расхождение или наоборот затухает

Если histogram растёт выше нуля, бычий импульс обычно усиливается. Если уходит всё глубже ниже нуля, усиливается медвежий импульс.

## Почему приходится выравнивать длины векторов

EMA с периодом 12 начинает выдавать значения раньше, чем EMA с периодом 26, потому что ей нужно меньше данных для старта.

Поэтому длины векторов отличаются:

1. `ema12` длиннее
2. `ema26` короче
3. для вычисления разницы нужно взять только общую пересекающуюся часть

Именно для этого в коде появляются `offset12`, `offset26` и `len`.

Если не выровнять ряды, ты начнёшь вычитать значения, относящиеся к разным временным точкам.

## Интерпретация сигналов

Самые популярные способы чтения MACD такие:

1. пересечение MACD выше Signal — бычий сигнал
2. пересечение MACD ниже Signal — медвежий сигнал
3. переход MACD через ноль — возможная смена фазы рынка
4. дивергенция между ценой и MACD — предупреждение об ослаблении тренда

Но важно помнить: MACD не предсказывает будущее сам по себе. Он описывает структуру импульса по уже пришедшим данным.

## Практический пример: поиск момента пересечения

```rust
fn crossover_points(macd: &[f64], signal: &[f64]) -> Vec<&'static str> {
    let mut events = Vec::new();

    for i in 1..macd.len().min(signal.len()) {
        let prev_diff = macd[i - 1] - signal[i - 1];
        let curr_diff = macd[i] - signal[i];

        if prev_diff <= 0.0 && curr_diff > 0.0 {
            events.push("bullish crossover");
        } else if prev_diff >= 0.0 && curr_diff < 0.0 {
            events.push("bearish crossover");
        } else {
            events.push("no crossover");
        }
    }

    events
}

fn main() {
    let prices: Vec<f64> = (0..60)
        .map(|i| 50_000.0 + i as f64 * 40.0 - ((i as f64) / 3.0).sin() * 300.0)
        .collect();

    let (macd_line, signal_line, _) = macd(&prices);

    let aligned_macd = &macd_line[macd_line.len() - signal_line.len()..];
    let events = crossover_points(aligned_macd, &signal_line);

    for (index, event) in events.iter().enumerate().rev().take(5).rev() {
        println!("{}: {}", index, event);
    }
}
```

Этот пример полезен тем, что переводит индикатор из «посчитали числа» в «получили события, на которые можно реагировать».

## Частые ошибки

### Ошибка 1: путать MACD и Signal по длине

Перед сравнением двух рядов убедись, что они выровнены по одинаковым индексам времени.

### Ошибка 2: использовать слишком мало данных

Для MACD нужен достаточный хвост истории, иначе первые значения будут малоинформативны.

### Ошибка 3: воспринимать одно пересечение как готовый торговый приказ

Индикатор лучше использовать вместе с контекстом рынка, объёмом и уровнем риска.

## Что мы узнали

| Компонент | Описание |
|-----------|----------|
| MACD Line | EMA(12) - EMA(26) |
| Signal Line | EMA(9) от MACD Line |
| Histogram | MACD - Signal (визуальная сила сигнала) |
| Пересечение снизу вверх | Бычий сигнал (покупка) |
| Пересечение сверху вниз | Медвежий сигнал (продажа) |

## Домашнее задание

1. Реализуй `macd(prices: &[f64]) -> (Vec<f64>, Vec<f64>, Vec<f64>)` возвращающий MACD, Signal, Histogram
2. Напиши функцию `macd_crossover_signal(macd: &[f64], signal: &[f64]) -> Vec<&str>` — buy/sell/hold
3. Определяй дивергенцию: цена растёт, MACD падает — медвежья дивергенция
4. Протестируй на реальных данных загрузив CSV с историческими ценами

## Навигация

[← Предыдущий день](../249-rsi-relative-strength-index/ru.md) | [Следующий день →](../251-bollinger-bands/ru.md)
