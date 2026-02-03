# Завершение работы сервера

Мы разобрались, как создавать HTTP сервер, теперь давайте разберёмся, как его правильно выключать.

## graceful shutdown

По умолчанию, если программа получает **сигнал** о необходимости завершения, она сразу же прекращает свою работу. При этом, если в программе был запущен Axum сервер, то обработка запросов, выполнявшихся в этот момент, прерывается.

> [!NOTE]
> В данной ситуации речь идёт о сигналах, отправляемых операционной системой в качестве реакции на действие, подразумевающее закрытие программы. Например в Unix-подобных системах приложению отправляется сигнал:
> 
> * `SIGTERM`, если родительский процесс был завершён
> * `SIGINT`, если в консоли, где запущена программа, нажали Ctrl+C.
> 
> Оба этих сигнала по умолчанию приводят к немедленному выключению программы.\
> (Также любой сигнал можно отправлять программно, или при помощи утилиты [kill](https://man7.org/linux/man-pages/man1/kill.1.html))

Если сервер содержит функциональность, которую нежелательно прерывать до её завершения, то применяют так называемый graceful shutdown — корректное выключение. Этот подход подразумевает, что сервер выключается не сразу, а ждёт завершения всех уже обрабатываемых запросов.

Для корректного выключения применяется метод [with_graceful_shutdown](https://docs.rs/axum/latest/axum/serve/struct.Serve.html#method.with_graceful_shutdown), который используется следующим образом:

```rust,noplayground
axum::serve(listener, app)
    .with_graceful_shutdown(шатдаун_фьючер)
    .await
    .unwrap();
```

Как это работает? Если мы добавляем к нашему серверу вызов `.with_graceful_shutdown()`, то это меняет поведение сервера так, что сразу после того, как сервер стартует свою работу, он начинает ждать завершения шатдаун фьючера. Шатдаун фьючер — это любой объект, чей тип реализует уже знакомый нам трэйт `Future`. После завершения этого шатдаун фьючера Axum начнёт выключаться следующим образом:

1. Сразу остановит приём новых запросов по сети
2. Дождётся завершения всех уже выполняемых запросов
3. Завершит работу сервера

Сперва нам надо разобраться, как работает этот шатдаун фьючер. Для этого рассмотрим следующий пример:

```rust,noplayground
use axum::{Router, routing::get};
use std::time::Duration;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(async || "Hello!"));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(tokio::time::sleep(Duration::from_secs(5)))
        .await
        .unwrap();
}
```

Если мы запустим это приложение (`cargo run`), то оно проработает 5 секунд, а после завершится. Почему так? Как мы сказали, добавление вызова `.with_graceful_shutdown(шатдаун_фьючер)` приводит к тому, что после завершения шатдаун фьючера Axum дождётся завершения всех обрабатываемых запросов и выключится. В примере выше в качестве шатдаун фьючера мы использовали результат вызова функции [sleep](https://docs.rs/tokio/latest/tokio/time/fn.sleep.html) из библиотеки Tokio. Эта функция возвращает фьючер, который завершается по истечении указанного временного интервала. В нашем случае — 5 секунд.

Этот пример наглядно показывает, что в качестве сигнала к завершению можно использовать любой фьючер, что даёт нам некую свободу в реализации механизма завершения приложения.

> [!IMPORTANT]
> Когда мы устанавливаем шатдаун фьючер вызовом `with_graceful_shutdown`, мы не перезатираем немедленное завершение программы при получении `SIGTERM` или `SIGINT`. Мы задаём дополнительный выключатель, которым можем управлять программно.

## Сигналы

Как мы уже сказали, по умолчанию, когда Axum приложение получает сигнал `SIGTERM` или `SIGINT`, срабатывает стандартный обработчик, который сразу завершает всё приложение. При этом graceful shutdown не происходит, что является для нас нежелательным.

Если мы хотим, чтобы graceful shutdown работал и при получении `SIGTERM` и `SIGINT`, то нам необходимо создать свой шатдаун фьючер, который завершается при получении сигнала `SIGTERM` или `SIGINT`, тем самым инициируя корректное выключение.

***

Для перехвата `SIGINT` библиотека Tokio предлагает функцию [ctrl_c](https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html). Эта функция перезатирает стандартный обработчик `SIGINT`, что позволит избежать поведения по умолчанию в виде немедленного завершения приложения.

> [!TIP]
> `SIGINT` — сигнал, специфичный для UNIX-подобных операционных систем, однако в Windows существует его аналог — событие `CTRL_C_EVENT`, которое работает по тем же правилам. При работе в Windows функция `ctrl_c` будет перехватывать `CTRL_C_EVENT`.

Можете попробовать запустить такую программу:

```rust,noplayground
#[tokio::main]
async fn main() {
    tokio::signal::ctrl_c().await.unwrap();
    println!("After Ctrl+C had been pressed");
}
```

После нажатия Ctrl+C вы увидите в консоли текст "After Ctrl+C had been pressed". Однако если вы запустите такой код:

```rust,noplayground
#[tokio::main]
async fn main() {
    tokio::time::sleep(std::time::Duration::from_hours(1)).await;
    println!("After Ctrl+C had been pressed");
}
```

и нажмёте Ctrl+C, то не увидите ничего. Это доказывает, что установка обработчика для `SIGINT` путём вызова функции `ctrl_c` перезатирает стандартный обработчик для `SIGINT`.

***

Второй сигнал, который нам надо перехватывать — `SIGTERM`. Этот сигнал обычно отправляется программе либо когда завершается операционная система, либо когда завершается родительский процесс программы.

> [!TIP]
> С точки зрения разработки бэкенд приложений надо отметить, что когда kubernetes начинает выключать pod, то именно `SIGTERM` отправляется программе, запущенной в контейнере. При этом по умолчанию kubernetes даёт программе 30 секунд на самостоятельное завершение, а после высылает `SIGKILL`, который сразу завершит программу на уровне ядра операционной системы.

В отличие от сигнала `SIGINT`, для `SIGTERM` в Windows отсутствует аналог, поэтому код для перехвата обоих сигналов обычно имеет вид:

```rust,noplayground
use tokio::signal;

#[tokio::main]
async fn main() {
    let ctrl_c = signal::ctrl_c();

    // SIGTERM доступен только на UNIX-подобных системах
    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Cannot create SIGTERM handler")
            .recv()
            .await;
    };
    
    // На не UNIX-подобных системах ожидание SIGTERM заменяем на бесконечное ожидание
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();
    
    // Ожидаем либо нажатие Ctrl+C, либо SIGTERM
    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
    println!("After Ctrl+C or SIGTERM");
}
```

Здесь мы используем макрос `select`, рассмотренный в главе про [Tokio](../async/tokio.md#channels).

***

Теперь давайте реализуем корректное выключение сервера при нажатии Ctrl+C в консоли (пока без `SIGTERM`, чтобы код был менее громоздким):

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(async || "Hello!"));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c().await.unwrap()
}
```

Эта версия сервера корректно выключается по нажатию Ctrl+C, при этом перед полным выключением сервера Axum дождётся окончания выполнения обработки всех HTTP запросов.

В том, что сервер дождётся завершения всех запросов, легко убедиться. Давайте модифицируем наш hello эндпоинт так, чтобы он никогда не завершался:

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new().route(
        "/hello",
        get(async || {
            std::future::pending::<()>().await; // Никогда не завершится
            "Hello!"
        }),
    );
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c().await.unwrap()
}
```

Теперь, если после запуска сервера мы перейдём на [ http://localhost:8080/hello](http://localhost:8080/hello), то мы инициируем обработку запроса, которая никогда не закончится. Поэтому, если после этого мы нажмём в консоли с запущенным сервером Ctrl+C, то мы инициируем корректное завершение, которое просто зависнет в ожидании окончания обработки hello запроса.

Решение проблемы зависания очевидно: нужно добавить к запросам таймаут. Из прошлой главы мы знаем, что существует стандартный Tower мидлваре, реализующий таймауты — [TimeoutLayer](https://docs.rs/tower-http/latest/tower_http/timeout/struct.TimeoutLayer.html).

Окончательный пример с таймаутом и обработкой `SIGINT` и `SIGTERM`:

```rust,noplayground
use axum::Router;
use tokio::signal;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route(
            "/hello",
            get(async || {
                std::future::pending::<()>().await; // Никогда не завершится
                "Hello!"
            }),
        )
        .layer(TimeoutLayer::with_status_code(
            StatusCode::REQUEST_TIMEOUT,
            Duration::from_secs(10),
        ));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = signal::ctrl_c();

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Cannot create SIGTERM handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

## Остановка фоновых задач

Напоследок, давайте разберём сценарий, когда кроме HTTP сервера у нас также имеется фоновый поток (Tokio таск), который по каналу получает сообщения, отправленные из обработчика HTTP запроса. В рамках корректного отключения нам сначала нужно прекратить приём новых HTTP запросов, далее обработать оставшиеся в канале сообщения и только потом завершить фоновый поток.

Для примера напишем простой сервер с одним эндпоинтом, который через аргумент пути получает слово и отправляет это слово в канал. Фоновая задача будет просто считывать сообщения из канала и печатать их на консоль.

```rust,noplayground
use axum::{Router, http::StatusCode, routing::get};
use axum::extract::{Path, State};
use std::time::Duration;
use tokio::{signal, sync::{broadcast, mpsc}};

#[tokio::main]
async fn main() {
    // Канал, для оповещения фоновых задач о необходимости завершиться
    let (shutdown_snd, mut shutdown_rcv) = broadcast::channel::<()>(1);
    // Канал, по которому эндпоинт передаёт сообщения фоновой задаче
    let (word_snd, mut word_rcv) = mpsc::unbounded_channel::<String>();

    // Стартуем фоновую задачу, которая принимает по каналу сообщения
    // и обрабатывает их.
    // Для простоты она будет принимать строковые значения и просто печатать их.
    let bg_job = tokio::spawn({
        async move {
            // Цикл обработки сообщений
            loop {
                tokio::select! {
                    // Если получено сообщение, то обрабатываем его
                    word_resp = word_rcv.recv() => {
                        if let Some(word) = word_resp {
                            process_msg(word).await;
                        }
                    }
                    // Если получен сигнал завершаться, то выходим из цикла
                    _ = shutdown_rcv.recv() => {
                        println!("> Worker received shutdown command");
                        break;
                    }
                }
            }
            // Обрабатываем остатки сообщений в канале
            while let Ok(word) = word_rcv.try_recv() {
                process_msg(word).await;
            }
            println!("> Worker is finished");
        }
    });

    let app = Router::new()
        .route("/enqueue/{word}", get(handle_request))
        .with_state(word_snd);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // Извещаем фоновый процесс о необходимости завершиться
    println!("> Sending shutdown command to workers");
    let _ = shutdown_snd.send(());

    let _ = bg_job.await;
}

// Эндпоинт, который получает слово из URL пути и отправляет это слово в канал.
// Далее фоновой задачей, это слово будет считано из канала и обработано.
async fn handle_request(
    Path(word): Path<String>,
    State(word_snd): State<mpsc::UnboundedSender<String>>,
) -> StatusCode {
    match word_snd.send(word) {
        Ok(_) => StatusCode::CREATED,
        Err(_) => StatusCode::INTERNAL_SERVER_ERROR,
    }
}

// Эмуляция бурной деятельности по обработке слова
async fn process_msg(w: String) {
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("Word: {w}");
}

async fn shutdown_signal() {
    let ctrl_c = signal::ctrl_c();

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Cannot create SIGTERM handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

Запустив сервер, мы можем перейти на [http://localhost:8080/enqueue/hello](http://localhost:8080/enqueue/hello), после чего в консоли будет напечатано "Word: hello".

После нажатия Ctrl+C в консоли с запущенным сервером приложения корректно завершится.
