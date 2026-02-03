# Версионирование структуры БД

SQLx предлагает утилиту командной строки [sqlx-cli](https://crates.io/crates/sqlx-cli), которая позволяет:

* вычитывать метаданные из БД и валидировать SQL запросы на этапе компиляции
* быстро создавать и удалять тестовую БД
* версионировать структуры БД, что часто еще называют **миграцией**

Именно версионирование структуры БД мы рассмотрим в этой главе.

Для начала нам потребуется установить саму утилиту при помощи команды:

`cargo install sqlx-cli`

Теперь давайте перенесём в файлы миграции наш SQL-скрипт для создания таблиц из прошлой главы.

Создадим первый файл при помощи команды:

```
cargo sqlx migrate add accounts -r --sequential
```

Эта команда создаст два пустых файла:

* `migrations/0001_accounts.up.sql` — файл для изменения структуры БД
* `migrations/0001_accounts.down.sql` — файл для отката изменений

Откроем `0001_accounts.up.sql` и напишем в нём следующее:

```sql
CREATE SEQUENCE accounts_seq START WITH 1000;

CREATE TABLE accounts ( -- mydb.public.accounts 
    id BIGINT PRIMARY KEY DEFAULT nextval('accounts_seq'),
    owner_name VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(10, 2)  NOT NULL DEFAULT 0.00 CHECK (balance >= 0)
);
```

Далее `0001_accounts.down.sql`:

```sql
DROP TABLE accounts;
DROP SEQUENCE accounts_seq;
```

Теперь создадим следующую пару файлов для таблицы `transactions`.

Можно также сделать это командой `cargo sqlx migrate add transactions -r --sequential`, но можно и создать файлы `0002_transactions.up.sql` и `0002_transactions.down.sql` вручную.

Заполним `0002_transactions.up.sql`:

```sql
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
```

И затем `0002_transactions.down.sql`:

```sql
DROP TABLE transactions;
DROP SEQUENCE transactions_seq;
```

Теперь, чтобы протестировать нашу миграцию, давайте создадим новую БД — my_migration.

```sql
CREATE DATABASE my_migration;
```

После создания БД запустим миграцию:

```
$ cargo sqlx migrate run \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 1/migrate accounts (3.99736ms)
Applied 2/migrate transactions (4.774571ms)
```

В базе данных должны появиться три таблицы:

* `accounts`
* `transactions`
* `_sqlx_migrations`

Таблица `_sqlx_migrations` является служебной таблицей SQLx и хранит метаданные применённых миграций. Механизм миграции использует эту таблицу, чтобы знать, какие миграции были уже выполнены. Таблица хранит номера применённых миграций, имена соответствующих выполненных файлов миграции, время выполнения, а также контрольную сумму файла миграции (нужна для проверки того, не был ли изменён "задним числом" файл от уже применённой миграции).

```
> select * from _sqlx_migrations;
 version | description  |  installed_on       | success | checksum | execution_time
---------+------------- +---------------------+---------+---------------------------
       1 | accounts     | 2025-12-12 02:19:32 | t       | \x3124df | 3997360
       2 | transactions | 2025-12-12 02:19:32 | t       | \xaded4e | 4774571
```

Как мы видим, обе наши миграции были успешно применены.

Утилита `cargo sqlx migrate` позволяет не только применять миграции, но и откатывать их. Например, давайте откатим структуру БД до версии 1, то есть к моменту, когда уже была создана таблица `accounts`, но еще не была создана таблица `transactions`.

```
$ cargo sqlx migrate revert \
    --target-version 1 \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 2/revert transactions (3.658186ms)
Skipped 1/revert accounts (0ns)
```

Если мы проверим, какие таблицы есть в БД `my_migration`, то мы увидим, что таблица `transactions` пропала.

Если мы запустим миграцию еще раз, то таблица `transactions` будет создана опять.

```
$ cargo sqlx migrate run \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 2/migrate transactions
```

