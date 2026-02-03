# SQLx

Настало время поговорить о работе с реляционными базами данных.

Для работы с реляционными СУБД мы будем использовать популярную библиотеку [SQLx](https://crates.io/crates/sqlx), которая поддерживает PostgreSQL, MySQL, SQLite и MS SQL Server.

SQLx предоставляет:

* драйверы для СУБД
* пул соединений
* API для выполнения SQL и получения ответов
* конверторы, преобразующие ответ от БД в объекты Rust структур

## Тестовая БД

Для примера будем использовать СУБД PostgreSQL. Вы можете поставить дистрибутив PostgreSQL локально или использовать docker образ.

Сначала создайте новую БД с именем `mydb`. Если вы предпочитаете использовать докер, то можете использовать следующие команды:

Запустить контейнер с PostgreSQL:

```
docker run --name my_pg_container \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=1111 \
  -d postgres
```

Зайти в консоль запущенного контейнера:

```
docker exec -it my_pg_container bash
```

Запустить psql (консольный клиент для PostgreSQL):

```
psql -U postgres
```

В консоли psql создать новую БД:

```sql
CREATE DATABASE mydb;
```

Выбрать mydb в качестве текущей активной БД:

```
\c mydb;
```

(Чтобы посмотреть все имеющиеся БД, используется команда `\list`)

***

Для наших примеров нам понадобится база данных с двумя таблицами: "банковские аккаунты" и "история транзакций".

<pre class="ascii-diagram" >
┌────────────┐         ┌────────────────┐
│  accounts  │         │  transactions  │
├────────────┤         ├────────────────┤
│ id         │───┐     │ id             │
│ owner_name │   │     │ amount         │
│ balance    │   ├────*│ src_account_id │
└────────────┘   └────*│ dst_account_id │
                       │ tx_timestamp   │
                       └────────────────┘
</pre>

Выполните следующий SQL, чтобы создать таблицы и тестовые данные к ним:

```sql
CREATE SEQUENCE accounts_seq START WITH 1000;

CREATE TABLE accounts ( -- mydb.public.accounts 
    id BIGINT PRIMARY KEY DEFAULT nextval('accounts_seq'),
    owner_name VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(10, 2)  NOT NULL DEFAULT 0.00 CHECK (balance >= 0)
);

CREATE SEQUENCE transactions_seq START WITH 1000;

CREATE TABLE transactions ( -- mydb.public.transactions 
    id BIGINT PRIMARY KEY DEFAULT nextval('transactions_seq'),
    amount NUMERIC(10, 2) DEFAULT 0.00,
    src_account_id BIGINT NOT NULL,
    dst_account_id BIGINT NOT NULL,
    tx_timestamp TIMESTAMP NOT NULL,
    FOREIGN KEY (src_account_id) REFERENCES accounts (id),
    FOREIGN KEY (dst_account_id) REFERENCES accounts (id)
);

INSERT INTO accounts(id, owner_name, balance) VALUES
(1, 'John Doe',    1000.00),
(2, 'Ivan Ivanov', 2000.00);

INSERT INTO transactions(amount, src_account_id, dst_account_id, tx_timestamp)
VALUES
(10.00, 1, 2, TO_TIMESTAMP('2025-12-11 14:00:00', 'YYYY-MM-DD HH24:MI:SS')),
(20.00, 2, 1, TO_TIMESTAMP('2025-12-12 15:00:00', 'YYYY-MM-DD HH24:MI:SS'));
```

Можете воспользоваться psql командой `\dt`, чтобы посмотреть список таблиц в базе данных и убедиться, что таблицы `accounts` и `transactions` присутствуют.

## Подключение к СУБД

Теперь когда база данных создана, можно приступать к программе на Rust.

Создадим новый проект:

```
cargo new test_sqlx
```

И добавим в `Cargo.toml` зависимости:

* [sqlx](https://crates.io/crates/sqlx) — сама библиотека SQLx. Фичи:
  * `postgres` включает в компиляцию реализации sqlx интерфейсов для PostgreSQL
  * `bigdecimal` подключает поддержку типа `BigDecimal`:  SQL тип `NUMERIC` обычно конвертируют именно в `BigDecimal`
* [chrono](https://crates.io/crates/chrono) — библиотека для работы с датой и временем
* [bigdecimal](https://crates.io/crates/bigdecimal) — библиотека, которая предоставляет тип [BigDecimal](https://docs.rs/bigdecimal/latest/bigdecimal/struct.BigDecimal.html) —  числовой тип большого размера и без потери точности при совершении операций над числами с плавающей запятой

```toml
[package]
name = "test_sqlx"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = "0.4"
```

Теперь мы можем написать программу, которая подключается к PostgreSQL базе данных. Для создания пула соединений к PostgreSQL используется билдер [PgPoolOptions](https://docs.rs/sqlx/latest/sqlx/postgres/type.PgPoolOptions.html), который позволяет сконфигурировать целый ряд параметров пула соединения, таких как:

* минимальное и максимальное количество соединений в пуле
* таймаут для получения соединения из пула, максимальное время жизни соединения, максимальное время жизни простаивающего соединения
* коллбэки, которые могут выполняться: после установки соединения с сервером СУБД, перед получением соединения из пула, после возврата соединения в пул
* различные опции логирования

Рассмотрим простейший пример подключения к нашей свежесозданной  БД:

```rust,noplayground
use sqlx::{Pool, Postgres, postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool: Pool<Postgres> = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();
}
```

Для указания непосредственно настроек подключения к БД используется один из двух методов: `connect` (его мы использовали в примере выше) или `connect_with`.

#### connect

Метод [connect](https://docs.rs/sqlx/latest/sqlx/postgres/type.PgPoolOptions.html#method.connect) задаёт настройки подключения при помощи URL строки.

Формат: `протокол://логин:пароль@хост/бд?параметры`.

Например:

- Для PostgreSQL: `postgres://mylogin:mypassword@localhost/mydb`\
- Для MySQL: `mysql://mylogin:mypassword@host/mydb`\
- Для SQLite: `sqlite::memory:` или `sqlite://my.db`

#### connect\_with

Метод [connect_with](https://docs.rs/sqlx/latest/sqlx/pool/struct.PoolOptions.html#method.connect_with) — задаёт настройки подключения при помощи структуры [PgConnectOptions](https://docs.rs/sqlx/latest/sqlx/postgres/struct.PgConnectOptions.html), которая инкапсулирует такие параметры, как хост, логин, пароль, имя БД и т.д.

`PgConnectionOption` позволяет выполнить более тонкую настройку по сравнению с `connect`.

Пример использования:

```rust,noplayground
use sqlx::postgres::{PgConnectOptions, PgPoolOptions};

#[tokio::main]
async fn main() {
    // Опции подключения
    let connection_option = PgConnectOptions::new()
        .host("localhost")
        .username("postgres")
        .password("1111")
        .database("mydb");

    // Создание пула соединений
    let pool = PgPoolOptions::new()
        .connect_with(connection_option)
        .await
        .unwrap();
}
```

## Выборка данных

### Типизированные запросы

Для выборки данных SQLx предлагает тип [QueryAs](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html), который позволяет выполнить SQL-запрос и сконвертировать ответ от БД в объекты структур соответствующего типа.

Объект `QueryAs`, как правило, создаётся при помощи функции [sqlx::query\_as](https://docs.rs/sqlx/latest/sqlx/fn.query_as.html).

```rust,noplayground
let query: QueryAs<'_, Postgres, ТипРезультата, PgArguments> = sqlx::query_as(
    "SELECT поле1, поле2, поле3 FROM таблица"
);
```

Далее объекту `QueryAs` необходимо передать объект пула соединений с БД, чтобы он мог получить соединение и выполнить запрос. Для этого используется один из следующих методов:

* [fetch_all](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch_all) — ожидает, что результатом запроса будет множество записей
* [fetch_one](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch_one) — ожидает, что результатом запроса будет ровно одна запись
* [fetch_optional](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch_optional) — ожидает, что результатом запроса будет не более одной записи

Если мы хотим выбрать множество записей и получить результат в форме вектора объектов, то код будет выглядеть примерно так:

```rust,noplayground
#[derive(FromRow)]
struct ТипРезультата {
   поле1: Тип1,
   поле2: Тип2,
   поле3: Тип3,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new().connect("PG_URL").await.unwrap();
    
    let query: QueryAs<'_, Postgres, ТипРезультата, PgArguments> = sqlx::query_as(
        "SELECT поле1, поле2, поле3 FROM таблица"
    );

    let result: Vec<ТипРезультата> = query.fetch_all(&pool).await.unwrap();
}
```

Как вы могли заметить, структура, в объекты которой перепаковывается ответ от БД, должна реализовать трэйт [FromRow](https://docs.rs/sqlx/latest/sqlx/trait.FromRow.html). При этом имена полей структуры должны совпадать с именами соответствующих колонок в результате SQL-запроса.

Рассмотрим пример выборки из таблицы `accounts` в нашей базы данных:

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, prelude::FromRow, types::BigDecimal};

#[derive(Debug, FromRow)]
struct Account {
    id: i64,
    owner_name: String,
    balance: BigDecimal,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    // Выборка списка записей
    let all_accounts: Vec<Account> = sqlx::query_as(
            "SELECT id, owner_name, balance FROM accounts"
        ).fetch_all(&pool).await.unwrap();
    for acc in all_accounts {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
    // 1: John Doe, 1000
    // 2: Ivan Ivanov, 2000

    // Выборка одной записи
    let opt_acc_1: Option<Account> = sqlx::query_as(r#"
            SELECT id, owner_name, balance FROM accounts WHERE owner_name=$1
        "#)
        .bind("John Doe") // Привязываем значение к плэйсхолдеру $1
        .fetch_optional(&pool)
        .await.unwrap();
    if let Some(acc) = opt_acc_1 {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
    // 1: John Doe, 1000
}
```

Как видите, принцип простой — нужно:

1. Написать SQL-запрос
2. Создать структуру с таким же набором полей, как и набор колонок в результате SQL-запроса
3. Воспользоваться функцией `query_as`

***

`query_as` используется только для выборки двух и более колонок. Если необходимо выбрать только одну колонку, то вместо `query_as` используется функция [query\_scalar](https://docs.rs/sqlx-core/latest/sqlx_core/query_scalar/fn.query_scalar.html), которая ведёт себя точно так же, но конвертирует результат от БД не в объекты структур, а в простые типы (строки и числа).

Например:

```rust,noplayground
use sqlx::{postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    // Выборка одной колонки
    let account_ids: Vec<i64> = sqlx::query_scalar("SELECT id FROM accounts")
        .fetch_all(&pool).await.unwrap();
    println!("All IDs: {account_ids:?}"); // All IDs: [1, 2]

    // Выборка одного значения
    let accounts_count: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM accounts")
        .fetch_one(&pool).await.unwrap();
    println!("Number of accounts: {accounts_count}"); // Number of accounts: 2
}
```

***

Теперь давайте рассмотрим пример с JOIN запросом: выберем всю историю транзакций с указанием имён отправителя и получателя денег. Чтобы получить имя по ID аккаунта, мы сделаем JOIN таблицы `transactions` на таблицу `accounts`.

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, prelude::FromRow, types::BigDecimal};
use chrono::NaiveDateTime;

#[derive(Debug, FromRow)]
struct TransactionFullInfo {
    id: i64,
    amount: BigDecimal,
    src_account_owner_name: String,
    dst_account_owner_name: String,
    tx_timestamp: NaiveDateTime,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();

    let transaction: Vec<TransactionFullInfo> = sqlx::query_as(r#"
            SELECT
                tx.id as id, tx.amount, tx.tx_timestamp,
                src_acc.owner_name as src_account_owner_name,
                dst_acc.owner_name as dst_account_owner_name
            FROM
                transactions tx
                JOIN accounts src_acc ON tx.src_account_id = src_acc.id
                JOIN accounts dst_acc ON tx.dst_account_id = dst_acc.id
    "#)
    .fetch_all(&pool).await.unwrap();

    for tx in transaction {
        println!(
            "TXID:{} amount={}, timestamp={}, src: {}, dst: {}",
            tx.id, tx.amount, tx.tx_timestamp,
            tx.src_account_owner_name, tx.dst_account_owner_name
        );
    }
// TXID:1 amount=10, timestamp=2025-12-11 14:00:00, src: John Doe, dst: Ivan Ivanov
// TXID:2 amount=20, timestamp=2025-12-12 15:00:00, src: Ivan Ivanov, dst: John Doe
}
```

***

Если в тексте запроса надо задать аргументы, то значения для них передаются при помощи метода [bind](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.bind).

Для примера сделаем запрос, который для заданных отправителя и получателя выводит общую сумму только тех транзакций, чья сумма превышает указанное значение.

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, types::BigDecimal};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let total_amount: BigDecimal = sqlx::query_scalar(r#"
            SELECT
                SUM(amount)
            FROM transactions tx
                JOIN accounts src_acc ON tx.src_account_id = src_acc.id
                JOIN accounts dst_acc ON tx.dst_account_id = dst_acc.id
            WHERE
                amount >= $1
                AND src_acc.owner_name = $2
                AND dst_acc.owner_name = $3
        "#)
        .bind(20.0)
        .bind("Ivan Ivanov")
        .bind("John Doe")
        .fetch_one(&pool).await.unwrap();

    println!("{total_amount}"); // 20
}
```

Важно отметить, что для передачи аргументов запроса, SQLx использует так называемые [prepared statement](https://www.postgresql.org/docs/current/sql-prepare.html). То есть для таких запросов на стороне сервера СУБД разбор SQL кода и формирование плана запроса производится только в первый раз. Все дальнейшие вызовы этого же запроса с другими значениями аргументов будут переиспользовать план запроса от предыдущих вызовов.

### Нетипизированные запросы

Тип `QueryAs` позволяет сделать выборку, и сразу конвертировать результат в объекты структуры. Однако есть другой тип запроса — [Query](https://docs.rs/sqlx/latest/sqlx/query/struct.Query.html), который не занимается такой конвертацией, а возвращает результат в виде коллекции нетипизированных записей `Row`, которые похожи на хеш-таблицы.

Объект `Query` создаётся функцией [sqlx::query](https://docs.rs/sqlx/latest/sqlx/fn.query.html), которая очень похожа на `sqlx::query_as`. Для того чтобы выполнить объект `Query`, как и в случае с `QueryAs`, нужно использовать один из `fetch_*` методов:

```rust,noplayground
let query: Query<'_, Postgres, PgArguments> = sqlx::query(
    "SELECT поле1, поле2, поле3 FROM таблица"
);
let rows: Vec<PgRow> = query.fetch_all(&pool).await.unwrap();
```

Результат запроса представлен объектами типа [PgRow](https://docs.rs/sqlx/latest/sqlx/postgres/struct.PgRow.html) (был бы [MySqlRow](https://docs.rs/sqlx-mysql/latest/sqlx_mysql/struct.MySqlRow.html) для MySQL или [SqliteRow](https://docs.rs/sqlx-sqlite/latest/sqlx_sqlite/struct.SqliteRow.html) для SQLite).

Для извлечения значений колонок из `PgRow` используется метод [try\_get](https://docs.rs/sqlx/latest/sqlx/trait.Row.html#method.try_get).

В качестве примера перепишем с использованием `sqlx::query` прошлый пример JOIN запроса:

```rust,noplayground
use chrono::NaiveDateTime;
use sqlx::postgres::PgRow;
use sqlx::{postgres::PgPoolOptions, types::BigDecimal};
use sqlx::Row;

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();

    let rows: Vec<PgRow> = sqlx::query(r#"
            SELECT
                tx.id as id, tx.amount, tx.tx_timestamp,
                src_acc.owner_name as src_account_owner_name,
                dst_acc.owner_name as dst_account_owner_name
            FROM
                transactions tx
                JOIN accounts src_acc ON tx.src_account_id = src_acc.id
                JOIN accounts dst_acc ON tx.dst_account_id = dst_acc.id
        "#)
        .fetch_all(&pool)
        .await
        .unwrap();
 
    for r in rows {
        let id: i64 = r.try_get("id").unwrap();
        let amount: BigDecimal = r.try_get("amount").unwrap();
        let ts: NaiveDateTime = r.try_get("tx_timestamp").unwrap();
        let src: String = r.try_get("src_account_owner_name").unwrap();
        let dst: String = r.try_get("dst_account_owner_name").unwrap();

        println!("TXID:{id} amount={amount}, timestamp={ts}, src: {src}, dst: {dst}");
    }
// TXID:1 amount=10, timestamp=2025-12-11 14:00:00, src: John Doe, dst: Ivan Ivanov
// TXID:2 amount=20, timestamp=2025-12-12 15:00:00, src: Ivan Ivanov, dst: John Doe
}
```

## Вставка данных

Теперь рассмотрим, как производить вставку данных в таблицы.

Для выполнения INSERT, UPDATE и DELETE запросов используется уже знакомый нам тип запроса — `Query`. Однако теперь для выполнения запроса вместо метода `fetch_*` применяется метод [execute](https://docs.rs/sqlx/latest/sqlx/query/struct.Query.html#method.execute).

Рассмотрим пример:

```rust,noplayground
use sqlx::{postgres::{PgPoolOptions, PgQueryResult}, types::BigDecimal};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let result: PgQueryResult = sqlx::query(
            "INSERT INTO accounts(owner_name, balance) VALUES($1, $2)"
        )
        .bind("Some Name")
        .bind(BigDecimal::default())
        .execute(&pool).await.unwrap();
    println!("{result:?}"); // PgQueryResult { rows_affected: 1 }
}
```

Метод `execute` возвращает объект структуры [PgQueryResult](https://docs.rs/sqlx-postgres/latest/sqlx_postgres/struct.PgQueryResult.html), хранящий число фактически вставленных/изменённых записей.

***

Если у нас имеется целая коллекция сущностей, которые мы хотим вставить в таблицу, то нам будет удобно воспользоваться типом [QueryBuilder](https://docs.rs/sqlx/latest/sqlx/struct.QueryBuilder.html), работа с которым имеет вид:

```rust,noplayground
struct Сущность { // Тип сущности, ассоциированной с таблицей в БД
    поле1: Тип1,
    поле2: Тип2,
    поле3: Тип3,
}

// Создаём объект QueryBuilder с заголовком INSERT запроса
let mut qb = QueryBuilder::new(r#"INSERT INTO таблица(поле1, поле2, поле3)"#);

let вектор_сущностей: Vec<Сущность> = ...; // коллекция сущностей для вставки

qb.push_values(&вектор_сущностей, |mut builder, сущность| {
    // привязываем поля сущностей к колонкам из заголовка INSERT запроса
    builder
        .push_bind(&сущность.поле1)
        .push_bind(&сущность.поле2)
        .push_bind(&сущность.поле3);
});

qb.build().execute(&pool).await.unwrap(); // выполняем запрос
```

Для примера рассмотрим программу, которая вставляет в таблицу `accounts` несколько новых записей:

```rust,noplayground
use sqlx::{
    Postgres, QueryBuilder, types::BigDecimal,
    postgres::{PgPoolOptions, PgQueryResult}
};

struct NewAcc {
    owner_name: String,
    balance: BigDecimal,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();

    let new_accounts = vec![
        NewAcc {owner_name: "Name 1".to_string(), balance: BigDecimal::default()},
        NewAcc {owner_name: "Name 2".to_string(), balance: BigDecimal::default()},
        NewAcc {owner_name: "Name 3".to_string(), balance: BigDecimal::default()},
    ];

    let mut qb: QueryBuilder<'_, Postgres> =
        QueryBuilder::new(r#"INSERT INTO accounts(owner_name, balance)"#);

    qb.push_values(&new_accounts, |mut builder, acc| {
        builder
            .push_bind(&acc.owner_name)
            .push_bind(&acc.balance);
    });

    let result: PgQueryResult = qb.build().execute(&pool).await.unwrap();
    println!("{result:?}"); // PgQueryResult { rows_affected: 3 }
}
```

## Транзакции

Теперь давайте разберёмся, как в SQLx работать с транзакциями.

Транзакцию можно создать путём вызова метода [begin](https://docs.rs/sqlx/latest/sqlx/struct.Pool.html#method.begin) на объекте пула соединений к БД. Этот вызов вернёт объект типа [Transaction](https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html), который инкапсулирует транзакцию. Далее этот объект транзакции можно использовать вместо пула соединений (`Pool`) для выполнения запросов в методах `fetch_all`, `fetch_one`, `execute` и т.д. Все запросы, выполненные на объекте транзакции, будут выполнены в рамках этой транзакции.

```rust,noplayground
#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new().connect("PG_URL").await.unwrap();
    
    let mut tx = pool.begin().await.expect("Cannot start transaction");
    
    sqlx::query("UPDATE ...").execute(&mut *tx).await?;
    sqlx::query("INSERT ...").execute(&mut *tx).await?;
    sqlx::query("DELETE ...").execute(&mut *tx).await?;

    tx.commit().await.expect("Cannot commit");
}
```

Как это работает?

Для начала, если мы посмотрим на сигнатуру метода [Query::execute](https://docs.rs/sqlx/latest/sqlx/query/struct.Query.html#method.execute), то увидим, что в качестве аргумента он принимает не пул соединений ([Pool](https://docs.rs/sqlx/latest/sqlx/struct.Pool.html)), а некий [Executor](https://docs.rs/sqlx/latest/sqlx/trait.Executor.html). Не вдаваясь в подробности, скажем просто, что и `Pool`, и `Transaction` реализуют этот трэйт `Executor`. Именно поэтому запросы можно исполнять как на объекте пула соединений, так и на объекте транзакции.

Теперь разберёмся с самим типом `Transaction`. Он работает следующим образом:

* Когда мы вызываем на пуле соединений метод `begin()`, то из пула выбирается соединение, в которое сразу отправляется вызов `BEGIN TRANSACTION`.
* Вызов метода `commit()` приведёт к тому, что в соединение будет отправлено `COMMIT`, что в свою очередь закрепит транзакцию.
* Если объект `Transaction` будет уничтожен до того, как на нём будет вызван метод `commit()`, то его деструктор отправит в соединение с БД команду `ROLLBACK`.

В качестве примера рассмотрим функцию, которая совершает пересылку денег с одного аккаунта на другой. Эта функция должна транзакционно совершить три операции:

1. снять деньги со счёта отправителя
2. добавить деньги на счёт получателя
3. создать запись о произошедшей транзакции

```rust,noplayground
use bigdecimal::FromPrimitive;
use sqlx::{PgPool, postgres::PgPoolOptions, types::BigDecimal};

struct Transfer {
    src_account_id: i64,
    dst_account_id: i64,
    amount: BigDecimal,
}

async fn make_transfer(transfer: &Transfer, pool: &PgPool) -> Result<(),sqlx::Error> {
    // Начинаем транзакцию
    let mut tx = pool.begin().await?;

    // Добавляем сумму трансфера к балансу получателя
    let _ = sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
        .bind(&transfer.amount)
        .bind(&transfer.dst_account_id)
        .execute(&mut *tx)
        .await?;

    // Уменьшаем баланс отправителя на сумму трансфера
    let _ = sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
        .bind(&transfer.amount)
        .bind(&transfer.src_account_id)
        .execute(&mut *tx)
        .await?;

    // Создаём новую запись о транзакции по пересылке денег
    let _ = sqlx::query(r#"
            INSERT INTO transactions(
                amount, src_account_id, dst_account_id, tx_timestamp
            ) VALUES ($1, $2, $3, NOW())
        "#)
        .bind(&transfer.amount)
        .bind(&transfer.src_account_id)
        .bind(&transfer.dst_account_id)
        .execute(&mut *tx)
        .await?;

    // Закрепляем транзакцию
    tx.commit().await?;

    Ok(())
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let transfer = Transfer {
        src_account_id: 1,
        dst_account_id: 2,
        amount: BigDecimal::from_f64(50.0).unwrap()
    };

    let _ = make_transfer(&transfer, &pool).await.unwrap();
}
```

Теперь, если мы посмотрим на таблицы `accounts` и `transactions`, то увидим изменения, сделанные программой:

```
mydb=# select * from accounts;
 id | owner_name  | balance
----+-------------+---------
  1 | John Doe    |  950.00
  2 | Ivan Ivanov | 2050.00
```

```
mydb=# select * from transactions;
 id | amount | src_account_id | dst_account_id |        tx_timestamp
----+--------+----------------+----------------+----------------------------
  1 |  10.00 |              1 |              2 | 2025-12-11 14:00:00
  2 |  20.00 |              2 |              1 | 2025-12-12 15:00:00
  3 |  50.00 |              1 |              2 | 2025-12-13 02:00:22.788004
```

Чтобы увидеть, как происходит откат транзакции, давайте попытаемся сделать перевод суммы, которая превышает текущий баланс на счету отправителя. Это вызовет срабатывание ограничения `CHECK (balance > 0)` в таблице accounts.

```rust,noplayground
#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let transfer = Transfer {
        src_account_id: 1,
        dst_account_id: 2,
        amount: BigDecimal::from_f64(5000.0).unwrap() // На аккаунте нет столько денег
    };

    let _ = make_transfer(&transfer, &pool).await.unwrap();
}
```

Как видите, мы попытались перевести 5000, что привело к тому, что вторая UPDATE операции (та, которая вычитает сумму со счета отправителя) в функции `make_transfer` завершилась с ошибкой.

```
Database(PgDatabaseError {
    severity: Error, code: "23514",
    message: "new row for relation 'accounts' violates check constraint 'accounts_balance_check'",
    detail: Some("Failing row contains (1, John Doe, -4050.00)."),
    schema: Some("public"),
    table: Some("accounts"),
    constraint: Some("accounts_balance_check"),
    routine: Some("ExecConstraints")
})
```

Транзакция при этом была полностью откачена.
