# День 33: Владение — кто держит актив

## Аналогия из трейдинга

На бирже каждый актив принадлежит ровно одному владельцу в каждый момент времени. Акция Apple не может одновременно числиться в двух портфелях — она либо у тебя, либо у покупателя. Когда ты её продаёшь, ты теряешь право на неё. Rust применяет тот же принцип к памяти: каждый кусок данных в куче имеет ровно одного владельца, и когда владелец уходит — данные уничтожаются автоматически. Никаких утечек, никаких висячих указателей.

---

## Три правила владения

Весь Rust держится на трёх законах. Если ты их понял — ты понял 80% языка:

1. **Каждое значение имеет одного владельца** — одну переменную, которая за него отвечает
2. **В один момент времени может быть только один владелец** — нельзя отдать одно значение двум переменным
3. **Когда владелец выходит из области видимости — значение уничтожается** — автоматически, без `free()`

```rust
fn main() {
    // Правило 1: ticker — единственный владелец строки "BTCUSDT"
    let ticker = String::from("BTCUSDT");

    // Правило 2: после этой строки ticker больше не владеет данными
    let new_ticker = ticker; // владение переходит к new_ticker

    // println!("{}", ticker); // ОШИБКА: ticker уступил владение!

    // Правило 3: когда main() завершится, new_ticker уничтожится,
    // Rust автоматически освободит память в куче
    println!("Владелец: {}", new_ticker);
} // ← здесь new_ticker уничтожается, память освобождается
```

Эти три правила — не ограничение, а гарантия безопасности: нет утечек памяти, нет двойного освобождения, нет null-pointer.

---

## Область видимости (Scope)

Область видимости — это блок `{}`, в котором переменная живёт. За его пределами она мертва. Как торговая сессия: ордера сессии не переходят в следующую.

```rust
fn main() {
    // portfolio ещё не существует

    {
        let portfolio = String::from("BTC: 0.5, ETH: 2.0, SOL: 10.0");
        println!("Портфель открыт: {}", portfolio);
        // portfolio живёт здесь
    } // ← portfolio уничтожается здесь

    // println!("{}", portfolio); // ОШИБКА: portfolio вне области видимости

    // Вложенные области видимости
    let market = String::from("NYSE");
    {
        let session = String::from("утренняя");
        println!("Рынок {} открылся, сессия: {}", market, session);

        {
            let order = String::from("BUY AAPL 100");
            println!("Ордер в сессии: {}", order);
        } // order уничтожен

        println!("Сессия {} продолжается", session);
    } // session уничтожен
    // market всё ещё жив
    println!("Рынок всё ещё работает: {}", market);
} // market уничтожается здесь
```

---

## Drop трейт

Когда владелец выходит из области видимости, Rust вызывает специальный метод `drop()` автоматически — это трейт `Drop`. Ты можешь реализовать его для своего типа, чтобы выполнить код при уничтожении (закрыть файл, соединение, записать лог).

```rust
struct TradingPosition {
    symbol: String,
    entry_price: f64,
    quantity: f64,
    is_open: bool,
}

impl Drop for TradingPosition {
    fn drop(&mut self) {
        if self.is_open {
            println!(
                "Позиция {} автоматически закрыта при выходе из scope. Entry: ${:.2}",
                self.symbol, self.entry_price
            );
        } else {
            println!("Позиция {} корректно закрыта", self.symbol);
        }
    }
}

fn main() {
    let btc = TradingPosition {
        symbol: String::from("BTCUSDT"),
        entry_price: 42000.0,
        quantity: 0.5,
        is_open: true,
    };

    {
        let eth = TradingPosition {
            symbol: String::from("ETHUSDT"),
            entry_price: 3000.0,
            quantity: 2.0,
            is_open: true,
        };
        println!("Открыты BTC и ETH позиции");
    } // ← eth.drop() вызывается здесь ("ETHUSDT закрыта")

    println!("ETH уже закрыта, BTC ещё открыта");
} // ← btc.drop() вызывается здесь ("BTCUSDT закрыта")

// Важно: уничтожение происходит в ОБРАТНОМ порядке создания
```

---

## drop() явно

Иногда нужно уничтожить значение раньше конца области видимости — например, освободить большой буфер памяти немедленно. Используй функцию `drop()`:

```rust
fn main() {
    // Загрузили миллион ценовых точек — 8 МБ в куче
    let large_history: Vec<f64> = (0..1_000_000).map(|i| i as f64).collect();
    let count = large_history.len();

    // Вычислили что нужно
    let max = large_history.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    println!("Загружено {} цен, максимум: {}", count, max);

    // Данные больше не нужны — освобождаем НЕМЕДЛЕННО
    drop(large_history);
    // large_history теперь недоступен, 8 МБ памяти освобождено

    // println!("{}", large_history.len()); // ОШИБКА: уже уничтожен

    // Теперь можем загрузить новые данные без двойного расхода памяти
    let new_data: Vec<f64> = (1_000_000..2_000_000).map(|i| i as f64).collect();
    let new_max = new_data.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    println!("Новые данные: максимум = {}", new_max);
}
```

Нельзя вызвать `drop()` дважды для одного значения — Rust это запрещает на уровне компилятора.

---

## Функции и владение

Когда ты передаёшь значение в функцию — владение переходит к параметру функции. После вызова оригинальная переменная недействительна:

```rust
fn analyze_portfolio(portfolio: String) -> usize {
    // portfolio теперь владеет строкой
    let count = portfolio.split(',').count();
    println!("Анализируем портфель: {}", portfolio);
    println!("Количество позиций: {}", count);
    count
} // portfolio уничтожается здесь

fn log_price(price: f64) {
    // price — тип Copy, создаётся битовая копия
    println!("[LOG] Цена: ${:.2}", price);
} // price уничтожается, но оригинал в caller жив

fn main() {
    let btc_price: f64 = 42000.0;
    let my_portfolio = String::from("BTC,ETH,SOL,ADA,DOT");

    log_price(btc_price);               // f64 копируется
    println!("Цена всё ещё: {}", btc_price); // ОК — Copy

    let positions = analyze_portfolio(my_portfolio); // String ПЕРЕМЕЩАЕТСЯ
    // println!("{}", my_portfolio); // ОШИБКА: my_portfolio уже передан
    println!("Итого позиций: {}", positions);
}
```

---

## Возврат владения из функции

Функция может вернуть владение обратно вызывающему коду:

```rust
fn build_ticker(base: String, quote: &str) -> String {
    // Берём владение base, создаём новую строку и возвращаем
    format!("{}{}", base, quote)
    // base уничтожается (вместе с quote используются данные)
    // новая String возвращается вызывающему
}

fn load_watchlist() -> Vec<String> {
    // Создаём новый Vec и отдаём владение вызывающему
    vec![
        String::from("BTCUSDT"),
        String::from("ETHUSDT"),
        String::from("SOLUSDT"),
        String::from("BNBUSDT"),
    ]
} // возвращаемое значение ПЕРЕМЕЩАЕТСЯ, не уничтожается

fn main() {
    let base = String::from("BTC");
    let ticker = build_ticker(base, "USDT");
    // base недоступен, ticker — новый владелец
    println!("Тикер: {}", ticker);

    // Получаем Vec из функции — мы владельцы
    let watchlist = load_watchlist();
    println!("Список: {:?}", watchlist);
    println!("Первый инструмент: {}", watchlist[0]);
}
```

---

## Паттерн "одолжи через return" (кортеж)

До заимствования (borrowing) иногда использовался паттерн: взял значение — использовал — вернул обратно через кортеж. Это многословно, но показывает суть владения:

```rust
fn calculate_stats(prices: Vec<f64>) -> (Vec<f64>, f64, f64, f64) {
    // Берём владение prices
    let min = prices.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = prices.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    let avg = prices.iter().sum::<f64>() / prices.len() as f64;

    // Возвращаем оригинал И результаты вычислений
    (prices, min, max, avg)
}

fn tag_expensive(tickers: Vec<String>, threshold: f64, prices: &[f64]) -> Vec<String> {
    // Трансформируем вектор и возвращаем новый
    tickers
        .into_iter()
        .enumerate()
        .map(|(i, t)| {
            if i < prices.len() && prices[i] > threshold {
                format!("{} [EXPENSIVE]", t)
            } else {
                t
            }
        })
        .collect()
}

fn main() {
    let price_data = vec![48000.0, 49500.0, 50000.0, 51200.0, 49800.0];

    // Передаём владение и получаем обратно через кортеж
    let (prices_back, min, max, avg) = calculate_stats(price_data);
    println!("Цены: {:?}", prices_back);
    println!("Мин: ${:.2}, Макс: ${:.2}, Ср: ${:.2}", min, max, avg);

    let tickers = vec![
        String::from("BTCUSDT"),
        String::from("ETHUSDT"),
        String::from("SOLUSDT"),
    ];
    let tagged = tag_expensive(tickers, 49000.0, &prices_back);
    // tickers недоступен, tagged — новый владелец
    for t in &tagged {
        println!("  {}", t);
    }
}
```

Этот паттерн многословен. Именно поэтому Rust придумал **заимствование** (`&T`), которое мы изучим в главах 37–38.

---

## Практический пример: Система управления позициями

```rust
#[derive(Debug)]
struct Position {
    symbol: String,
    entry_price: f64,
    quantity: f64,
    is_open: bool,
}

impl Position {
    fn new(symbol: String, entry_price: f64, quantity: f64) -> Self {
        println!("[OPEN] Открываем позицию {} @ ${:.2}", symbol, entry_price);
        Position { symbol, entry_price, quantity, is_open: true }
    }

    // Принимает self по значению — "потребляет" позицию
    fn close(mut self, exit_price: f64) -> f64 {
        self.is_open = false;
        let pnl = (exit_price - self.entry_price) * self.quantity;
        println!(
            "[CLOSE] Закрываем {} @ ${:.2}, PnL: ${:.2}",
            self.symbol, exit_price, pnl
        );
        pnl
        // self уничтожается здесь (is_open == false, Drop не печатает предупреждение)
    }
}

impl Drop for Position {
    fn drop(&mut self) {
        if self.is_open {
            let unrealized = self.entry_price * 0.01 * self.quantity;
            println!(
                "[WARNING] Позиция {} уничтожена без закрытия! Нереализованный P/L: ~${:.2}",
                self.symbol, unrealized
            );
        }
    }
}

fn run_strategy(symbol: String, entry: f64, exit: f64, qty: f64) -> f64 {
    let position = Position::new(symbol, entry, qty);
    // Работаем с позицией...
    println!("  Удерживаем позицию...");
    position.close(exit) // владение потребляется, PnL возвращается
}

fn main() {
    println!("=== Торговый день ===");

    let pnl1 = run_strategy(String::from("BTCUSDT"), 42000.0, 45000.0, 0.1);
    let pnl2 = run_strategy(String::from("ETHUSDT"), 3000.0, 2800.0, 1.0);

    println!("\n=== Итоги ===");
    println!("BTC PnL: ${:.2}", pnl1);
    println!("ETH PnL: ${:.2}", pnl2);
    println!("Общий PnL: ${:.2}", pnl1 + pnl2);

    // Демонстрация предупреждения при незакрытой позиции
    println!("\n=== Незакрытая позиция ===");
    {
        let _risky = Position::new(String::from("SOLUSDT"), 150.0, 10.0);
        println!("Позиция открыта, выходим из блока...");
    } // _risky.drop() вызывается — печатает WARNING
}
```

---

## Что мы узнали

| Синтаксис | Описание | Пример в трейдинге |
|-----------|----------|--------------------|
| `let x = value` | x становится владельцем | Купить актив в портфель |
| `{ let x = ... }` | x уничтожается при `}` | Закрытие торговой сессии |
| `let y = x` (heap) | Move: x больше недействителен | Продать актив другому |
| `fn f(x: T)` | f берёт владение x | Передать ордер брокеру |
| `fn f() -> T` | f передаёт владение caller | Брокер возвращает исполненный ордер |
| `drop(x)` | Явное уничтожение | Принудительное закрытие позиции |
| `impl Drop for T` | Код при уничтожении | Логирование закрытия сделки |

---

## Домашнее задание

1. Создай структуру `Order` с полями `id: u32`, `symbol: String`, `price: f64`, `qty: f64`.
   - Реализуй `Drop` для `Order`, печатающий `"[LOG] Ордер {id} на {symbol} уничтожен"`
   - Создай несколько ордеров в разных областях видимости `{}`
   - Запусти и проследи: уничтожение идёт в обратном порядке создания

2. Напиши функцию `process_order(order: Order) -> String`:
   - Принимает владение ордером
   - Формирует строку `"Ордер #{id}: {symbol} x{qty} @ ${price} — ИСПОЛНЕН"`
   - После вызова убедись, что исходная переменная недоступна

3. Реализуй паттерн "одолжи через return":
   - Функция `validate_orders(orders: Vec<Order>) -> (Vec<Order>, Vec<u32>)` 
   - Возвращает кортеж: (оригинальный вектор, список id ордеров с price > 40000)
   - Используй `.into_iter()` и пересоздай вектор через `.collect()`

4. Изучи порядок Drop в структурах:
   - Создай структуру `Portfolio { name: String, positions: Vec<String> }` с `impl Drop`
   - Вложи один `Portfolio` внутрь другого через `Box`
   - Выведи порядок уничтожения и объясни его

---

## Навигация

[← Предыдущий день](../032-stack-and-heap/ru.md) | [Следующий день →](../034-move-selling-asset/ru.md)
