# Несколько исполняемых файлов

Как мы знаем, когда мы создаём исполняемую программу, то главным файлом является `src/main.rs`. Но что делать, если мы хотим сделать несколько исполняемых файлов, которые переиспользуют одни и те же модули?

В таком случае мы можем создать сколько угодно дополнительных главных файлов в каталоге `src/bin`.

Например, нам нужны три утилиты, работающие с последовательностью Фибоначчи:

```
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
```

* Первая утилита будет печатать печатать первые 10 элементов последовательности.
* Вторая будет печатать сумму первых 10 элементов последовательности.
* Третья будет печатать произведение элементов со 2-го по 10-й (потому что произведение на 0 всегда даёт 0).

Создадим новый Cargo проект:

```
cargo new fibonacci_util
```

Никаких зависимостей у нас не будет, поэтому `Cargo.toml` мы не трогаем.

Создаём модуль `src/fibonacci.rs` с реализацией генерации последовательности Фибоначчи в виде итератора:

```rust
// Сколько элементов из последовательности Фибоначчи мы хотим взять
pub struct FibonacciSequence(pub usize);

// Итератор для последовательности Фибоначчи заданной длины
pub struct FibonacciIter {
    current_index: usize,
    len: usize,
    prev: u64,
    before_prev: u64,
}

impl Iterator for FibonacciIter {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        match self.current_index {
            0 => {
                self.current_index += 1;
                Some(0)
            },
            1 => {
                self.current_index += 1;
                Some(1)
            },
            n if n < self.len => {
                let nth = self.before_prev + self.prev;
                self.before_prev = self.prev;
                self.prev = nth;
                self.current_index += 1;
                Some(nth)
            },
            _ => None,
        }
    }
}

impl IntoIterator for FibonacciSequence {
    type Item = u64;
    type IntoIter = FibonacciIter;

    fn into_iter(self) -> Self::IntoIter {
        FibonacciIter {
            current_index: 0,
            len: self.0,
            prev: 1,
            before_prev: 0,
        }
    }
}
```

Поскольку эта функциональность будет использоваться тремя программами, оформим её как библиотеку. Создадим `src/lib.rs`:

```rust
pub mod fibonacci;
```

К этому моменту дерево файлов должно выглядеть так:

```
fibonacci_util/
├── Cargo.toml
└── src/
    ├── fibonacci.rs
    ├── lib.rs
    └── main.rs
```

---

Теперь настало время первой программы, которая просто печатает 10 элементов последовательности. Поместим её в `src/main.rs`:

```rust,noplayground
use fibonacci_util::fibonacci::FibonacciSequence;

fn main() {
    let s = FibonacciSequence(10).into_iter().collect::<Vec<_>>();
    println!("First 10 elements of fibonacci sequence: {s:?}");
}
```

Здесь мы видим кое-что новое: вместо того, чтобы подключать модули через `mod`, мы просто импортировали модуль через `use`, причём первой компонентой имени является имя самого проекта (крэйта) — `fibonacci_util`. Это стало возможным, так как у нас уже есть библиотека `lib.rs`, которая по умолчанию подключена с тем же именем, что и у нашего крэйта, и которая импортирует модуль `fibonacci`. Таким образом, обращаясь к имени текущего проекта `fibonacci_util`, мы обращаемся к нашей библиотеке `lib.rs`.

Могли ли мы вместо этого, как раньше, просто подключить модуль `fibonacci` в наш `main.rs`?

```rust,noplayground
mod fibonacci;
use fibonacci::FibonacciSequence;

fn main() {
    let s = FibonacciSequence(10).into_iter().collect::<Vec<_>>();
    println!("First 10 elements of fibonacci sequence: {s:?}");
}
```

Да, могли. Такой код тоже работает. Но этот вариант не подойдёт для следующих двух утилит, которые мы напишем.

В Cargo проекте может быть только один основной главный файл программы — `src/main.rs`. И только он может подключать в себя модули через `mod`. Дополнительные главные файлы исполняемых программ помещаются в каталог `/src/bin`, и для них не доступны модули, которые располагаются непосредственно в `src/`, поэтому они могут обращаться к ним только через `src/lib.rs`.

---

Создадим нашу вторую утилиту, которая печатает сумму первых 10 элементов последовательности — `src/bin/fibonacci_sum.rs`:

```rust,noplayground
use fibonacci_util::fibonacci::FibonacciSequence;

fn main() {
    let sum: u64 = FibonacciSequence(10)
        .into_iter()
        .sum();
    println!("Sum of first 10 elements of fibonacci sequence: {sum}");
}
```

Как мы видим, функциональность из библиотеки `src/lib.rs` доступна через пространство имён `fibonacci_util` точно таким же образом, как и в нашем `src/main.rs`. Как мы уже сказали, для главных файлов в `src/bin` никакие модули из `src/` напрямую не доступны, и единственный способ использовать эти модули — посредством библиотеки `lib.rs`.

---

Аналогично создадим последнюю утилиту, которая печатает произведение элементов последовательности со 2-го по 10-й — `src/bin/fibonacci_prod.rs`:

```rust,noplayground
use fibonacci_util::fibonacci::FibonacciSequence;

fn main() {
    let prod: u64 = FibonacciSequence(10).into_iter()
        .skip(1) // Отбрасываем первый нулевой элемент
        .product();
    println!("Product of fibonacci sequence elements from 2nd to 10th: {prod}");
}
```

К этому моменту, дерево файлов проекта должно выглядеть так:

```
fibonacci_util/
├── Cargo.toml
└── src/
    ├── bin/
    │   ├── fibonacci_sum.rs
    │   └── fibonacci_prod.rs
    ├── fibonacci.rs
    ├── lib.rs
    └── main.rs
```

Выполнив команду сборки `cargo build`, мы должны увидеть в `target/debug`:

* `libfibonacci_util.rlib` — Rust библиотека, собранная из `src/lib.rs`
* `fibonacci_util` — исполняемый файл, собранный из `src/main.rs`
* `fibonacci_sum` — исполняемый файл, собранный из `src/bin/fibonacci_sum.rs`
* `fibonacci_prod` — исполняемый файл, собранный из `src/bin/fibonacci_prod.rs`

Если мы хотим собрать только какой-то конкретный исполняемый файл, то мы должны указать его при помощи опции `--bin`. Например, чтобы собрать только `fibonacci_sum`:

```
cargo build --bin fibonacci_sum
```

## [[bin]]

По умолчанию исполняемые файлы будут иметь те же имена, как и у соответствующих им `src/bin/*.rs` файлов. Однако имена можно изменить при помощи секции `[[bin]]` в файле `Cargo.toml`.

Допустим, мы хотим, чтобы `src/bin/fibonacci_sum.rs` собирался в исполняемый файл с именем `sum_10`, а `src/bin/fibonacci_prod.rs` — в файл `prod_10`.

```toml
[package]
name = "fibonacci_util"
version = "0.1.0"
edition = "2024"

[[bin]]
name = "sum_10"
path = "src/bin/fibonacci_sum.rs"

[[bin]]
name = "prod_10"
path = "src/bin/fibonacci_prod.rs"

[dependencies]
```

Теперь, после выполнения `cargo build`, в директории `target/debug` мы увидим `sum_10` и `prod_10`.

Мы также можем задать другое имя и для исполняемого файла, который собирается из `src/main.rs`:

```toml
[package]
name = "fibonacci_util"
version = "0.1.0"
edition = "2024"

[[bin]]
name = "sequence_10"
path = "src/main.rs"

[[bin]]
name = "sum_10"
path = "src/bin/fibonacci_sum.rs"

[[bin]]
name = "prod_10"
path = "src/bin/fibonacci_prod.rs"

[dependencies]
```

