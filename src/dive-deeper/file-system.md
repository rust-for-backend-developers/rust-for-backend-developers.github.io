# Файловая система

> [!NOTE]
> Работа с файловой системой редко нужна при написании бекенд приложений, поэтому мы касаемся этой темы очень поверхностно.

Для работы с файловой системой стандартная библиотека Rust предоставляет такие модули:

* [std::fs](https://doc.rust-lang.org/std/fs/index.html) — содержит функциональность для работы непосредственно с объектами файловой системы: файлами, директориями, ссылками.
* [std::io](https://doc.rust-lang.org/std/io/index.html) — содержит функциональность для работы с операциями ввода/вывода

## Чтение и запись файла

Для примера работы с файлом напишем простую программу, которая:

* открывает новый файл, записывает в него текст, закрывает файл
* открывает этот же файл, добавляет в него строку, закрывает файл
* опять открывает файл и считывает из него всё содержимое в строку

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};

fn main() -> io::Result<()> {
    {
        // Создаёт новый файл (или перезаписывает имеющийся)
        let mut file = File::create("file.txt")?;
        // Записывает в файл байты
        file.write_all("First line\n".as_bytes())?;
        file.flush()?; // Очистка буфера вывода
        file.write_all("Second line\n".as_bytes())?;
    }

    {
        // Открываем файл для добавления
        let mut file = OpenOptions::new()
            .append(true)
            .create(false)
            .open("file.txt")?;
        file.write_all("Third line\n".as_bytes())?;
    }

    {
        // Открываем файл для чтения
        let mut file = File::open("file.txt")?;
        let mut buffer = String::new();
        file.read_to_string(&mut buffer)?;
        println!("{buffer}");
    }

    Ok(())
}
```

Первое, что бросается в глаза: мы открываем файл для записи, но не закрываем его. Дело в том, что для типа `File` реализован трэйт `Drop`: при выходе объекта из скоупа деструктор сбросит буфер вывода (вызовом `flush`), а после закроет файл.

Тип `io::Result<T>`, в который завёрнут результат всех методов ввода/вывода — это псевдоним, который объявлен как:

```rust,noplyground
pub type Result<T> = result::Result<T, std::io::Error>;
```

то есть это обычный `Result`, который параметризирован стандартной ошибкой для I/O операций.

## Чтение директории

Напишем программу, которая выводит имена файлов и директорий, находящихся в текущем каталоге:

```rust
use std::{ffi::OsString, fs::FileType};

fn main() -> std::io::Result<()> {
    for read_entry in std::fs::read_dir(".")? {
        if let Ok(entry) = read_entry {
            let entry_name: OsString = entry.file_name();
            let file_type: FileType = entry.file_type()?;
            println!("{entry_name:?} {file_type:?}");
        }
    }
    Ok(())
}
```

Вывод программы:

```
"target" FileType { is_file: false, is_dir: true, is_symlink: false, .. }
"Cargo.lock" FileType { is_file: true, is_dir: false, is_symlink: false, .. }
"src" FileType { is_file: false, is_dir: true, is_symlink: false, .. }
"Cargo.toml" FileType { is_file: true, is_dir: false, is_symlink: false, .. }
```

Как видите, имя директории хранится в объекте типа `OsString`. Это важный момент, который следует разобрать подробнее.

## OsString

Почти все функции для работы с файловой системой в качестве строк используют не `String`, а [OsString](https://doc.rust-lang.org/std/ffi/struct.OsString.html). Этот тип хранит строки в том представлении, в котором строки хранятся в текущей операционной системе.

Отдельный тип строки используется, потому что:

* В Unix-подобных системах имена файлов хранятся в виде последовательности ненулевых байт, которая содержит строку в UTF-8 или другой кодировке.
* На Windows имена файлов представлены последовательностями ненулевых двухбайтных значений (UTF-16).
* Rust строки всегда хранятся в UTF-8 кодировке, причем нулевые символы допустимы.

Для `OsString` реализованы `From<String>` и `From<&str>`, которые позволяют легко преобразовать "обычную" строку в `OsString`.

```rust
use std::ffi::OsString;

fn main() {
    let string = "text";
    let os_string = OsString::from(string);
}
```

Как для типа `String` есть парный тип — строковый-слайс `&str`, так и для `OsString` есть соответствующий тип — `&OsStr`.

```rust
use std::ffi::{OsStr, OsString};

fn main() {
    let string = "text";
    let os_string = OsString::from(string);
    let os_str: &OsStr = &os_string;
}
```

***

В большинстве функций для работы с путями и именами файлов используется тип [Path](https://doc.rust-lang.org/std/path/struct.Path.html). Фактически, этот тип является просто обёрткой над `OsStr`:

```rust
pub struct Path {
    inner: OsStr,
}
```

Стандартная библиотека предоставляет `From`/`Into` и `AsRef` преобразования, которые позволяют бесшовно конвертировать Rust строки в `Path`.

Например, сигнатура метода `File::create`, который мы использовали в примере создания файла, имеет такой вид:

```rust
pub fn create<P: AsRef<Path>>(path: P) -> io::Result<File>
```

И именно потому что для `&str` определён `AsRef<Path>`, мы смогли вызвать этот метод как:

```rust
let mut file = File::create("file.txt")?;
```
