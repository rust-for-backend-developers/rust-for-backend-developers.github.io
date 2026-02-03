# SQLx макросы

Будучи языком со строгой статической типизацией, Rust пытается обнаружить максимум ошибок еще на этапе компиляции. Однако если мы сделаем ошибку в тексте SQL запроса, то узнаем мы об этом только во время его исполнения. Хорошей практикой является написание интеграционных тестов с реальной СУБД для всех запросов, однако это не единственный способ уберечься от ошибок в SQL.

Утилита `cargo sqlx`, с которой мы познакомились в прошлой главе, имеет возможность подключиться к базе данных и проверить корректность кода SQL запросов. Принцип работы следующий:

1. Вместо функций `query_as`, `query_scalar` и `query` мы должны использовать соответствующие макросы: [query_as!](https://docs.rs/sqlx/latest/sqlx/macro.query_as.html), [query_scalar!](https://docs.rs/sqlx/latest/sqlx/macro.query_scalar.html) и [query!](https://docs.rs/sqlx/latest/sqlx/macro.query.html). Эти макросы работают по тому же принципу, что и их собратья-функции.\
   Сразу после написания кода с этими макросами компилятор будет на них ругаться и рекомендовать запустить команду `cargo sqlx prepare`
2. Поэтому запускаем эту команду:\
   `cargo sqlx prepare --database-url postgres://postgres:1111@localhost/mydb`\
   Утилита sqlx подключится к базе данных и с её помощью провалидирует все SQL запросы, используемые в макросах `query_as!`, `query_scalar!` и `query!`.\
   Также утилита создаст каталог `.sqlx/`, в который сложит JSON файлы с метаинформацией о проверенных запросах.
3. При компиляции приложения макросы `query_as!`, `query_scalar!` и `query!` будут валидировать текст SQL запросов, используя метаинформацию из каталога `.sqlx`.

Таким образом, мы получим проверку корректности SQL запросов на этапе компиляции.

## query_as!, query_scalar! и query!

Формат использования макроса `query_as!` практически такой же, как и функции `query_as`, но с минимальными отличиями:

```rust,noplayground
struct ТипРезультата {
   поле1: Тип1,
   поле2: Тип2,
   поле3: Тип3,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new().connect("PG_URL").await.unwrap();
    
    let query = sqlx::query_as!(
        ТипРезультата,
        "SELECT поле1, поле2, поле3 FROM таблица WHERE поле1 = $1 AND поле2 = $2",
        ЗначениеАргумента$1,
        ЗначениеАргумента$2,
    );

    let result: Vec<ТипРезультата> = query.fetch_all(&pool).await.unwrap();
}
```

Как видите, структура, в объекты которой конвертируется результат, больше не должна реализовывать трэйт `FromRow`. Это логично: макрос выполняется на этапе компиляции, поэтому у него и так есть доступ к полям структуры.

Также макрос `sqlx::query_as!` принимает минимум два аргумента: структура, в виде объектов которой будет представлен результат запроса, и сам SQL запрос. Напомним, функция `query_as` имеет только один аргумент — SQL запрос.

Если код запроса содержит аргументы, то значения для них передаются сразу после самого запроса, в то время как с функциями мы использовали метод `bind` на объекте `Query`.

***

Используя макрос, перепишем наш пример использования `query_as` из прошлой главы.

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, types::BigDecimal};

#[derive(Debug)]
struct Account {
    id: i64,
    owner_name: String,
    balance: BigDecimal,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let all_accounts: Vec<Account> = sqlx::query_as!(
            Account,
            "SELECT id, owner_name, balance FROM accounts"
        ).fetch_all(&pool).await.unwrap();
    for acc in all_accounts {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
 
    let opt_acc_1: Option<Account> = sqlx::query_as!(
            Account,
            "SELECT id, owner_name, balance FROM accounts WHERE owner_name=$1",
            "John Doe" // Значение для аргумента $1
        )
        .fetch_optional(&pool)
        .await.unwrap();
    if let Some(acc) = opt_acc_1 {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
}
```

Теперь выполняем команду:

```
$ cargo sqlx prepare --database-url postgres://postgres:1111@localhost/mydb
query data written to .sqlx in the current directory;
please check this into version control
```

Если команда `cargo sqlx prepare` завершилась успешно, то можно запустить программу:

```
$ cargo run
1: John Doe, 1000
2: Ivan Ivanov, 2000
1: John Doe, 1000
```

***

Макрос `query_scalar!` работает по тому же принципу:

```rust,noplayground
use sqlx::{postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let account_ids: Vec<i64> = sqlx::query_scalar!("SELECT id FROM accounts")
        .fetch_all(&pool).await.unwrap();
    println!("All IDs: {account_ids:?}");

    let accounts_count: Option<i64> = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM accounts"
        )
        .fetch_one(&pool).await.unwrap();
    println!("Number of accounts: {}", accounts_count.unwrap_or_default());
}
```

После выполнения `cargo sqlx prepare`, запускаем приложение:

```
$ cargo run
All IDs: [1, 2]
Number of accounts: 2
```

***

И макрос `query!`, как вы уже могли догадаться, тоже работает по той же самой схеме:

```rust,noplayground
use sqlx::{postgres::{PgPoolOptions, PgQueryResult}, types::BigDecimal};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let result: PgQueryResult = sqlx::query!(
            "INSERT INTO accounts(owner_name, balance) VALUES($1, $2)",
            "Some Name",
            BigDecimal::default(),
        )
        .execute(&pool).await.unwrap();
    println!("{result:?}"); // PgQueryResult { rows_affected: 1 }
}
```

После выполнения `cargo sqlx prepare`, запускаем приложение:

```
$ cargo run
PgQueryResult { rows_affected: 1 }
```

## query* функции vs. query*! макросы

После прочтения этой главы у вас, скорее всего, возник вопрос: "Когда следует использовать макросы, а когда — просто функции?".

Распространено мнение, что функции следует использовать только в ситуациях, когда SQL конструируется динамически во время выполнения программы, а во всех остальных случаях следует использовать макросы.

На практике же макросы не являются абсолютной защитой от некорректного SQL, так как структура БД, относительно которой вы выполнили `cargo sqlx prepare`, может, ввиду разных обстоятельств, оказаться не такой, какая будет в итоге у продакшн базы данных. Поэтому, в первую очередь, важно покрывать интеграционными тестами всю функциональность, которая работает с БД.

Также следует заметить, что макросы не дают никакого преимущества в плане скорости выполнения, так как компилируются в те же структуры, которые создают их собратья-функции.
