# День 125: std::fs — операции с файлами

## Аналогия из трейдинга

Торговый терминал постоянно читает и пишет файлы: сохраняет историю сделок, загружает конфигурацию стратегии, экспортирует отчёты в CSV. Модуль `std::fs` — это твой файловый менеджер в Rust: он умеет создавать, читать, копировать и удалять файлы, точно так же как ты работаешь с торговыми данными на диске.

## fs::write / fs::read_to_string / fs::read

Три самых базовых операции — записать, прочитать как текст, прочитать как байты:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Записать текст в файл (создаст или перезапишет)
    fs::write("trades.csv", "ticker,entry,exit,pnl\nBTCUSDT,65000,68000,1500\n")?;
    println!("Файл записан");

    // Прочитать файл как строку
    let content = fs::read_to_string("trades.csv")?;
    println!("Содержимое:\n{}", content);

    // Прочитать файл как байты (для бинарных данных)
    let bytes = fs::read("trades.csv")?;
    println!("Размер в байтах: {}", bytes.len());

    Ok(())
}
```

`fs::write` принимает путь и любое значение, реализующее `AsRef<[u8]>` — строку, `Vec<u8>`, байтовый срез. Если файл существует — перезапишет, если нет — создаст.

## fs::copy / fs::rename / fs::remove_file

Операции перемещения и удаления файлов:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Подготовим тестовый файл
    fs::write("btc_trades.csv", "ticker,price\nBTCUSDT,65000")?;

    // Копировать файл (оригинал остаётся)
    let bytes_copied = fs::copy("btc_trades.csv", "btc_trades_backup.csv")?;
    println!("Скопировано байт: {}", bytes_copied);

    // Переименовать / переместить файл
    // fs::rename("btc_trades.csv", "archive/btc_trades_2024.csv")?;

    // Удалить файл
    fs::remove_file("btc_trades_backup.csv")?;
    fs::remove_file("btc_trades.csv")?;
    println!("Файлы удалены");

    Ok(())
}
```

`fs::rename` работает как атомарная операция на Unix — файл либо перемещён, либо нет. Это важно при архивировании торговых логов.

## fs::create_dir / create_dir_all / remove_dir_all

Управление директориями:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Создать все директории в пути (как mkdir -p)
    fs::create_dir_all("data/2024/trades/btc")?;
    println!("Структура папок создана");

    // Записать данные в созданную папку
    fs::write("data/2024/trades/btc/january.csv", "date,price\n2024-01-01,42000")?;

    // Удалить директорию со всем содержимым (осторожно!)
    fs::remove_dir_all("data")?;
    println!("Папка data и всё содержимое удалены");

    Ok(())
}
```

`create_dir_all` — незаменимая функция при создании структуры папок для хранения торговых данных по датам: `data/2024/01/`, `data/2024/02/` и т.д.

## fs::read_dir — список файлов

Перечислить содержимое директории:

```rust
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::create_dir_all("reports")?;
    fs::write("reports/btc.csv", "ticker\nBTCUSDT")?;
    fs::write("reports/eth.csv", "ticker\nETHUSDT")?;
    fs::write("reports/bnb.csv", "ticker\nBNBUSDT")?;

    println!("Файлы в папке reports:");
    for entry in fs::read_dir("reports")? {
        let entry = entry?;
        let path = entry.path();
        let file_name = entry.file_name();
        println!("  {:?}", file_name);

        // Обрабатываем только CSV файлы
        if path.extension().map(|e| e == "csv").unwrap_or(false) {
            let content = fs::read_to_string(&path)?;
            println!("    -> {} байт", content.len());
        }
    }

    fs::remove_dir_all("reports")?;
    Ok(())
}
```

`read_dir` возвращает итератор по `DirEntry`. Каждый `DirEntry` содержит путь, имя файла и метаданные.

## metadata — размер и время

Получить информацию о файле без его чтения:

```rust
use std::fs;
use std::time::SystemTime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::write("trades.log", "много данных о сделках...")?;

    let meta = fs::metadata("trades.log")?;

    println!("Размер файла: {} байт", meta.len());
    println!("Это файл: {}", meta.is_file());
    println!("Это папка: {}", meta.is_dir());
    println!("Только чтение: {}", meta.permissions().readonly());

    // Время последнего изменения
    if let Ok(modified) = meta.modified() {
        let duration = SystemTime::now()
            .duration_since(modified)
            .unwrap_or_default();
        println!("Изменён {} секунд назад", duration.as_secs());
    }

    fs::remove_file("trades.log")?;
    Ok(())
}
```

`metadata` полезна для проверки: не устарел ли кэш с котировками, существует ли файл конфигурации, достаточно ли места на диске.

## Обработка ошибок через ?

При работе с файлами ошибки неизбежны. Оператор `?` делает их обработку удобной:

```rust
use std::fs;
use std::io;

fn load_strategy_config(path: &str) -> Result<String, io::Error> {
    // Если файл не существует — вернём ошибку автоматически
    let config = fs::read_to_string(path)?;
    Ok(config)
}

fn save_trade_report(trades: &[&str], path: &str) -> Result<(), io::Error> {
    let mut output = String::from("ticker,pnl\n");
    for trade in trades {
        output.push_str(trade);
        output.push('\n');
    }
    fs::write(path, output)?;
    Ok(())
}

fn main() {
    // Пробуем загрузить конфиг — файл может не существовать
    match load_strategy_config("strategy.toml") {
        Ok(config) => println!("Конфиг загружен: {} байт", config.len()),
        Err(e) if e.kind() == io::ErrorKind::NotFound => {
            println!("Конфиг не найден, используем настройки по умолчанию");
        }
        Err(e) => println!("Ошибка чтения: {}", e),
    }

    // Сохраняем отчёт
    let trades = ["BTCUSDT,1500", "ETHUSDT,-200", "BNBUSDT,300"];
    match save_trade_report(&trades, "report.csv") {
        Ok(()) => println!("Отчёт сохранён"),
        Err(e) => println!("Ошибка сохранения: {}", e),
    }

    let _ = fs::remove_file("report.csv");
}
```

Паттерн `e.kind() == io::ErrorKind::NotFound` позволяет различать "файл не существует" и другие ошибки ввода-вывода.

## Практический пример: менеджер торговых данных

```rust
use std::fs;
use std::path::Path;

struct TradeDataManager {
    base_dir: String,
}

impl TradeDataManager {
    fn new(base_dir: &str) -> Result<Self, std::io::Error> {
        fs::create_dir_all(format!("{}/trades", base_dir))?;
        fs::create_dir_all(format!("{}/reports", base_dir))?;
        fs::create_dir_all(format!("{}/archive", base_dir))?;
        println!("Менеджер данных инициализирован в '{}'", base_dir);
        Ok(TradeDataManager { base_dir: base_dir.to_string() })
    }

    fn save_trade(&self, ticker: &str, entry: f64, exit: f64, qty: f64)
        -> Result<(), std::io::Error>
    {
        let pnl = (exit - entry) * qty;
        let path = format!("{}/trades/{}.csv", self.base_dir, ticker.to_lowercase());

        let existing = fs::read_to_string(&path).unwrap_or_default();
        let header = if existing.is_empty() { "ticker,entry,exit,qty,pnl\n" } else { "" };
        let line = format!("{},{:.2},{:.2},{:.4},{:.2}\n", ticker, entry, exit, qty, pnl);
        fs::write(&path, format!("{}{}{}", existing, header, line))?;

        println!("Сделка {} сохранена (PnL: {:.2})", ticker, pnl);
        Ok(())
    }

    fn list_tickers(&self) -> Result<Vec<String>, std::io::Error> {
        let trades_dir = format!("{}/trades", self.base_dir);
        let mut tickers = Vec::new();

        for entry in fs::read_dir(&trades_dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.extension().map(|e| e == "csv").unwrap_or(false) {
                if let Some(stem) = path.file_stem() {
                    tickers.push(stem.to_string_lossy().to_uppercase().to_string());
                }
            }
        }
        tickers.sort();
        Ok(tickers)
    }

    fn generate_report(&self) -> Result<String, std::io::Error> {
        let trades_dir = format!("{}/trades", self.base_dir);
        let mut total_pnl = 0.0f64;
        let mut trade_count = 0usize;
        let mut report = String::from("=== ТОРГОВЫЙ ОТЧЁТ ===\n\n");

        for entry in fs::read_dir(&trades_dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.extension().map(|e| e == "csv").unwrap_or(false) {
                let content = fs::read_to_string(&path)?;
                for line in content.lines().skip(1) {
                    let parts: Vec<&str> = line.split(',').collect();
                    if parts.len() >= 5 {
                        if let Ok(pnl) = parts[4].parse::<f64>() {
                            total_pnl += pnl;
                            trade_count += 1;
                        }
                    }
                }
            }
        }

        report.push_str(&format!("Сделок всего: {}\n", trade_count));
        report.push_str(&format!("Суммарный PnL: ${:.2}\n", total_pnl));
        report.push_str(&format!(
            "Средний PnL: ${:.2}\n",
            if trade_count > 0 { total_pnl / trade_count as f64 } else { 0.0 }
        ));

        let report_path = format!("{}/reports/summary.txt", self.base_dir);
        fs::write(&report_path, &report)?;
        println!("Отчёт сохранён в '{}'", report_path);

        Ok(report)
    }

    fn archive_all(&self) -> Result<(), std::io::Error> {
        let trades_dir = format!("{}/trades", self.base_dir);
        let archive_dir = format!("{}/archive", self.base_dir);

        for entry in fs::read_dir(&trades_dir)? {
            let entry = entry?;
            let src = entry.path();
            if let Some(name) = src.file_name() {
                let dst = Path::new(&archive_dir).join(name);
                fs::copy(&src, &dst)?;
                fs::remove_file(&src)?;
                println!("Архивировано: {:?}", name);
            }
        }
        Ok(())
    }

    fn cleanup(&self) -> Result<(), std::io::Error> {
        fs::remove_dir_all(&self.base_dir)?;
        println!("Все данные удалены");
        Ok(())
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let manager = TradeDataManager::new("trading_data")?;

    manager.save_trade("BTCUSDT", 65000.0, 68500.0, 0.5)?;
    manager.save_trade("ETHUSDT", 3200.0, 3050.0, 2.0)?;
    manager.save_trade("BTCUSDT", 68500.0, 70000.0, 0.3)?;
    manager.save_trade("BNBUSDT", 420.0, 445.0, 10.0)?;

    let tickers = manager.list_tickers()?;
    println!("\nТикеры с данными: {:?}", tickers);

    println!("\n{}", manager.generate_report()?);

    println!("Архивируем сделки...");
    manager.archive_all()?;

    manager.cleanup()?;

    Ok(())
}
```

## Что мы узнали

| Синтаксис | Описание | Применение |
|-----------|----------|------------|
| `fs::write(path, data)` | Записать данные в файл | Сохранение торговых данных, отчётов |
| `fs::read_to_string(path)` | Прочитать файл как строку | Загрузка конфигов, CSV файлов |
| `fs::read(path)` | Прочитать файл как байты | Бинарные файлы, сериализованные данные |
| `fs::copy(src, dst)` | Скопировать файл | Резервное копирование |
| `fs::rename(src, dst)` | Переместить/переименовать | Архивирование файлов |
| `fs::remove_file(path)` | Удалить файл | Очистка временных файлов |
| `fs::create_dir_all(path)` | Создать все папки в пути | Инициализация структуры хранилища |
| `fs::read_dir(path)` | Перечислить содержимое папки | Обход файлов данных |
| `fs::metadata(path)` | Получить метаданные файла | Проверка размера, времени изменения |
| `io::ErrorKind::NotFound` | Тип ошибки "не найдено" | Различение типов ошибок ввода-вывода |

## Домашнее задание

1. Создай функцию `append_trade(path: &str, ticker: &str, price: f64, qty: f64)`:
   - Если файл не существует — создай его с заголовком `ticker,price,qty`
   - Если файл существует — дозапиши новую строку
   - Используй `fs::read_to_string` и `fs::write`

2. Создай функцию `count_trades(path: &str) -> Result<usize, io::Error>`:
   - Прочитай файл и посчитай количество строк (не считая заголовок)
   - Верни 0 если файл не существует (обработай `NotFound`)

3. Создай функцию `backup_file(path: &str) -> Result<String, io::Error>`:
   - Копирует файл, добавляя суффикс `.bak`
   - Возвращает путь к резервной копии

4. Напиши функцию `find_csv_files(dir: &str) -> Result<Vec<String>, io::Error>`:
   - Возвращает список имён всех `.csv` файлов в директории
   - Сортирует список по алфавиту

5. Создай структуру `TradeLogger` с методами:
   - `new(dir: &str)` — создаёт папку если её нет
   - `log(&self, ticker: &str, msg: &str)` — дописывает в `{dir}/{ticker}.log`
   - `get_logs(&self, ticker: &str) -> Vec<String>` — все строки лога

## Навигация

[← Предыдущий день](../124-bufreader-bufwriter-efficient-io/ru.md) | [Следующий день →](../126-path-pathbuf-file-paths/ru.md)
