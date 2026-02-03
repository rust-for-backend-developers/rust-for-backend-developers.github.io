# Workspace проект

Как правило, большие проекты разбивают на подпроекты.

Например, при написании back-end приложений, функциональность очень часто разбивают на слои:

* Слой взаимодействия с хранилищем данных: подключение к БД, функции с SQL запросами, структуры для работы с данными
* Слой с бизнес логикой: функции и структуры, реализующие логику программы
* Слой представления: REST/GRPC эндпоинты, web сокеты, и т.д.

Слои решают проблемы не только разбиения функциональности на подмножества, но и проблему логической изоляции этих подмножеств друг от друга. Например:

* Слой работы с данными не должен "видеть" функциональность из других модулей.
* Слой бизнес логики должен "видеть" сущности, хранимые в БД, и функции для работы с данными, но не должен видеть конкретную реализацию работы с хранилищем. И, очевидно, не должен видеть слой представления.
* А слой представления должен знать и про данные, и про бизнес функциональность, но без деталей реализации.

![](img/workspace_layered_project.svg)

Во многих языках, например в Java, такой тип проектов, состоящий из отдельных слоёв-подпроектов, принято называть многомодульными проектами (multimodule project). Но так как в Rust слово "модуль" связано с другой сущностью, такие проекты называют воркспейс проектами (workspace).

## workspace

**Workspace** — это такой тип пакета, который вместо кода, содержит в себе другие пакеты.

```
workspace-package/
  ├── Cargo.toml
  ├── пакет-1/
  │   ├── Cargo.toml
  │   └── src/
  │       └── lib.rs
  └── пакет-2/
      ├── Cargo.toml
      └── src/
          └── main.rs
```

Файл `Cargo.toml` в корневой директории имеет вид:

```toml
[workspace]
members = ["пакет-1", "пакет-2"]
```

При запуске Cargo (например, `cargo build`) в корневой workspace директории, Cargo считает `Cargo.toml`, поймёт, что мы имеем дело с workspace проектом, и проведёт сборку для всех дочерних пакетов, корректно разрешая зависимости между ними.

## Пример workspace проекта

Для примера создадим простую программу, которая печатает некий текст, а также информацию о том, сколько в этом тексте слов.

Оформим программу в виде workspace проекта, который состоит из трёх пакетов:

* Пакет 1: data (библиотека) — функциональность для получения текста
* Пакет 2: processor (библиотека) — функциональность для подсчёта количества слов
* Пакет 3: cli (исполняемый файл) — печатает текст на консоль

В удобном для вас месте, создайте новую директорию с именем `workspace_test` для нашего workspace проекта.

В этой директории создайте файл `Cargo.toml` с таким содержимым:

```toml
[workspace]
members = [ ]
```

Теперь создадим дочерние пакеты: находясь консолью внутри `workspace_test` выполните:

```sh
cargo new data --lib
cargo new processor --lib
cargo new cli --bin
```

Cargo автоматически обновит корневой `Cargo.toml` после чего он должен выглядеть так:

```toml
[workspace]
members = ["cli", "processor","data"]
```

При этом дерево файлов проекта должно иметь вид:

```
workspace_test/
  ├── Cargo.toml
  ├── data/
  │    ├── Cargo.toml
  │    └── src/
  │         └── lib.rs
  ├── processor/
  │    ├── Cargo.toml
  │    └── src/
  │         └── lib.rs
  └── cli/
       ├── Cargo.toml
       └── src/
            └── main.rs
```

---

Для начала напишем код библиотеки, которая предоставляет текст. В файле `data/src/lib.rs` напишем такое:

```rust,noplayground
const TEXT: &str = "One two three four five six seven eight nine ten.";

pub fn get_text() -> String {
    TEXT.to_string()
}
```

***

Теперь в пакете `processor` в файле `processor/Cargo.toml` добавим зависимость на пакет `data`.

```toml
[package]
name = "processor"
version = "0.1.0"
edition = "2024"

[dependencies]
data = { path = "../data" }
```

После этого в пакете `processor` можно использовать функции, импортированные из пакета `data`. Напишем функцию, которая возвращает наш текст и количество слов в нём. Файл `processor/src/lib.rs`:

```rust,noplayground
use data;

pub fn get_text_with_info() -> (String, usize) {
    let text = data::get_text();
    let words_count = text.split(" ").count();
    (text, words_count)
}
```

***

Теперь аналогичным образом добавим зависимость на `processor` в модуль `cli`.\
`cli/Cargo.toml`:

```toml
[package]
name = "cli"
version = "0.1.0"
edition = "2024"

[dependencies]
processor = { path = "../processor" }
```

И наконец, создадим исполняемую программу, которая печатает на консоль текст и количество слов в нём. Файл `cli/src/main.rs`:

```rust,noplayground
use processor;

fn main() {
    let (text, words_count) = processor::get_text_with_info();
    println!("Text: {text}");
    println!("Words count: {words_count}");
}
```

***

Всё готово. Мы можем собрать и запустить нашу программу:

```
$ cargo build
   Compiling data v0.1.0 (/home/user/project/rust/workspace_test/data)
   Compiling processor v0.1.0 (/home/user/project/rust/workspace_test/processor)
   Compiling cli v0.1.0 (/home/user/project/rust/workspace_test/cli)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.28s

$ ./target/debug/cli 
Text: One two three four five six seven eight nine ten.
Words count: 10

```

Обратите внимание, что в workspace проекте, мы запускаем программу из директории `target`, которая находится в корне проекта, а не из `cli/target`.

> [!NOTE]
> Также, в workspace проекте мы можем выполнять команду Cargo для отдельного пакета при помощи опции `-p`. Например, чтобы выпонить `cargo run` для пакета `cli`:
> 
> ```
> $ cargo run -p cli
>     Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
>      Running `target/debug/cli`
> Text: One two three four five six seven eight nine ten.
> Words count: 10
> ```

## workspace зависимости

Поскольку в workspace проекте у нас имеется несколько пакетов, легко может возникнуть ситуация, когда в разных пакетах в качестве зависимости нужна одна и те же библиотека. Например, нескольким пакетам может понадобиться библиотека _rand_.

Таким образом нам придётся быть внимательными, и при обновлении версий зависимостей, следить, чтобы версия была одинаково обновлена по всех пакетах.

В качестве решения этой проблемы, можно прописать версии зависимостей в workspace `Cargo.toml`, а в дочерних пакетах в `Cargo.toml` ссылаться на них.

Например, в workspace `Cargo.toml` объявляем версию библиотеки _rand_:

```toml
[workspace]
members = ["child_package"]

[workspace.dependencies]
rand = "0.8"
```

Теперь в дочернем пакете можно подключить зависимость _rand_ так:

```toml
[package]
name = "child_package"
version = "0.1.0"
edition = "2024"

[dependencies]
rand = { workspace = true }
```

Теперь обновление версии зависимости в корневом `Cargo.toml`, будет автоматически отражено во всех дочерних пакетах.
