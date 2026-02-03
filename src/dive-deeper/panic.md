# Паника

## panic

Когда в программе происходит какое-то недопустимое действие, например, деление на ноль или вызов `unwrap()` на объекте `None`, то происходит паника.

Например:

```rust,should_panic
fn main() {
    let a: Option<i32> = None;
    a.unwrap();
}
```

выведет

```
$ cargo run
thread 'main' panicked at src/main.rs:3:7:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Паника аварийно завершает программу и печатает на стандартный вывод информацию о случившейся ошибке.

Панику также можно инициировать вручную при помощи макроса [panic](https://doc.rust-lang.org/std/macro.panic.html). Синтаксис его использования такой же, как у макроса `println!`, но вместо вывода на консоль, он завершит программу и напечатает переданное сообщение.

Например, такой код:

```rust,should_panic
fn main() {
    let a = 5;
    panic!("Ending the program here. Num: {}", a);
}
```

выведет:

```
$ cargo run
thread 'main' panicked at src/main.rs:3:5:
Ending the program here. Num: 5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

## Перехват паники

По умолчанию паника аварийно завершает выполнение программы. Однако это не всегда является желаемым поведением. Например, при разработке веб-сервера при возникновении паники в обработчике запроса от пользователя, как правило, мы предпочитаем просто возвращать HTTP 500, а не завершать работу всего сервера.

Стандартная библиотека предоставляет функцию-обёртку [catch_unwind](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html), которая принимает на вход замыкание и выполняет его.

```rust,noplayground
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R>
```

Если замыкание отрабатывает без ошибок, то его результат возвращается завёрнутым в `Ok`, если же при выполнении возникла паника, то будет возвращено `Err`.

> [!NOTE]
> Напомним, что если функция ожидает в качестве аргумента `FnOnce` замыкание, то в неё можно передавать и `FnMut`, и `Fn`, и просто указатель на функцию `fn`.

Рассмотрим пример:

```rust
use std::panic;
use std::panic::catch_unwind;
use std::any::Any;

fn main() {
    let result: Result<(), Box<dyn Any + Send + 'static>> = catch_unwind(|| {
        panic!("My panic msg");
    });
    println!("Result: '{:?}'", result);
    println!("Continue working...");
}
```

эта программа печатает

```
$ cargo run
thread 'main' panicked at src/main.rs:5:9:
My panic msg
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Result: 'Err(Any { .. })'
Continue working...
```

Тот факт, что `catch_unwind` возвращает панику как `Err`, содержащий `Any`, словно намекает, что паника может содержать некий объект. И это так. Для случаев, когда через панику надо передать объект, существует функция [panic_any](https://doc.rust-lang.org/std/panic/fn.panic_any.html), которая принимает объект любого типа, реализующего `Any`.

```rust
pub fn panic_any<M: 'static + Any + Send>(msg: M) -> !
```

Для примера напишем простую функцию, которая эмулирует обработчик запроса от пользователя. Эта функция будет принимать текстовый запрос и возвращать текстовый ответ. В случае возникновения ошибки, функция может выбросить панику, содержащую объект, который описывает проблему.

```rust
use std::{any::Any, panic, panic::panic_any};

/// Тип для ошибки, которая будет пробрасываться через панику
#[derive(Debug)]
enum ProcessingErr {
    NotAuthorized,
    AnotherImportantError,
}

/// Некий обработчик ошибки, который может паниковать
fn serve_request(req: &str) -> String {
    panic_any(ProcessingErr::NotAuthorized);
}

fn main() {
    let request = "Some request"; // Эмулирует некий запрос

    let closure_result: Result<String, Box<dyn Any + Send + 'static>> =
        panic::catch_unwind(|| serve_request(request));

    if let Err(a) = closure_result {
        if let Some(panic_obj) = a.downcast_ref::<ProcessingErr>() {
            println!("Panic object: {panic_obj:?}");
        }
    }
}
```

Программа напечатает:

```
# cargo run
thread 'main' panicked at src/main.rs:12:5:
Box<dyn Any>
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Panic object: NotAuthorized
```

Таким образом, мы смогли через панику получить информацию о проблеме, которая привела к ошибке.

> [!CAUTION]
> Имейте в виду, что мы написали такую программу исключительно для демонстрации возможностей проброса объектов через панику. Использовать панику подобно исключениям для обработки ошибок — очень плохая затея. Паника должна использоваться только для серьёзных проблем, которые не являются нормой. Для остальных случаев используйте `Result`.

## Обработчик паники

Как мы заметили в примере выше, даже при том, что мы перехватываем панику функцией `catch_unwind`, на консоль всё равно печатается сообщение паники, и место, откуда она была выброшена:

```
thread 'main' panicked at src/main.rs:5:9:
My panic msg
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Так происходит, потому что по умолчанию для паники установлен стандартный предобработчик, который выполняется до того, как паника будет перехвачена в `catch_unwind`.

Если нам нужно другое поведение, то мы можем задать свой предобработчик для паники при помощи функции [set_hook](https://doc.rust-lang.org/std/panic/fn.set_hook.html).

```rust
use std::panic::{self, PanicHookInfo};

fn my_panic_hook<'a>(p: &PanicHookInfo<'a>) {
    println!("Hello from panic hook");
    println!("Location: {:?}", p.location());
    println!("Payload: {:?}", p.payload());
}

fn main() {
    panic::set_hook(Box::new(my_panic_hook));

    panic!("Original panic msg");
}
```

Такая программа напечатает:

```
$cargo run
Hello from panic hook
Location: Some(Location { file: "src/main.rs", line: 12, column: 5 })
Payload: Any { .. }
```

## Макросы, инициирующие панику

Чтобы завершить тему макросов, стоит упомянуть несколько макросов, которые также инициируют панику.

### todo и unimplemented

Макросы [todo](https://doc.rust-lang.org/std/macro.todo.html) и [unimplemented](https://doc.rust-lang.org/std/macro.unimplemented.html) служат одной и той же цели: они являются заполнителями для кода, который еще не имплементирован. Отличительной особенностью этих макросов является то, что они "возвращают" тот тип, который от них ожидается. Например:

```rust
fn my_func() -> String {
    todo!("Don't forget to implement")
}

fn main() {
    let s = my_func();
    println!("{s}");
}
```

Этот код компилируется без ошибок, но при попытке выполнить программу она паникует:

```
$ cargo run
thread 'main' panicked at src/main.rs:2:5:
not yet implemented: Don't forget to implement
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Функция `my_func` возвращает значение типа `String`, поэтому вызов макроса `todo` как бы возвращает тип `String`. Это очень удобно, так как можно вставлять `todo` в любом месте, какой бы сложный тип там ни ожидался, и компилятор будет удовлетворён.

Причём `todo` можно использовать в любом контексте, а не только при возврате значений из функции:

```rust
let r: Result<(), Box<dyn Error>> = todo!();
```

Макрос `unimplemented` работает точно таким же образом, как и `todo`, различие исключительно семантическое: `todo` подразумевает, что функциональность должна быть реализована в текущей итерации разработки, а `unimplemented` предполагает, что функциональность может быть реализована позже.

### unreachable

Макрос [unreachable](https://doc.rust-lang.org/std/macro.unreachable.html) также инициирует панику. Однако в отличие от `todo` и `unimplemented`, он используется для участков кода, которые чисто технически могут быть выполнены, но на практике выполнение никогда не должно доходить до них (обычно вследствие предварительной проверки аргументов).

Например:

```rust
/// Данная функция ожидает версию API либо 1, либо 2.
/// Проверка версии API ложится на вызывающий код
fn get_user_by_id(id: u64, api_version: u32) {
    match api_version {
        1 => v1_get_user_by_id(id),
        2 => v2_get_user_by_id(id),
        _ => unreachable!("Unknown api version"),
    }
}
```
