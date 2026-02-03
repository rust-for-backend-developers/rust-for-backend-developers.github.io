# Роутинг

В этой главе мы подробнее разберёмся с возможностями роутинга эндпоинтов.

### Фоллбэк (fallback)

Если путь из введённого пользователем URL не соответствует ни одному эндпоинту, зарегистрированному в роутере, то Axum по умолчанию просто ответит HTTP кодом 404.

Если вам необходимо обрабатывать такие запросы к несуществующим эндпоинтам "вручную", то можно задать фоллбэк (fallback) обработчик. Фоллбэк — это обычная функция-обработчик запросов, которая регистрируется в роутере при помощи метода [fallback](https://docs.rs/axum/latest/axum/struct.Router.html#method.fallback) и вызывается для всех запросов, для которых не был найден соответствующий эндпоинт.

В качестве примера сделаем фоллбэк обработчик, который просто отображает HTTP метод и URL запроса.

```rust,noplayground
use axum::{Router, http::request::Parts, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/hello", get(hello))
        .fallback(my_fallback);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}

async fn my_fallback(parts: Parts) -> String {
    format!("Method: {}\nURL: {}", parts.method, parts.uri)
}
```

После запуска программы, если перейти на [http://localhost:8080/hello](http://localhost:8080/hello), то мы увидим "Hello!", но попытка указать любой другой путь, например, [http://localhost:8080/non-existing-page?a=1\&b=2](http://localhost:8080/non-existing-page?a=1\&b=2) приведёт к ответу вида:

```
Method: GET
URL: /non-existing-page?a=1&b=2
```

### Слияние роутеров (merging) { #merging-routers }

Роутер эндпоинтов можно собирать из других под-роутеров. То есть мы можем определить одну часть эндпоинтов в одном объекте роутера, другую — в другом и потом "склеить" эти роутеры в главный роутер методом [merge](https://docs.rs/axum/latest/axum/struct.Router.html#method.merge).

Для примера создадим один роутер для эндпоинтов, связанных с данными пользователей, а другой роутер для эндпоинтов, связанных с товарами.

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let users_router = Router::new()
        .route("/users", get(list_users));

    let products_router = Router::new()
        .route("/products", get(list_products));

    let app = Router::new()
        .merge(users_router)
        .merge(products_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users() -> &'static str {
    "Users"
}

async fn list_products() -> &'static str {
    "Products"
}
```

Такое разнесение эндпоинтов по отдельным роутерам позволяет писать более модульный код. Но читабельность кода — не единственное преимущество, которое мы получаем, собирая роутер из под-роутеров. Этот подход так же позволяет задавать отдельные объекты состояния (`State`) для каждого из роутеров, обеспечивая тем самым изоляцию доступа к частям состояния приложения.

Модифицируем наш пример так, что для каждого из под-роутеров будет использоваться свой объект состояния.

```rust,noplayground
use std::sync::Arc;

use axum::{Router, extract::State, routing::get};

#[derive(Debug)]
struct UserState {}

#[derive(Debug)]
struct ProductState {}

#[tokio::main]
async fn main() {
    let users_router = Router::new()
        .route("/users", get(list_users))
        .with_state(Arc::new(UserState {}));

    let products_router = Router::new()
        .route("/products", get(list_products))
        .with_state(Arc::new(ProductState {}));

    let app = Router::new().merge(users_router).merge(products_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users(State(state): State<Arc<UserState>>) -> String {
    format!("Users endpoint. State: {state:?}")
}

async fn list_products(State(state): State<Arc<ProductState>>) -> String {
    format!("Products endpoint. State: {state:?}")
}
```

Если мы запустим приложение и перейдём по адресу [http://localhost:8080/users](http://localhost:8080/users), то мы увидим "Users endpoint. State: UserState".

Если перейти на [http://localhost:8080/products](http://localhost:8080/products), то должно отобразиться — "Products endpoint. State: ProductState".

Как видим, эндпоинты работают с тем объектом состояния, который был задан в их под-роутере. Но что случится, если мы добавим еще один объект состояния на уровень объединяющего роутера?

```rust,noplayground
#[derive(Debug)]
struct MainState {}

let app = Router::new()
    .merge(users_router)
    .merge(products_router)
    .with_state(Arc::new(MainState));
```

Не изменится ничего: обработчики используют тот объект состояния, который объявлен "ближе всего" к ним.

Однако если мы удалим объект состояния, объявленный на уровне `users_router`, и добавим объект состояния типа `UserState` на уровне главного роутера, тогда при обращении к [http://localhost:8080/users](http://localhost:8080/users) будет использоваться объект состояния из главного роутера.

### Встраивание под-роутеров (nesting)

Встраивание (nesting) подобно вышерассмотренному слиянию (merging) с тем отличием, что при встраивании пути эндпоинтов из встраиваемого роутера предваряются указанным префиксом.

Например, имеется роутер:

```rust,noplayground
let users_router = Router::new().route("/users", get(list_users));
```

Если его встроить в главный роутер так:

```rust,noplayground
let main_router = Router::new().nest("/api", users_router);
```

то функция-обработчик _list\_users_ будет срабатывать для запроса с URL http://localhost:8080/api/users.

Для чего это нужно? Это позволяет еще больше повысить модульность кода. Например, мы можем вынести некоторые эндпоинты в отдельную библиотеку и встроить их в роутер в другом приложении, не опасаясь получить конфликт путей с другими эндпоинтами.

Приведём пример. Создадим два роутера, содержащие эндпоинты с конфликтующими путями. Используя разные префиксы, встроим эти два роутера в главный роутер.

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let users_v1_router = Router::new().route("/users", get(list_users_v1));

    let users_v2_router = Router::new().route("/users", get(list_users_v2));

    let app = Router::new()
        .nest("/api/v1", users_v1_router)
        .nest("/api/v2", users_v2_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users_v1() -> &'static str {
    "Users endpoint Version 1"
}

async fn list_users_v2() -> &'static str {
    "Users endpoint Version 2"
}
```

При переходе на [http://localhost:8080/api/v1/users](http://localhost:8080/api/v1/users) отображается "Users endpoint Version 1", а при переходе на [http://localhost:8080/api/v2/users](http://localhost:8080/api/v2/users) — "Users endpoint Version 2".
