# День 27: Shadowing — обновляем цену тем же именем

## Аналогия из трейдинга

Цена актива постоянно обновляется на экране терминала, но мы по-прежнему называем её "ценой BTC" — имя не меняется, меняется только значение. Shadowing в Rust работает похожим образом: ты можешь объявить новую переменную с тем же именем, и она "перекрывает" старую. При этом — в отличие от `mut` — новая переменная может быть совсем другого типа. Это особенно удобно при парсинге: сначала `price` — строка из API, потом `price` — число, потом `price` — число с учётом комиссии. Одно имя, три формы.

## Что такое shadowing

Shadowing — это не изменение переменной, а создание новой под тем же именем:

```rust
fn main() {
    let price = 50000;            // i32 — целое
    println!("Цена (int): {}", price);

    let price = price as f64;     // shadowing: теперь f64
    println!("Цена (f64): {}", price);

    let price = price * 1.05;     // shadowing: значение с +5%
    println!("Цена +5%: {:.2}", price);

    // price всё ещё f64 — тип не менялся в последнем shadowing
    let price = format!("${:.0}", price); // shadowing: теперь String
    println!("Цена (String): {}", price);

    // Старые "версии" price недоступны — только последняя
}
```

Каждый `let price = ...` создаёт новую переменную. Старая при этом "уходит в тень" — отсюда название "shadowing" (затенение).

## Shadowing vs mut: смена типа

Главное отличие от `mut` — shadowing позволяет менять тип переменной:

```rust
fn main() {
    // SHADOWING: можно менять тип
    let input = "   50000.50   "; // &str — строка
    let input = input.trim();      // &str без пробелов (ещё строка)
    let input: f64 = input.parse().expect("Неверный формат");  // теперь f64
    let input = input + 0.5;       // f64 + delta
    println!("Shadowing: {:.2}", input); // 50001.00

    // MUT: тип изменить нельзя
    let mut price = 49000.0_f64;
    price = 50000.0;  // OK: тот же тип
    // price = "дорого"; // ОШИБКА КОМПИЛЯЦИИ: expected f64, found &str

    // MUT не может делать то что делает shadowing:
    let mut s = "hello";
    // s = s.len(); // ОШИБКА: тип должен остаться &str
    let s = s.len(); // OK через shadowing: s теперь usize
    println!("Длина: {}", s);
}
```

`mut` — это изменение значения одной и той же переменной. Shadowing — это создание новой переменной под старым именем.

## Shadowing в блоках: локальное затенение

Shadowing в блоке `{}` действует только внутри блока:

```rust
fn main() {
    let price = 50000.0;

    // Внутри блока затеняем price
    let discounted = {
        let price = price * 0.98;  // -2% скидка (shadowing только здесь)
        println!("  Внутри блока: {:.2}", price); // 49000.0
        price  // возвращаем из блока
    };

    // Снаружи блока — оригинальное значение сохранено
    println!("Снаружи блока: {:.2}", price);      // 50000.0
    println!("Со скидкой:    {:.2}", discounted);  // 49000.0

    // Ещё пример: обработка котировки в блоке
    let processed_price = {
        let price = price / 1000.0;  // переводим в тысячи
        let price = price.round();   // округляем
        price * 1000.0               // возвращаем обратно в обычные единицы
    };
    println!("Округлённая цена: {:.2}", processed_price); // 50000.0
}
```

Это полезный паттерн: временные вычисления изолированы в блоке, не "засоряя" пространство имён.

## Практический паттерн парсинга: str → f64 → обработанное значение

Shadowing — идеальный инструмент для цепочки преобразований при парсинге:

```rust
fn process_ticker_price(raw: &str) -> f64 {
    // Шаг 1: trim (убираем пробелы, &str → &str)
    let raw = raw.trim();

    // Шаг 2: парсинг строки в число (меняем тип!)
    let raw: f64 = match raw.parse() {
        Ok(v)  => v,
        Err(_) => {
            println!("Предупреждение: '{}' не является числом, используем 0.0", raw);
            0.0
        }
    };

    // Шаг 3: валидация
    let raw = if raw < 0.0 { 0.0 } else { raw };

    // Шаг 4: нормализация (например, цена в копейках → доллары)
    let raw = if raw > 1_000_000.0 { raw / 100.0 } else { raw };

    raw // возвращаем результат
}

fn main() {
    let test_cases = ["  50000.0  ", "  -100  ", "  not_a_number  ", "  5000000  ", "  48500.50  "];

    for input in &test_cases {
        let price = process_ticker_price(input);
        println!("'{}' → ${:.2}", input.trim(), price);
    }
}
```

Каждый шаг описывает одно преобразование, а имя `raw` остаётся неизменным — читается как история обработки.

## Shadowing в функциях: одно имя, разные стадии

Функции с shadowing часто выглядят как "конвейер обработки":

```rust
fn normalize_price(price_str: &str) -> Option<f64> {
    // Стадия 1: trim
    let price_str = price_str.trim();
    if price_str.is_empty() {
        return None;
    }

    // Стадия 2: убираем символы валюты если есть
    let price_str = price_str.trim_start_matches('$').trim_start_matches("USD");

    // Стадия 3: парсинг (меняем тип)
    let price: f64 = price_str.parse().ok()?;

    // Стадия 4: проверка диапазона
    let price = price.abs(); // всегда положительная
    let price = (price * 100.0).round() / 100.0; // 2 знака после запятой

    Some(price)
}

fn main() {
    let inputs = ["$50000.123", " 48500.5 ", "USD51000", "", "bad", "-49000.0"];

    println!("Нормализация цен:");
    for input in &inputs {
        match normalize_price(input) {
            Some(p) => println!("  {:>15} → ${:.2}", format!("'{}'", input), p),
            None    => println!("  {:>15} → (невалидное)", format!("'{}'", input)),
        }
    }
}
```

## Когда shadowing хорош, когда плох

Shadowing — мощный инструмент, но у него есть место применения:

```rust
fn main() {
    // ХОРОШО: паттерн парсинга — логичная цепочка
    let order_qty = "100";
    let order_qty: u32 = order_qty.parse().expect("Число");
    let order_qty = order_qty.min(500); // ограничение
    println!("Количество ордеров: {}", order_qty);

    // ХОРОШО: временная трансформация в блоке
    let price = 50123.456;
    let display = {
        let price = (price / 1000.0).floor(); // целые тысячи
        format!("${:.0}k", price)
    };
    println!("Цена: {} (точно: {:.3})", display, price);

    // ПЛОХО: shadowing для "исправления опечатки"
    let enty_price = 50000.0; // опечатка в имени
    let entry_price = enty_price; // псевдоним вместо переименования
    // Лучше сразу назвать правильно!

    // ПЛОХО: сложные цепочки без смысла
    let x = 5;
    let x = x + 1;
    let x = x * 2;
    // Лучше использовать разные имена: x_base, x_adjusted, x_final
    println!("{}", x);
}
```

Shadowing хорош когда имя действительно остаётся semantически тем же ("это всё ещё цена"), но форма меняется (строка → число → обработанное число).

## Что мы узнали

| Синтаксис | Когда использовать | Пример |
|-----------|-------------------|--------|
| `let x = ...; let x = ...;` | Новое значение/тип под тем же именем | `let price = "50000"; let price: f64 = price.parse()...` |
| Shadowing меняет тип | В отличие от `mut`, тип может стать другим | `let s = "3"; let s = s.len();` (usize) |
| Shadowing в блоке `{}` | Изолированные временные вычисления | `let result = { let x = x * 2; x };` |
| Паттерн парсинга | Цепочка str → trim → parse → validate | Обработка котировок из API |
| `mut` vs shadowing | `mut` — то же имя и тип, shadowing — то же имя, любой тип | Для смены типа только shadowing |
| Когда НЕ применять | Когда логически это разные переменные | Лучше `price_raw` и `price_clean` |

## Домашнее задание

1. Реализуй функцию `parse_ticker_input(s: &str) -> Option<(String, f64)>` через shadowing:
   - `let s = s.trim()`
   - `let parts = s.split(':')` 
   - Проверь что частей ровно 2
   - `let ticker = parts[0].to_uppercase()`
   - `let price: f64 = parts[1].parse().ok()?`
   - Вход: `"btc: 50000"`, выход: `Some(("BTC", 50000.0))`

2. Напиши конвейер нормализации объёма: входная строка `"  1500000 "`, результат — строка `"1.5M"`
   - `let v = v.trim()`
   - `let v: f64 = v.parse()...`
   - `let v = v / 1_000_000.0`
   - `let v = format!("{:.1}M", v)`

3. Покажи разницу между shadowing и `mut` на примере: попробуй сменить тип через `mut` (покажи ошибку в комментарии), затем покажи что через shadowing это работает

4. Создай функцию обработки массива цен: принимает `&[&str]`, возвращает `Vec<f64>`
   - Для каждого элемента делай shadowing: trim → parse → validate (> 0) → round
   - Пропускай невалидные значения через `continue`

## Навигация

[← Предыдущий день](../026-constants-fixed-exchange-fee/ru.md) | [Следующий день →](../028-user-input-position-size/ru.md)
