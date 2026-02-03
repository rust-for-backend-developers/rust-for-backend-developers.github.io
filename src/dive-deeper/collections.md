# Коллекции

Стандартная библиотека Rust предоставляет такие типы коллекций:

* [Vec\<T>](https://doc.rust-lang.org/std/vec/struct.Vec.html) — Вектор: с ним мы уже знакомы
* [LinkedList\<T>](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) — Двусвязный список
* [VecDeque\<T>](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — Динамически расширяемый кольцевой буфер. Ведёт себя как вектор, который позволяет добавлять элементы как в начало, так и в конец.
* [HashMap\<R,V>](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — Хеш-таблица
* [HashSet\<T>](https://doc.rust-lang.org/stable/std/collections/struct.HashSet.html) — Хеш-множество
* [BTreeMap\<K,V>](https://doc.rust-lang.org/stable/std/collections/struct.BTreeMap.html) — Словарь (ключ-значение), основанный на структуре B-дерево (B-Tree).
* [BinaryHeap\<T>](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — Очередь с приоритетом, основанная на [двоичной куче](https://ru.algorithmica.org/cs/basic-structures/heap/).

## Vec

Мы уже познакомились с вектором и его устройством в памяти в главе [Вектор](../rust-basics/vector.md).

Давайте просто взглянем на примеры часто используемых методов.

```rust
use std::cmp::Ordering;

fn main() {
    // Создаём вектор с буфером на 8 элементов
    let mut v: Vec<i32> = Vec::with_capacity(10);

    // Добавляем элементы
    v.push(0);
    v.push(4);
    v.push(1);
    v.push(3);
    v.push(2);

    // Добавляем в вектор сразу несколько элементов из слайса
    v.extend_from_slice(&[5, 6]);

    // Добавляем в вектор сразу несколько элементов из итератора
    v.extend([8, 7, 9].iter());

    println!("{v:?}"); // [0, 4, 1, 3, 2, 5, 6, 8, 7, 9]

    // Сортируем вектор (по возрастанию)
    v.sort();

    println!("{v:?}"); // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

    // Сортируем по убыванию
    v.sort_by(|a, b| {
        if a > b {
            Ordering::Less
        } else if a < b {
            Ordering::Greater
        } else {
            Ordering::Equal
        }
    });

    println!("{v:?}"); // [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]

    // Извлекаем последний элемент из вектора
    println!("Last: {:?}", v.pop()); // Last: Some(0)

    v.extend_from_slice(&[1, 2, 3]);

    println!("{v:?}"); // [9, 8, 7, 6, 5, 4, 3, 2, 1, 1, 2, 3]

    // Удаляем рядомстоящие дубликаты
    v.dedup();

    println!("{v:?}"); // [9, 8, 7, 6, 5, 4, 3, 2, 1, 2, 3]
}

```

## LinkedList

`LinkedList` представляет из себя классический двусвязный список.

Каждый элемент представлен типом `Node` — структурой с тремя полями:

* значение элемента
* указатель на предыдущий элемент в списке
* указатель на следующий элемент.

Сам же список `LinkedList` — структура с полями:

* указатель на первый элемент списка
* указатель на последний элемент списка
* длина списка

![](img/collections-linked_list.svg)

В отличие от вектора, двусвязный список позволяет эффективно вставлять/удалять элементы в середину и начало списка.

Пример:

```rust
use std::collections::LinkedList;

fn main() {
    let mut list = LinkedList::<i32>::new();
    list.push_back(1); // [1]
    list.push_back(2); // [1,2]
    list.push_back(3); // [1,2,3]
    list.push_front(0); // [0,1,2,3];

    println!("{list:?}"); // [0, 1, 2, 3]

    let last: Option<i32> = list.pop_back();
    println!("{:?}", last); // Ok(3)

    let first: Option<i32> = list.pop_front();
    println!("{:?}", first); // 0

    println!("{list:?}"); // [1, 2]
}
```

## VecDeque

Двусвязный список позволяет вставлять и удалять элементы в середине и в начале списка, но его недостатком является то, что операция доступа к элементу по индексу имеет сложность O(n/2), в отличие от сложности O(1) у вектора. Этот недостаток стремится исправить коллекция `VecDeque`.

`VecDeque` реализован как кольцевой буфер, что позволяет эффективно добавлять элементы в начало списка и относительно эффективно в середину списка. При этом `VecDeque`, как и вектор, имеет O(1) сложность доступа к элементу по индексу.

![](img/collections-vecdeque.svg)

Пример использования:

```rust
use std::collections::VecDeque;

fn main() {
    let mut v = VecDeque::from([1, 2, 3]);
    v.insert(2, 99);
    v.push_front(0);
    println!("{v:?}"); // [0, 1, 2, 99, 3]

    let last: Option<i32> = v.pop_back();
    println!("{:?}", last); // Ok(3)

    let first: Option<i32> = v.pop_front();
    println!("{:?}", first); // 0

    println!("{v:?}"); // [1, 2, 99]
}
```

## HashMap — Хеш-таблица

`HashMap<K,V>` — словарь, который хранит пары ключ-значение в виде хеш-таблицы.

Элементы хранятся в непрерывном буфере, поделённом на секции — бакеты (bucket).

При вставке новой пары:

1. для ключа вычисляется хеш-код
2. к хеш-коду применяется маска (bucket_mask), чтобы определить номер бакета, в котором будет храниться эта пара
3. По номеру бакета (и "Control Bytes" преамбуле в начале буфера) определяется позиция в буфере, куда должна быть вставлена пара ключ-значение
4. Если после вставки количество элементов (хранится в Control Bytes) в бакете превысило предельное значение (growth_left), то выделяется новый буфер большего размера и в него переносятся значения из текущего буфера.

![](img/collections-hash_amp.svg)

Тип ключа должен реализовать трэйты `Eq` и `Hash`, а тип значения — `PartialEq`.

Пример использования:

```rust
use std::collections::HashMap;

fn main() {
    let mut h: HashMap<i32, String> = HashMap::new();
    h.insert(1, "one".to_string());
    h.insert(2, "two".to_string());
    h.insert(3, "three".to_string());
    h.insert(4, "three".to_string());
    h.insert(5, "three".to_string());

    println!("Pairs:");
    for (key, value) in h.iter() {
        println!("  {key} -> {value}");
    }

    println!("Keys:");
    for key in h.keys() {
        println!("  {key}");
    }

    println!("Values:");
    for value in h.values() {
        println!("  {value}");
    }

    println!("Getting reference to value for key=1:");
    if let Some(reference) = h.get(&1) {
        println!("  {reference}");
    }

    println!("Extracting value for key=1:");
    if let Some(value) = h.remove(&1) {
        println!("  {value}");
    }

    println!("Contains key 1: {}", h.contains_key(&1));
}
```

## HashSet — Хеш-множество

`HashSet<T>` — хеш-множество, позволяющее быстро вставлять и искать  элементы. Реализовано как обёртка над `HashMap<T, ()>`, поэтому всё, что справедливо для `HashMap`, справедливо и для `HashSet`.

Пример использования:

```rust
use std::collections::HashSet;

fn main() {
    let mut s: HashSet<String> = HashSet::new();
    s.insert("one".to_string());
    s.insert("two".to_string());
    s.insert("three".to_string());

    for value in s.iter() {
        println!("{value}");
    }

    if let Some(reference) = s.get(&"one".to_string()) {
        println!("{reference}");
    }

    if let Some(value) = s.take(&"one".to_string()) {
        println!("{value}");
    }

    println!("HashSet contains 'one': {}", s.contains(&"one".to_string()));
}
```

## BTreeMap

`BTreeMap<K,V>`, как и `HashMap<K,V>`, является словарём, который хранит пары ключ-значение. Разница лишь в том, что `BTreeMap` хранит данные в виде структуры B-дерево (B-Tree).

![](img/collections-btree_map.svg)

С точки зрения API, `BTreeMap` практически идентичен `HashMap`: в примере для `HashMap` вы можете заменить тип словаря на с `HashMap` на `BTreeMap`, и программа скомпилируется.

Единственная разница: для `HashMap` тип ключа должен реализовать трэйты `Hash` и `Eq`, а для `BTreeMap` тип ключа должен реализовать трэйт `Ord`.

Операции вставки и поиска для `BTreeMap` работают немного медленнее, чем для `HashMap`, особенно при большом размере словаря (больше тысячи элементов). Однако `BTreeMap` хранит элементы в отсортированном по ключу порядке, что может быть удобно в некоторых сценариях использования.

## BinaryHeap

`BinaryHeap<T>` — очередь с приоритетом (priority queue), реализованная как двоичная куча (binary heap).

Куча реализована поверх вектора.

![](img/collections-binary_heap.svg)

> [!NOTE]
> Если читатель не знаком с двоичной кучей, то может создаться впечатление, что это не что иное, как двоичное дерево. Однако это не так. В двоичном дереве значения в левой ветви обязательно должны быть меньше значений в правой ветви, в то время как куча такого ограничения не имеет. Главное, только чтобы значение в родительском узле было больше/меньше (в зависимости от типа кучи), чем в дочерних.

При добавлении новых элементов или удалении существующих куча перестраивается таким образом, что самый верхний узел всегда содержит наибольший элемент. Просто снимая с кучи самый верхний элемент, мы получаем последовательность, отсортированную по убыванию.

Пример использования:

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut queue: BinaryHeap<i32> = BinaryHeap::new();
    queue.push(2);
    queue.push(7);
    queue.push(1);
    queue.push(4);
    queue.push(9);

    while let Some(element) = queue.pop() {
        print!(" {element}"); // 9 7 4 2 1
    }
}
```

Так как `BinaryHeap` упорядочивает элементы путём сравнения, тип элементов должен реализовать трэйт `Ord`.
