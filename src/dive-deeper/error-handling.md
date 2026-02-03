# Обработка ошибок

В главе про тип [Result](../rust-basics/result.md) мы познакомились с основами обработки ошибок. В этой главе мы рассмотрим, как ошибки принято обрабатывать в бекенд приложениях.

## thiserror

Давайте представим, что мы разрабатываем некий сервис, который отвечает за резервирование товаров. Для начала сервис будет содержать только одну функцию, которая принимает два параметра: ID товара для резервирования и желаемое количество экземпляров.

Очевидно, что в такой функциональности могут возникнуть минимум две ошибки:

* попытка резервирования неизвестного товара
* попытка зарезервировать больше экземпляров, чем имеется в наличии

Мы можем написать код этого сервиса следующим образом:

```rust,noplayground
/// Тип объект успешно созданного резерва
struct Reservation {
    reservation_id: u64,
    product_id: u64,
    quantity: u64,
}

trait ReservationService {
    /// Резервирует указанный товар в указанном количестве
    /// * ID товара
    /// * количество экземпляров
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError>;
}

enum ReserveError {
    NoSuchProduct { id: u64 },
    NotEnough { asked: u64, available: u64 },
}
```

Также из секции [про трэйт Error](../rust-basics/result.md#trait-error) мы знаем, что для типов, представляющих ошибку, рекомендуется реализовать трэйт [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html), поэтому реализуем его для нашего типа ошибки — `ReserveError`:

```rust,noplayground
#[derive(Debug)]
enum ReserveError {
  NoSuchProduct { id: u64 },
  NotEnough { asked: u64, available: u64 },
}

impl std::fmt::Display for ReserveError {
  fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
    use ReserveError::*;
    match self {
      NoSuchProduct { id } =>
        write!(f, "No product with ID {id}"),
      NotEnough {asked, available} =>
        write!(f, "Asked {asked}, but available {available}"),
    }
  }
}

impl std::error::Error for ReserveError { }
```

Нетрудно заметить, что реализация трэйта `Error` является громоздкой, и содержит в себе шаблонный код, который был бы одинаковым и в реализациях `Error` для других типов. К счастью, существует сторонняя библиотека [thiserror](https://crates.io/crates/thiserror), которая сильно упрощает создание типов ошибок.

Вот как выглядит эквивалентное определение нашего типа `ReserveError` при помощи thiserror:

```rust,noplayground
use thiserror::Error;

#[derive(Debug, Error)]
enum ReserveError {
  #[error("No product with ID {id}")]
  NoSuchProduct { id: u64 },
  #[error("Asked {asked}, but available {available}")]
  NoEnoughQuantity { asked: u64, available: u64 },
}
```

Библиотека thiserror содержит в себе процедурный макрос, который вызывается для перечислений и структур, аннотированных с `#[derive(thiserror::Error)]`. Этот макрос генерирует реализацию `std::fmt::Display` и `std::error::Error`, фактически делая то, что до этого мы сделали вручную.

Чтобы лучше понять, как это работает в целом, давайте напишем небольшую программу, использующую нашу функциональность по резервированию товаров.

Создайте новый проект:

```
cargo new test_rust
```

Добавьте thiserror в `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
thiserror = "1"
```

Теперь `src/main.rs`. Напишем реализацию хранилища, из которого мы будем резервировать товары. Для простоты будем хранить товары в хеш-таблице.

```rust
use std::{
    collections::HashMap,
    sync::{Mutex, atomic::{AtomicU64, Ordering}},
};

#[derive(Debug)]
struct Reservation {
    reservation_id: u64,
    product_id: u64,
    quantity: u64,
}

trait ReservationService {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError>;
}

#[derive(Debug, thiserror::Error)]
enum ReserveError {
    #[error("No product with ID {id}")]
    NoSuchProduct { id: u64 },
    #[error("Asked {asked}, but available {available}")]
    NotEnough { asked: u64, available: u64 },
}

// Примитивная реализация склада в виде хеш-таблицы: ID товара->количество
struct ReservationImpl {
    storage: Mutex<HashMap<u64, u64>>,
    last_id: AtomicU64,
}

impl ReservationService for ReservationImpl {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError> {
        let mut guard = self.storage.lock().unwrap();
        if let Some(stock) = guard.get_mut(&id) {
            if *stock < quantity {
                Err(ReserveError::NotEnough {
                    asked: quantity,
                    available: *stock,
                })
            } else {
                *stock -= quantity;
                Ok(Reservation {
                    reservation_id: self.last_id.fetch_add(1, Ordering::Relaxed),
                    product_id: id,
                    quantity,
                })
            }
        } else {
            Err(ReserveError::NoSuchProduct { id })
        }
    }
}

fn main() {
    let mut products = HashMap::new();
    products.insert(111, 50); // Товар 111 в количестве 50 экземпляров

    let reservation_service = ReservationImpl {
        storage: Mutex::new(products),
        last_id: AtomicU64::new(0),
    };

    println!("{:?}", reservation_service.reserve(112, 1));
    // Err(NoSuchProduct { id: 112 })

    println!("{:?}", reservation_service.reserve(111, 51));
    // Err(NoEnoughQuantity { asked: 51, available: 50 })

    println!("{:?}", reservation_service.reserve(111, 10));
    // Ok(Reservation { reservation_id: 1, product_id: 111, quantity: 10 })
}
```

## Переоборачивание ошибок

Давайте теперь расширим наше приложение: добавим еще функциональность для планирования адресной доставки и функциональность покупки, которая будет объединять в себе резервирование и доставку.

Предполагается такая логика.

![](img/error_handling-wrapping.svg)

API будет представлен тремя сервисами, каждый из которых состоит из:
* трэйта, описывающего интерфейс сервиса
* структуры для представления результата успешного вызова сервиса
* ошибки для представления проблемы в случае неуспешного вызова

Резервирование:

```rust,noplayground
struct Reservation {
    reservation_id: u64,
    product_id: u64,
    quantity: u64,
}

#[derive(Debug, thiserror::Error)]
enum ReserveError {
    #[error("No product with ID {id}")]
    NoSuchProduct { id: u64 },
    #[error("Asked {asked}, but available {available}")]
    NotEnough { asked: u64, available: u64 },
}

trait ReservationService {
    fn reserve(
        &self, id: u64, quantity: u64
    ) -> Result<Reservation, ReserveError>;
}
```

Доставка:

```rust,noplayground
struct Shipment {
    shipment_id: u64,
    address: String,
    reservation_id: u64,
}

#[derive(Debug, thiserror::Error)]
enum ShipmentError {
    #[error("Invalid address: {address}")]
    InvalidAddress { address: String },
}

trait ShipmentService {
    fn schedule_shipment(
        &self, reservation: &Reservation, address: &str
    ) -> Result<Shipment, ShipmentError>;
}
```

Покупка (резервирование + доставка):

```rust,noplayground
struct Purchase {
    purchase_id: u64,
    reservation_id: u64,
    shipment_id: u64,
}

#[derive(Debug, Error)]
enum PurchaseError {
    #[error("Nested servation error: (0)")]
    ReservationFailed(#[from] ReserveError)
    #[error("Nested shipping error: (0)")]
    ShippingFailed(#[from] ShipmentError)
}

trait PurchaseService {
    fn purchase(
        &self, id: u64, quantity: u64, addr: &str
    ) -> Result<Purchase, PurchaseError>;
}
```

Обратите внимание, что `PurchaseError` просто оборачивает ошибки от низлежащих сервисов при помощи аннотации `#[from]`. Эта аннотация говорит thiserror, что нужно сгенерировать соответствующую реализацию трэйта [From](https://doc.rust-lang.org/std/convert/trait.From.html), с которым мы познакомились в главе про [основные трэйты](common-traits.md#from-into).

Например, для примера выше будут сгенерированы:

```rust,noplayground
impl From<ReservationError> for PurchaseError { ... }
impl From<ShipmentError> for PurchaseError { ... }
```

Таким образом, "оборачивая" одну ошибку в другую, мы одновременно и сохраняем информацию о причинах возникновения проблемы, и имеем ошибки, специфичные для текущего API.

Давайте расширим наш пример так, чтобы продемонстрировать, как ошибка, возникшая в низлежащих `ReservationService` и `ShipmentService`, возвращается из метода `PurchaseService::purchase` обёрнутой в `PurchaseError`.

Код для `src/main.rs`:

```rust,edition2024
# use thiserror;
use std::{
    collections::HashMap,
    sync::{Arc, Mutex, atomic::{AtomicU64, Ordering}},
};

// ----- Код для функциональности резервирования
#[derive(Debug)]
struct Reservation {
    id: u64,
    product_id: u64,
    quantity: u64,
}

#[derive(Debug, thiserror::Error)]
enum ReserveError {
    #[error("No product with ID {id}")]
    NoSuchProduct { id: u64 },
    #[error("Asked {asked}, but available {available}")]
    NotEnough { asked: u64, available: u64 },
}

trait ReservationService {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError>;
}

struct ReservationImpl {
    storage: Mutex<HashMap<u64, u64>>,
    last_id: AtomicU64,
}

impl ReservationImpl {
    fn new(storage: HashMap<u64, u64>) -> ReservationImpl {
        ReservationImpl {
            storage: Mutex::new(storage),
            last_id: AtomicU64::new(0),
        }
    }
}

impl ReservationService for ReservationImpl {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError> {
        let mut guard = self.storage.lock().unwrap();
        if let Some(stock) = guard.get_mut(&id) {
            if *stock < quantity {
                Err(ReserveError::NotEnough {
                    asked: quantity,
                    available: *stock,
                })
            } else {
                *stock -= quantity;
                Ok(Reservation {
                    id: self.last_id.fetch_add(1, Ordering::Relaxed),
                    product_id: id,
                    quantity,
                })
            }
        } else {
            Err(ReserveError::NoSuchProduct { id })
        }
    }
}

// ----- Код для функциональности доставки
struct Shipment {
    id: u64,
    address: String,
    reservation_id: u64,
}

#[derive(Debug, thiserror::Error)]
enum ShipmentError {
    #[error("Invalid address: {address}")]
    InvalidAddress { address: String },
}

trait ShipmentService {
    fn schedule_shipment(
        &self,
        reservation: &Reservation,
        address: &str,
    ) -> Result<Shipment, ShipmentError>;
}

#[derive(Debug)]
struct ShipmentImpl {
    last_id: AtomicU64,
}

impl ShipmentImpl {
    fn new() -> ShipmentImpl {
        ShipmentImpl {
            last_id: AtomicU64::new(0),
        }
    }
}

impl ShipmentService for ShipmentImpl {
    fn schedule_shipment(
        &self, reservation: &Reservation, address: &str,
    ) -> Result<Shipment, ShipmentError> {
        if address.split(" ").count() < 2 {
            Err(ShipmentError::InvalidAddress { address: address.to_string() })
        } else {
            Ok(Shipment {
                id: self.last_id.fetch_add(1, Ordering::Relaxed),
                address: address.to_string(),
                reservation_id: reservation.id,
            })
        }
    }
}

// ----- Код для функциональности покупки
#[derive(Debug)]
struct Purchase {
    id: u64,
    reservation_id: u64,
    shipment_id: u64,
}

#[derive(Debug, thiserror::Error)]
enum PurchaseError {
    #[error("Nested servation error: (0)")]
    ReservationFailed(#[from] ReserveError),
    #[error("Nested shipping error: (0)")]
    ShippingFailed(#[from] ShipmentError),
}

trait PurchaseService {
    fn purchase(
        &self, id: u64, quantity: u64, addr: &str
    ) -> Result<Purchase, PurchaseError>;
}

struct PurchaseImpl {
    reservation_service: Arc<dyn ReservationService>,
    shipment_service: Arc<dyn ShipmentService>,
    last_id: AtomicU64,
}

impl PurchaseImpl {
    fn new(
        reservation_service: Arc<dyn ReservationService>,
        shipment_service: Arc<dyn ShipmentService>,
    ) -> PurchaseImpl {
        PurchaseImpl {
            reservation_service,
            shipment_service,
            last_id: AtomicU64::new(0),
        }
    }
}

impl PurchaseService for PurchaseImpl {
    fn purchase(
        &self, id: u64, quantity: u64, addr: &str
    ) -> Result<Purchase, PurchaseError> {
        let reservation = self.reservation_service.reserve(id, quantity)?;
        let shipment = self
            .shipment_service
            .schedule_shipment(&reservation, addr)?;
        Ok(Purchase {
            id: self.last_id.fetch_add(1, Ordering::Relaxed),
            reservation_id: reservation.id,
            shipment_id: shipment.id,
        })
    }
}

// Подготовка заглушек экземпляров сервисов
fn initialize_purchase_service() -> Arc<dyn PurchaseService> {
    let mut products = HashMap::new();
    products.insert(111, 50); // Товар 111 в количестве 50 экземпляров
    let reservation_service = ReservationImpl::new(products);

    let shipment_service = ShipmentImpl::new();

    let purchase_service =
        PurchaseImpl::new(Arc::new(reservation_service), Arc::new(shipment_service));

    Arc::new(purchase_service)
}

fn main() {
    let purchase_service = initialize_purchase_service();

    println!("{:?}", purchase_service.purchase(112, 1, "addr 1"));
    // Err(ReservationFailed(NoSuchProduct { id: 112 }))

    println!("{:?}", purchase_service.purchase(111, 51, "addr 1"));
    // Err(ReservationFailed(NotEnough { asked: 51, available: 50 }))

    println!("{:?}", purchase_service.purchase(111, 10, "invalid"));
    // Err(ShippingFailed(InvalidAddress { address: "invalid" }))

    println!("{:?}", purchase_service.purchase(111, 10, "addr 1"));
    // Ok(Purchase { id: 0, reservation_id: 1, shipment_id: 0 })
}
```



## Box\<dyn Error>

Существует ряд ситуаций, когда отсутствует возможность как-то корректно обработать ошибку. Например, если в бэкенд-приложении в процессе обработки клиентского запроса возникает ошибка, то довольно часто единственное, что можно сделать — это залогировать ошибку и ответить клиенту 500-м HTTP кодом.

В этом случае переоборачивание ошибок может оказаться бесполезной тратой усилий или даже, наоборот, усложнить код. Поэтому вместо переоборачивания ошибок можно просто "пробрасывать их наверх" в виде "обезличенного" трэйт-объекта `Box<dyn Error>`.

Рассмотрим пример: мы напишем две функции, которые возвращают два разных типа ошибок, и функцию, которая внутри себя вызывает эти две функции и пробрасывает полученные от них ошибки в виде `Box<dyn std::error::Error>>`.

```rust
#[derive(Debug, thiserror::Error)]
#[error("Error A")]
struct ErrA;

#[derive(Debug, thiserror::Error)]
#[error("Error B")]
struct ErrB;

fn fail_a() -> Result<(), ErrA> {
    Err(ErrA)
}
fn fail_b() -> Result<(), ErrB> {
    Err(ErrB)
}

fn fail_something(is_a: bool) -> Result<(), Box<dyn std::error::Error>> {
    if is_a {
        // Оператор ? перепакует ошибку в ожидаемый тип - Box<dyn std::error::Error>>)
        // как если бы мы явно вызвали:
        // fail_a().map_err(|e| Box::new(e) as Box<dyn std::error::Error>)
        let r = fail_a()?;
        Ok(r)
    } else {
        let r = fail_b()?;
        Ok(r)
    }
}

fn main() {
    if let Err(e) = fail_something(true) {
        println!("Underlying error: {}", e.to_string());
    }
    if let Err(e) = fail_something(false) {
        println!("Underlying error: {}", e.to_string());
    }
}
```

Программы выводит:

```
Underlying error: Error A
Underlying error: Error B
```

Итак, теперь мы умеем возвращать любую ошибку, не заботясь о её конкретном типе. Но иногда есть необходимость отдельно обработать какой-то один вид ошибок. Это можно сделать при помощи метода [downcast_ref](https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast_ref), который определён для трэйт-объекта `dyn Error`.

```rust
# #[derive(Debug, thiserror::Error)]
# #[error("Error A")]
# struct ErrA;
# 
# #[derive(Debug, thiserror::Error)]
# #[error("Error B")]
# struct ErrB;
# 
# fn fail_a() -> Result<(), ErrA> {
#     Err(ErrA)
# }
# fn fail_b() -> Result<(), ErrB> {
#     Err(ErrB)
# }
# 
# fn fail_something(is_a: bool) -> Result<(), Box<dyn std::error::Error>> {
#     if is_a {
#         // Оператор ? перепакует ошибку в ожидаемый тип - Box<dyn std::error::Error>>)
#         // как если бы мы явно вызвали:
#         // fail_a().map_err(|e| Box::new(e) as Box<dyn std::error::Error>)
#         let r = fail_a()?;
#         Ok(r)
#     } else {
#         let r = fail_b()?;
#         Ok(r)
#     }
# }
# 
fn main() {
    if let Err(e) = fail_something(true) {
        if let Some(err_a) = e.downcast_ref::<ErrA>() {
            println!("Handle ErrA separately: {err_a}")
        } else {
            println!("Underlying error: {}", e.to_string());
        }
    }
}
```

## Anyhow

Если вы не в восторге от ручной работы с `Box<dyn std::error::Error>`, то экосистема Rust предлагает библиотеку [anyhow](https://crates.io/crates/anyhow), которая упрощает работу с "обезличенными" ошибками.

anyhow предлагает свой тип "обезличенной" ошибки [anyhow::Error](https://docs.rs/anyhow/latest/anyhow/struct.Error.html) и свой тип результата [anyhow::Result\<T>](https://docs.rs/anyhow/latest/anyhow/type.Result.html), который является псевдонимом для `std::result::Result<T, anyhow::Error>`.

Для начала рассмотрим простейшую программу, которая демонстрирует, как anyhow встраивается в процесс обработки ошибок.

```rust,noplayground
#[derive(Debug, thiserror::Error)]
#[error("My custom error")]
struct MyError;

fn fail_with_specific_error() -> Result<(), MyError> { // вернёт специфичную ошибку
    Err(MyError)
}

fn call_failable() -> anyhow::Result<()> { // вернёт обезличенную anyhow::Error
    let r = fail_with_specific_error()?;
    Ok(r)
}

fn main() {
    match call_failable() {
        Ok(_) => println!("It was fine"),
        Err(e) => {
            if let Some(_my_err) = e.downcast_ref::<MyError>() {
                eprintln!("It failed MyError")
            } else {
                eprintln!("It failed with: {}", e.root_cause())
            }
        }
    }
}
```

Как видите, в теле функции `call_failable` конкретная ошибка перепаковывается в `anyhow::Error`, подобно тому, как мы уже перепаковывали конкретную ошибку в `Box<dyn Error>` в [предыдущей секции](error-handling.md#boxdyn-error).

Так какие же преимущества предлагает anyhow?

### backtrace

`anyhow::Error` не просто оборачивает ошибку, но также может добавить бэктрэйс, который позволит легко идентифицировать точное место возникновения ошибки.

По умолчанию бэктрэйсы не создаются (так как это ресурсозатратный процесс) и чтобы их включить, необходимо перед запуском программы выставить переменную окружения `RUST_LIB_BACKTRACE=1`.

Давайте перепишем предыдущий пример так, чтобы в нём выводился бэктрэйс:

```rust,noplayground
#[derive(Debug, thiserror::Error)]
#[error("My custom error")]
struct MyError;

fn fail_with_specific_error() -> Result<(), MyError> {
    Err(MyError)
}

fn call_failable() -> anyhow::Result<()> {
    let r = fail_with_specific_error()?;
    Ok(r)
}

fn main() {
    match call_failable() {
        Ok(_) => println!("It was fine"),
        Err(e) => {
            eprintln!("It failed with: {}", e.root_cause());
            eprintln!("Backtrace:\n{}", e.backtrace());
        }
    }
}
```

Перед запуском выставляем переменную окружения (в Windows cmd это делается командой `set RUST_LIB_BACKTRACE=1`):

```
$ export RUST_LIB_BACKTRACE=1
$ cargo run
It failed with: My custom error
Backtrace:
   0: anyhow::error::<impl core::convert::From<E> for anyhow::Error>::from
             at /home/stas/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/anyhow-1.0.100/src/backtrace.rs:27:14
   1: <core::result::Result<T,F> as core::ops::try_trait::FromResidual<core::result::Result<core::convert::Infallible,E>>>::from_residual
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/result.rs:2177:27
   2: test_rust::call_failable
             at ./src/main.rs:10:13
   3: test_rust::main
             at ./src/main.rs:15:11
   4: core::ops::function::FnOnce::call_once
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250:5
   5: std::sys::backtrace::__rust_begin_short_backtrace
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/backtrace.rs:158:18
   6: std::rt::lang_start::{{closure}}
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/rt.rs:206:18
   7: core::ops::function::impls::<impl core::ops::function::FnOnce<A> for &F>::call_once
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/core/src/ops/function.rs:287:21
   8: std::panicking::catch_unwind::do_call
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:590:40
   9: std::panicking::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:553:19
  10: std::panic::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panic.rs:359:14
  11: std::rt::lang_start_internal::{{closure}}
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/rt.rs:175:24
  12: std::panicking::catch_unwind::do_call
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:590:40
  13: std::panicking::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:553:19
  14: std::panic::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panic.rs:359:14
  15: std::rt::lang_start_internal
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/rt.rs:171:5
  16: std::rt::lang_start
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/rt.rs:205:5
  17: main
  18: __libc_start_call_main
             at ./csu/../sysdeps/nptl/libc_start_call_main.h:58:16
  19: __libc_start_main_impl
             at ./csu/../csu/libc-start.c:360:3
  20: _start
```

Во втором элементе цепочки бэктрэйса видно, что конкретная ошибка была перепакована в `anyhow::Error` в строке `./src/main.rs:10:13`.

### context

Иногда текстовое описание оригинальной ошибки является малоинформативным. anyhow позволяет добавить к ошибке дополнительное текстовое описание — контекст (context). Потом этот контекст может будет получить просто вызовом `to_string()` на объекте ошибки `anyhow::Error`.

Чтобы добавить к ошибке контекст, надо использовать метод `.context()` на результате (объекте `Result`).

```rust,noplayground
fn my_func() -> anyhow::Result<Тип> {
    let result = func_that_can_fail()
            .context("информативное описание")?;
    Ok(result)
}
```

Рассмотрим пример: напишем программу, которая пытается считать несуществующий файл, и используем контекст для указания информации о том, какой файл не удалось считать.

```rust,noplayground
use anyhow::Context;

fn read_non_existing_file() -> anyhow::Result<String> {
    let text = std::fs::read_to_string("non_existing_file.txt")
        .context("Cannot read non_existing_file.txt")?;
    Ok(text)
}

fn main() {
    match read_non_existing_file() {
        Ok(text) => println!("File content: {text}"),
        Err(e) => {
            eprintln!("Failed with error: {}", e.root_cause());
            eprintln!("Error context: {}", e.to_string());
        }
    }
}
```

Запускаем:

```
$ cargo run
Failed with error: No such file or directory (os error 2)
Error context: Cannot read non_existing_file.txt
```

Как видите, если бы мы не использовали контекст, то получили бы малоинформативное описание ошибки "No such file or directory (os error 2)". Однако при помощи контекста мы смогли указать, какой именно файл отсутствует.

## Общие рекомендации по работе с ошибками

* Когда вы пишете библиотеку, то рекомендуется использовать конкретные и детальные типы ошибок. Для облегчения создания типов ошибок рекомендуется использовать thiserror.
* При написании конечного приложения в участках, где нет возможности должным образом обработать каждый тип ошибки отдельно, используйте anyhow, так как он сильно упрощает работу с ошибками.
