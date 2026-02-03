# Структура бекенда
К этому моменту мы уже умеем создавать HTTP сервер, работать с базой данных и знаем, как тестировать эндопоинты и функциональность, работающую с БД. Давайте теперь рассмотрим, как всё это обычно компонуют в реальных проектах.

## Создание проекта

Мы создадим многомодульный (workspace) проект, состоящий из следующих крэйтов:

* `persist` — Слой работы с базой данных: библиотека с функциями для работы с БД.
* `server` — Слой веб сервера: исполняемое приложение, которое для работы с хранилищем импортирует библиотеку `persist`.

1\) В удобном для вас месте создайте новую директорию `test_backend`.

2\) В директории `test_backend/` создайте файл `Cargo.toml` со следующим содержимым:

```toml
[workspace]
resolver = "3"
```

Корневой workspace готов. Теперь можно добавлять дочерние крэйты.

3\) Откройте консоль в директории `test_backend/` и создайте модули `persist` и `server`:

```
cargo new persist --lib
cargo new server
```

После этого корневой `Cargo.toml` должен иметь содержимое:

```toml
[workspace]
resolver = "3"
members = ["persist", "server"]
```

4\) В корневом `Cargo.toml` объявите зависимости, которые мы будем использовать в дочерних крэйтах:

```toml
[workspace]
members = ["server", "persist"]

[workspace.dependencies]
async-trait = "0.1"
thiserror = "1"
tokio = { version = "1", features = ["full"] }
sqlx = {version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = { version = "0.4", features = ["serde"]}

axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

axum-test = "18"
testcontainers = "0.26"
testcontainers-modules = { version = "0.14", features = ["postgres"] }
```

Чтобы вам проще было ориентироваться, финальная структура проекта будет такой:

```
test_backend/
├── Cargo.toml
├── persist
│    ├── Cargo.toml
│    ├── src/
│    │   └── lib.rs
│    └── tests/
│        └── account_storage_tests.rs
├── server
│    ├── Cargo.toml
│    ├── src/
│    │   ├── service.rs
│    │   ├── endpoints.rs
│    │   ├── lib.rs    
│    │   └── main.rs
│    └── tests/
│        └── endpoints_tests.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```



## Крэйт persist

Сначала добавим все необходимые зависимости в `persist/Cargo.toml`:

```toml
[package]
name = "persist"
version = "0.1.0"
edition = "2024"

[dependencies]
async-trait = { workspace = true }
sqlx = { workspace = true }
chrono = { workspace = true }
bigdecimal = { workspace = true }
serde = { workspace = true }

[dev-dependencies]
tokio = { workspace = true }
testcontainers = { workspace = true }
testcontainers-modules = { workspace = true }
```

Весь код крэйта будет в файле `persist/src/lib.rs`:

```rust,noplayground
use bigdecimal::BigDecimal;
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool, postgres::PgPoolOptions};

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct Account {
    pub id: i64,
    pub owner_name: String,
    pub balance: BigDecimal,
}

// Интерфейс для работы с хранилищем
#[async_trait::async_trait]
pub trait Storage {
    async fn fetch_accounts(&self) -> Result<Vec<Account>, sqlx::Error>;
    async fn create_accounts(
        &self, owner_name: &str, initial_balance: BigDecimal
    ) -> Result<Account, sqlx::Error>;
}

pub struct StorageImpl {
    db: PgPool,
}

impl StorageImpl {
    pub fn new(db: PgPool) -> StorageImpl {
        StorageImpl { db }
    }
    pub async fn from_connection_url(url: &str) -> Result<StorageImpl, sqlx::Error> {
        let pool = PgPoolOptions::new()
            .connect(url).await?;
        Ok(StorageImpl { db: pool })
    }
}

#[async_trait::async_trait]
impl Storage for StorageImpl {
    async fn fetch_accounts(&self) -> Result<Vec<Account>, sqlx::Error> {
        sqlx::query_as("SELECT id, owner_name, balance FROM accounts")
            .fetch_all(&self.db)
            .await
    }

    async fn create_accounts(
        &self, owner_name: &str, initial_balance: BigDecimal
    ) -> Result<Account, sqlx::Error> {
        let mut tx = self.db.begin().await?;
        sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
            .bind(owner_name)
            .bind(initial_balance)
            .execute(&mut *tx)
            .await?;
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
}
```

Добавим интеграционный тест для функциональности, работающей с базой данных — `persist/tests/account_storage_tests.rs`:

```rust,noplayground
use bigdecimal::{BigDecimal, FromPrimitive};
use persist::{Storage, StorageImpl};
use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
use testcontainers::runners::AsyncRunner;

#[tokio::test]
async fn test_create_and_fetch_account() {
    let container = testcontainers_modules::postgres::Postgres::default()
        .with_password("1111")
        .start().await.unwrap();

    let connection_options = PgConnectOptions::new()
        .host(&container.get_host().await.unwrap().to_string())
        .port(container.get_host_port_ipv4(5432).await.unwrap())
        .database("postgres")
        .username("postgres")
        .password("1111");
    let pool = PgPoolOptions::new()
        .connect_with(connection_options).await.unwrap();

    Migrator::new(std::path::Path::new("../migrations")).await.unwrap()
        .run(&pool).await.unwrap();

    // Тестируемый объект хранилища
    let sut = StorageImpl::new(pool);

    // Изначально таблица с аккаунтами пуста
    let accounts = sut.fetch_accounts().await.unwrap();
    assert!(accounts.is_empty());

    // Создаём новый аккаунт
    let created_acc = sut.create_accounts(
            "Test-Account-1", BigDecimal::from_f64(1000.0).unwrap()
        ).await
        .unwrap();

    // Выбираем все аккаунты, чтобы убедиться, что свежесозданный аккаунт присутствует
    let accounts = sut.fetch_accounts().await.unwrap();
    assert_eq!(accounts.len(), 1);
    assert_eq!(accounts[0].id, created_acc.id);
    assert_eq!(accounts[0].owner_name, created_acc.owner_name);
    assert_eq!(accounts[0].balance, created_acc.balance);
}
```

> [!NOTE]
> Как вы могли заметить, переменная, которая содержит тестируемый объект типа `StorageImpl`, называется **sut**. Аббревиатура SUT расшифровывается как System Under Test — тестируемая система. Использование имени `sut` для тестируемой сущности, облегчает чтение кода теста.

## Крэйт server

Теперь крэйт сервера, который использует крэйт `persist` в качестве зависимости. Схему взаимодействия модулей можно проиллюстрировать следующий образом:

<pre class="ascii-diagram" >
┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐   ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
┆ server crate               ┆   ┆ persist crate ┆
┆┌───────────┐   ┌──────────┐┆   ┆┌──────────┐   ┆
┆│ endpoints ├──>│ services ├────>│  lib.rs  │   ┆
┆└───────────┘   └──────────┘┆   ┆└──────────┘   ┆
└╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘   └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘
</pre>


Добавим всё необходимое в `server/Cargo.toml`:

```toml
[package]
name = "server"
version = "0.1.0"
edition = "2024"

[dependencies]
persist = { path = "../persist" }
thiserror = { workspace = true }
async-trait = { workspace = true }
tokio = { workspace = true }
axum = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
sqlx = { workspace = true }

[dev-dependencies]
bigdecimal = { workspace = true }
axum-test = { workspace = true }
testcontainers = { workspace = true }
testcontainers-modules = { workspace = true }
```

В крэйте `server` у нас следующие файлы с кодом:

* `service.rs` — модуль с бизнес-логикой, которая построена вокруг вызовов функциональности из крэйта `persist`
* `endpoints.rs` — модуль, содержащий функции обработчики эндпоинтов, которые вызывают функциональность, определённую в модуле `service.rs`
* `lib.rs` — содержит в себе непосредственно создание axum сервера
* `main.rs` — содержит функцию `main`, которая запускает функцию создания сервера из `lib.rs`

Файл `server/src/service.rs`:

```rust,noplayground
use std::sync::Arc;
use persist::{Account, Storage};
use crate::endpoints::NewAcc;

#[derive(Debug, thiserror::Error)]
pub enum AccountServiceError {
    #[error("Database error: (0)")]
    StorageError(#[from] sqlx::Error),
}

// Интерфейс для работы с бизнес логикой
#[async_trait::async_trait]
pub trait AccountService {
    async fn get_all_accounts(&self) -> Result<Vec<Account>, AccountServiceError>;
    async fn create_new_account(&self, new_acc: NewAcc) -> Result<Account, String>;
}

pub struct AccountServiceImpl {
    storage: Arc<dyn Storage + Send + Sync>,
}

impl AccountServiceImpl {
    pub fn new(storage: Arc<dyn Storage + Send + Sync>) -> AccountServiceImpl {
        AccountServiceImpl { storage }
    }
}

#[async_trait::async_trait]
impl AccountService for AccountServiceImpl {
    async fn get_all_accounts(&self) -> Result<Vec<Account>, AccountServiceError> {
        self.storage.fetch_accounts().await
            .map_err(AccountServiceError::from)
    }
    async fn create_new_account(&self, new_acc: NewAcc) -> Result<Account, String> {
        self.storage.create_accounts(&new_acc.owner_name, new_acc.init_balance).await
            .map_err(|e|e.to_string())
    }
}
```

Обратите внимание, что структура `AccountServiceImpl` инкапсулирует хранилище посредством `Arc<dyn Storage + Send + Sync>`. Это позволит нам иметь возможность заинжектить в `AccountServiceImpl` как реальное хранилище — `StorageImpl`, так и какую-то заглушку для тестов.  Ограничения `Send` и `Sync` необходимы, так как методы трэйта асинхронны.

Теперь файл, в котором создаётся Axum сервер — `server/src/lib.rs`. Обратите внимание, что объект нашей бизнес-логики — `AccountServiceImpl` мы храним в объекте состояния, причём также не напрямую, а посредством `Arc<dyn AccountService + Send + Sync>`.

```rust,noplayground
use std::sync::Arc;
use axum::{Router, routing::{get, post}};
use persist::StorageImpl;
use crate::service::{AccountService, AccountServiceImpl};

pub mod service;
pub mod endpoints;

pub struct AppState {
    pub account_service: Arc<dyn AccountService + Send + Sync>,
}

pub async fn run_server() {
    let storage = StorageImpl::from_connection_url(
        "postgres://postgres:1111@localhost/mydb"
    ).await.unwrap();
    let account_service = AccountServiceImpl::new(Arc::new(storage));

    let app = build_app(Arc::new(account_service));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

pub fn build_app(account_service: Arc<dyn AccountService + Send + Sync>) -> Router {
    let state = AppState { account_service };
    let app = Router::new()
        .route("/accounts", get(endpoints::list_accounts))
        .route("/accounts", post(endpoints::create_new_account))
        .with_state(Arc::new(state));
    app
}
```

И теперь сами эндпоинты — `server/src/endpoints.rs`. С ними всё просто: они лишь вызывают соответствующие методы из сервиса бизнес-логики и перепаковывают результат в HTTP ответ.

```rust,noplayground
use std::sync::Arc;
use axum::{Json, extract::State, http::StatusCode};
use serde::{Deserialize, Serialize};
use sqlx::types::BigDecimal;
use persist::Account;
use crate::AppState;

#[derive(Serialize, Deserialize)]
pub struct NewAcc {
    pub owner_name: String,
    pub init_balance: BigDecimal,
}

pub async fn list_accounts(
    State(state): State<Arc<AppState>>
) -> Result<Json<Vec<Account>>, (StatusCode, String)> {
    match state.account_service.get_all_accounts().await {
        Ok(accounts) => Ok(Json(accounts)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}

pub async fn create_new_account(
    State(state): State<Arc<AppState>>, Json(acc): Json<NewAcc>
) -> Result<(StatusCode, Json<Account>), (StatusCode, String)> {
    match state.account_service.create_new_account(acc).await {
        Ok(account) => Ok((StatusCode::CREATED, Json(account))),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}
```

Как видите, эндпоинты просто достают `AccountService` из состояния и вызывают его методы.

И напоследок — `server/src/main.rs`. В главной функции мы просто вызываем функциональность из `lib.rs`.

```rust,noplayground
#[tokio::main]
async fn main() {
    server::run_server().await
}
```

Также напишем интеграционный тест для эндпоинтов `server/tests/endpoints_tests.rs`. Он будет посредством вызова эндпоинтов сначала создавать новый аккаунт, а затем проверять его наличие в хранилище.

```rust,noplayground
use std::sync::Arc;
use axum::http::StatusCode;
use bigdecimal::{BigDecimal, FromPrimitive};
use persist::{Account, StorageImpl};
use server::{build_app, endpoints::NewAcc, service::AccountServiceImpl};
use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
use testcontainers::runners::AsyncRunner;
use axum_test::TestServer;

#[tokio::test]
async fn test_account_endpoints() {
    let container = testcontainers_modules::postgres::Postgres::default()
        .with_password("1111")
        .start().await.unwrap();

    let connection_options = PgConnectOptions::new()
        .host(&container.get_host().await.unwrap().to_string())
        .port(container.get_host_port_ipv4(5432).await.unwrap())
        .database("postgres")
        .username("postgres")
        .password("1111");
    let pool = PgPoolOptions::new()
        .connect_with(connection_options).await.unwrap();

    Migrator::new(std::path::Path::new("../migrations")).await.unwrap()
        .run(&pool).await.unwrap();

    let storage = StorageImpl::new(pool);
    let account_service = AccountServiceImpl::new(Arc::new(storage));
    let app = build_app(Arc::new(account_service));

    let sut = TestServer::new(app).unwrap();

    // Тестовый аккаунт
    let new_acc = NewAcc {
        owner_name: "John Doe".to_string(),
        init_balance: BigDecimal::from_f64(1000.0).unwrap()
    };

    // Проверяем создание аккаунта
    let create_acc_resp = sut.post("/accounts")
        .json(&new_acc)
        .await;
    create_acc_resp.assert_status(StatusCode::CREATED);
    let created_acc = create_acc_resp.json::<Account>();
    assert_eq!(created_acc.owner_name, new_acc.owner_name);
    assert_eq!(created_acc.balance, new_acc.init_balance);

    // Проверяем получение всех аккаунтов
    let list_accs_resp = sut.get("/accounts").await;
    list_accs_resp.assert_status_ok();
    let fetched_accs = list_accs_resp.json::<Vec<Account>>();
    assert_eq!(fetched_accs.len(), 1);
    assert_eq!(fetched_accs[0].id, created_acc.id);
    assert_eq!(fetched_accs[0].owner_name, created_acc.owner_name);
    assert_eq!(fetched_accs[0].balance, created_acc.balance);

}
```

## Миграции

Файлы для миграции структуры БД скопируйте из главы [Версионирование структуры БД](db-migrate.md) и поместите в каталог `migrations`.

## Запуск

Теперь можете прогнать тесты:

```
cargo test
```

Или запустить сервер:

```
cargo run
```

и вызвать эндпоинты, чтобы убедиться в том, что они работают (не забудьте убедиться, что сервер СУБД запущен).

Создать аккаунт:

```
curl -i -X POST \
    -H "Content-Type: application/json" \
    -d '{"owner_name": "John Doe", "init_balance": 1000.0}' \
    http://localhost:8080/accounts
```

Получить все аккаунты:

```
curl -i http://localhost:8080/accounts
```

***

Вот и всё: мы разобрались, каким образом на Axum и SQLx можно писать бекенды с классической слоёной архитектурой, где присутствуют слой данных, слой бизнес-логики и слой представления.
