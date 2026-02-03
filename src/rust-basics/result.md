# Result

Рассмотрим еще один вездесущий тип — перечисление [Result](https://doc.rust-lang.org/std/result/).

```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

Как мы видим, `Result` чем-то напоминает `Option`, только если `Option` хранит либо значение, либо "пустоту", то `Result` хранит либо значение, либо ошибку.

`Result` используется в качестве типа результата для функций, которые могут завершиться с ошибкой.

Давайте рассмотрим простейший пример:

```rust
fn square_root(num: f32) -> Result<f32, String> {
    if num < 0.0 {
        Err("Cannot calculate for negative number".to_string())
    } else {
        Ok(num.sqrt())
    }
}

fn main() {
    println!("sqrt(-4) = {:?}", square_root(-4.0));
    // sqrt(-4) = Err("Cannot calculate for negative number")
    
    println!("sqrt(4) = {:?}", square_root(4.0));
    // sqrt(4) = Ok(2.0)
}
```

Функция `square_root` извлекает квадратный корень из переданного ей аргумента. Если аргумент имеет неотрицательное значение, то она возвращает `Ok(значение)`, иначе — `Err` (так как в арифметике нельзя извлекать корень из отрицательных чисел).

Извлечение значения из `Result` очень похоже на извлечение значения из `Option`:

* методом `unwrap` / `unwrap_or`
* оператором `match`
* оператором `if-let`

```rust
# fn square_root(num: f32) -> Result<f32, String> {
#     if num < 0.0 {
#         Err("Cannot calculate for negative number".to_string())
#     } else {
#         Ok(num.sqrt())
#     }
# }
# 
fn main() {
    match square_root(-4.0) {
        Ok(v) => println!("sqrt(-4)={v}"),
        Err(e) => println!("Cannot calculate sqrt(-4): {e}"),
    }
    
    let v = square_root(4.0).unwrap_or(0.0);
    println!("sqrt(4)={v}");
}
```

## Представление ошибок { #error-representation }

В примере выше в качестве ошибки мы использовали просто строку с описанием проблемы. Разумеется, такой подход крайне неудобен для программной обработки ошибки. Гораздо удобнее хранить информацию об ошибке в перечислении, в котором каждый элемент представляет отдельную проблему.

Для примера напишем функцию, которая принимает строку, содержащую ФИО, и возвращает отчество.

```rust
#[derive(Debug)]
enum NameParseError{
    EmptyString,  // Попытка парсить пустую строку
    NoMiddleName, // Строка не содержит отчество
}

// Принимает строку, содержающую имя, отчество и фамилию, разделённые
// символом пробела, и возвращает либо отчество, либо ошибку.
fn get_middle_name(full_name: &str) -> Result<String, NameParseError> {
    if full_name.is_empty() {
        return Err(NameParseError::EmptyString);
    }
    let mut words = split_to_words(full_name);
    if words.len() < 3 {
        return Err(NameParseError::NoMiddleName);
    }
    let middle_name = words.remove(1);
    Ok(middle_name)
}

// Эта функция разбивает строку на слова.
// В стандартной библиотеке для этого имеется готовая функциональность,
// но мы пока не готовы её использовать.
fn split_to_words(text: &str) -> Vec<String> {
    let mut words: Vec<String> = Vec::new();
    let mut current_word = String::new();
    for c in text.chars() {
        if c.is_whitespace() {
            if !current_word.is_empty() {
                words.push(current_word);
                current_word = String::new();
            }
        } else {
            current_word.push(c);
        }
    }
    if !current_word.is_empty() {
        words.push(current_word);
    }
    words
}

fn main() {
    let m_name_1 = get_middle_name("");
    println!("{m_name_1:?}"); // Err(EmptyString)

    let m_name_2 = get_middle_name("John Doe");
    println!("{m_name_2:?}"); // Err(NoMiddleName)

    let m_name_3 = get_middle_name("Сергей Петрович Иванов");
    println!("{m_name_3:?}"); // Ok("Петрович")
}
```

## Трэйт Error { #trait-error }

Как мы сказали, тип `Result` позволяет использовать любой тип для представления ошибки.
Однако стандартная библиотека предоставляет трэйт [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html), который создан специально для типов, используемых в качестве ошибки в `Result`:

```rust
pub trait Error: Debug + Display {
    // Если данная ошибка является обёрткой для другой ошибки,
    // то этот метод возвращает оборачиваемую ошибку.
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }

    // Устаревший, используйте fmt::Debug вместо него
    fn description(&self) -> &str { ... }

    // Устаревший, используйте source
    fn cause(&self) -> Option<&dyn Error> { ... }

    // Доступен только в ночной сборке RustSDK
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

Этот трэйт используется для унификации ошибок. Все типы ошибок из API стандартной библиотеки Rust реализуют `std::error::Error`.

Давайте реализуем трэйт `Error` для типа ошибки из секции [Представление ошибок](#error-representation).

```rust
#[derive(Debug)]
enum NameParseError{
    EmptyString,
    NoMiddleName,
}

impl std::fmt::Display for NameParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            NameParseError::EmptyString =>
                write!(f, "Attempt to parse empty string"),
            NameParseError::NoMiddleName =>
                write!(f, "No middle name found"),
        }
    }
}

impl std::error::Error for NameParseError {}
```

После этого нашу ошибку можно будет связывать в цепочку с другими ошибками из стандартной библиотеки.

> [!NOTE]
> Подобной ручной реализацией трэйта `std::error::Error` занимаются очень редко, так как в экосистеме Rust есть библиотеки, заметно упрощающие этот процесс. Но о них мы поговорим позже.

## Композиция  объектов Result

Давайте взглянем на простую программу, которая печатает на консоль текстовый файл.

> [!NOTE]
> Мы еще не разбирали API для работы с файловой системой и сделаем это только в главе [Файловая система](../dive-deeper/file-system.md). Однако если вы хотя бы раз работали с файловой системой на других языках программирования, то этот код должен выглядеть понятным:

```rust,noplayground
use std::fs::File;
use std::io::{Error, prelude::*};

/// Функция, которая читает содержимое текстового файла с заданным именем,
/// и возвращает содержимое файла в виде объекта строки.
fn read_text_file(file_name: &str) -> Result<String, Error> {
    // Открытие файла
    let mut file = match File::open(file_name) {
        Ok(file) => file,
        Err(e)   => return Err(e),
    };

    // Создание строки буфера, в которую будет произведено считывание
    let mut contents = String::new();
    // Читаем содержимое файла в строку
    match file.read_to_string(&mut contents) {
        Ok(read_bytes) => Ok(contents),
        Err(e)         => return Err(e),
    }
} 

fn main() {
    // Файл /etc/fstab присутствует в каждой Linux системе.
    // Если у вас Windows, то замените его на путь к любому текстовому файлу
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

Нетрудно заметить, что в функции `read_text_file` шаблонного кода, пробрасывающего ошибку, не меньше, чем "полезного". Можно ли как-то улучшить ситуацию?

Тип `Result`, подобно типу `Option`, имеет комбинаторы `map` и `and_then`. Используя их, мы можем переписать функцию `read_text_file` так:

```rust,noplayground
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    File::open(file_name).and_then(|mut file| {
        let mut contents = String::new();
        file.read_to_string(&mut contents).map(|_| {
            contents
        })
    })
}

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

Такой вариант `read_text_file` значительно короче, однако теперь он потерял в читабельности.

К счастью, Rust предоставляет специальный оператор `?`, который позволяет и сохранить линейность кода, и избавиться от шаблонного пробрасывания ошибки. Этот оператор  работает так:

* если объект типа `Result` содержит `Ok(значение)`, то значение просто извлекается
* если объект типа `Result` содержит `Err(ошибка)`, то функция завершает свою работу, а ошибка возвращается из функции.

```rust
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    let mut file = File::open(file_name)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
} 

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

Как видите, благодаря оператору `?`, код функции стал абсолютно линейным: словно у нас и нет никакой обёртки `Result`. При этом ни одна ошибка не осталась незамеченной. Именно благодаря этой мощной комбинации типа `Result`, трэйта `Error` и оператора `?` в Rust удобно работать с ошибками, несмотря на то что в языке отсутствует механизм исключений.

Разумеется, оператор `?` можно использовать только внутри функций, которые сами возвращают `Result`.

## Игнорирование Result

Если мы вызываем функцию, которая возвращает `Result`, и при этом не присваиваем её результат какой-то переменной и не проверяем `Result` на наличие ошибки, то компилятор выдаст предупреждение, так как будет считать, что мы просто забыли проверить результат на возможную ошибку.

Если мы сознательно хотим проигнорировать результат, то следует его явно отбросить путём присваивания ["выброшенной" переменной](variables.md#discarded-variable) — `let _ =` .

Например:

```rust
fn function_that_may_fail() -> Result<(), String> {
    Err("Something is wrong".to_string())
}

fn main() {
    let _ = function_that_may_fail();
}
```

