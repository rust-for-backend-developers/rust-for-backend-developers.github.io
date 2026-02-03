# Any

## Проблема даункаста

Как мы знаем, Rust не является ООП языком. Тем не менее, благодаря системе трэйтов, Rust предлагает мощные возможности полиморфизма.

Благодаря [трэйт-объектам](../rust-basics/traits.md#dynamic_dispatching), мы в любой момент можем сделать "upcast", то есть начать общаться с типом через его трэйт, абстрагируясь от того, какой конкретно тип скрыт за трэйт-объектом.

```rust
trait Person {}

struct Student {}
impl Person for Student {}

struct Teacher {}
impl Person for Teacher {}

fn work_with_person(person: &dyn Person) {}

fn main() {
    let child1 = Student {};
    work_with_base(&child1); // upcast &Student к &dyn Person
    
    let child2 = Teacher {};
    work_with_base(&child2); // upcast &Teacher к &dyn Person
}
```

Но что делать в ситуации, когда необходимо узнать, какой конкретный тип скрывается за трэйт-объектом? То есть, обращаясь к нашему примеру выше, как, имея ссылку `&dyn Person`, получить из неё ссылку `&Student` или `&Teacher`? Другими словами, сделать "downcast".


> [!NOTE]
> Upcast и downcast — термины из мира ООП, где классы могут наследовать друг друга. Приведение указателя на объект дочернего класса к указателю на родительский класс называется upcast. Словно движение вверх (up) по иерархии. Соответственно, приведение типа указателя с родительского класса на дочерний называется downcast.
> 
> Допустим, у нас есть такая иерархия классов на C++:
> 
> ```cpp
> class Shape {
> protected:
>    float x, y; // location
> }
> 
> class Circle : public Shape {
>     float radius;
> }
> ```
> 
> Тогда
> 
> ```cpp
> Circle* circle1 = new Circle();
> Shape*  shape   = dynamic_cast<Shape*>(circle1); // upcast
> Circle* circle2 = dynamic_cast<Circle*>(shape);  // downcast
> ```

К сожалению, на данный момент в Rust нет никакой возможности узнать, какой реальный тип скрывается за трэйт-объектом. Напомним, ссылка на трэйт-объект — это просто пара указателей: первый — на сам объект, второй — на vtable. При этом vtable не содержит название или ID типа, только указатели на методы, указатель на деструктор (если таков имеется), а также информацию о размере и выравнивании.

Однако есть обходной путь. При компиляции каждому типу присваивается уникальный идентификатор — [TypeId](https://doc.rust-lang.org/std/any/struct.TypeId.html), который является просто обёрткой над 128-битным числом:

```rust
pub struct TypeId {
    t: (u64, u64),
}
```

Чтобы узнать `TypeId` для некого типа, используется метод-конструктор [TypeId::of](https://doc.rust-lang.org/std/any/struct.TypeId.html#method.of).

Например:

```rust
use std::any::TypeId;

struct Student {}
struct Teacher {}

fn main() {
    println!("{:?}", TypeId::of::<Student>()); // TypeId(0xaf8ebb053b28606d84daaf35f1eae84c)
    println!("{:?}", TypeId::of::<Teacher>()); // TypeId(0x318ec7010fde8de82dedcdc9562881fd)
}
```

При этом важно заметить, что `TypeId::of` выполняется на этапе компиляции.

Имея в арсенале `TypeId`, мы можем самостоятельно реализовать проверку: является ли трэйт-объект экземпляром нужного типа или нет. Давайте перепишем самый первый пример из главы, добавив в него downcast.

```rust
use std::any::TypeId;

trait Person where Self: 'static {
    fn exact_type(&self) -> TypeId {
        TypeId::of::<Self>()
    }
}

/// Обратите внимание, что impl не для Person, а для dyn Person - трэйт-объекта
impl dyn Person {
    /// Этот метод даункастит трэйт-объект к ссылке на конкретный тип, если 
    /// этот конкретный тип соответствует типу объекта, сокрытого за трэйт-объектом
    fn downcast<T: 'static>(&self) -> Option<&T> {
        if TypeId::of::<T>() == self.exact_type() {
            unsafe {
                let (data, _vtable): (*const u8, *const u8) =
                    std::mem::transmute(self);
                let data: *const T = std::mem::transmute(data);
                data.as_ref()
            }
        } else {
            None
        }
    }
}

struct Student {
    name: String,
    year_of_education: u32,
}
impl Person for Student {}

struct Teacher {
    name: String,
    subject: String,
}
impl Person for Teacher {}

fn work_with_person(base: &dyn Person) {
    // При помощи нашего метода downcast, проверяем кто скрыт
    // за трэйт-объектом: Student или Teacher
    if let Some(s) = base.downcast::<Student>() {
        println!("This is {}, a {}-year student", s.name, s.year_of_education);
    } else if let Some(t) = base.downcast::<Teacher>() {
        println!("This is {}, a teacher of {}", t.name, t.subject);
    }
}

fn main() {
    let student = Student {
        name: "John".to_string(),
        year_of_education: 3,
    };
    work_with_person(&student);

    let teacher = Teacher {
        name: "Ivan".to_string(),
        subject: "Programming".to_string(),
    };
    work_with_person(&teacher);
}
```

Программа работает как от неё и требовалось:

```
$ cargo run
This is John, a 3-year student
This is Ivan, a teacher of Programming
```

Теперь давайте разберёмся в коде.

Для трэйта `Person` мы добавили метод, который возвращает `TypeId` того типа, который этот трэйт реализует. Если `Person` реализуется структурой `Student`, то вернётся `TypeId` для `Student`, а если структурой `Teacher`, то соответственно `TypeId` для `Teacher`.

```rust
trait Person where Self: 'static {
    fn exact_type(&self) -> TypeId {
        TypeId::of::<Self>()
    }
}
```

> [!TIP]
> <details>
> 
> <summary>Про границу трэйта <code>'static</code>.</summary>
> 
> Граница трэйта `'static` говорит, что **объект**, реализуюший этот трэйт, не должен содержать ссылок, чей лайфтайм короче, чем `'static` лайфтайм.
> 
> Рассмотрим пример:
> 
> ```rust
> struct MyIntRef<'a> {
>     r: &'a i32,
> }
> 
> /// Функция, которая принимает объект со статическим лайфтаймом
> fn prove_static<T: 'static>(v: T) {}
> 
> fn main() {
>     // Это работает
>     static VAR_STATIC: i32 = 5;
>     let my1 = MyIntRef { r: &VAR_STATIC };
>     prove_static(my1);
> 
>     // Это не скомпилируется
>     let var_local = 5;
>     let my2 = MyIntRef { r: &var_local }; // doesn not live long enough
>     prove_static(my2);
> }
> ```
> 
> Получается, что наш пример даункаста от _Person_ к _Student_/_Teacher_ не будет работать, если мы добавим еще одну стукруту, которая реализует трэйт `Person`, но содержит не `'static` ссылки? Да, именно так. Увы, но на сегодняшний существует такое ограничение. Впрочем, на практике, врядли вы когда-либо столкнётесь с ситуаций, когда вам нужно делать downcast для типа содержащего не `static` ссылки.
> 
> </details>

Далее в коде, специально для трэйт-объекта `dyn Person`, мы определили метод, который проверяет, равно ли значение `TypeId` для типа, к которому хотят даункастить наш трэйт-обект, значению `TypeId` для типа объекта, который сокрыт за трэйт-объектом (`Student` или `Teacher`). Если да, то мы извлекаем из трэйт-объекта указатель на данные и приводим к желаемому типу ссылки. Иначе — возвращаем `None`.

```rust
impl dyn Base {
    fn downcast<T: 'static>(&self) -> Option<&T> {
        if TypeId::of::<T>() == self.exact_type() {
            unsafe {
                // Деструктурируем сслыку на трэйт-объект в пару указателей
                let (data, _vtable): (*const u8, *const u8) =
                    std::mem::transmute(self);
                // Приводим сырой указатель, к указателю на конкретный тип
                let data: *const T = std::mem::transmute(data);
                // Превращаем указатель в ссылку: as_ref() возвращает Option
                data.as_ref()
            }
        } else {
            None
        }
    }
}
```

Чтобы понять, каким образом мы получаем ссылку на объект, сокрытый за трэйт-объектом, мы должны еще раз вспомнить, что из себя представляет ссылка на трэйт-объект. Она состоит из пары указателей:

* первый — непосредственно на данные
* второй — на vtable (таблицу виртуальных вызовов).

Поэтому мы сначала преобразовываем `self` (ссылку на трэйт-объект) в пару указателей, затем берём первый указатель и приводим его к ссылке желаемого типа.

Для приведения `self` к кортежу из двух указателей мы используем функцию [transmute](https://doc.rust-lang.org/std/mem/fn.transmute.html). Эта функция позволяет как бы посмотреть на одну и ту же память (последовательность байт) через призму другого типа. Разумеется, этой функцией надо пользоваться очень осторожно и учитывать не только размеры значений, но и возможные выравнивания. К счастью, в нашем случае мы имеем дело с двумя указателями, а они всегда имеют размер, равный размеру машинного слова, поэтому не нуждаются в выравнивании.

> [!TIP]
> На самом деле фрагмент
> 
> ```rust
> let (data, _vtable): (*const u8, *const u8) = std::mem::transmute(self);
> let data: *const T = std::mem::transmute(data);
> data.as_ref()
> ```
> 
> можно записать куда проще:
> 
> ```rust
> Some(&*(self as *const dyn Person as *const T));
> ```
> 
> однако такая запись прячет те детали работы с трэйт-объектом, которые мы хотели продемонстрировать явно.

Остальной код из примера должен быть понятен.

## Даункаст с Any

Разумеется, реализовывать такой даункаст вручную неудобно и опасно, и в примере выше мы сделали это лишь для того, чтобы показать, как происходит даункаст на уровне памяти.

На практике же стандартная библиотека Rust предоставляет удобную (относительно) трэйт обёртку [Any](https://doc.rust-lang.org/std/any/trait.Any.html), которая инкапсулирует в себе `TypeId`, тем самым позволяя делать даункаст.

Трэйт `Any` имеет вид:

```rust
pub trait Any: 'static {
    fn type_id(&self) -> TypeId;
}

impl<T: 'static + ?Sized> Any for T {
    fn type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

Как видно из объявления, абсолютно для любого типа, реализующего `'static`, определена реализация `Any`, которая просто получает `TypeId` типа. По сути, это то же самое, что мы делали методом _exact\_type_ в примере выше.

Также для `dyn Any` определены такие методы:

* `pub fn is<T: Any>(&self) -> bool`\
  Возвращает `true`, если реальный тип, сокрытый за трэйт-объектом — это `T`
* `pub fn downcast_ref<T: Any>(&self) -> Option<&T>`\
  Если реальный тип, сокрытый за трэйт-объектом — это `T`, то возвращает `Some(&T)`, иначе `None`.
* `pub fn downcast_mut<T: Any>(&mut self) -> Option<&mut T>`\
  Если реальный тип, сокрытый за трэйт-объектом — это `T`, то возвращает `Some(&mut T)`, иначе `None`.

Пример простого даункаста посредством `Any`:

```rust
use std::any::Any;

fn main() {
    let s = "hello".to_string();
    let any = &s as &dyn Any;

    println!("any is i32: {}", any.is::<i32>());       // any is i32: false
    println!("any is String: {}", any.is::<String>()); // any is String: true

    println!("{:?}", any.downcast_ref::<i32>());    // None
    println!("{:?}", any.downcast_ref::<String>()); // Some("hello")
}
```

Если трэйт наследует `Any`, то его трэйт-объект автоматически получает возможность делать даункаст.

Перепишем с использованием `Any` наш пример с самописным даункастом.

```rust
use std::any::Any;

trait Person: Any {}

struct Student {
    name: String,
    year_of_education: u32,
}
impl Person for Student {}

struct Teacher {
    name: String,
    subject: String,
}
impl Person for Teacher {}

fn work_with_person(person: &dyn Person) {
    let any = person as &dyn Any;
    if let Some(s) = any.downcast_ref::<Student>() {
        println!("This is {}, a {}-year student", s.name, s.year_of_education);
    } else if let Some(t) = any.downcast_ref::<Teacher>() {
        println!("This is {}, a teacher of {}", t.name, t.subject);
    }
}

fn main() {
    let student = Student {
        name: "John".to_string(),
        year_of_education: 3,
    };
    work_with_person(&student);

    let teacher = Teacher {
        name: "Ivan".to_string(),
        subject: "Programming".to_string(),
    };
    work_with_person(&teacher);
}
```

Результат работы программы такой же, как и с нашим собственным даункастом:

```
$ cargo run
This is John, a 3-year student
This is Ivan, a teacher of Programming
```

