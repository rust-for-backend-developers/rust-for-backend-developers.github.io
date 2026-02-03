# Мидлваре

Мидлваре (middleware) — функциональность, которая вызывается перед функцией-обработчиком, а также имеет возможность обработать результат функции обработчика после её завершения. Можно сказать, что мидлваре оборачивает вызов функции-обработчика запроса.

<pre class="ascii-diagram">
╭────╮     ┌──────┐ request ┌──────────┐ request ┌──────────┐
│    ├────▶│      ├────────▶│          ├────────▶│          │
│Сеть│ TCP │ Axum │         │middleware│         │обработчик│
│    │◀────┤роутер│◀────────┤          │◀────────┤ запроса  │
╰────╯     └──────┘ response└──────────┘ response└──────────┘
</pre>

Если мы хотим выполнять какие-то действия перед обработкой любого запроса, то мидлваре — как раз тот механизм, который позволит нам это сделать.

При этом мы можем создать несколько мидлваре, которые будут цепочкой выполняться перед обработчиком запроса и после него.

<pre class="ascii-diagram">
    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
───▶│          ├───▶│          ├───▶│          ├───▶│          ├───▶
    │middleware│    │middleware│    │middleware│    │middleware│
◀───┤    1     │◀───┤    2     │◀───┤    3     │◀───┤    4     │◀───
    └──────────┘    └──────────┘    └──────────┘    └──────────┘
</pre>

По диаграмме сверху видно, что объекты мидлваре выстраиваются подобно слоям, оборачивающим обработчик запроса. Именно поэтому зарегистрированное в роутере мидлваре называют слоем (layer).

## Простейшее мидлваре

Самый простой вариант мидлваре выглядит как просто функция, которая принимает объект [Request](https://docs.rs/axum/latest/axum/extract/type.Request.html) — HTTP запрос и объект [Next](https://docs.rs/axum/latest/axum/middleware/struct.Next.html) — ссылка на следующий в цепочке мидлваре или обработчик запроса и возвращает объект `Response`, полученный от вызова обработчика запроса.

```rust,noplayground
async fn мидлваре(request: Request, next: Next) -> Response {
    // ... действия перед обработкой запроса
    
    // вызываем следующий в цепочке обработчик
    let response = next.run(request).await;
    
    // ... действия после обработки запроса

    // возвращаем результат из мидлваре
    response
}
```

Такую функцию надо:

* превратить в мидлваре при помощи функции [from_fn](https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html)
* зарегистрировать в роутере при помощи метода [layer](https://docs.rs/axum/latest/axum/struct.Router.html#method.layer)

После этого мидлваре начнёт отрабатывать для всех эндпоинтов, зарегистрированных в этом роутере.

```rust,noplayground
let app = Router::new()
    .route("/путь", get(функция-обработчик))
    .layer(middleware::from_fn(мидлваре));
```

Чтобы понять, как это работает, напишем классический пример мидлваре — мидлваре, который логирует время работы обработчика:

```rust,noplayground
use axum::{Router, extract::Request, response::Response, routing::get};
use axum::middleware::{self, Next};
use tokio::time::Instant;

async fn log_exec_time(request: Request, next: Next) -> Response {
    let start = Instant::now();
    let response = next.run(request).await;
    tracing::info!("Request took: {} micros", start.elapsed().as_micros());
    response
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/hello", get(hello))
        .layer(middleware::from_fn(log_exec_time));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

Запустим сервер:

```
$ cargo run
```

и перейдём в браузере на [http://localhost:8080/hello](http://localhost:8080/hello).

В консоли должна появиться лог-запись с уровнем `INFO` от приложения test_axum.

```
2025-12-02T23:09:30.342625Z  INFO test_axum: Request took: 52 micros
```

## Слои

Теперь давайте рассмотрим, как взаимодействуют между собой разные мидлваре.

Добавим в наш hello сервер два мидлваре, каждый из которых:

* печатает в лог одно сообщение перед вызовом следующего обработчика по цепочке
* печатает в лог другое сообщение после того, как следующий по цепочке обработчик закончил свою работу

```rust,noplayground
use axum::middleware::{Next, from_fn};
use axum::{Router, extract::Request, routing::get};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/hello", get(hello))
        .layer(from_fn(async |request: Request, next: Next| {
            tracing::info!("Middleware-1: before call");
            let response = next.run(request).await;
            tracing::info!("Middleware-1: after call");
            response
        }))
        .layer(from_fn(async |request: Request, next: Next| {
            tracing::info!("Middleware-2: before call");
            let response = next.run(request).await;
            tracing::info!("Middleware-2: after call");
            response
        }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    tracing::info!("Request handler");
    "Hello!"
}
```

Запустим сервер и перейдём по адресу [http://localhost:8080/hello](http://localhost:8080/hello)

```
stas@slim:~/dev/proj/rust/test_axum$ cargo run
   Compiling test_axum v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.16s
     Running `target/debug/test_axum`

2025-12-03T22:36:42.563720Z  INFO test_axum: Middleware-2: before call
2025-12-03T22:36:42.563776Z  INFO test_axum: Middleware-1: before call
2025-12-03T22:36:42.563800Z  INFO test_axum: Request handler
2025-12-03T22:36:42.563823Z  INFO test_axum: Middleware-1: after call
2025-12-03T22:36:42.563834Z  INFO test_axum: Middleware-2: after call
```

Как видите, мидлваре оборачивают друг друга подобно матрёшке. При этом тот мидлваре, который был добавлен в роутер последним, становится первым в цепочке.

## Работа со стэйтом

Нередко для логики работы мидлваре может понадобиться доступ к данным, хранящимся в состоянии. В этом случае функция-мидлваре будет иметь дополнительный аргумент — `State`.

```rust,noplayground
async fn мидлваре(state: State<Стэйт>, request: Request, next: Next) -> Response {
    // Действия перед обработкой запроса
    let response = next.run(request).await;
    // действия после обработки запроса
    response
}
```

Такая функция превращается в мидлваре при помощи функции [from_fn_with_state](https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html).

```rust,noplayground
let app = Router::new()
    .route("/путь", get(функция-обработчик))
    .with_state(state.clone())
    .layer(middleware::from_fn_with_state(state, мидлваре));
```

Как вы могли заметить, состояние приходится передавать отдельно и в `with_state`, и в `from_fn_with_state`. С одной стороны, это не очень удобно, но, с другой стороны, позволяет иметь разные объекты состояния для мидлвари и для роутера.

В качестве примера напишем мидлваре, который извлекает из HTTP заголовка ID сессии, потом достаёт из состояния соответствующий объект сессии пользователя и помещает его в [task-local переменную](../async/tokio.md#task-local). Далее эта task-local переменная используется из функции-обработчика запроса.

```rust,noplayground
use std::{collections::HashMap, sync::Arc};
use axum::{Router, extract::{Request, State}, http::StatusCode, routing::get};
use axum::middleware::{self, Next};
use axum::response::{IntoResponse, Response};
use tokio::sync::{Mutex, RwLock};

tokio::task_local! {
    pub static SESSION: Arc<Mutex<SessionData>>;
}

// Сессия пользователям
struct SessionData {
    user_name: String,
}

// Состояние бекенда, хранящее сессии
struct AppState {
    sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>>,
}

async fn set_session_for_request(
    State(state): State<Arc<AppState>>, request: Request, next: Next
) -> Response {
    // получаем объект сессии
    let session = if let Some(value) = request.headers().get("sessionid") {
        if let Ok(string_value) = value.to_str() {
            let session_id = string_value.to_string();
            let read_guard = state.sessions.read().await;
            if let Some(session) = read_guard.get(&session_id) {
                session.clone()
            } else {
                return StatusCode::UNAUTHORIZED.into_response();
            }
        } else {
            return StatusCode::BAD_REQUEST.into_response();
        }
    } else {
        return StatusCode::UNAUTHORIZED.into_response();
    };

    // Записываем объект сессии в task-local переменную
    let response = SESSION.scope(session, async {
            next.run(request).await // вызываем обработчик по цепочке
        }).await;
    response
}

#[tokio::main]
async fn main() {
    let sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>> = {
        let mut data = HashMap::new();
        data.insert( // Создаём тестовую сессию пользователя
            "1111-1111-1111".to_string(),
            Arc::new(Mutex::new(SessionData {
                user_name: "John Doe".to_string(),
            })),
        );
        RwLock::new(data)
    };
    let app_state = Arc::new(AppState { sessions });

    let app = Router::new()
        .route("/hello", get(hello))
        .with_state(app_state.clone())
        .layer(middleware::from_fn_with_state(app_state, set_session_for_request));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> String {
    // Извлекаем сессию из task-local переменной
    let session = SESSION.with(|session| session.clone());
    // используем сессию
    format!("Hello, {}!", session.lock().await.user_name)
}
```

Протестируем наш эндпоинт, чтобы убедиться, что мидлваре корректно помещает сессию в task-local переменную, доступную из функции-обработчика.

```
$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello

HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 15
date: Wed, 03 Dec 2025 14:03:22 GMT

Hello, John Doe!
```

## Стандартные мидлваре

Экосистема Axum предоставляет ряд стандартных мидлваре для таких вещей, как CORS, компрессия, таймауты и т.д., но перед тем как мы посмотрим на то, как ими пользоваться, мы должны познакомиться с библиотекой [Tower](tower.md).
