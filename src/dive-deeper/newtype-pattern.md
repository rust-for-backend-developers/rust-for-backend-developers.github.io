# Newtype паттерн

Одно из самых неудобных ограничений Rust — Orphan rule, о котором мы уже упоминали в главе [Трэйты](../rust-basics/traits.md#impl-for-foreign-types). Напомним, Orphan Rule гласит:

> трэйт можно реализовать для типа только в том случае, если либо трэйт, либо тип (либо оба) принадлежит крэйту (библиотеке или программе), в которой осуществляется реализация.

Другими словами, если мы хотим реализовать трэйт A для типа B, то код реализации должен располагаться либо в крэйте, где объявлен тип B, либо в крэйте, где объявлен трэйт A.

А теперь давайте представим, что у нас есть задача, в рамках которой нам надо сортировать вектор с объектами файлов:

```rust,compile_fail
use std::fs::File;

fn main() {
    let mut v = vec![
        File::open("/etc/fstab").unwrap(),
        File::open("/etc/resolv.conf").unwrap(),
        File::open("/etc/hosts").unwrap(),
    ];
    v.sort();
}
```

> [!NOTE]
> Для пользователей, не знакомых с Linux:
> 
> * `/etc/fstab` — стандартный файл конфигурации разделов жесткого диска
> * `/etc/resolv.conf` — файл с адресами DNS серверов
> * `/etc/hosts` — файл для задания соответствий доменных имён и IP адресов
> 
> Автор взял эти файлы без какого-то специального умысла. Проверяя примеры из главы, вы вольны использовать любые имеющиеся у вас файлы или создать новые.

Метод `.sort()` требует, чтобы тип сортируемых объектов реализовал трэйт [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html). Тип [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) не реализует `Ord`, поэтому компиляция завершится с ошибкой:

```
error[E0277]: the trait bound `File: Ord` is not satisfied
   --> src/main.rs:8:7
    |
  8 |     v.sort(); // the trait `Ord` is not implemented for `File`
    |       ^^^^ the trait `Ord` is not implemented for `File`
```

Допустим, мы хотим реализовать `Ord` для `File` так, чтобы сортировка происходила на основании размера файла. Здесь и проявляется Orphan Rule: и тип `File`, и трэйт `Ord` объявлены не в нашем крэйте, а в стандартной библиотеке.

Стандартным решением этой проблемы является **Newtype паттерн**. Смысл его заключается в том, что мы оборачиваем "чужой" тип в кортежную структуру, и получается, что эта обёртка уже располагается в нашем крэйте.

```rust
struct Обёртка(ЧужойТип);
```

Теперь для этой обёртки мы можем реализовать `Ord`. Давайте сделаем это, и сразу напишем нашу программу для сортировки файлов по возрастанию их размера.

```rust
use std::{cmp::Ordering, fs::File};

// Newtype обёртка для File.
struct FileWrapper(File);

impl PartialEq for FileWrapper {
    fn eq(&self, other: &Self) -> bool {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len() == m2.len(),
            _ => false,
        }
    }
}
impl Eq for FileWrapper {}

impl Ord for FileWrapper {
    fn cmp(&self, other: &Self) -> Ordering {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len().cmp(&m2.len()),
            _ => Ordering::Equal,
        }
    }
}

impl PartialOrd for FileWrapper {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut v: Vec<FileWrapper> = vec![
        FileWrapper(File::open("/etc/fstab").unwrap()),
        FileWrapper(File::open("/etc/resolv.conf").unwrap()),
        FileWrapper(File::open("/etc/hosts").unwrap()),
    ];

    println!("Before sorting");
    for file in v.iter() {
        println!("Size: {}", file.0.metadata().unwrap().len());
    }

    v.sort();

    println!("After sorting");
    for file in v.iter() {
        println!("Size: {}", file.0.metadata().unwrap().len());
    }
}
```

Запустим программу:

```
$ cargo run
Before sorting
Size: 866
Size: 920
Size: 219
After sorting
Size: 219
Size: 866
Size: 920
```

Всё работает. Однако, как вы могли заметить, нам приходится явно запаковывать объекты `File` в обёртку `FileWrapper`. К тому же для доступа к объекту файла внутри обёртки, нам приходится использовать индекс `.0` (в примере: `file.0.metadata()`), что не очень элегантно.

Для решения этой проблемы, при использовании newtype паттерна, для типа обёртки принято реализовывать трэйты `From` и `Deref`. Это позволит:

* заворачивать оригинальный тип в обёртку вызовом метода `.into()`
* обращаться к обёрнутому объекту без использования поля `.0`

Полный пример программы:

```rust
use std::{cmp::Ordering, fs::File, ops::Deref};

struct FileWrapper(File);

impl PartialEq for FileWrapper {
    fn eq(&self, other: &Self) -> bool {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len() == m2.len(),
            _ => false,
        }
    }
}
impl Eq for FileWrapper {}

impl Ord for FileWrapper {
    fn cmp(&self, other: &Self) -> Ordering {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len().cmp(&m2.len()),
            _ => Ordering::Equal,
        }
    }
}

impl PartialOrd for FileWrapper {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl From<File> for FileWrapper {
    fn from(value: File) -> Self {
        FileWrapper(value)
    }
}

impl Deref for FileWrapper {
    type Target = File;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let mut v: Vec<FileWrapper> = vec![
        // Вызовом .into() преобразовываем File в Newtype обёртку
        File::open("/etc/fstab").unwrap().into(),
        File::open("/etc/resolv.conf").unwrap().into(),
        File::open("/etc/hosts").unwrap().into(),
    ];

    println!("Before sorting");
    for file in v.iter() {
        // Ниже мы вызываем file.metadata(), а не file.0.metadata()
        // так как мы реализовали Deref
        println!("Size: {}", file.metadata().unwrap().len());
    }

    v.sort();

    println!("After sorting");
    for file in v.iter() {
        println!("Size: {}", file.metadata().unwrap().len());
    }
}
```
