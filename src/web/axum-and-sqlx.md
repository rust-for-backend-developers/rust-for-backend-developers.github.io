# Axum и SQLx

Теперь, когда мы разобрались, как работать с SQLx, давайте посмотрим, как использовать его совместно с Axum.

Расширим наш test_sqlx проект (который мы создали в главе [SQLx](sqlx.md)), добавив в него Axum-сервер. Для простоты примера пускай наш сервер будет предоставлять только два эндпоинта:

* `GET /accounts` — получить список всех аккаунтов
* `POST /account` — создать новый аккаунт

Сначала добавим axum зависимость в `Cargo.toml`:

```toml
[package]
name = "test_sqlx"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
sqlx = {version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = { version = "0.4", features = ["serde"]}

axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Теперь создадим пустые файлы — заготовки под модули:

* `persist.rs` — для кода взаимодействия с БД
* `server.rs` — для кода HTTP сервера

В результате у нас должно получиться такое дерево файлов:

```
test_sqlx/
├── Cargo.toml
├── src/
│   ├── persist.rs
│   ├── server.rs
│   └── main.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```

Сначала напишем функциональность для работы с базой данных. У нас будет две функции:

* для выборки всех аккаунтов из таблицы `accounts`
* для вставки нового аккаунта

Итак, наш файл `src/persist.rs`:

```rust,noplayground
use bigdecimal::BigDecimal;
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool};

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct Account {
    id: i64,
    owner_name: String,
    balance: BigDecimal,
}

pub async fn fetch_accounts(db: &PgPool) -> Result<Vec<Account>, sqlx::Error> {
    sqlx::query_as("SELECT id, owner_name, balance FROM accounts")
        .fetch_all(db)
        .await
}

pub async fn create_accounts(
    db: &PgPool, owner_name: &str, initial_balance: BigDecimal
) -> Result<Account, sqlx::Error> {
    let mut tx = db.begin().await?;
    // Вставляем новый аккаунт
    sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
        .bind(owner_name)
        .bind(initial_balance)
        .execute(&mut *tx)
        .await?;
    // Вычитываем только что вставленную запись.
    // Функция currval(имя сиквенса), которая возвращает
    // последнее полученное значение сиквенса,
    // должна быть вызвана в той же сессии, что и INSERT запрос,
    // использовавший этот сиквенс. Поэтому мы используем транзакцию.
    let result = sqlx::query_as(r#"
            SELECT id, owner_name, balance
            FROM accounts
            WHERE id = currval('accounts_seq')
        "#)
        .fetch_one(&mut *tx)
        .await?;
    tx.commit().await?;
    Ok(result)
}
```

Теперь код нашего сервера. Для того чтобы эндпоинты могли использовать базу данных, мы просто поместим объект пула соединений с БД в объект состояния.

Таким образом файл `src/server.rs` будет иметь вид:

```rust,noplayground
use std::sync::Arc;

use axum::{Json, Router, extract::State, http::StatusCode, routing::{get, post}};
use serde::Deserialize;
use sqlx::{PgPool, postgres::PgPoolOptions, types::BigDecimal};
use crate::persist::{self, Account};

struct AppState {
    db: PgPool,
}

/// Создаёт и запускает Axum сервер
pub async fn run_server() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let state = AppState { db: pool };

    let app = Router::new()
        .route("/accounts", get(list_accounts))
        .route("/accounts", post(create_new_account))
        .with_state(Arc::new(state)); // Помещаем пул соединений к БД в состояние

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

#[derive(Deserialize)]
struct NewAcc {
    owner_name: String,
    init_balance: BigDecimal,
}

async fn list_accounts(
    state: State<Arc<AppState>>
) -> Result<Json<Vec<Account>>, (StatusCode, String)> {
    // Извлекаем объект пула соединений из состояния, и используем его для
    // вызова функции, которая вычитывает все аккаунты из БД.
    match persist::fetch_accounts(&state.db).await {
        Ok(accounts) => Ok(Json(accounts)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}

async fn create_new_account(
    state: State<Arc<AppState>>, Json(acc): Json<NewAcc>
) -> Result<Json<Account>, (StatusCode, String)> {
    match persist::create_accounts(&state.db, &acc.owner_name, acc.init_balance).await {
        Ok(account) => Ok(Json(account)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}
```

В главном файле `src/main.rs` мы просто вызовем функцию запуска сервера:

```rust,noplayground
mod persist;
mod server;

#[tokio::main]
async fn main() {
    server::run_server().await
}
```

Для чистоты эксперимента удалим из таблиц `transactions` и `accounts` все записи, которые остались от примеров из предыдущих глав.

Если вы использовали Docker, чтобы запускать сервер Postgres, то можете воспользоваться командами:

* `docker start my_pg_container` — запустить контейнер, если он неактивен
* `docker exec -it my_pg_container bash` — запустить консоль в контейнере
* `psql -U postgres` — подключиться к самой БД
* `\c mydb;` — выбрать _mydb_ как текущую базу данных
* `delete from transactions; delete from accounts;` — очистить таблицы

***

Теперь запустим наше приложение.

```
cargo run
```

Сначала вызовем эндпоинт, возвращающий все аккаунты. Так как мы только что удалили все записи из таблиц, то ответом должен быть пустой список:

```
$ curl -i http://localhost:8080/accounts
HTTP/1.1 200 OK
content-type: application/json
content-length: 2
date: Sat, 03 Jan 2026 02:02:18 GMT

[]
```

Теперь добавим новый аккаунт:

```
$ curl -i -X POST \
    -H "Content-Type: application/json" \
    -d '{"owner_name": "Acc-1", "init_balance": 1000.0}' \
    http://localhost:8080/accounts
</strong>HTTP/1.1 200 OK
content-type: application/json
content-length: 49
date: Sat, 03 Jan 2026 02:02:20 GMT

{"id":1000,"owner_name":"Acc-1","balance":"1000"}
```

Снова вызовем эндпоинт, возвращающий все аккаунты.

```
$ curl -i http://localhost:8080/accounts
HTTP/1.1 200 OK
content-type: application/json
content-length: 51
date: Sat, 03 Jan 2026 02:02:22 GMT

[{"id":1000,"owner_name":"Acc-1","balance":"1000"}]
```

Всё работает, как и ожидалось.

***

Как видите, объект состояния — именно то место, где предпочтительно хранить долгоживущие сущности, которые могут использоваться разными эндпоинтами.
