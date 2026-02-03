# Тестирование

## Unit тестирование

В Rust принято писать юнит-тесты в том же файле, в котором находятся тестируемые функции.

Юнит-тест — представляет из себя функцию, помеченную аннотацией `#[test]`.

Например, у нас есть программа, которая инкрементирует число. Делает она это при помощи функции `inc`.

```rust
fn inc(a: i32) -> i32 {
    a + 1
}

fn main() {
    let a = 5;
    let b = inc(a);
    println!("{b}");
}
```

Давайте напишем для функции `inc` пару юнит-тестов: `test_inc_1` и `test_inc_2`.

```rust
fn inc(a: i32) -> i32 {
    a + 1
}

#[test]
fn test_inc_1() {
    assert_eq!(inc(1), 2);
}

#[test]
fn test_inc_2() {
    assert_eq!(inc(7), 8);
}

fn main() {
    let a = 5;
    a = inc(a);
    println!("{a}");
}
```

Макрос `assert_eq` сравнивает аргументы, и если они не равны, то инициируется паника.


***

Для того чтобы запустить все тесты в пакете, используется команда `cargo test`:

```
$ cargo test
running 2 tests
test my_module::test_inc_1 ... ok
test my_module::test_inc_2 ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Запустить отдельный тест можно так:

```
$ cargo test -- --exact my_module::test_inc_1
```

> [!NOTE]
> С точки зрения организации, все юнит тесты компилируются в отдельный исполняемый бинарный файл, который можно найти в каталоге `target/debug/deps/`. Он носит имя `имя_крэйта-хеш`. Этот бинарный файл содержит в себе всё содержимое тестируемого модуля, плюс сами функции юнит-тесты, плюс код для запуска тестов и генерации отчёта.

## Интеграционные тесты

Если юнит-тесты располагаются в тех же модулях, что и тестируемые функции, то интеграционные тесты располагаются в отдельной директории пакета — `tests`. Как следствие, интеграционные тесты имеют доступ только к публичному API тестируемого крэйта.

```
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── module1.rs
│   └── module2.rs
└── tests/
    ├── integration-tests_1.rs
    └── integration-tests_2.rs
```

В отличие от юнит тестов, каждый `tests/*.rs` файл компилируется в отдельный исполняемый файл. Фактически \*.rs файл с интеграционными тестами является отдельным тестовым крэйтом.

Поэтому имеет смысл, по возможности, группировать интеграционные тесты по типу ресурсов, которые им необходимы для исполнения. Например, собрать в один файл все тесты, которым нужна только реляционная БД, в другой файл — все тесты, которым нужен только распределённый кеш.

***

Давайте посмотрим на пример, который поможет понять структуру тестов.

Объектом нашего интеграционного тестирования будет крэйт-библиотека с двумя модулями:

* `data` — функциональность для работы с хранилищем
* `user_service` — функциональность работы с записями пользователей

Создадим новый проект:

```
cargo new test_project --lib
```

Файл `src/data.rs`:

```rust,noplayground
// Отражает пользователя, который хранится в неком хранилище.
pub struct User {
    pub first_name: String,
    pub last_name: String,
}

// Объявляет интерфейс получения объекта пользователя из хранилища.
// Конкретная реализация работы с конкретным хранилищем отдаётся на откуп
// пользователю библиотеки, который должен будет реализовать этот трэйт.
pub trait DataSource {
    // Извлекает пользователя с указанным ID из хранилища
    fn find_user_by_id(&self, id: u64) -> Option<User>;
}
```

Файл `src/user_service.rs`:

```rust,noplayground
use crate::data::DataSource;

// Функция, которая принимает реализацию работы с хранилищем и ID пользователя,
// и возвращает полное имя этого пользователя
pub fn get_user_full_name(ds: &dyn DataSource, user_id: u64) -> Option<String> {
    ds.find_user_by_id(user_id)
      .map(|user| format!("{} {}", user.first_name, user.last_name))
}
```

Файл `src/lib.rs`:

```rust,noplayground
pub mod data;
pub mod user_service;
```

---

Теперь напишем интеграционный тест, в котором мы протестируем функцию `get_user_full_name` из модуля `user_service`.

Эта функция требует какую-то реализацию трэйта `DataSource`, чтобы использовать её для извлечения пользователя. В реальной программе реализацией, скорее всего, была бы интеграция с базой данных или неким identity сервисом, но для тестовых целей мы просто сделаем заглушку (mock), которая возвращает заранее подготовленные тестовые данные.

Файл `tests/user_service_test.rs`:

```rust,noplayground
use test_project::{data::{DataSource, User}, user_service::get_user_full_name};

// Заглушка для тестов
struct DataSourceMock;

impl DataSource for DataSourceMock {
    fn find_user_by_id(&self, id: u64) -> Option<User> {
        Some(User { first_name: "John".to_string(), last_name: "Doe".to_string() })
    }
}

#[test]
fn test_get_user_full_name() {
    let result = get_user_full_name(&DataSourceMock, 1);
    assert_eq!(Some("John Doe".to_string()), result);
}
```

Теперь запустим наш тест:

```
$ cargo test
     Running tests/user_service_test.rs (target/debug/deps/user_service_test-9f98ad2948b56b39)

running 1 test
test test_get_user_full_name ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Тест проходит.

***

> [!TIP]
> Если какие-то зависимости (библиотеки) необходимы только для интеграционных тестов (например, Test containers), то в `Cargo.toml` существует отдельная секция зависимостей, которые доступны только для интеграционных тестов — секция `[dev-dependencies]`.

На данном этапе нам нет смысла углубляться в интеграционные тесты. Мы поговорим о них подробнее в главе [Тест контейнеры](../web/testcontainers.md), когда будем рассматривать написание бекендов.

## cargo-nextest

Если вам не хватает гибкости стандартного `cargo test` раннера тестов, то обратите внимание на [nextest](https://nexte.st/). Этот — альтернативный раннер, который:

* выдаёт более информативный вывод о запуске тестов
* позволяет запускать легковесные тесты параллельно, а тяжеловесные — последовательно
* позволяет указывать ретраи для тестов
* позволяет генерировать отчёты в JUnit XML формате
* и много другое

Чтобы установить nextest, выполните команду:

```
cargo install cargo-nextest --locked
```

После этого вы можете запускать ваши тесты при помощи:

```
cargo nextest run
```

Сравните вывод `cargo test` и `cargo nextest`.

test:

```
$ cargo test
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.21s
     Running unittests src/main.rs (target/debug/deps/test_rust-ba9e2d97573eceb4)

running 3 tests
test test_1 ... ok
test test_2 ... ok
test test_3 ... ok

test result: ok.
3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

nextest:

```
$ cargo nextest run
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.05s
────────────
 Nextest run ID 06dccce6-29f5-45a0-9ec4-1d9328eaae82 with nextest profile: default
    Starting 3 tests across 1 binary
        PASS [   0.006s] test_rust::bin/test_rust test_3
        PASS [   0.006s] test_rust::bin/test_rust test_2
        PASS [   0.006s] test_rust::bin/test_rust test_1
────────────
     Summary [   0.007s] 3 tests run: 3 passed, 0 skipped
```

