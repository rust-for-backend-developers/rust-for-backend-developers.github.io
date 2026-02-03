# Состояние бекенда

В прошлой главе мы научились создавать эндпоинты, которые работают только со значениями, полученными из запроса. Однако обычно в реальных бекенд приложениях существует некое состояние самого приложения, которое может содержать конфигурацию, пул соединений с базой данных, HTTP и GRPC клиенты для общения с другими сервисами, данные метрик, кеш и т.д.

Для работы с таким состоянием бэкенда в Axum существует обёртка [State](https://docs.rs/axum/latest/axum/extract/struct.State.html). Принцип работы следующий:

1\) Мы создаём объект произвольной структуры, и добавляем его в конфигурацию роутера при помощи метода [with_state](https://docs.rs/axum/latest/axum/struct.Router.html#method.with_state).

```rust,noplayground
struct AppState {
  // поля
}

#[tokio::main]
async fn main() {
    let my_app_data = Arc::new(AppState { ... });

    let app = Router::new()
        .route("/чего-то", get(обработчик))
        .with_state(my_app_data);
    ...
}
```

2\) Далее, при помощи обёртки `State` этот объект можно будет заинжектить в любую функцию-обработчик, зарегистрированную в этом роутере.

```rust,noplayground
async fn обработчик(State(my_app_data): State<Arc<AppState>>) -> Response { ... } 
```

Для примера создадим простой бекенд с эндпоинтом счётчиком. Каждый раз, когда мы будем делать GET запрос на http://localhost:8080/count, значение счётчика будет инкрементироваться, и в ответ мы будем получать строку, содержащую новое значение счётчика.

```rust,noplayground
use std::sync::{ Arc, atomic::{AtomicU64, Ordering} };
use axum::{Router, extract::State, routing::get};

// Эта структура будет определять состояние нашего бэкенда
struct AppState {
    counter: AtomicU64,
}

#[tokio::main]
async fn main() {
    let shared_state = Arc::new(AppState {
        counter: AtomicU64::new(0),
    });

    let app = Router::new()
        .route("/count", get(tick_counter))
        .with_state(shared_state);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn tick_counter(State(state): State<Arc<AppState>>) -> String {
    let prev_value = state.counter.fetch_add(1, Ordering::Relaxed);
    format!("New value: {}", prev_value + 1)
}
```

Первый переход на [http://localhost:8080/count](http://localhost:8080/count) должен отобразить "New value: 1", второй "New value: 2" и т.д.

***

У вас может возникнуть логичный вопрос: а почему бы просто не использовать глобальную переменную counter? Использование `State` имеет несколько преимуществ:

* Это нагляднее, так как сразу понятно, что данные относятся к состоянию приложения
* Это облегчает тестирование эндпоинтов
* В Axum приложение может иметь несколько роутеров, и для каждого из них можно указать свой объект состояния, что мы увидим в [следующей главе](routing.md#merging-routers)
