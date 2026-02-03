# Тестирование эндпоинтов

Экосистема Axum предлагает крэйт [axum-test](https://crates.io/crates/axum-test), который позволяет проводить полноценное тестирование эндпоинтов, минуя сетевую часть. Это очень удобно, так как позволяет запускать тесты быстрее и не переживать о коллизии портов.

Крэйт axum-test предоставляет обёртку [TestServer](https://docs.rs/axum-test/latest/axum_test/struct.TestServer.html), при помощи которой можно вызывать эндпоинты "напрямую":

```rust,noplayground
// Создаём роутер с эндпоинтами, которые будем тестировать
let app = Router::new()
    .route("/give-5", get(|| async { "5" }))

// Создаём тестовый сервер, который работает без сетевого слушателя
let server = TestServer::new(app).unwrap();

// Делаем запрос к эндпоинту "напрямую"
let response = server.get("/give-5").await;

// Проверяем результат запроса
response.assert_text("5");
```

---

Давайте добавим тест для нашего hello эндпоинта. При этом мы немного модифицируем код создания роутера, чтобы его было проще тестировать.

Для начала включим axum-test в `Cargo.toml`.

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"

[dev-dependencies]
axum-test = "18"
```

Функциональность из крэйта axum-test понадобится только в тестах, поэтому мы объявили её в секции зависимостей для тестов — `[dev-dependencies]`.

Теперь вынесем создание роутера в отдельную функцию, чтобы её можно было вызывать из теста, и допишем сам тест:

```rust,noplayground
use std::collections::HashMap;
use axum::{Router, extract::Query, routing::get};

#[tokio::main]
async fn main() {
    let app = setup_app();
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

// Создание роутера
fn setup_app() -> Router {
    Router::new()
        .route("/hello", get(hello))
}

async fn hello(Query(map):Query<HashMap<String, String>>) -> String {
    if let Some(name) = map.get("name") {
        format!("Hello, {name}!")
    } else {
        String::from("Hello!")
    }
}

// Этот модуль, содержащий тест, будет компилироваться только при запуске тестов.
#[cfg(test)]
mod test {
    use axum_test::TestServer;
    use axum::http::StatusCode;
    use super::*;

    #[tokio::test]
    async fn test_hello_endpoint() {
        let app = setup_app();
 
        let server = TestServer::new(app).unwrap();

        // Тестируем вызов без квери параметра
        let response1 = server.get("/hello").await;
        response1.assert_status(StatusCode::OK);
        response1.assert_text("Hello!");

        // Тестируем вызов с квери параметром
        let response2 = server.get("/hello?name=Stas").await;
        response2.assert_status(StatusCode::OK);
        response2.assert_text("Hello, Stas!");
    }
}
```

