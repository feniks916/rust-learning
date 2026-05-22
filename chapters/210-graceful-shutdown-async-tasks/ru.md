# День 210: Graceful shutdown асинхронных задач

## Аналогия из трейдинга

Перед выключением торгового робота нужно: закрыть все позиции, дождаться исполнения отложенных ордеров, сохранить состояние. Резкое завершение — это риск незакрытых позиций. Graceful shutdown — корректное завершение.

## Что значит graceful shutdown на практике

Это не просто «остановить программу». Корректное завершение обычно включает несколько шагов:

1. перестать принимать новые задачи
2. сообщить уже работающим задачам, что пора завершаться
3. дать им время закончить текущую работу
4. сохранить состояние
5. закрыть соединения и только потом выйти

Для торгового софта это особенно важно: потерянное состояние, незакрытый сокет или незафиксированный лог могут сильно усложнить восстановление после рестарта.

## CancellationToken

```rust
use tokio_util::sync::CancellationToken;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();

    let monitor_token = token.clone();
    let monitor = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = monitor_token.cancelled() => {
                    println!("Монитор: получен сигнал завершения");
                    // Сохраняем состояние
                    break;
                }
                _ = sleep(Duration::from_millis(500)) => {
                    println!("Монитор: проверяем цены...");
                }
            }
        }
        println!("Монитор остановлен корректно");
    });

    // Работаем 2 секунды
    sleep(Duration::from_secs(2)).await;

    println!("Отправляем сигнал завершения...");
    token.cancel();

    monitor.await.unwrap();
    println!("Все задачи завершены");
}
```

## Shutdown через канал

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    let mut rx1 = shutdown_tx.subscribe();
    let task1 = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = rx1.recv() => { println!("Задача 1: завершение"); break; }
                _ = sleep(Duration::from_millis(300)) => { println!("Задача 1: работает"); }
            }
        }
    });

    let mut rx2 = shutdown_tx.subscribe();
    let task2 = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = rx2.recv() => { println!("Задача 2: завершение"); break; }
                _ = sleep(Duration::from_millis(400)) => { println!("Задача 2: работает"); }
            }
        }
    });

    sleep(Duration::from_secs(1)).await;
    shutdown_tx.send(()).unwrap();

    let _ = tokio::join!(task1, task2);
    println!("Graceful shutdown завершён");
}
```

## `CancellationToken` vs канал

Оба подхода решают похожую задачу, но немного по-разному:

1. `CancellationToken` хорошо подходит, когда нужен простой сигнал «всем завершиться»
2. `broadcast::channel` удобен, если у тебя уже архитектура на каналах или нужны сообщения нескольким подписчикам

Практический ориентир такой:

1. простой общий стоп-сигнал — `CancellationToken`
2. более событийная координация — канал

## Почему `tokio::select!` здесь центральный инструмент

Graceful shutdown почти всегда выглядит как ожидание двух вещей одновременно:

1. либо пришла полезная работа
2. либо пришёл сигнал остановки

Именно это и делает `tokio::select!`:

```rust
tokio::select! {
    _ = token.cancelled() => {
        println!("Пора завершаться");
    }
    _ = sleep(Duration::from_millis(500)) => {
        println!("Делаем обычную работу");
    }
}
```

Так код остаётся отзывчивым к сигналу остановки.

## Практический пример: координатор с сохранением состояния

```rust
use tokio::time::{sleep, Duration};
use tokio_util::sync::CancellationToken;

async fn worker(name: &str, token: CancellationToken) {
    loop {
        tokio::select! {
            _ = token.cancelled() => {
                println!("{}: сохраняем состояние перед выходом", name);
                sleep(Duration::from_millis(100)).await;
                println!("{}: завершено", name);
                break;
            }
            _ = sleep(Duration::from_millis(250)) => {
                println!("{}: рабочий цикл", name);
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();

    let w1 = tokio::spawn(worker("market-data", token.clone()));
    let w2 = tokio::spawn(worker("risk-engine", token.clone()));

    sleep(Duration::from_secs(1)).await;
    println!("main: отправляем shutdown");
    token.cancel();

    let _ = tokio::join!(w1, w2);
    println!("main: все задачи закрыты");
}
```

Это уже хороший базовый шаблон для фоновых async-задач.

## Частые ошибки

### Ошибка 1: просто abort-ить задачу без финализации

Иногда это допустимо, но для сервисных задач и торговых ботов лучше сначала дать им корректно завершиться.

### Ошибка 2: послать сигнал остановки, но не дождаться `JoinHandle`

Если не `await`-ить задачи, программа может завершиться раньше их финальных действий.

### Ошибка 3: держать задачи в бесконечном цикле без ветки shutdown

Тогда остановить их корректно будет трудно.

## Что мы узнали

| Инструмент | Описание |
|------------|----------|
| `CancellationToken` | Отмена через токен |
| `broadcast::channel` | Сигнал всем задачам |
| `tokio::select!` | Ожидание нескольких событий |
| `tokio::join!` | Ожидание завершения всех |

## Домашнее задание

1. Реализуй 3 async задачи с graceful shutdown через CancellationToken
2. Добавь "финализацию" в каждой задаче (сохранение данных перед выходом)
3. Установи таймаут на завершение: если задача не остановилась за 5 сек — принудительно
4. Реализуй обработку Ctrl+C: `tokio::signal::ctrl_c().await`

## Навигация

[← Предыдущий день](../209-parallel-websocket-subscriptions/ru.md) | [Следующий день →](../211-debugging-async-code/ru.md)
