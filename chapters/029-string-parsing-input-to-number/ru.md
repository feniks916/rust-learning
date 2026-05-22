# День 29: Парсинг строк — преобразуем ввод в число

## Аналогия из трейдинга

Биржевые API возвращают цены в виде строк: `"49500.50"`. Чтобы с ними считать, нужно преобразовать строку в число. Это называется парсинг — и Rust делает это безопасно через `parse()`.

## parse(): преобразование строки в число

```rust
fn main() {
    let price_str = "49500.50";
    let price: f64 = price_str.parse().expect("Неверный формат цены");
    println!("Цена: ${:.2}", price);

    let qty_str = "10";
    let qty: u32 = qty_str.parse().expect("Неверное количество");
    println!("Количество: {}", qty);

    let total = price * qty as f64;
    println!("Итого: ${:.2}", total);
}
```

## Что такое `parse()` на самом деле

Метод `parse()` пытается превратить строку в другой тип. Но сам по себе он не знает, во что именно нужно парсить строку, поэтому тип должен быть понятен из контекста.

Есть два обычных способа дать этот контекст:

1. указать тип слева: `let price: f64 = s.parse().unwrap();`
2. указать тип прямо в `parse`: `s.parse::<f64>().unwrap()`

```rust
fn main() {
    let a: i32 = "42".parse().unwrap();
    let b = "3.14".parse::<f64>().unwrap();

    println!("{} и {}", a, b);
}
```

`parse()` почти всегда возвращает `Result`, потому что преобразование может не получиться.

## Безопасный парсинг через Result

```rust
fn parse_price(s: &str) -> Result<f64, String> {
    s.trim()
     .parse::<f64>()
     .map_err(|e| format!("Ошибка парсинга '{}': {}", s, e))
}

fn main() {
    let inputs = vec!["50000.0", "invalid", "48500.5", ""];

    for input in inputs {
        match parse_price(input) {
            Ok(price)  => println!("Цена: ${:.2}", price),
            Err(e)     => println!("Ошибка: {}", e),
        }
    }
}
```

## Парсинг с проверкой диапазона

```rust
fn parse_quantity(s: &str) -> Result<f64, String> {
    let qty: f64 = s.trim()
        .parse()
        .map_err(|_| "Не число".to_string())?;

    if qty <= 0.0 {
        return Err("Количество должно быть положительным".to_string());
    }
    if qty > 1000.0 {
        return Err("Слишком большой объём".to_string());
    }
    Ok(qty)
}

fn main() {
    for s in &["5.5", "-1", "2000", "abc"] {
        println!("{:?} -> {:?}", s, parse_quantity(s));
    }
}
```

## `expect`, `match` и `Result` — когда что использовать

Есть три типовых уровня строгости:

1. `expect` — быстро и удобно для учебного кода, но программа падает при ошибке
2. `match` — можно показать пользователю понятное сообщение
3. возврат `Result` из функции — лучший вариант для переиспользуемой логики

```rust
fn main() {
    let good = "50000";
    let bad = "abc";

    let price: f64 = good.parse().expect("Число обязательно");
    println!("Цена: {}", price);

    match bad.parse::<f64>() {
        Ok(value) => println!("OK: {}", value),
        Err(error) => println!("Ошибка: {}", error),
    }
}
```

## Цепочка: trim -> parse -> validate

В реальной программе парсинг редко ограничивается одной строкой `parse()`. Обычно есть три шага:

1. очистить ввод
2. преобразовать строку в число
3. проверить бизнес-правила

```rust
fn parse_risk_percent(input: &str) -> Result<f64, String> {
    let input = input.trim();
    let value: f64 = input.parse().map_err(|_| "Введите число".to_string())?;

    if !(0.0..=100.0).contains(&value) {
        return Err("Риск должен быть в диапазоне от 0 до 100".to_string());
    }

    Ok(value)
}
```

Такой стиль читается заметно лучше, чем одна длинная строка без промежуточных смысловых шагов.

## Практический пример: парсим строку ордера

```rust
fn parse_order_line(line: &str) -> Result<(String, f64, f64), String> {
    let line = line.trim();
    let parts: Vec<&str> = line.split(',').collect();

    if parts.len() != 3 {
        return Err("Ожидается формат: SYMBOL,PRICE,QTY".to_string());
    }

    let symbol = parts[0].trim().to_uppercase();
    let price: f64 = parts[1].trim().parse().map_err(|_| "Неверная цена".to_string())?;
    let qty: f64 = parts[2].trim().parse().map_err(|_| "Неверное количество".to_string())?;

    if price <= 0.0 {
        return Err("Цена должна быть положительной".to_string());
    }

    if qty <= 0.0 {
        return Err("Количество должно быть положительным".to_string());
    }

    Ok((symbol, price, qty))
}

fn main() {
    for sample in ["BTC,50000,0.2", "eth, 3200, 1.5", "bad,data"] {
        println!("{:?}", parse_order_line(sample));
    }
}
```

Это уже похоже на реальную задачу: строка приходит извне, а мы по шагам превращаем её в безопасные типизированные данные.

## Частые ошибки

### Ошибка 1: забыть, что `parse()` может не сработать

Никогда не думай о парсинге как о гарантированном действии, если ввод пришёл извне.

### Ошибка 2: валидировать строку до `trim()`

Сначала очистка, потом преобразование, потом проверка бизнес-ограничений.

### Ошибка 3: пытаться вернуть `f64` без `Result`, когда ошибка реальна

Если парсинг может не удаться, лучше явно описать это в типе результата.

## Что мы узнали

| Синтаксис | Описание |
|-----------|----------|
| `s.parse::<f64>()` | Парсинг строки в конкретный тип |
| `s.parse().expect("msg")` | Парсинг с паникой при ошибке |
| `s.parse().map_err(\|e\| ...)` | Преобразование ошибки |
| `.trim()` перед parse | Убираем пробелы и переносы строки |

## Домашнее задание

1. Напиши функцию `parse_price(s: &str) -> Result<f64, String>` с проверкой что цена > 0
2. Парси вектор строк `["100.5", "bad", "200.0"]` и выводи Ok/Err для каждой
3. Напиши `parse_percent(s: &str) -> Result<f64, String>` — значение от 0.0 до 100.0
4. Объедини парсинг цены и количества в функцию вычисления общей стоимости позиции

## Навигация

[← Предыдущий день](../028-user-input-position-size/ru.md) | [Следующий день →](../030-simple-profit-calculator/ru.md)
