# Генерики

Генерики (generics) — это механизм, который позволяет писать функциональность, работающую со значениями, абстрагируясь от типов этих значений.

> [!NOTE]
> Мы будем одинаково использовать названия и "генерик", и "обобщённый".

Например, если мы создаём структуру данных "список", то мы будем сфокусированы на работе с элементами в списке (добавление элемента, удаление, поиск, вставка), а не над тем, какой тип у этих элементов. Функциональность списка не изменится в зависимости от того, храним мы в нём числа или строки.

Уже знакомый нам тип вектор — `Vec` как раз является генерик-типом: в векторе мы можем хранить значения любого типа данных.

## Генерик типы и функции

Для того чтобы разобраться с генериками, давайте создадим максимально простой обобщённый тип — структуру-обёртку. Эта обёртка не имеет никакой специальной функциональности, она просто хранит в себе значение, но эта обёртка подходит для хранения значений любого типа.

```rust
struct Holder<T> {
    v: T
}
```

Здесь `<T>` — это так называемый генерик тип-аргумент (generic type argument): тип, над которым мы абстрагируемся.

> [!NOTE]
> В примере выше мы назвали наш абстрагированный тип как `T` (самое популярное имя для генерик типа-аргумента), но это имя может быть абсолютно любым, например "Element".
> 
> ```rust
> struct Holder<Element> {
>     v: Element
> }
> ```

`Holder<T>` — это, скорее, не тип, а шаблон для создания типа. Когда компилятор встречает использование `Holder<T>` для значения какого-то типа, например `i32`, он генерирует конкретный тип `Holder<i32>`.

```rust
struct Holder<T> {
    v: T
}

fn main() {
    let bool_holder: Holder<bool> = Holder { v: true };
    let i32_holder: Holder<i32> = Holder { v: 5 };
    let string_holder: Holder<String> = Holder { v: "aaa".to_string() };
}
```

---

Обобщёнными могут быть не только структуры, но и функции. Напишем для нашей генерик структуры `Holder<T>` соответствующую обобщённую функцию-конструктор:

```rust
fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}
```

Как видим, генерик аргумент для функции указывается так же, как и для структуры: в угловых скобках после имени.

Теперь мы можем создавать экземпляры `Holder`, используя эту функцию:

```rust
struct Holder<T> {
    v: T
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
  let bool_holder: Holder<bool> = make_holder(true);
  let i32_holder: Holder<i32> = make_holder(5);
  let string_holder: Holder<String> = make_holder("aaa".to_string());
}
```

> [!NOTE]
> Заметьте, генерик тип-аргумент `T` у функции `make_holder<T>` никак не связан с генерик аргументом `T` в объявлении структуры `Holder<T>`. Просто, как мы уже сказали, `T` — самое популярное имя. Если бы мы написали `fn make_holder<A>(v: A) -> Holder<A>`, то ничего бы не изменилось.

---

Чтобы сложить полную картину, давайте посмотрим, как выглядят методы для генерик структур. Сделаем два метода: _get_ — для получения значения из нашей обёртки, и _set_ — для записи нового значения.

```rust,noplayground
impl<T> Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}
```

В этой конструкции мы видим два генерик тип-аргумента: в `impl<T>` и в `Holder<T>`. Тот, который в `impl<T>` — <ins>объявляет</ins> имя генерик тип-аргумента для всего `impl` блока. Генерик аргумент в `Holder<T>` просто устанавливает связь между генерик аргументом из `impl<T>` и генерик аргументом в объявлении структуры `Holder<T>`.

Мы словно говорим:

> Для некоего типа `T` мы реализуем шаблонный тип `Holder`, который при параметризации этим типом `T` имеет такие имплементации методов `get` и `set`.

Немного запутано, но станет понятнее в более сложных примерах далее.

А теперь всё вместе:

```rust
struct Holder<T> {
    v: T
}

impl<T> Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
    let mut h = make_holder(1);
    println!("{}", h.get()); // 1
    h.set(5);
    println!("{}", h.get()); // 5
}
```

## Мономорфизация генериков

Как мы уже сказали, генерик типы вроде `Holder<T>` — это скорее шаблоны типов, а конкретные типы получаются при параметризации генерика конкретным типом, например `Holder<i32>`.

Каждый раз, когда компилятор встречает использование генерик типа с новым тип-аргументом, он генерирует конкретный вариант генерика под этот тип. Это генерирование конкретных типов для генерик-типов называют **мономорфизацией**.

Например, мы использовали наш генерик тип `Holder<T>` для типов `i32` и `bool`. Компилятор мономорфизирует `Holder<T>` для `i32` и для `bool` примерно так (в реальности имена будут выглядеть сложнее):

```rust,noplayground
struct Holder_i32 {
    v: i32
}
struct Holder_bool {
    v: bool
}
```

То же самое произойдёт и с методами, и с функциями: под каждый тип-аргумент будут сгенерированы конкретные варианты функций.

<table class="two-side">
<tr>
<td width="50%">

```rust,noplayground
fn make_holder<T>(v: T) -> Holder<T> {
    Holder { v: v }
}
```

</td>
<td width="50%">

```rust,noplayground
let h1 = make_holder(5);
let h2 = make_holder(true);
```

</td>
</tr>

<tr>
<td width="50%"><center>↓</center></td>
<td width="50%"><center>↓</center></td>
</tr>

<tr>
<td width="50%">

```rust,noplayground
fn make_holder_i32(v: i32) -> Holder_i32 {
    Holder_i32 { v: v }
}

fn make_holder_bool(v: bool) -> Holder_bool {
    Holder_bool { v: v }
}
```

</td>
<td width="50%">

```rust,noplayground
let h1 = make_holder_i32(5);
let h2 = make_holder_bool(true);
```

</td>
</tr>
</table>

> [!NOTE]
> В языках Java и C# также имеются генерики. Однако в этих языках отличительной чертой генериков является так называемое "стирание" генерик тип-аргумента при компиляции. Другими словами, в Java генерики используются просто как дополнительный контроль типов во время компиляции.\
> Но после компиляции информация о генерик тип-аргументах исчезает и недоступна во время работы программы.\
> В Rust же генерики больше похожи на шаблоны в C++, где при компиляции также происходит мономорфизация шаблона.

## Турборыба ::<>

Мы уже видели пример использования генерик-функции для создания объекта генерик-структуры.

```rust
struct Holder<T> {
    v: T
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
    let mut h = make_holder(1); // тип переменной - Holder<i32>
}
```

Здесь компилятор смог самостоятельно вывести тип генерик-функции `make_holder` по типу аргумента, с которым функция была вызвана: раз тип аргумента — `i32`, то и тип результата функции — `Holder<i32>`.

Но если у функции нет аргументов, то компилятор не сможет сам вывести тип генерика, и его придётся указать явно. Для примера рассмотрим функцию, которая возвращает пустой вектор:

```rust,noplayground
fn make_empty_vec<T>() -> Vec<T> {
    Vec::new()
}
```

При вызове `make_empty_vec` тип генерика можно указать через тип для переменной, которой присваивается результат.

```rust,noplayground
let v: Vec<i32> = make_empty_vec();
```

Но можно указать генерик тип-аргумент прямо на вызове функции:

```rust,noplayground
let v = make_empty_vec::<i32>();
```

Этот синтаксис задания генерик тип-аргумента `::<>` прозвали турборыбой (turbofish).

## Генерик трэйты { #generic-traits }

Обобщёнными могут быть не только структуры и функции, но и трэйты.

```rust
// Задаёт интерфейс доступа к некому значению внутри типа
// при помощи методов get и set.
// При этом тип элемента, к которому осуществляется доступ,
// задаётся генериком.
trait CanBeAccessed<T> {
    fn get(&self) -> &T;
    fn set(&mut self, new_v: T);
}

// Задаёт интерфейс для создания нового объекта.
trait HasGenericConstructor<T> {
    // Из объекта типа T cоздаёт объект типа,
    // реализующего HasGenericConstructor<T>
    fn new(value: T) -> Self;
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed<T> for Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

impl<T> HasGenericConstructor<T> for Holder<T> {
    fn new(value: T) -> Self {
        Holder { v: value }
    }
}

fn main() {
    let mut h = Holder::new(5);
    h.set(7);
    println!("{}", h.get());
}
```

Как мы видим из примера, генерик трэйты ведут себя ровно так же, как и обычные, только теперь у них появился генерик тип-аргумент.

## Ассоциированные типы

Для генерик трэйтов существует альтернативный синтаксис для указания генерик тип-аргумента: через специальное поле — ассоциированный тип.

```rust,noplayground
trait Имя {
    type АссоциированныйТип;
}
```

Следующий пример показывает, как соотносятся между собой генерик с тип-аргументом и генерик с ассоциированным типом.

<table>
<tr>
<td width="50%" style="border: none;">
Трэйт с генерик тип-аргументом

```rust,noplayground
trait Трэйт<A> {
    ...
}


struct S {
    ...
}

impl Трэйт<i32> for S {
    ...

}
```

</td>
<td width="50%" style="border: none;">
Трэйт с ассоциированным типом

```rust,noplayground
trait Трэйт {
    type A;
    ...
}

struct S {
    ...
}

impl Трэйт for S {
    type A = i32;
    ...
}
```

</td>
</tr>
</table>

Перепишем наш пример из раздела про генерик трэйты с использованием ассоциированного типа:

```rust
trait CanBeAccessed {
    type ElementType;
    fn get(&self) -> &Self::ElementType;
    fn set(&mut self, new_v: Self::ElementType);
}

trait HasGenericConstructor {
    type TypeArg;
    fn new(value: Self::TypeArg) -> Self;
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed for Holder<T> {
    type ElementType = T;
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

impl<T> HasGenericConstructor for Holder<T> {
    type TypeArg = T;
    fn new(value: T) -> Self {
        Holder { v: value }
    }
}

fn main() {
    let mut h = Holder::new(5);
    h.set(7);
    println!("{}", h.get());
}
```

---

Хоть генерик трэйты и трэйты с ассоциированным типом служат одной цели, у них всё же имеются два различия.

1\) Первое отличие генерик-трэйтов от трэйтов с ассоциированным типом заключается в том, где определяется тип, которым будет параметризирован генерик:

* Для генерик трэйта фактический тип генерик тип-аргумента определяется в коде, где <ins>используется</ins> структура, реализующая этого генерик трэйт.

* Для трэйта с ассоциированным типом тип ассоциированного поля определяется в блоке реализации трэйта для структуры (`impl Трэйт for Тип`).
  В примере выше в блоке `impl<T> CanBeAccessed for Holder<T>` мы определили, что тип поля-ассоциированного типа для трэйта `CanBeAccessed` будет таким же, как и генерик тип-аргумент в структуре `Holder<T>`. И это никак нельзя переопределить в месте создания объекта `Holder`.

2\) Второе отличие заключается в реализации трэйтов для типов.

Один и тот же генерик трэйт можно несколько раз реализовать для одного и того же типа, указав разные типы для тип-аргумента:

```rust
trait ValueProducer<T> {
    fn produce() -> T;
}

struct ProducerImpl;

impl ValueProducer<i32> for ProducerImpl {
    fn produce() -> i32 {
        5
    }
}

impl ValueProducer<String> for ProducerImpl {
    fn produce() -> String {
        "Hello".to_string()
    }
}

fn main() {
    let n: i32 = ProducerImpl::produce();
    println!("{n}");

    let s: String = ProducerImpl::produce();
    println!("{s}");
}
```

Написать аналогичный код, используя трэйт с ассоциированным типом, не получится.


## Границы генериков { #generics-border }

Когда мы объявляем обобщённый тип или функцию, мы можем указать, что генерик тип-аргумент должен реализовывать некий трэйт. Другими словами, мы можем ограничить его трэйтом.

```rust,noplayground
fn my_func<T: Трэйт>(...) -> ... {
    ...
}

struct MyStruct<T: Трэйт> {
    v: T
}
```

В таком случае компилятор позволит параметризировать наш обобщённый тип/функцию только теми типами, которые реализуют этот указанный трэйт.

```rust,compile_fail
// Функция принимает аргумент любого типа, который реализует трэйт Copy
fn duplicate<T: Copy>(v: T) -> T {
    v
}

fn main() {
    let num = 5;
    let num_dup = duplicate(num); // i32 реализует трэйт Copy

    let s = "Hello".to_string();
    let s_dup = duplicate(s); // String не реализует трэйт Copy
             // ^^^^^^^^^^^^ the trait `Copy` is not implemented for `String
}
```


Если генерик тип-аргумент должен реализовывать несколько трейтов, то их надо перечислить через знак `+`

```rust,noplayground
fn my_func<T: Трэйт1 + Трэйт2 + Трэйт3>(...) -> ... {
    ...
}
```

На этом моменте вам может показаться, что это ограничение типов по реализуемому ими трэйту очень похоже на передачу аргументов по `impl Трэйт`, с которым мы познакомились в главе [Трэйты / Статическая диспетчеризация](traits.md#static-dispatching). Ведь по сути, `impl Трэйт` тоже может быть заменён только на тип, который этот трэйт реализует.

<table>
<tr>
<td width="50%" style="border: none;">

генерик с границей

```rust,noplayground
fn my_func<T: Трэйт>(v: T) -> ... {
    ...
}
```

</td>
<td width="50%" style="border: none;">

`impl Трэйт`

```rust,noplayground
fn my_func(v: impl Трэйт) -> ... {
    ...
}
```

</td>
</tr>
</table>


То есть: передача аргументов по генерику с границей и передача аргументов через `impl Трэйт` — это просто разный синтаксис для одного и того же механизма.

Чтобы продемонстрировать это, давайте перепишем пример, который мы использовали в главе [Трэйты / Статическая диспетчеризация](traits.md#static-dispatching), с использованием границ генерика.

> <details>
>   <summary>Оригинальный пример с трэйтами</summary>
> 
> ```rust
> trait CanIntroduce {
>     fn introduce(&self) -> String;
> }
> 
> struct Person { name: String }
> struct Dog { name: String }
> 
> impl CanIntroduce for Person {
>     fn introduce(&self) -> String {
>         format!("Hello, I'm {}", self.name)
>     }
> }
> 
> impl CanIntroduce for Dog {
>     fn introduce(&self) -> String {
>         String::from("Waf-waf")
>     }
> }
> 
> fn print_introduction(v: &impl CanIntroduce) {
>     println!("{}", v.introduce());
> }
> 
> fn main() {
>     let person = Person { name: String::from("John") };
>     let dog    = Dog    { name: String::from("Bark") };
> 
>     print_introduction(&person); // Hello, I'm John
>     print_introduction(&dog);    // Waf-waf
> }
> ```
> </details>


```rust
trait CanIntroduce {
    fn introduce(&self) -> String;
}

struct Person { name: String }
struct Dog    { name: String }

fn create_person(name: String) -> Person {
    Person { name }
}

fn create_dog(name: String) -> Dog {
    Dog { name }
}

impl CanIntroduce for Person {
    fn introduce(&self) -> String {
        format!("Hello, I'm {}", self.name)
    }
}

impl CanIntroduce for Dog {
    fn introduce(&self) -> String {
        String::from("Waf-waf")
    }
}

fn print_introduction<T: CanIntroduce>(v: T) { // Отличие здесь
    println!("{}", v.introduce ());
}

fn main() {
    let person = create_person("John".to_string());
    let dog    = create_dog("Bark".to_string());

    print_introduction(person); // Hello, I'm John
    print_introduction(dog);    // Waf-waf
}
```

Как видите, это минимальное различие в синтаксисе приводит к одному и тому же результату.

---

Однако есть ситуации, где можно использовать только генерик с границей. Например, компилятор не позволяет написать замыкание, которое возвращает `impl Трэйт`, но позволит — замыкание, которое возвращает генерик с границей:

```rust
use std::fmt::Display;

fn produce_number() -> i32 {
    5
}

// Так нельзя:
// fn print_produced(f: fn() -> impl Format) {
//     println!("{}", f());
// }

fn print_produced<R: Display>(f: fn() -> R) {
    println!("{}", f());
}

fn main() {
    print_produced(produce_number);
}
```

Также `impl Трэйт` синтаксис имеет существенные ограничения при описании обобщённых асинхронных функций, с которыми мы познакомимся позже.


## where

Для указания границы генерика существует альтернативный синтаксис — блок **where**.

<table>
<tr>
<td width="50%" style="border: none;">
Без where блока

```rust,noplayground
fn my_func<A: Трэйт1, B: Трэйт2>(
    аргумент1: A, аргумент2: B,
) -> Тип {
    ...

}
```

</td>
<td width="50%" style="border: none;">
С where блоком

```rust,noplayground
fn my_func<A, B>(
    аргумент1: A, аргумент2: B,
) -> Тип
where A: Трэйт1, B: Трэйт2 {
    ...
}
```

</td>
</tr>
</table>

Блок `where` позволяет сделать сигнатуру функции более читабельной, так как делает объявление генерик тип-аргументов менее громоздкими.

Перепишем функцию `print_introduction` из предыдущего примера, с использованием блока `where`.

```rust,noplayground
fn print_introduction<T>(v: T) where T: CanIntroduce {
    println!("{}", v.introduce ());
}
```

Блок where можно использовать не только с функциями, но и с типами:

<table>
<tr>
<td width="50%" style="border: none;">

генерик с границей

```rust,noplayground
struct Holder<T: Clone> {
    v: T
}
```

</td>
<td width="50%" style="border: none;">

блок `where`

```rust,noplayground
struct Holder<T> where T: Clone {
    v: T
}
```

</td>
</tr>
</table>

Блок `where` особенно удобен для описания генериков, которые в своём составе имеют другие генерики. Рассмотрим пример:

```rust
use std::fmt::Display;

fn print_produced<F,R>(mut f: F)
    where
        F: FnMut() -> R,
        R: Display
{
    println!("{}", f());
}

fn main() {
    let sequence_producer = {
        let mut counter = 0;
        move || {
            counter += 1;
            counter
        }
    };
    
    print_produced(sequence_producer);
}
```

Здесь функция `print_produced` в качестве аргумента принимает `FnMut` замыкание, которое возвращает значение любого типа, который реализует трэйт `Display`. Попытка написать эту функцию с использованием `impl Трэйт` синтаксиса привела бы к ошибке компиляции:

```rust,noplayground
fn print_produced(mut f: impl FnMut() -> impl Display) {
    println!("{}", f());
}
```

Ошибка:

```
`impl Trait` is not allowed in the return type of `Fn` trait bounds
`impl Trait` is only allowed in arguments and return types of functions and methods
```


## Специализация трэйта

Для генерик структур можно задавать методы, которые будут доступны только при мономорфизации определённым типом. Это позволяет добавить дополнительную функциональность для определённых типов.

Давайте сделаем для нашего `Holder` метод инкремент, который будет доступен только если объект `Holder` хранит `i32`.

```rust
struct Holder<T> {
    v: T
}

impl Holder<i32> {
    fn inc(&mut self) {
        self.v += 1;
    }
}

fn main() {
    let mut h = Holder { v: 1 };
    h.inc();
}
```

Специализировать генерик можно не только под конкретный тип, но и под трэйт. Давайте сделаем метод, который будет доступен только если генерик `Holder` параметризирован типом, реализующим трэйт `Clone`.

```rust
struct Holder<T> {
    v: T
}

impl<T: Clone> Holder<T> {
    fn clone_value(&self) -> T {
        self.v.clone()
    }
}

fn main() {
    let h: Holder<String> = Holder { v: "text".to_string() };
    let s2: String = h.clone_value();
}
```

## const генерики

Генерики позволяют параметризировать типы и функции не только не только типами, но и константами.

Например, тип массива состоит из двух частей: тип элемента и размер массива.

```rust
let arr: [i32; 3] = [1, 2, 3];
```

Поэтому если мы захотим написать генерик функцию, которая возвращает массив определённого размера, то в дополнение к генерик тип-аргументу для элементов массива нам придётся указать еще и константу для размера самого массива. Константа в генериках определяется тоже в угловых скобках, но в отличие от генерик тип-аргумента, к ней прибавляется ключевое слово `const` и тип константы.

`<const Константа: ТипКонстанты>`

Для примера напишем функцию, которая создаёт массив заданного размера, и инициализирует его элементы заданным значением:

```rust
/// Создаёт массив указанного размера, где все элементы
/// инициализированы заданным значением
fn make_array<T: Copy, const SIZE: usize>(init_value: T) -> [T; SIZE] {
    [init_value; SIZE]
}

fn main() {
    let arr: [i32; 5] = make_array::<i32, 5>(1);
    println!("{arr:?}"); // [1, 1, 1, 1, 1]
}
```

---

К значению генерик константы можно обращаться как к обычной константе. Например, напишем функцию, которая несколько раз печатает на консоль переданное в неё значение. При этом количество раз, которое значение будет напечатано на консоль, зададим `const` генерик-аргументом.

```rust
use std::fmt::Display;

fn print_times<const QUANTITY: usize>(v: impl Display) {
    for _ in 0 .. QUANTITY {
        println!("{v}");
    }
}

fn main() {
    print_times::<3>("Hello");
}
```

