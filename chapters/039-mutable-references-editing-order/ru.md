# День 39: Изменяемые ссылки — редактируем чужой ордер

## Аналогия из трейдинга

Иногда брокеру нужно не только посмотреть ордер, но и изменить его — например, скорректировать цену. Изменяемая ссылка `&mut T` даёт такой доступ: можно изменять данные, не становясь владельцем.

## Чем `&mut` отличается от `mut`

Эти вещи часто путают, хотя они отвечают за разное:

1. `let mut price = 50000.0;` означает: владелец может менять своё значение
2. `price: &mut f64` означает: кто-то получил временное право менять чужое значение

То есть `mut` относится к переменной-владельцу, а `&mut` относится к ссылке-заимствованию.

```rust
fn add_fee(value: &mut f64) {
    *value += 25.0;
}

fn main() {
    let mut price = 50_000.0;

    add_fee(&mut price);
    println!("Новая цена: {}", price);
}
```

Если убрать `mut` у переменной `price`, создать `&mut price` уже не получится.

## &mut T: изменяемая ссылка

```rust
fn apply_slippage(price: &mut f64, slippage: f64) {
    *price *= 1.0 + slippage;
}

fn main() {
    let mut entry_price = 50000.0;
    println!("До: ${}", entry_price);

    apply_slippage(&mut entry_price, 0.001); // изменяем через ссылку
    println!("После slippage: ${:.2}", entry_price);
}
```

## Изменение Vec через &mut

```rust
fn add_fee_to_all(prices: &mut Vec<f64>, fee: f64) {
    for price in prices.iter_mut() {
        *price += fee;
    }
}

fn main() {
    let mut prices = vec![50000.0, 51000.0, 52000.0];
    println!("До: {:?}", prices);

    add_fee_to_all(&mut prices, 50.0);
    println!("После: {:?}", prices);
}
```

## Только одна &mut в момент времени

```rust
fn main() {
    let mut order = String::from("BUY BTC");

    let r1 = &mut order;
    // let r2 = &mut order; // ОШИБКА: вторая &mut недопустима

    r1.push_str(" 0.1");
    println!("{}", r1);
}
```

## Изменяемая ссылка в цикле

```rust
fn normalize_prices(prices: &mut [f64]) {
    let max = prices.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    for p in prices.iter_mut() {
        *p /= max;
    }
}

fn main() {
    let mut data = vec![48000.0, 50000.0, 52000.0];
    normalize_prices(&mut data);
    println!("Нормализованные: {:?}", data);
}
```

## Что именно делает `*price` внутри функции

В этой строке:

```rust
*price *= 1.0 + slippage;
```

происходит следующее:

1. `price` — это не число, а ссылка `&mut f64`
2. `*price` — это доступ к самому числу
3. через этот доступ мы меняем исходное значение у владельца

Именно поэтому после вызова функции обновляется переменная из `main`, а не только локальная копия.

## Когда использовать `&mut Vec<T>`, а когда `&mut [T]`

Если функция должна изменить элементы, но не менять размер коллекции, чаще удобнее `&mut [T]`.

```rust
fn apply_discount(prices: &mut [f64], percent: f64) {
    for price in prices.iter_mut() {
        *price *= 1.0 - percent;
    }
}
```

Если функция должна добавлять или удалять элементы, нужен `&mut Vec<T>`.

```rust
fn append_price(prices: &mut Vec<f64>, new_price: f64) {
    prices.push(new_price);
}
```

Это полезное различие:

1. `&mut [T]` — меняем содержимое
2. `&mut Vec<T>` — можем менять и содержимое, и длину

## Почему изменяемая ссылка только одна

`&mut` означает эксклюзивный доступ. Пока она жива, никто другой не должен ни читать эти данные, ни менять их.

Так Rust гарантирует, что в каждый момент времени есть ровно один редактор значения.

```rust
fn main() {
    let mut order = String::from("BUY BTC 0.5");

    {
        let editor = &mut order;
        editor.push_str(" @ 50000");
        println!("Внутри редактора: {}", editor);
    }

    println!("После редактирования: {}", order);
}
```

## Практический пример: обновление торгового плана

```rust
fn apply_slippage(prices: &mut [f64], percent: f64) {
    for price in prices.iter_mut() {
        *price *= 1.0 + percent;
    }
}

fn cap_risk(position_sizes: &mut [f64], max_size: f64) {
    for size in position_sizes.iter_mut() {
        if *size > max_size {
            *size = max_size;
        }
    }
}

fn main() {
    let mut entries = vec![50_000.0, 51_500.0, 49_800.0];
    let mut sizes = vec![1.2, 0.7, 2.4];

    apply_slippage(&mut entries, 0.001);
    cap_risk(&mut sizes, 1.5);

    println!("Скорректированные входы: {:?}", entries);
    println!("Ограниченные размеры: {:?}", sizes);
}
```

## Частые ошибки

### Ошибка 1: забыли `mut` у самой переменной

Если значение не объявлено как изменяемое, `&mut` создать нельзя.

### Ошибка 2: пытаемся использовать переменную, пока её держит `&mut`

Пока живёт изменяемая ссылка, владелец временно не должен обращаться к тем же данным.

### Ошибка 3: выбираем `&mut Vec<T>` там, где достаточно `&mut [T]`

Если функция не меняет длину вектора, более общий интерфейс через срез обычно лучше.

## Что мы узнали

| Синтаксис | Описание |
|-----------|----------|
| `&mut T` | Изменяемая ссылка |
| `&mut variable` | Создаём изменяемую ссылку |
| `*ref = value` | Изменяем через разыменование |
| Только одна &mut | Правило эксклюзивного доступа |

## Домашнее задание

1. Напиши `reset_balance(b: &mut f64)` которая устанавливает баланс в 10000.0
2. Напиши `mark_winning(trades: &mut Vec<f64>)` которая удваивает прибыльные сделки
3. Попробуй создать две &mut одновременно — изучи ошибку компилятора
4. Напиши функцию `scale_prices(prices: &mut [f64], factor: f64)` умножающую все цены

## Навигация

[← Предыдущий день](../038-borrowing-temporary-access/ru.md) | [Следующий день →](../040-one-mutable-reference-rule/ru.md)
