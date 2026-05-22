# День 34: Move — продажа актива

## Аналогия из трейдинга

Когда ты продаёшь акцию на бирже — она уходит из твоего портфеля навсегда. Ты больше не можешь ею распоряжаться: ни продать снова, ни получить дивиденды. Покупатель — новый полноправный владелец. В Rust операция присваивания для типов на куче работает точно так же: `let b = a` перемещает владение из `a` в `b`, и `a` становится недействительным.

---

## Что такое Move

Move (перемещение) — это передача владения от одной переменной к другой. После Move оригинальная переменная **недействительна**: компилятор не позволит её использовать.

Важно: Move — это **не копирование данных**! Данные остаются в том же месте в куче. Просто меняется то, кто за них отвечает.

```rust
fn main() {
    // s1 владеет строкой "BTCUSDT" в куче
    let s1 = String::from("BTCUSDT");
    //      s1 на стеке:
    //      [ ptr → куча, len=7, cap=7 ]
    //                ↓
    //         куча: "BTCUSDT"

    // Move: s2 берёт владение. s1 становится пустым указателем
    let s2 = s1;
    //      s1 на стеке: [ INVALID ]
    //      s2 на стеке: [ ptr → куча, len=7, cap=7 ]
    //                              ↓
    //                       куча: "BTCUSDT"  (ТЕ ЖЕ данные, не копия!)

    // println!("{}", s1); // ОШИБКА: use of moved value: `s1`
    println!("s2 = {}", s2); // ОК
}
```

---

## Move при присваивании

Move происходит всегда, когда ты присваиваешь типы, не реализующие `Copy`:

```rust
fn main() {
    // Vec<f64> — не Copy, поэтому Move
    let prices = vec![42000.0, 43000.0, 44000.0];
    let prices2 = prices; // Move!

    // println!("{:?}", prices); // ОШИБКА
    println!("prices2: {:?}", prices2);

    // Строки тоже Move
    let order1 = String::from("BUY BTCUSDT 0.1");
    let order2 = order1; // Move!
    // println!("{}", order1); // ОШИБКА
    println!("order2: {}", order2);

    // f64 — Copy, поэтому копирование
    let price1 = 42000.0_f64;
    let price2 = price1; // Copy!
    println!("price1={}, price2={}", price1, price2); // ОБА работают
}
```

---

## Move в функцию

Передача аргумента в функцию — это тоже Move для типов без `Copy`:

```rust
fn print_portfolio(p: Vec<String>) {
    // p получает владение
    println!("Портфель ({} позиций):", p.len());
    for ticker in &p {
        println!("  - {}", ticker);
    }
} // p уничтожается здесь

fn calculate_total(prices: Vec<f64>) -> f64 {
    // prices перемещён сюда
    prices.iter().sum()
} // prices уничтожается здесь

fn main() {
    let my_portfolio = vec![
        String::from("BTCUSDT"),
        String::from("ETHUSDT"),
        String::from("SOLUSDT"),
    ];

    print_portfolio(my_portfolio); // Move в функцию
    // println!("{:?}", my_portfolio); // ОШИБКА: перемещён

    let price_list = vec![42000.0, 3000.0, 150.0];
    let total = calculate_total(price_list); // Move
    // println!("{:?}", price_list); // ОШИБКА: перемещён
    println!("Общая стоимость: ${}", total);
}
```

---

## Возврат назад — Move из функции

Функция может вернуть владение обратно, "переместив" значение через return:

```rust
fn add_timestamp(order: String) -> String {
    // order получен по Move
    format!("[2024-01-15 10:30:00] {}", order)
    // Новая строка перемещается к вызывающему
}

fn filter_profitable(trades: Vec<f64>) -> Vec<f64> {
    // trades получен по Move, возвращаем новый Vec
    trades.into_iter().filter(|&t| t > 0.0).collect()
    // Vec перемещается к вызывающему
}

fn main() {
    let raw_order = String::from("BUY BTCUSDT 0.5 @ 42000");
    let stamped_order = add_timestamp(raw_order); // raw_order перемещён
    println!("Ордер с временем: {}", stamped_order);

    let all_trades = vec![150.0, -50.0, 300.0, -100.0, 200.0];
    let profitable = filter_profitable(all_trades); // all_trades перемещён
    println!("Прибыльные сделки: {:?}", profitable);
    println!("Количество: {}", profitable.len());
}
```

---

## Частичное перемещение структуры

Можно переместить отдельное поле структуры. После этого вся структура считается "частично перемещённой" — некоторые поля доступны, некоторые нет:

```rust
#[derive(Debug)]
struct Trade {
    ticker: String,    // String — не Copy
    price: f64,        // f64 — Copy
    quantity: f64,     // f64 — Copy
    notes: String,     // String — не Copy
}

fn main() {
    let trade = Trade {
        ticker: String::from("BTCUSDT"),
        price: 42000.0,
        quantity: 0.1,
        notes: String::from("Пробой уровня сопротивления"),
    };

    // Перемещаем только ticker
    let ticker = trade.ticker; // Move поля ticker

    // trade.ticker теперь недоступен, но остальные поля — OK
    println!("Тикер: {}", ticker);
    println!("Цена: {}", trade.price);    // f64 — Copy, OK
    println!("Количество: {}", trade.quantity); // f64 — Copy, OK

    // println!("{}", trade.ticker); // ОШИБКА: перемещён
    // println!("{:?}", trade); // ОШИБКА: структура частично перемещена

    // Переместим и notes
    let notes = trade.notes; // Move
    println!("Заметки: {}", notes);
    // Теперь из trade доступны только price и quantity (Copy)
}
```

---

## Copy vs Move — ключевое различие

```rust
fn demo_copy_vs_move() {
    // === Copy типы (стековые, маленькие) ===
    let price: f64 = 42000.0;
    let volume: u64 = 1000;
    let is_long: bool = true;
    let side: char = 'B';

    let p2 = price;     // Copy — побитовое копирование стека
    let v2 = volume;    // Copy
    let b2 = is_long;   // Copy
    let s2 = side;      // Copy

    // ВСЕ оригиналы живы:
    println!("price={}, p2={}", price, p2);
    println!("volume={}, v2={}", volume, v2);

    // === Move типы (куча, большие, не-Copy) ===
    let ticker = String::from("BTCUSDT");
    let history = vec![1.0, 2.0, 3.0];
    let big_data: Box<[f64; 100]> = Box::new([0.0; 100]);

    let t2 = ticker;   // Move — указатель переходит от ticker к t2
    let h2 = history;  // Move
    let b2 = big_data; // Move

    // Оригиналы НЕДОСТУПНЫ:
    // println!("{}", ticker); // ОШИБКА
    // println!("{:?}", history); // ОШИБКА

    println!("t2={}", t2);
    println!("h2 len={}", h2.len());
    println!("b2 len={}", b2.len());
}

fn main() {
    demo_copy_vs_move();
}
```

---

## Когда Move — проблема и как решить

Move мешает, когда нужно использовать значение несколько раз. Есть три решения:

```rust
fn process(data: Vec<f64>) -> f64 {
    data.iter().sum()
}

fn main() {
    let prices = vec![42000.0, 43000.0, 44000.0];

    // ПРОБЛЕМА: после первого вызова prices перемещён
    // let total1 = process(prices);
    // let total2 = process(prices); // ОШИБКА!

    // РЕШЕНИЕ 1: Clone — дорого (копирует данные)
    let total1 = process(prices.clone());
    let total2 = process(prices.clone()); // prices всё ещё жив
    println!("total1={}, total2={}", total1, total2);

    // РЕШЕНИЕ 2: Передавай ссылку &Vec вместо Vec
    fn process_ref(data: &Vec<f64>) -> f64 {
        data.iter().sum()
    }
    let ref_total1 = process_ref(&prices);
    let ref_total2 = process_ref(&prices); // prices живёт!
    println!("ref totals: {} {}", ref_total1, ref_total2);

    // РЕШЕНИЕ 3: Return ownership (кортеж)
    fn process_and_return(data: Vec<f64>) -> (Vec<f64>, f64) {
        let sum: f64 = data.iter().sum();
        (data, sum)
    }
    let (prices_back, sum) = process_and_return(prices);
    println!("prices back: {:?}, sum={}", prices_back, sum);
}
```

---

## Практический пример: Конвейер обработки ордеров

```rust
#[derive(Debug)]
struct RawOrder {
    symbol: String,
    price: f64,
    quantity: f64,
    side: String,
}

#[derive(Debug)]
struct ValidatedOrder {
    symbol: String,
    price: f64,
    quantity: f64,
    side: String,
    notional: f64,
}

#[derive(Debug)]
struct ExecutedOrder {
    symbol: String,
    fill_price: f64,
    quantity: f64,
    side: String,
    notional: f64,
    order_id: u64,
}

// Каждая функция потребляет (Move) ордер и возвращает следующий этап
fn validate(order: RawOrder) -> Result<ValidatedOrder, String> {
    if order.price <= 0.0 {
        return Err(format!("Неверная цена: {}", order.price));
    }
    if order.quantity <= 0.0 {
        return Err(format!("Неверный объём: {}", order.quantity));
    }
    let notional = order.price * order.quantity;
    Ok(ValidatedOrder {
        symbol: order.symbol,    // Move поля
        price: order.price,      // Copy
        quantity: order.quantity, // Copy
        side: order.side,        // Move
        notional,
    })
}

fn execute(order: ValidatedOrder) -> ExecutedOrder {
    let slippage = 0.0005;
    let fill_price = if order.side == "BUY" {
        order.price * (1.0 + slippage)
    } else {
        order.price * (1.0 - slippage)
    };

    ExecutedOrder {
        symbol: order.symbol,     // Move
        fill_price,
        quantity: order.quantity, // Copy
        side: order.side,         // Move
        notional: order.notional, // Copy
        order_id: 1001,
    }
}

fn main() {
    let raw = RawOrder {
        symbol: String::from("BTCUSDT"),
        price: 42000.0,
        quantity: 0.1,
        side: String::from("BUY"),
    };

    println!("Сырой ордер: {:?}", raw);

    match validate(raw) { // raw перемещён
        Ok(validated) => {
            println!("Проверен: {:?}", validated);
            let executed = execute(validated); // validated перемещён
            println!("Исполнен: {:?}", executed);
            println!(
                "Заявка #{}: {} {} {} @ ${:.2} (исполнено @ ${:.2})",
                executed.order_id,
                executed.side,
                executed.quantity,
                executed.symbol,
                executed.notional / executed.quantity,
                executed.fill_price
            );
        }
        Err(e) => println!("Ошибка валидации: {}", e),
    }
}
```

---

## Что мы узнали

| Синтаксис | Описание | Пример в трейдинге |
|-----------|----------|--------------------|
| `let b = a` (String/Vec) | Move: a недействителен | Продажа актива покупателю |
| `fn f(x: T)` | Move в функцию | Передача ордера на исполнение |
| `fn f() -> T` | Move из функции | Получение исполненного ордера |
| `let (a, b) = f(x)` | Move и получение обратно | Ордер + результат исполнения |
| `trade.field` (String) | Частичный Move поля | Перевод одного актива из структуры |
| `.clone()` | Избежать Move через копию | Дублировать портфель для теста |

---

## Домашнее задание

1. Напиши функцию `sell_asset(portfolio: Vec<String>, symbol: &str) -> (Vec<String>, bool)`:
   - Принимает владение вектором
   - Удаляет первый элемент, совпадающий с `symbol`
   - Возвращает кортеж `(новый_вектор, был_ли_продан)`
   - Убедись, что оригинальный вектор недоступен после вызова

2. Создай цепочку Move:
   - `let a = String::from("ORDER")` → переместить в `b` → переместить в `c` → передать в функцию
   - Подтверди, что только `c` (и затем функция) могут использовать значение

3. Реализуй частичное перемещение:
   - Структура `Position { symbol: String, price: f64, notes: String }`
   - Извлеки `symbol` и `notes` по отдельности через Move
   - Покажи, что `price` (Copy) всё ещё доступна после Move строк

4. Сравни три подхода для одной задачи "вычислить скользящее среднее и сохранить данные":
   - С `clone()` перед вызовом
   - С `&Vec` в параметре функции
   - С возвратом `(Vec<f64>, Vec<f64>)` кортежа

---

## Навигация

[← Предыдущий день](../033-ownership-who-holds-asset/ru.md) | [Следующий день →](../035-clone-copying-portfolio/ru.md)
