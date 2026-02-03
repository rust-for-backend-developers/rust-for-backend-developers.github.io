# Экстракторы

Экстрактор (Extractor) — механизм, позволяющий извлечь некое значение из (или на основе) HTTP-запроса и заинжектить его в функцию-обработчик запроса.

Мы уже знакомы с рядом стандартных экстракторов:

* экстрактор для инжекции аргументов пути — [Path](axum-basics.md#path-parameters)
* экстрактор для инжекции квери-параметров — [Query](axum-basics.md#query-parameters)
* экстрактор для инжекции тела запроса, переданного как JSON-документ — [Json](axum-basics.md#чтение-тела-запроса)

Теперь давайте разберёмся, как экстракторы устроены.

Чтобы создать экстрактор, нужно реализовать один из двух трэйтов: [FromRequestParts](https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html) или [FromRequest](https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html).

### FromRequestParts

Трэйт `FromRequestParts` объявлен так:

```rust,noplayground
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection>;
}
```

Как видим, через аргументы он имеет доступ к заголовочной части HTTP-запроса — [Parts](https://docs.rs/http/latest/http/request/struct.Parts.html) и к объекту состояния. Строить свой экстрактор на базе `FromRequestParts` следует в том случае, если экстрактору необходимо иметь доступ только к заголовочной части HTTP-запроса, а доступ к телу запроса не нужен.

***

Чтобы понять, как это работает, давайте напишем свою упрощённую версию экстрактора [Query](axum-basics.md#query-parameters), который инжектит квери-параметры в функцию-обработчик.

```rust,noplayground
use std::collections::HashMap;
use axum::{Router, extract::FromRequestParts, http::request::Parts, routing::get};

struct MyQueryParams(HashMap<String, String>);

impl<S: Send + Sync> FromRequestParts<S> for MyQueryParams {
    type Rejection = ();

    async fn from_request_parts(
        parts: &mut Parts, _state: &S
    ) -> Result<Self, Self::Rejection> {
        let mut params = HashMap::new();
        // Получает квери-строку вида key_1=val_1&key_2=val_2
        if let Some(query_string) = parts.uri.query() {
            // По разделителю &, разбиваем квери-строку на подстроки вида key=val
            for pair in query_string.split("&") {
                // вычленяем из строки key=value ключ и значение
                let mut kv = pair.split("=");
                if let Some(k) = kv.next() && let Some(v) = kv.next() {
                    params.insert(k.to_string(), v.to_string());
                }
            }
        }
        Ok(MyQueryParams(params))
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(hello));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello(MyQueryParams(map): MyQueryParams) -> String {
    format!("Query params: {map:?}")
}
```

Если запустить сервер (`cargo run`) и перейти по адресу [http://localhost:8080/hello?a=1\&b=2\&c=3](http://localhost:8080/hello?a=1\&b=2\&c=3), то мы должны увидеть:

```
Query params: {"b": "2", "c": "3", "a": "1"}
```

***

Теперь давайте рассмотрим довольно распространённый пример: экстрактор, который извлекает ID сессии из HTTP заголовка.

```rust,noplayground
use axum::{ Router, extract::FromRequestParts, routing::get };
use axum::http::{StatusCode, request::Parts};

// Экстрактор ID сессии
struct SessionId(String);

impl<S: Send + Sync> FromRequestParts<S> for SessionId {
    type Rejection = StatusCode;

    async fn from_request_parts(
        parts: &mut Parts, _state: &S
    ) -> Result<Self, Self::Rejection> {
        // ID сессии передаётся в заголовке sessionid
        if let Some(value) = parts.headers.get("sessionid") {
            if let Ok(string_value) = value.to_str() {
                Ok(SessionId(string_value.to_string()))
            } else {
                Err(StatusCode::BAD_REQUEST)
            }
        } else {
            Err(StatusCode::UNAUTHORIZED)
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(hello));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello(SessionId(session_id): SessionId) -> String {
    format!("Session ID: {session_id}")
}
```

Протестируем наш эндпоинт:

```
$ curl -i http://localhost:8080/hello
HTTP/1.1 401 Unauthorized
content-length: 0
date: Mon, 01 Dec 2025 22:20:47 GMT

$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 26
date: Mon, 01 Dec 2025 22:08:03 GMT

Session ID: 1111-1111-1111
```

***

С практической точки зрения, будет гораздо удобнее, если в обработчик будет инжектиться не ID сессии, а сразу объект сессии пользователя. Экстракторы имеют доступ не только к заголовку HTTP-запроса, но и к объекту состояния приложения, где мы будем хранить сессию. Давайте перепишем наш экстрактор так, чтобы он сначала извлекал из заголовков запроса ID сессии, а далее из состояния доставал уже сам объект сессии.

```rust,noplayground
use std::{collections::HashMap, sync::Arc};
use axum::{Router, extract::FromRequestParts, routing::get};
use axum::http::{StatusCode, request::Parts};
use tokio::sync::{Mutex, RwLock};

// Данные пользователя, хранимые в сессии
struct SessionData {
    user_name: String,
}

struct AppState {
    // ID сессии -> данные сессии
    sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>>,
}

// Экстрактор сессии. Хранит и ID сессии, и объект данных сессии
struct Session(String, Arc<Mutex<SessionData>>);

impl FromRequestParts<Arc<AppState>> for Session {
    type Rejection = StatusCode;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &Arc<AppState>,
    ) -> Result<Self, Self::Rejection> {
        if let Some(value) = parts.headers.get("sessionid") {
            if let Ok(string_value) = value.to_str() {
                let session_id = string_value.to_string();
                let read_guard = state.sessions.read().await;
                if let Some(session) = read_guard.get(&session_id) {
                    Ok(Session(session_id, session.clone()))
                } else {
                    Err(StatusCode::UNAUTHORIZED)
                }
            } else {
                Err(StatusCode::BAD_REQUEST)
            }
        } else {
            Err(StatusCode::UNAUTHORIZED)
        }
    }
}

#[tokio::main]
async fn main() {
    // Предподготовленное хранилище сессий для тестирования нашего экстрактора
    let sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>> = {
        let mut data = HashMap::new();
        // тестовая сессия
        data.insert(
            "1111-1111-1111".to_string(),
            Arc::new(Mutex::new(SessionData {user_name: "John Doe".to_string()})),
        );
        RwLock::new(data)
    };
    let app_state = AppState { sessions };

    let app = Router::new()
        .route("/hello", get(hello))
        .with_state(Arc::new(app_state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello(Session(id, session): Session) -> String {
    format!(
        "Session ID: {id}, User name: {}",
        session.lock().await.user_name
    )
}
```

Проверяем, что если передать ID несуществующей сессии, мы получим в ответ 401-й код.

```
$ curl -i -H "sessionid: 2222-2222-2222" http://localhost:8080/hello
HTTP/1.1 401 Unauthorized
content-length: 0
date: Mon, 01 Dec 2025 22:56:52 GMT
```

Теперь проверяем успешный сценарий:

```
$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 47
date: Mon, 01 Dec 2025 22:48:05 GMT

Session ID: 1111-1111-1111, User name: John Doe
```

### FromRequest

Трэйт `FromRequest` имеет вид:

```rust,noplayground
pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;

    async fn from_request(
        req: Request<Body>,
        state: &S,
    ) -> Result<Self, Self::Rejection>
}
```

Он имеет доступ к объекту [Request](https://docs.rs/http/latest/http/request/struct.Request.html), который инкапсулирует все данные запроса, включая тело. Экстрактор следует строить на основе `FromRequest` только в случае, если необходим доступ к телу запроса.

***

Как вы уже могли догадаться, типы `Json` и `Form<FormData>`, которые мы рассматривали в [разделе про чтение данных запроса](axum-basics.md#read-request-body),  реализуют `FromRequest`, что позволяет им инжектиться в функцию обработчик.

Чтобы понять, как работать с `FromRequest`, давайте напишем экстрактор, который будет иметь возможность десериализовать данные запроса, переданные и в JSON, и в XML формате.

Для начала добавим зависимость [serde-xml-rs](https://crates.io/crates/serde-xml-rs) в файл `Cargo.toml`:

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"

serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde-xml-rs = "0.8"
```

Теперь сама программа — `src/main.rs`:

```rust,noplayground
use std::io::Cursor;
use axum::{Router, body::{Body, Bytes, to_bytes}, http::StatusCode, routing::post};
use axum::extract::{FromRequest, Request};
use serde::{Deserialize, de::DeserializeOwned};

// Экстрактор для данных запроса
struct AnyFormat<D: DeserializeOwned>(D);

impl<S: Send + Sync, D: DeserializeOwned> FromRequest<S> for AnyFormat<D> {
    type Rejection = (StatusCode, &'static str);

    async fn from_request(
        request: Request<Body>, _state: &S
    ) -> Result<Self, Self::Rejection> {
        // Извлекаем из запроса заголовок запроса и его тело
        let (parts, body) = request.into_parts();
        let Some(content_type) = parts.headers.get("content-type") else {
            return Err((StatusCode::BAD_REQUEST, "Missing content-type"));
        };
        // Вычитываем байты запроса в буфер, максимальный размер которого 100MB
        let body_bytes: Bytes = to_bytes(body, 100 * 1024 * 1024).await.unwrap();

        let result = match content_type.to_str().unwrap() {
            "application/json" => match serde_json::from_slice::<D>(&body_bytes) {
                Ok(entity) => entity,
                Err(_) => return Err((StatusCode::BAD_REQUEST, "Malformed JSON")),
            },
            "application/xml" => {
                let cursor = Cursor::new(body_bytes);
                match serde_xml_rs::from_reader::<'_, D, _>(cursor) {
                    Ok(entity) => entity,
                    Err(_) => return Err((StatusCode::BAD_REQUEST, "Malformed XML")),
                }
            }
            _ => return Err((StatusCode::BAD_REQUEST, "Unsupported format")),
        };
        Ok(AnyFormat(result))
    }
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/users", post(create_user));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn create_user(AnyFormat(req): AnyFormat<CreateUserRequest>) -> String {
    format!("Received: {req:?}")
}
```

Теперь, запустив сервер, мы можем на один и тот же эндпоинт http://localhost:8080/users сделать запрос, отправив тело и в формате JSON:

```
$ curl -i -X POST -H "content-type: application/json" -d '{"name":"John Doe"}' \
    http://localhost:8080/users
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 48
date: Tue, 02 Dec 2025 00:31:16 GMT

Received: CreateUserRequest { name: "John Doe" }
```

и в формате XML:

```
$ curl -i -X POST -H "content-type: application/xml" \
    -d '<CreateUserRequest><name>John Doe</name></CreateUserRequest>' \
    http://localhost:8080/users
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 48
date: Tue, 02 Dec 2025 00:56:08 GMT

Received: CreateUserRequest { name: "John Doe" }
```
