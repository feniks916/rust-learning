# День 65: Несколько блоков impl

## Аналогия из трейдинга

Представь себе большую торговую фирму. В ней есть отдел создания сделок, отдел расчётов прибыли, отдел отчётности и отдел проверки рисков. Все они работают с одним и тем же объектом — сделкой (Trade). Так и несколько блоков `impl` в Rust: каждый блок отвечает за свою группу методов, но все они относятся к одной структуре.

## Базовый синтаксис

В Rust можно написать несколько блоков `impl` для одной и той же структуры. Компилятор объединит их все — это абсолютно легально:

```rust
#[derive(Debug)]
struct Trade {
    ticker: String,
    entry_price: f64,
    exit_price: f64,
    quantity: f64,
}

// Первый блок impl — создание
impl Trade {
    fn new(ticker: &str, entry_price: f64, exit_price: f64, quantity: f64) -> Self {
        Trade {
            ticker: ticker.to_string(),
            entry_price,
            exit_price,
            quantity,
        }
    }
}

// Второй блок impl — совершенно отдельный, но для той же структуры
impl Trade {
    fn pnl(&self) -> f64 {
        (self.exit_price - self.entry_price) * self.quantity
    }
}

fn main() {
    let t = Trade::new("AAPL", 150.0, 165.0, 10.0);
    println!("PnL: ${:.2}", t.pnl()); // PnL: $150.00
}
```

Оба метода `new` и `pnl` вызываются абсолютно одинаково — Rust не делает разницы между тем, в каком блоке `impl` они определены.

## Зачем нужны несколько блоков?

Один огромный блок `impl` из 30 методов читать трудно. Разбив его на логические группы, ты получаешь:

- **Читаемость**: сразу понятно, где создание, где расчёты, где вывод
- **Навигацию**: легко найти нужный метод
- **Команду**: разные разработчики могут работать с разными блоками без конфликтов

```rust
// Плохо: один большой блок
impl Trade {
    fn new(...) -> Self { ... }
    fn pnl(&self) -> f64 { ... }
    fn summary(&self) { ... }
    fn is_valid(&self) -> bool { ... }
    fn pnl_percent(&self) -> f64 { ... }
    // ... ещё 20 методов вперемешку
}

// Хорошо: блоки по смыслу
impl Trade { /* создание */ }
impl Trade { /* расчёты */ }
impl Trade { /* вывод */ }
impl Trade { /* валидация */ }
```

## Блок создания

Первый блок — конструкторы и фабричные методы:

```rust
impl Trade {
    /// Создать новую сделку
    fn new(ticker: &str, entry_price: f64, exit_price: f64, quantity: f64) -> Self {
        Trade {
            ticker: ticker.to_string(),
            entry_price,
            exit_price,
            quantity,
        }
    }

    /// Создать убыточную тестовую сделку
    fn test_loss() -> Self {
        Trade::new("TEST", 100.0, 90.0, 1.0)
    }

    /// Создать с округлением цены до 2 знаков
    fn with_rounded_prices(ticker: &str, entry: f64, exit: f64, qty: f64) -> Self {
        Trade::new(
            ticker,
            (entry * 100.0).round() / 100.0,
            (exit * 100.0).round() / 100.0,
            qty,
        )
    }
}
```

## Блок расчётов

Второй блок — вся математика:

```rust
impl Trade {
    /// Абсолютный PnL в долларах
    fn pnl(&self) -> f64 {
        (self.exit_price - self.entry_price) * self.quantity
    }

    /// Процентное изменение цены
    fn pnl_percent(&self) -> f64 {
        (self.exit_price - self.entry_price) / self.entry_price * 100.0
    }

    /// Стоимость позиции на входе
    fn position_value(&self) -> f64 {
        self.entry_price * self.quantity
    }

    /// Прибыльная ли сделка?
    fn is_winner(&self) -> bool {
        self.pnl() > 0.0
    }

    /// Соотношение прибыль/убыток к стоимости позиции
    fn return_on_position(&self) -> f64 {
        self.pnl() / self.position_value() * 100.0
    }
}
```

## Блок отчётности

Третий блок — вывод и форматирование:

```rust
impl Trade {
    /// Краткое резюме сделки
    fn summary(&self) {
        let status = if self.is_winner() { "ПРИБЫЛЬ ✓" } else { "УБЫТОК ✗" };
        println!("=== Сделка {} ===", self.ticker);
        println!("  Вход:   ${:.2}", self.entry_price);
        println!("  Выход:  ${:.2}", self.exit_price);
        println!("  Объём:  {:.4}", self.quantity);
        println!("  PnL:    ${:.2} ({:+.2}%)", self.pnl(), self.pnl_percent());
        println!("  Статус: {}", status);
    }

    /// Одна строка для таблицы
    fn to_csv_row(&self) -> String {
        format!(
            "{},{:.2},{:.2},{:.4},{:.2}",
            self.ticker, self.entry_price, self.exit_price,
            self.quantity, self.pnl()
        )
    }
}
```

## Блок валидации

Четвёртый блок — проверки перед использованием:

```rust
impl Trade {
    /// Все поля корректны?
    fn is_valid(&self) -> bool {
        !self.ticker.is_empty()
            && self.entry_price > 0.0
            && self.exit_price > 0.0
            && self.quantity > 0.0
    }

    /// Не слишком ли большая позиция?
    fn check_size(&self, max_position: f64) -> Result<(), String> {
        let value = self.position_value();
        if value > max_position {
            Err(format!(
                "Позиция ${:.2} превышает лимит ${:.2}",
                value, max_position
            ))
        } else {
            Ok(())
        }
    }

    /// Допустимое ли изменение цены (не более 50%)?
    fn check_price_move(&self) -> Result<(), String> {
        let change = (self.exit_price - self.entry_price).abs() / self.entry_price;
        if change > 0.5 {
            Err(format!("Подозрительное изменение цены: {:.1}%", change * 100.0))
        } else {
            Ok(())
        }
    }
}
```

## Порядок блоков неважен

Rust не требует определённого порядка блоков `impl`. Ты можешь сначала написать валидацию, потом создание — всё равно скомпилируется:

```rust
// Порядок блоков не влияет на компиляцию
impl Trade { /* валидация */ }  // можно первым
impl Trade { /* создание */  }  // можно вторым
impl Trade { /* расчёты */   }  // можно третьим

fn main() {
    // Методы из всех блоков доступны одинаково
    let t = Trade::new("ETHUSDT", 2000.0, 2200.0, 5.0);

    // Из блока расчётов
    println!("PnL: ${:.2}", t.pnl());

    // Из блока валидации
    if t.is_valid() {
        println!("Сделка корректна");
    }

    // Из блока отчётности
    t.summary();
}
```

## Практический пример: Trade с 4 блоками impl

```rust
#[derive(Debug, Clone)]
struct Trade {
    ticker: String,
    entry_price: f64,
    exit_price: f64,
    quantity: f64,
    commission: f64,
}

// === БЛОК 1: Создание ===
impl Trade {
    fn new(ticker: &str, entry: f64, exit: f64, qty: f64) -> Self {
        Trade {
            ticker: ticker.to_string(),
            entry_price: entry,
            exit_price: exit,
            quantity: qty,
            commission: 0.001, // 0.1% комиссия по умолчанию
        }
    }

    fn with_commission(mut self, commission: f64) -> Self {
        self.commission = commission;
        self
    }
}

// === БЛОК 2: Расчёты ===
impl Trade {
    fn gross_pnl(&self) -> f64 {
        (self.exit_price - self.entry_price) * self.quantity
    }

    fn commission_cost(&self) -> f64 {
        (self.entry_price + self.exit_price) * self.quantity * self.commission
    }

    fn net_pnl(&self) -> f64 {
        self.gross_pnl() - self.commission_cost()
    }

    fn pnl_percent(&self) -> f64 {
        self.net_pnl() / (self.entry_price * self.quantity) * 100.0
    }

    fn is_winner(&self) -> bool {
        self.net_pnl() > 0.0
    }
}

// === БЛОК 3: Отчётность ===
impl Trade {
    fn summary(&self) {
        println!("╔══════════════════════════╗");
        println!("║  Сделка: {:>14}  ║", self.ticker);
        println!("╠══════════════════════════╣");
        println!("║  Вход:  {:>15.2}$ ║", self.entry_price);
        println!("║  Выход: {:>15.2}$ ║", self.exit_price);
        println!("║  Объём: {:>15.4}  ║", self.quantity);
        println!("╠══════════════════════════╣");
        println!("║  Валовый PnL: {:>10.2}$ ║", self.gross_pnl());
        println!("║  Комиссия:   {:>10.2}$ ║", self.commission_cost());
        println!("║  Чистый PnL: {:>10.2}$ ║", self.net_pnl());
        println!("╠══════════════════════════╣");
        let result = if self.is_winner() { "ПРИБЫЛЬ" } else { "УБЫТОК " };
        println!("║  Итог: {:>17}  ║", result);
        println!("╚══════════════════════════╝");
    }
}

// === БЛОК 4: Валидация ===
impl Trade {
    fn is_valid(&self) -> bool {
        !self.ticker.is_empty()
            && self.entry_price > 0.0
            && self.exit_price > 0.0
            && self.quantity > 0.0
            && self.commission >= 0.0
            && self.commission < 1.0
    }

    fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        if self.ticker.is_empty() {
            errors.push("Тикер не может быть пустым".to_string());
        }
        if self.entry_price <= 0.0 {
            errors.push(format!("Цена входа должна быть > 0, получено: {}", self.entry_price));
        }
        if self.quantity <= 0.0 {
            errors.push(format!("Объём должен быть > 0, получено: {}", self.quantity));
        }
        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}

fn main() {
    let trade = Trade::new("BTCUSDT", 65000.0, 68500.0, 0.5)
        .with_commission(0.001);

    match trade.validate() {
        Ok(()) => {
            println!("Сделка прошла валидацию\n");
            trade.summary();
            println!("\nЧистый PnL: ${:.2}", trade.net_pnl());
            println!("Доходность: {:.2}%", trade.pnl_percent());
        }
        Err(errors) => {
            println!("Ошибки валидации:");
            for e in &errors {
                println!("  - {}", e);
            }
        }
    }
}
```

## Что мы узнали

| Синтаксис | Описание | Применение |
|-----------|----------|------------|
| `impl Struct { }` несколько раз | Несколько блоков impl для одной структуры | Организация кода по смысловым группам |
| Порядок блоков | Не важен для компилятора | Можно расставить в любом порядке |
| Методы из разных блоков | Вызываются одинаково | Нет разницы для пользователя структуры |
| Блок создания | `new()`, `with_*()` | Конструкторы и строители |
| Блок расчётов | `pnl()`, `percent()` | Математика и бизнес-логика |
| Блок отчётности | `summary()`, `to_csv()` | Форматирование и вывод |
| Блок валидации | `is_valid()`, `validate()` | Проверка корректности данных |

## Домашнее задание

1. Создай структуру `Order` с полями: `symbol`, `side` (Buy/Sell), `price`, `quantity`, `status`
   - Добавь перечисление `OrderSide { Buy, Sell }` и `OrderStatus { Pending, Filled, Cancelled }`

2. Напиши блок создания с методами:
   - `new(symbol, side, price, quantity) -> Self` — создание нового ордера
   - `market_buy(symbol, quantity) -> Self` — рыночный ордер на покупку (price = 0.0)

3. Напиши блок исполнения:
   - `fill(&mut self, fill_price: f64)` — исполняет ордер по указанной цене
   - `cancel(&mut self)` — отменяет ордер
   - `is_active(&self) -> bool` — ордер ещё не исполнен и не отменён?

4. Напиши блок отчётности:
   - `display(&self)` — красивый вывод информации об ордере
   - `to_log_string(&self) -> String` — строка для записи в лог

5. Напиши блок валидации:
   - `check_price(&self) -> Result<(), String>` — цена не отрицательная
   - `check_quantity(&self) -> Result<(), String>` — объём в допустимом диапазоне (0.0001..=1000.0)

## Навигация

[← Предыдущий день](../064-associated-functions-order-new/ru.md) | [Следующий день →](../066-tuple-structs/ru.md)
