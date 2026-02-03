# Дата и время

## Таймеры

В стандартной библиотеке Rust отсутствует функциональность для работы с датами и астрономическим временем, но имеются следующие три типа:

* [std::time::Duration](https://doc.rust-lang.org/stable/std/time/struct.Duration.html) — представляет собой временной отрезок.\
  Например: 5 секунд, 3 минуты, 1 час.
* [std::time::SystemTime](https://doc.rust-lang.org/std/time/struct.SystemTime.html) — API для работы с системными часами.
* [std::time::Instant](https://doc.rust-lang.org/std/time/struct.Instant.html) — API для работы с монотонным таймером.

#### Duration

Тип `Duration` устроен очень просто:

```rust
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
pub struct Duration {
    secs: u64,
    nanos: Nanoseconds,
}
```

Он позволяет представлять временные промежутки большой длины (до 584 миллиардов лет) с точностью до наносекунды.

```rust
use std::time::Duration;

fn main() {
    let d1 = Duration::from_secs(5); // 5 seconds
    let d2 = Duration::from_mins(1); // 1 minute
    let d3 = d2.saturating_sub(d1); // 55 seconds
    println!("{}", d3.as_secs()); // 55
}
```

Сам по себе тип `Duration` умеет немного, однако он используется как входной аргумент в различной функциональности. Например, стандартная функция [sleep](https://doc.rust-lang.org/std/thread/fn.sleep.html), которая останавливает выполнение потока на указанное время, принимает аргумент типа `Duration`.

#### SystemTime

Системный таймер хранит астрономическое время. Как правило, системное время реализовано при помощи чипа с кварцевым генератором, находящимся на системной плате и питаемым от батарейки, даже когда компьютер выключен.

Если мы хотим получить, например, текущее Unix время (количество секунд, прошедшее с 1 января 1970 года), то мы должны использовать как раз системный таймер.

```rust
use std::time::{Duration, SystemTime};

fn main() {
    // Получаем текущее значение системного времени
    let sys_time_now: SystemTime = SystemTime::now();

    // Конвертируем системное время в Duration, начальной
    // точкой которого является начало эпохи Unix
    let unix_time_duration: Duration =
            sys_time_now.duration_since(SystemTime::UNIX_EPOCH).unwrap();

    // Конвертируем Duration в количество секунд
    let unix_time_now: u64 = unix_time_duration.as_secs();

    println!("Now (Unix-time): {unix_time_now}");
}
```

Однако у системного таймера есть проблема: если вызвать `SystemTime::now()` и сразу после этого вызвать `SystemTime::now()` еще раз (особенно из другого потока), то есть вероятность, что время, полученное в результате второго вызова, будет раньше, чем время, полученное в результате первого вызова. Почему так происходит — отдельная история, но главное, что нам нужно знать: не следует использовать системный таймер для того, чтобы замерять время выполнения участков кода.

#### Instant

Монотонный таймер хранит не астрономическое время, а время, прошедшее с момента старта операционной системы. В зависимости от архитектуры компьютера и операционной системы, монотонный таймер может быть реализован по-разному. Например, на x86-64 процессорах монотонный таймер реализован в виде регистра TSC (Time Stamp Counter). Самое важное для нас то, что монотонный таймер гарантирует, что каждый последующий запрос текущего времени будет выдавать значение не меньшее, чем выдал предыдущий запрос. Поэтому монотонный таймер безопасно использовать для замера времени выполнения участков кода.

```rust
use std::time::{Duration, Instant};

fn main() {
    let start = Instant::now();
    //... какая-то функциональность
    let took: Duration = start.elapsed();
}
```

## chrono

Стандартом де-факто для работы с датой и временем считается библиотека [chrono](https://crates.io/crates/chrono).

Для представления даты и времени chrono предлагает следующие типы:

* [NaiveDate](https://docs.rs/chrono/latest/chrono/struct.NaiveDate.html) — ISO 8601 дата без учёта часового пояса
* [NaiveTime](https://docs.rs/chrono/latest/chrono/struct.NaiveTime.html) — ISO 8601 время без учёта часового пояса
* [NaiveDateTime](https://docs.rs/chrono/latest/chrono/struct.NaiveDateTime.html) — ISO 8601 дата и время без учёта часового пояса
* [DateTime](https://docs.rs/chrono/latest/chrono/struct.DateTime.html)<[TimeZone](https://docs.rs/chrono/latest/chrono/trait.TimeZone.html)> — ISO 8601 дата и время с учётом часового пояса

Давайте подключим chrono в качестве зависимости в `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
chrono = { version = "0.4", features = ["serde"]}
```

> [!NOTE]
> Обратите внимание, что типы данных из библиотеки chrono поддерживают сериализацию посредством [serde](https://crates.io/crates/serde), однако эту функциональность надо отдельно включить при помощи фичи "serde".

### NaiveDate

Рассмотрим тип `NaiveDate`, который позволяет:

* создавать объект даты
* конвертировать дату в строковое представление
* парсить дату из строки
* добавлять и вычитать годы, месяцы и дни
* итерироваться по дням, месяцам, годам

Рассмотрим эти возможности на примере:

```rust,edition2024
# use chrono;
use chrono::{Days, Months, NaiveDate};

fn main() {
    let date: NaiveDate = NaiveDate::from_ymd_opt(2025, 12, 15).unwrap();
    println!("{:?}", date); // 2025-12-15
    println!("{}", date.format("%Y-%m-%d")); // 2025-12-15
    println!("{}", date.format("%Y/%d/%m")); // 2025/15/12

    let date = NaiveDate::parse_from_str("2025-01-01", "%Y-%m-%d").unwrap();
    let three_days = date.iter_days()
        .skip(1)
        .take(3)
        .map(|d|d.to_string())
        .collect::<Vec<_>>()
        .join(",");
    println!("Three days since {date}: {three_days}");
    // Three days since 2025-01-01: 2025-01-02,2025-01-03,2025-01-04

    let new_date = date.checked_add_months(Months::new(2)).unwrap()
        .checked_add_days(Days::new(5)).unwrap();
    println!("{}", new_date); // 2025-03-06
}
```

### NaiveTime

Тип `NaiveTime` предоставляет похожую функциональность, но для времени:

```rust,edition2024
# use chrono;
use chrono::NaiveTime;

fn main() {
    let time1: NaiveTime = NaiveTime::from_hms_opt(13, 51, 10).unwrap();
    println!("{:?}", time1); // 13:51:10
    println!("{}", time1.format("%H-%M-%S")); // 13-51-10

    let time2 = NaiveTime::parse_from_str("12:01:00", "%H:%M:%S").unwrap();
    let time_diff = time1.signed_duration_since(time2);
    println!("{} seconds", time_diff.num_seconds()); // 6610 seconds
}
```

### NaiveDateTime

`NaiveDateTime` — это комбинация `NaiveDate` и `NaiveTime`.

```rust,edition2024
# use chrono;
use chrono::{NaiveDate, NaiveDateTime, NaiveTime};

fn main() {
    let dt: NaiveDateTime = NaiveDateTime::new(
        NaiveDate::from_ymd_opt(2025, 11, 15).unwrap(),
        NaiveTime::from_hms_nano_opt(13, 51, 10,123456789).unwrap()
    );
    println!("{}", dt); // 2025-11-15 13:51:10.123456789
    println!("{}", dt.format("%Y-%m-%dT%H:%M:%S.%3f")); // 2025-11-15T13:51:10.123

    let dt = NaiveDateTime::parse_from_str(
        "2025-11-15T13:51:10.123", "%Y-%m-%dT%H:%M:%S.%3f"
    ).unwrap();
    let date = dt.date();
    let time = dt.time();
}
```

### DateTime

`DateTime<Tz>` хранит дату и время с учетом часового пояса, причём часовой пояс задаётся генерик тип-аргументом, реализующим трэйт [TimeZone](https://docs.rs/chrono/latest/chrono/trait.TimeZone.html).

Библиотека chrono содержит три стандартных реализации для трэйта `TimeZone`:

* [Utc](https://docs.rs/chrono/latest/chrono/struct.Utc.html) — UTC часовой пояс (нулевой меридиан)
* [Local](https://docs.rs/chrono/latest/chrono/struct.Local.html) — текущий часовой пояс
* [FixedOffset](https://docs.rs/chrono/latest/chrono/struct.FixedOffset.html) — хранит фиксированный сдвиг в <ins>секундах</ins> относительно UTC

Эти реализации проще понять на примере:

```rust,edition2024
# use chrono;
use chrono::{DateTime, FixedOffset, Local, Utc};
fn main() {
    // Получаем текущие дату и время в UTC и локальной тайм зоне
    let utc: DateTime<Utc> = Utc::now();
    let local: DateTime<Local> = Local::now();
    // Конвертируем в форму с фиксированным сдвигом
    let utc_fixed: DateTime<FixedOffset> = utc.fixed_offset();
    let local_fixed: DateTime<FixedOffset> = local.fixed_offset();

    println!("UTC:   {utc}");
    println!("Local: {local}");
    println!("UTC fixed:   {}", utc_fixed);
    println!("Local fixed: {}", local_fixed);

    // Конвертируем дату и время из формы с таймзоной в форму без таймзоны
    println!("UTC -> naive UTC:     {}", utc.naive_utc());
    println!("UTC -> naive Local:   {}", utc.naive_local());
    println!("Local -> naive Local: {}", local.naive_local());
    println!("Local -> naive UTC:   {}", local.naive_utc());
}
```

Вывод программы (компьютер, запускавший пример, находился в часовом поясе +2):

```
UTC:   2025-12-15 14:42:07.078561733 UTC
Local: 2025-12-15 16:42:07.078569548 +02:00
UTC fixed:   2025-12-15 14:42:07.078561733 +00:00
Local fixed: 2025-12-15 16:42:07.078569548 +02:00
UTC -> naive UTC:     2025-12-15 14:42:07.078561733
UTC -> naive Local:   2025-12-15 14:42:07.078561733
Local -> naive Local: 2025-12-15 16:42:07.078569548
Local -> naive UTC:   2025-12-15 14:42:07.078569548
```

Объект `DateTime` также может быть создан:

* из компонентов времени и даты
* из объекта `NaiveDateTime` путём указания тайм зоны
* распарсив строку

```rust,edition2024
# use chrono;
use chrono::{DateTime, NaiveDate, TimeZone, Utc};

fn main() {
    let utc1: DateTime<Utc> = Utc.with_ymd_and_hms(2025, 12, 15, 13, 51, 10).unwrap();
    println!("{utc1}"); // 2025-12-15 13:51:10 UTC

    let utc2: DateTime<Utc> =
        NaiveDate::from_ymd_opt(2025, 12, 15).unwrap() // Создаём NavieDate
            .and_hms_opt(13, 51, 10).unwrap() // Превращаем в NaiveDateTime
            .and_utc(); // Добавляем тайм зону
    println!("{utc2}"); // 2025-12-15 13:51:10 UTC

    let dt = DateTime::parse_from_str(
            "2025-12-15 13:51:10 +0000",
            "%Y-%m-%d %H:%M:%S %z"
        ).unwrap();
    println!("{dt}"); // 2025-12-15 13:51:10 +00:00

    let utc3 = dt.to_utc();
    println!("{utc3}"); // 2025-12-15 13:51:10 UTC
}
```

Чтобы получить строковое представление объекта `DateTime`, так же как и для `Naive*` собратьев, используется метод [format](https://docs.rs/chrono/latest/chrono/struct.DateTime.html#method.format). Плюс имеются отдельные методы для форматирования в RFC2822 и RFC3339.

```rust,edition2024
# use chrono;
use chrono::{TimeZone, Utc};

fn main() {
    let utc = Utc.with_ymd_and_hms(2025, 12, 15, 13, 51, 10).unwrap();
    println!("RFC3339: {}", utc.to_rfc3339());
    // RFC3339: 2025-12-15T13:51:10+00:00
    
    println!("RFC2822: {}", utc.to_rfc2822());
    // RFC2822: Mon, 15 Dec 2025 13:51:10 +0000
    
    println!("{}", utc.format("%Y-%m-%d %H:%M:%S.%3f %z"));
    // 2025-12-15 13:51:10.000 +0000
}
```

Последнее, с чем нам осталось разобраться — `DateTime<FixedOffset>`. Тип `FixedOffset` позволяет задавать любую тайм зону путём указания смещения в секундах относительно UTC. Для примера давайте создадим тайм зону UTC+2:30, т.е. смещение на восток от Гринвича на 2 часа 30 минут.

```rust,edition2024
# use chrono;
use chrono::{DateTime, FixedOffset, TimeZone, Utc};

fn main() {
    // Количество секунд в 2 часах и 30 минутах
    let hours_02_minutes_30 = 2 * 3600 + 30 * 60;
    // Таймзона со сдвигом +02:30
    let tz_02_30: FixedOffset = FixedOffset::east_opt(hours_02_minutes_30).unwrap();

    // Создание объекта DateTime из временных компонентов
    let dt: DateTime<FixedOffset> =
        tz_02_30.with_ymd_and_hms(2025, 12,15, 13, 51, 10).unwrap();
    println!("{dt}"); // 2025-12-15 13:51:10 +02:30

    // Конвертирование UTC времени в UTC+2:30
    let now_utc: DateTime<Utc> = Utc::now();
    let now_02_30: DateTime<FixedOffset> = now_utc.with_timezone(&tz_02_30);
    println!("UTC:    {now_utc}");   // UTC:    2025-12-15 16:42:19.043035374 UTC
    println!("+02:30: {now_02_30}"); // +02:30: 2025-12-15 19:12:19.043036305 +02:30
}
```

