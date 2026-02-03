# Итераторы

Как мы знаем, цикл `for` можно использовать для перебора элементов разных типов: массив, слайс, вектор, диапазон и т.д. Но за счёт чего достигается такая гибкость?

Дело в том, что цикл `for` умеет перебирать только итераторы, которые, в свою очередь, являются единым интерфейсом для работы с последовательностями. Итератор представлен трэйтом [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html).

```rust
pub trait Iterator {
    type Item;
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // ... более 70 других методов
}
```

Фактически, итератор определяет интерфейс, который позволяет последовательно перебирать объекты в некой коллекции (массив, слайс, вектор), с которой итератор связан.

Самый главный метод этого трэйта — `next`, который возвращает:

* либо следующий элемент в перебираемой последовательности в форме `Some(значение)`
* либо `None`, если все элементы перебраны

Именно метод `next` используется циклом `for` для получения следующего элемента.

> [!NOTE]
> Следует отметить, что <ins>обычно</ins> трэйт `Iterator` реализуется не для типа самой коллекции (массив, вектор и т.д.), а для отдельного типа, чей объект используется для перебора элементов в связанной с итератором коллекции (массиве, векторе и т.д.).
> 
> ![](img/iterator.svg)

## Делаем итератор

Чтобы было проще разобраться, давайте создадим свой итератор для вектора. Стандартная библиотека уже содержит стандартную реализацию итератора для вектора — тип [Iter](https://doc.rust-lang.org/beta/core/slice/struct.Iter.html). Поэтому чтобы не смущать компилятор попытками понять, какую же реализацию итератора он должен использовать (стандартную или нашу), мы сделаем обёртку над вектором.

```rust,noplayground
struct MyVec<T>(Vec<T>);
```

Теперь сделаем итератор для нашей обёртки:

```rust,noplayground
struct MyVecIter<'a, T> {
    data: &'a MyVec<T>,
    current_ind: usize,
}

impl <'a, T> Iterator for MyVecIter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.current_ind < self.data.0.len() {
            let current_element = &self.data.0[self.current_ind];
            self.current_ind += 1;
            Some(current_element)
        } else {
            None
        }
    }
}
```

Наш итератор — просто структура, которая хранит ссылку на объект `MyVec` и индекс того элемента, который будет извлечен при вызове `next()`.

Если индекс еще не вышел за границы итерируемого вектора, то мы возвращаем ссылку на элемент, находящийся по этому индексу, после чего инкрементируем индекс. Если индекс вышел за границы вектора, то возвращаем `None`, который служит сигналом того, что итерация завершена.

Пример использования нашего итератора:

```rust
# struct MyVec<T>(Vec<T>);
# 
# struct MyVecIter<'a, T> {
#     data: &'a MyVec<T>,
#     current_ind: usize,
# }
# 
# impl <'a, T> Iterator for MyVecIter<'a, T> {
#     type Item = &'a T;
# 
#     fn next(&mut self) -> Option<Self::Item> {
#         if self.current_ind < self.data.0.len() {
#             let current_element = &self.data.0[self.current_ind];
#             self.current_ind += 1;
#             Some(current_element)
#         } else {
#             None
#         }
#     }
# }
# 
fn main() {
    let my_vec = MyVec(vec![1,2,3]);
    let iterator = MyVecIter { data: &my_vec, current_ind: 0 }; 
    for n in iterator {
        print!("{n} ");
    }
    // Напечатает: 1 2 3
}
```

Разумеется, создавать объект итератора вот так вручную — очень неудобно. Поэтому в коллекцию, по которой можно итерироваться, часто добавляют метод `iter()`, который сам конструирует итератор.

```rust,noplayground
impl <T> MyVec<T> {
    // Возвращает итератор для нашей обёртки MyVec
    fn iter(&self) -> MyVecIter<'_, T> {
        MyVecIter { data: self, current_ind: 0 }
    }
}
```

Теперь итерироваться циклом `for` гораздо удобнее:

```rust
# struct MyVec<T>(Vec<T>);
# 
# impl <T> MyVec<T> {
#     fn iter(&self) -> MyVecIter<'_, T> {
#         MyVecIter { data: self, current_ind: 0 }
#     }
# }
# 
# struct MyVecIter<'a, T> {
#     data: &'a MyVec<T>,
#     current_ind: usize,
# }
# 
# impl <'a, T> Iterator for MyVecIter<'a, T> {
#     type Item = &'a T;
# 
#     fn next(&mut self) -> Option<Self::Item> {
#         if self.current_ind < self.data.0.len() {
#             let current_element = &self.data.0[self.current_ind];
#             self.current_ind += 1;
#             Some(current_element)
#         } else {
#             None
#         }
#     }
# }
# 
fn main() {
    let my_vec = MyVec(vec![1,2,3]);
    for n in my_vec.iter() {
        print!("{n}, ");
    }
}
```

> [!TIP]
> Имя метода `iter()` не регламентировано никаким трейтом и является просто понятным общепринятым именем.

## Цикл for и итераторы

Теперь, когда мы выяснили, что цикл `for` на самом деле работает только с итераторами, давайте выясним, как же цикл `for` итерируется непосредственно по вектору.

Всё просто: цикл `for` принимает либо непосредственно итератор, либо тип, который реализует трэйт [IntoIterator](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html).

```rust,noplayground
pub trait IntoIterator {
    // Тип элемента итератора
    type Item;

    // Тип итератора
    type IntoIter: Iterator<Item = Self::Item>;

    // Возвращает объект итератора
    fn into_iter(self) -> Self::IntoIter;
}
```

Этот трэйт реализуется типом самой коллекции и позволяет ей создать объект итератора по своим элементам.

То есть если мы хотим передавать объект `MyVec` в цикл `for` непосредственно, то мы должны реализовать для него трэйт `IntoIterator`. Давайте сделаем это.

```rust,noplayground
impl <'a, T> IntoIterator for &'a MyVec<T> {
    type Item = &'a T;

    type IntoIter = MyVecIter<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        MyVecIter { data: self, index: 0 }
    }
}
```

Поскольку мы итерируемся по элементам через ссылку, а не по значению, то `IntoIterator` мы реализуем не для `MyVec`, а для ссылки `&MyVec`.

Теперь, когда `IntoIterator` определён, мы можем итерироваться по объекту `MyVec` так:

```rust
# struct MyVec<T>(Vec<T>);
# 
# struct MyVecIter<'a, T> {
#     data: &'a MyVec<T>,
#     index: usize,
# }
# 
# impl <T> MyVec<T> {
#     fn iter(&self) -> MyVecIter<'_, T> {
#         MyVecIter { data: self, index: 0 }
#     }
# }
# 
# impl <'a, T> Iterator for MyVecIter<'a, T> {
#     type Item = &'a T;
#     fn next(&mut self) -> Option<Self::Item> {
#         if self.index < self.data.0.len() {
#             let current_element = &self.data.0[self.index];
#             self.index += 1;
#             Some(current_element)
#         } else {
#             None
#         }
#     }
# }
# 
# impl <'a, T> IntoIterator for &'a MyVec<T> {
#     type Item = &'a T;
# 
#     type IntoIter = MyVecIter<'a, T>;
# 
#     fn into_iter(self) -> Self::IntoIter {
#         self.iter()
#     }
# }
# 
fn main() {
    let my_vec = MyVec(vec![1,2,3]);

    for n in &my_vec {
        print!("{n}, ");
    }
}
```

А можно ли итерироваться по элементам нашей обёртки не по ссылке, а по значению? Да, однако нам придётся написать второй итератор, который будет захватывать нашу обёртку по значению:

```rust
struct MyVec<T>(Vec<T>);

struct MyVecIterVal<T> {
    data: MyVec<T>,
}

impl <T> MyVec<T> {
    fn iter_val(self) -> MyVecIterVal<T> {
        MyVecIterVal { data: self }
    }
}

impl <T> Iterator for MyVecIterVal<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.data.0.pop() // извлекаем и возвращаем первый элемент
    }
}

impl <T> IntoIterator for MyVec<T> {
    type Item = T;

    type IntoIter = MyVecIterVal<T>;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_val()
    }
}

fn main() {
    let my_vec = MyVec(vec![1,2,3]);

    for n in my_vec {
        print!("{n}, ");
    }
}
```

## Еще раз про итерирование по вектору

Теперь, когда мы познакомились с трэйтами `Iterator` и `IntoIterator`, давайте посмотрим на несколько примеров итерирования по уже знакомым типам.

```rust,noplayground
let v = vec![1, 2, 3];

// Метод iter() определен для типа Vec.
// Он явно возвращает объект итератора, который перебирает ссылки на элементы
// Тип i: &i32
for i in v.iter() { }

// На объекте &Vec будет неявно вызван метод into_iter(),
// который вернёт итератор. Тип i: &i32
for i in &v { }

// Явный вызов into_iter() для объекта Vec. Вернёт итератор для перебора
// элементов по значению, т.е. итерирование "поглотит вектор". Тип i: i32
for i in v.into_iter() { }

// Неявно вызовет into_iter() для объекта Vec.
for i in v { }
```

Точно такая же картина и для массивов:

```rust,noplayground
let arr = [1, 2, 3];

// Перебор по ссылке - i: &i32
for i in arr.iter() { }

// Перебор по ссылке - i: &i32
for i in &arr { }

// Перебор по значению - i: i32
for i in arr.into_iter() { }

// Перебор по значению - i: i32
for i in arr { }
```

Резюмируя:

* если мы хотим проитерироваться по коллекции, перебирая значения по ссылке, тогда следует передавать в оператор `for` либо ссылку на коллекцию, либо вызывать на объекте коллекции метод `iter()` (если таков имеется).
* если нам нужны именно значения элементов (и нам подходит разрушение самой коллекции), то следует передавать в оператор `for` либо объект коллекции непосредственно, либо вызвать метод `into_iter()`

## Range

В главе [Циклы](loops.md) мы видели такую форму перебора чисел:

```rust
for i in 0 .. 20 {
    println!("{i}");
}
```

Наконец-то мы готовы разобраться с тем, как это работает.

Дело в том, что оператор `..` создаёт диапазон — объект структуры [Range](https://doc.rust-lang.org/std/ops/struct.Range.html), которая имеет вид:

```rust,noplayground
pub struct Range<Idx> {
    pub start: Idx, // начальный элемент диапазона
    pub end: Idx,   // конечный элемент диапазона
}
```

Как мы видим, `Range` просто хранит начальное и конечное значения диапазона. При этом тип `Range` реализует интерфейс `Iterator`, что позволяет итерироваться по его элементам.

При желании мы можем использовать `Range` и без цикла `for`.

```rust
use std::ops::Range;

fn main() {
    let mut range: Range<i32> = 0..20;
    println!("{:?}", range.next()); // Some(0)
    println!("{:?}", range.next()); // Some(1)
}
```

> [!NOTE]
> В начале главы мы сказали, что `Iterator` обычно реализуется не для самой коллекции, а для отдельного типа, который используется для перебора элементов этой коллекции. Тип `Range` является одним из исключений, так как трэйт `Iterator` реализован непосредственно для него самого. При этом итерирование реализовано просто инкрементированием поля `start`.

## API итераторов

Как мы заметили в самом начале, в трэйте `Iterator`, кроме метода `next()`, определено еще более 70 методов, которые имеют реализацию по умолчанию. Что же это за методы?

Это методы, позволяющие декларативно обрабатывать и преобразовывать коллекции.

> [!TIP]
> Программисты на Java могут провести аналогию с Stream API, а программисты на C# — с Linq.

Начнём с примера: у нас имеется вектор чисел, и мы хотим:

1) Взять из него только чётные числа.
2) Далее каждое из этих чётных чисел возвести в квадрат.

```rust
let v1 = vec![1,2,3,4,5];
let v2 = v1.into_iter()     // Превращаем вектор в итератор
    .filter(|x| x % 2 == 0) // фильтруем только чётные элементы
    .map(|x| x * x)         // возводим элементы в квадрат
    .collect::<Vec<_>>();   // перепаковываем итератор в вектор
println!("{v2:?}"); // [4, 16]
```

Давайте подробно разберём этот пример.

1\) Сначала мы получаем итератор для вектора. Так как после завершения обработки нам больше не нужен оригинальный вектор, мы используем метод `into_iter()`, который создаст итератор, перебирающий элементы вектора по значению, а значит, оригинальный вектор будет "поглощён".

2\) На полученном итераторе мы вызываем метод [filter](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter), который имеет вид:

```rust
fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where Self: Sized, P: FnMut(&Self::Item) -> bool
{
    Filter::new(self, predicate)
}
```

Как мы видим, этот метод принимает некий предикат и возвращает объект [Filter](https://doc.rust-lang.org/std/iter/struct.Filter.html).

```rust
pub struct Filter<I, P> {
    pub(crate) iter: I,
    predicate: P,
}
```

Тип `Filter` — это обёртка над итератором, которая хранит в себе итератор и предикат, которым его надо профильтровать. И разумеется, тип `Filter` тоже реализует трэйт `Iterator`.

3\) Уже на объекте `Filter` мы вызываем другой метод, определённый в трэйте `Iterator` — метод [map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map):

```rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where Self: Sized, F: FnMut(Self::Item) -> B,
{
    Map::new(self, f)
}
```

Как видим, этот метод очень напоминает `filter`, за исключением того, что он возвращает объект типа [Map](https://doc.rust-lang.org/std/iter/struct.Map.html), который объявлен так:

```rust
pub struct Map<I, F> {
    pub(crate) iter: I,
    f: F,
}
```

То есть `Map` — это обёртка над итератором, которая хранит функцию, при помощи которой нужно преобразовать элементы обёрнутого итератора.

4\) На объекте `Map` мы вызываем метод [collect()](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect), определённый в трэйте `Iterator`, и только в этот момент начинается раскрутка обёрток и вычисление результата.\
Каким будет тип результата, зависит от генерик тип-аргумента метода `collect()`.

Эта "матрёшка" из вектор-итератора, завёрнутого в `Filter`, и завёрнутого в `Map`, выглядит примерно так:

![](img/iterator_nesting.svg)

В примере выше мы обработали вектор и получили результат тоже в форме вектора (так как в метод `collect` передали тип-аргумент `Vec`). Если бы мы хотели получить результирующие элементы в виде, например, [HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html) (хеш-множество — мы поговорим о нём позже), то просто должны были бы указать этот вид коллекции в методе `collect()`.

```rust
use std::collections::HashSet;

fn main() {
    let v1: Vec<i32> = vec![1,2,3,4,5];
    let v2: HashSet<i32> = v1.into_iter()
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
        .collect::<HashSet<_>>();
    println!("{v2:?}"); // {16, 4}
}
```

Для большинства коллекций из стандартной библиотеки можно получить итератор, и большинство коллекций из стандартной библиотеки могут быть построены из итератора методом `collect`. Получается, что итератор — универсальное API для обработки и преобразования типов-коллекций.

![](img/iterator_for_collections.svg)

Большинство коллекций из сторонних библиотек также работают вместе с итераторами.

## Свёртки

Итератор не обязательно использовать для создания другой коллекции, он также позволяет "сворачивать" (агрегировать) элементы. Для этого трэйт `Iterator` предоставляет метод [fold](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold):

```rust,noplayground
fn fold<B, F>(mut self, init: B, mut f: F) -> B
    where F: FnMut(B, Self::Item) -> B
{ ... }
```

Этот метод принимает два аргумента:

1. начальное значение, с которым мы будем "сворачивать" (агрегировать)
2. агрегирующую функцию, которая принимает уже предагрегированное значение и следующий элемент из итератора, и возвращает результат агрегации

Для примера давайте суммируем элементы массива при помощи свёртки.

```rust
let arr = [1, 2, 3];
let sum = arr.into_iter()
    .fold(0, |x, y| x + y);
println!("{sum}"); // 6
```

Здесь свёртка происходит примерно так:

![](img/iterator_fold.svg)

Возможно, у вас появился вопрос: а для чего нам начальное значение `0`, если в качестве начального значения агрегации мы просто могли использовать первый элемент?

Во-первых, на самом деле в трэйте `Iterator` есть метод [reduce](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.reduce), который работает практически так же, как и `fold`, только в качестве начального значения берётся как раз первый элемент.

```rust,edition2024
# fn main() {
let arr = [1, 2, 3];
let sum: Option<i32> = arr.into_iter()
    .reduce(|x, y| x + y);
println!("{sum:?}"); // Some(6)
# }
```

(`reduce` возвращает `Option`, так как в сворачиваемой коллекции может вообще не быть элементов)

А во-вторых, при использовании `fold`, результат агрегации может иметь тип, отличный от типа итерируемых элементов. Например, при помощи `fold` мы можем проитерироваться по массиву строк и подсчитать количество символов во всех строках:

```rust
let arr = ["aa", "bbb", "cccc"];
let char_count = arr.iter()
    .fold(0, |count, s| count + s.len());
println!("{char_count}"); // 9
```

***

Также следует заметить, что для итератора определён ряд [специализаций](generics.md#specializaciya-treita).

Например, для итератора с элементами числового типа, имеется дополнительный метод [sum](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.sum), который считает сумму всех элементов:

```rust
fn main() {
    let arr = [1, 2, 3];
    let sum: i32 = arr.into_iter().sum();
    println!("{sum}");
}
```

## Другие методы

### filter_map

Метод [filter_map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map) работает как обычный `map`, однако функция-преобразователь, которую он принимает в качестве аргумента, должна возвращать значение, завёрнутое в `Option`.

* Если функция-преобразователь после применения к очередному элементу вернёт `Some(значение)`, то значение будет автоматически извлечено и пойдёт "дальше по конвейеру итератора".
* Если функция-преобразователь вернёт `None`, то этот `None` будет просто отброшен.

Пример: получение квадратных корней всех элементов, из которых можно извлечь корень.

```rust,edition2024
fn safe_sqrt(n: f32) -> Option<f32> {
    if n < 0.0 {
        None
    } else {
        Some(n.sqrt())
    }
}

fn main() {
    let arr = [4.0, -25.0, 9.0];
    let result = arr.into_iter()
        .filter_map(safe_sqrt)
        .collect::<Vec<_>>();
    println!("{result:?}"); // [2.0, 3.0]
}
```

### find

Метод [find](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.find) принимает предикат и возвращает первый элемент, который удовлетворяет этому предикату.

Пример: поиск первого чётного элемента.

```rust,edition2024
let arr = [1, 3, 5, 7, 8, 9];
let first_even: Option<i32> = arr.into_iter()
    .find(|x| x % 2 == 0);
println!("{first_even:?}");
```

Метод возвращает значение, завёрнутое в `Option`, так как есть вероятность, что ни один элемент не удовлетворит предикату.

> [!TIP]
> В трэйте `Iterator` определено много полезных методов, тем не менее на практике вам, скорее всего, окажется их недостаточно. Однако существует замечательная библиотека [itertools](https://crates.io/crates/itertools), которая предлагает целое множество дополнительных методов.

