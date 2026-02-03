# Тест контейнеры

Считается, что полноценное интеграционное тестирование функциональности, взаимодействующей с базой данных, нужно производить только с использованием реального сервера СУБД.
Однако устанавливать и подготавливать сервер PostgreSQL отдельно для тестов — очень неудобно, особенно, если речь идёт о тестировании на CI сервере. Именно поэтому для тестирования работы с СУБД, как правило, используют Docker контейнер с нужным СУБД сервером.

Существует популярная библиотека [Testcontainers](https://testcontainers.com/), которая значительно упрощает интеграционное тестирование с реальными системами, так как берёт на себя всю рутину по:

* запуску тестового контейнера в начале теста
* взаимодействию с контейнером
* завершению работы контейнера после окончания теста

> [!TIP]
> Если вы писали бэкенд приложения на Java, C#, Python, Ruby и т.д., то вы, скорее всего, уже сталкивались с вариацией библиотеки Testcontainers для вашего языка.

***

На основе структуры БД, созданной в предыдущих главах, напишем простую программу, которая сначала добавляет в БД новый аккаунт, а потом вычитывает все аккаунты из БД, и печатает их на консоль.

Для этого мы напишем две функции:

* `fetch_accounts()` — вычитывает все аккаунты из БД
* `insert_accounts(owner_name, initial_balance)` — вставляет новый аккаунт с заданным именем и начальным балансом

Однако в этот раз нас больше интересует не сама программа, а тестирование функций, взаимодействующих с БД. Тест, который мы напишем при помощи Testcontainers, будет:

1. Поднимать новый контейнер с PostgreSQL
2. При помощи SQLx миграции создавать таблицы в базе данных
3. Вызывать функцию `insert_accounts`, чтобы вставить новый аккаунт
4. Вызывать функцию `fetch_accounts`, и проверять, что результат содержит только что созданный аккаунт

Для начала добавим зависимость на [testcontainers](https://crates.io/crates/testcontainers), в `Cargo.toml`:

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

[dev-dependencies]
testcontainers = "0.26"
```

Скопируйте файлы миграции из проекта из прошлой главы [Версионирование структуры БД](db-migrate.md), после чего дерево файлов проекта должно выглядеть так:

```
/
├── Cargo.toml
├── src/
│   └── main.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```

Теперь наш `src/main.rs`:

```rust,noplayground
use bigdecimal::{BigDecimal, FromPrimitive};
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool, postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    insert_accounts(&pool, "John Doe", BigDecimal::from_i32(1000).unwrap())
        .await.unwrap();

    let accounts = fetch_accounts(&pool).await.unwrap();
    println!("{accounts:?}");
}

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

pub async fn insert_accounts(
    db: &PgPool, owner_name: &str, initial_balance: BigDecimal
) -> Result<(), sqlx::Error> {
    sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
        .bind(owner_name)
        .bind(initial_balance)
        .execute(db)
        .await?;
    Ok(())
}

#[cfg(test)]
mod test {
    use super::*;
    use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
    use testcontainers::{
        core::{IntoContainerPort, WaitFor},
        runners::AsyncRunner, GenericImage, ImageExt
    };

    #[tokio::test]
    async fn test_create_account() {
        // Запуск контейнера с PostgreSQL
        let container = GenericImage::new("postgres", "18")
            .with_wait_for(WaitFor::message_on_stderr(
                "database system is ready to accept connections"
            ))
            .with_exposed_port(5432.tcp())
            .with_env_var("POSTGRES_PASSWORD", "1111")
            .start()
            .await
            .expect("Postgres started");

        // Подключение к БД в контейнере
        let connection_options = PgConnectOptions::new()
            .host(&container.get_host().await.unwrap().to_string())
            .port(container.get_host_port_ipv4(5432).await.unwrap())
            .database("postgres")
            .username("postgres")
            .password("1111");

        let pool = PgPoolOptions::new()
            .connect_with(connection_options).await.unwrap();

        // Создание таблиц в базе данных при помощи скриптов SQLx миграции
        Migrator::new(std::path::Path::new("./migrations")).await.unwrap()
            .run(&pool).await.unwrap();

        // Вставляем новый аккаунт
        insert_accounts(&pool, "Test-Account-1", BigDecimal::from_i32(1000).unwrap())
            .await.unwrap();

        // Вычитываем все аккаунты
        let accounts = fetch_accounts(&pool).await.unwrap();

        // Проверяем правильность ответа из БД
        assert_eq!(accounts.len(), 1);
        assert_eq!(accounts[0].owner_name, "Test-Account-1".to_string());
        assert_eq!(accounts[0].balance, BigDecimal::from_i32(1000).unwrap());
    }
}
```

Запустите тест командой `cargo test`:

```
running 1 test
test test::test_create_account ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.58s
```

***

Теперь давайте разберём участок кода, где создаётся контейнер:

```rust,noplayground
let container = GenericImage::new("postgres", "18")
    .with_wait_for(WaitFor::message_on_stderr(
        "database system is ready to accept connections"
    ))
    .with_exposed_port(5432.tcp())
    .with_env_var("POSTGRES_PASSWORD", "1111")
    .start()
    .await
    .expect("Postgres started");
```

Здесь:

* `GenericImage::new("postgres", "18")` — указывает, что мы хотим создать контейнер на основе образа [postgres](https://hub.docker.com/_/postgres) из [https://hub.docker.com/](https://hub.docker.com/). Аргумент `"18"` — это тег образа.
* `.with_wait_for` — задаёт условие, которого необходимо дождаться перед тем, как с контейнером можно будет начинать работать. В нашем случае мы ожидаем момент, когда postgresql сервер внутри контейнера, напечатает в консоль строку "database system is ready to accept connections".
* `.with_exposed_port(5432.tcp())` — указывает, что мы хотим отобразить порт 5432 из контейнера на случайный свободный порт на хостовой системе. Номер порта на хостовой системе можно будет получить из объекта контейнера.
* `.with_env_var("POSTGRES_PASSWORD", "1111")` — проталкивает в контейнер переменную окружения `POSTGRES_PASSWORD`, равную `1111`. Согласно документации образа на Docker hub странице, таким способом мы задаём пароль для базы данных в контейнере. Логин по умолчанию — "postgres"

Аналогичным образом можно поднять и любой другой, доступный на Docker hub образ. По завершению теста, все контейнеры будут автоматически потушены, даже если тест завершился с ошибкой.

## Testcontainers Modules

В пару к крэйту testcontainers существует еще крэйт [testcontainers-modules](https://crates.io/crates/testcontainers-modules), который содержит удобные обёртки для популярных Docker образов. Разумеется, обёртка для PostgreSQL образа присутствует среди них.

Добавим в `Cargo.toml` зависимость на testcontainers-modules:

```toml
[dev-dependencies]
testcontainers = "0.26"
testcontainers-modules = { version = "0.14", features = ["postgres"] }
```

После этого можно запустить контейнер так:

```rust,noplayground
// Запуск контейнера с PostgreSQL
let container = testcontainers_modules::postgres::Postgres::default()
    .with_password("1111")
    .start().await.unwrap();

// Подключение к БД в контейнере
let connection_options = PgConnectOptions::new()
    .host(&container.get_host().await.unwrap().to_string())
    .port(container.get_host_port_ipv4(5432).await.unwrap())
    .database("postgres")
    .username("postgres")
    .password("1111");
```

Как видите, код запуска postgresql контейнера стал заметно короче и проще.

Также testcontainers-modules предлагает подобные обёртки для Anvil, Azurite, CockroachDB, Clickhouse, Consul, DynamoDB, ElasticSearch, Kafka, Localstack, Minio, MongoDB, MS SQL Server, MySQL, Nats, Neo4J, OpenLDAP, Oracle, OrientDB, RabbitMQ, Redis, RQLitem Scylladb, Solr, SurrealDB и Zookeeper.
