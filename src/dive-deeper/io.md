# Ввод/вывод

Стандартная библиотека Rust предоставляет модуль [std::io](https://doc.rust-lang.org/std/io/index.html), который содержит универсальные интерфейсы и типы для операций ввода/вывода.

В этом модуле находятся два базовых трэйта, задающих единый интерфейс для чтения и записи: [Read](https://doc.rust-lang.org/std/io/trait.Read.html) и [Write](https://doc.rust-lang.org/std/io/trait.Write.html).

## Read

Трэйт `Read` определяет один необходимый к реализации метод — `read` и несколько вспомогательных методов, работающих поверх него.

```rust
pub trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result;
    // ...
}
```

Вызов `read` заполняет переданные ему слайс байтами и возвращает количество байт, которые были записаны в слайс. Если размер слайса меньше, чем количество доступных байт, то метод `read` придётся вызвать несколько раз.

Остальные методы трэйта `Read` имеют реализацию по умолчанию. Некоторые из них:

* `fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize>`\
  Считывает всё содержимое в вектор байт.
* `fn read_to_string(&mut self, buf: &mut String) -> Result<usize>`\
  Считывает всё содержимое в в строку. Предполагается, что считываемые данные — текст.
* `fn read_buf(&mut self, buf: BorrowedCursor<'_>) -> Result<()>`\
  Считывает содержимое в курсор (о них мы поговорим позже).

Трэйт `Read` реализован для целого ряда типов: файл, сетевой сокет, Unix канал, STDIN (стандартный ввод), т.д.

***

Давайте рассмотрим работу с трэйтом `Read` на примере коллекции `VecDeque<u8>`, которую мы изучили в прошлой главе и которая также реализует трэйт `Read`.

> [!NOTE]
> Программисты на Java могут провести аналогию с чтением из массива байт при помощи класса `java.io.ByteArrayInputStream`.

```rust,edition2024
use std::{collections::VecDeque, io::Read};

fn main() {
    // Создаём VecDeque, из которого будем читать посредством трэйта Read
    let mut v: VecDeque<u8> = VecDeque::from(vec![1, 2, 3, 4, 5, 6, 7, 8, 9]);

    // Буфер размером в 2 байта. В него мы будем производить чтение
    let mut buf: [u8; 2] = [0; 2];

    // Считываем первый байт
    let read_bytes: Result<usize, std::io::Error> = v.read(&mut buf);
    println!("Buffer: {buf:?}, Number of read bytes: {read_bytes:?}");

    // В цикле считываем оставшиеся байты
    while let Ok(read_bytes) = v.read(&mut buf) && read_bytes > 0 {
        println!("Buffer: {buf:?}, Number of read bytes: {read_bytes:?}");
    }
}
```

Вывод программы:

```
Buffer: [1, 2], Number of read bytes: Ok(2)
Buffer: [3, 4], Number of read bytes: 2
Buffer: [5, 6], Number of read bytes: 2
Buffer: [7, 8], Number of read bytes: 2
Buffer: [9, 8], Number of read bytes: 1
```

Если источник данных небольшой (все данные можно легко считать в оперативную память), то удобнее воспользоваться методом `read_to_end`:

```rust
use std::{collections::VecDeque, io::Read};

fn main() {
    let mut v = VecDeque::from(vec![1, 2, 3, 4, 5, 6, 7, 8, 9]);

    let mut buf: Vec<u8> = Vec::new();
    let read_bytes: Result<usize, std::io::Error> = v.read_to_end(&mut buf);
    println!("Buffer: {buf:?}, Number of read bytes: {read_bytes:?}");
    // Buffer: [1, 2, 3, 4, 5, 6, 7, 8, 9], Number of read bytes: Ok(9)
}
```

В случае если считываемые данные — текст, то подойдёт метод `read_to_string`:

```rust
use std::{collections::VecDeque, io::Read};

fn main() {
    // 65 - 'A', 66 - 'B', 67 - 'C', 68 - 'D', 69 - 'E'
    let mut v = VecDeque::from(vec![65, 66, 67, 68, 69]);

    let mut buf = String::new();
    let read_bytes: Result<usize, std::io::Error> = v.read_to_string(&mut buf);
    println!("Buffer: {buf}, Number of read bytes: {read_bytes:?}");
    // Buffer: ABCDE, Number of read bytes: Ok(5)
}
```

## Cursor

Коллекция `VecDeque<u8>` реализует трэйт `Read`, но коллекция `Vec<u8>` — нет. Так было сделано потому, что `VecDeque` позволяет эффективно извлекать элементы из начала коллекции (элементы, считанные из `VecDeque` посредством `read`, удаляются), а из `Vec` извлекать элементы эффективно можно только с конца. Однако если необходимо иметь возможность работать с `Vec<u8>` как с `Read`, то это можно сделать при помощи обёртки [std::io::Cursor](https://doc.rust-lang.org/std/io/struct.Cursor.html).

`Cursor` имеет два поля: оборачиваемая коллекция и текущая позиция курсора.

```rust
pub struct Cursor<T> {
    inner: T,
    pos: u64,
}
```

Эта позиция позволяет эффективно реализовать операцию чтения: вместо того, чтобы удалять вычитанные элементы из вектора, курсор будет просто "сдвигать" позицию к еще не считанным байтам. Именно таким образом и реализован трэйт `Read` для `Cursor`.

```rust
impl<T> Read for Cursor<T> where T: AsRef<[u8]> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let n = Read::read(&mut Cursor::split(self).1, buf)?;
        self.pos += n as u64;
        Ok(n)
    }
}
```

Как видим, `Read` реализован для любой вариации `Cursor`, которая хранит в себе любой тип, реализующий `AsRef<u8>`. Тип `Vec<u8>` как раз реализует `AsRef<u8>`.

```rust
use std::{io::Read, io::Cursor};

fn main() {
    let v: Vec<u8> = vec![65, 66, 67, 68, 69];
    let mut c: Cursor<Vec<u8>> = Cursor::new(v);
    let mut buf = String::new();
    let read_bytes: Result<usize, std::io::Error> = c.read_to_string(&mut buf);
    println!("String: {buf}, Number of read bytes: {read_bytes:?}");
}
```

Вывод программы:

```
String: ABCDE, Number of read bytes: Ok(5)
```

## Write

Трэйт `Write` определяет универсальный интерфейс для записи данных.

В трэйте объявлены два обязательных к реализации метода и еще несколько методов с реализацией по умолчанию.

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
    // ...
}
```

Метод `write` записывает все байты из буфера-аргумента и возвращает количество фактически записанных байт. В конкретной реализации допускается записывать только часть байт, если эта часть может быть записана быстро, а запись оставшихся байт займёт существенно больше времени.

Метод `flush` имеет смысл для типов, которые перед фактической записью данных, предварительно буферизируют их в памяти. Вызов метода должен приводить к сбросу буферов в конечное хранилище данных.

Также трэйт содержит метод с реализацией по умолчанию — `write_all`.

```rust
fn write_all(&mut self, buf: &[u8]) -> Result<()>;
```

Этот метод, в отличие от `write`, должен записывать все байты, даже если это займёт существенно больше времени.

Как и `Read`, трэйт `Write` реализован для файла, сетевого сокета, Unix канала, STDOUT, т.д.

***

В отличие от `Read`, трэйт `Write` реализован не только для `VecDeque<u8>`, но и для `Vec<u8>`, так как вектор позволяет эффективно добавлять элементы в свой конец.

Рассмотрим пример записи байт в `Vec<u8>` посредством трэйта `Write`:

```rust
use std::io::Write;

fn main() {
    let mut v: Vec<u8> = vec![1, 2, 3, 4, 5];
    let count = v.write(&[6, 7, 8, 9]);
    println!("Written: {count:?}, vec: {v:?}");
    // Written: Ok(4), vec: [1, 2, 3, 4, 5, 6, 7, 8, 9]
}
```

Стандартный вывод STDOUT также позволяет работать с собой через трэйт `Write`:

```rust
use std::io::Write;

fn main() {
    let mut stdout = std::io::stdout();
    let _ = stdout.write(&[65, 66, 67, 68, 69, 10]);
}
```

Программа напечатает:

```
ABCDE
```

***

В следующей главе мы рассмотрим работу с `Read` и `Write` на примере файловой системы.
