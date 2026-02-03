# Prometheus метрики

Последняя важная тема, которую мы рассмотрим в рамках изучения бэкендов — метрики.

В этой главе мы узнаем, как собирать и предоставлять [Prometheus](https://prometheus.io/docs/introduction/overview/) метрики, работа с которыми обычно выглядит так:

![](img/prometheus-system.svg)

Принцип следующий:

* Сервисы, с которых нужно собирать метрики, должны иметь эндпоинт (для простоты мы будем считать, что его путь — `/metrics`, но его путь может быть любым), который выдаёт метрики в [формате prometheus метрик](https://prometheus.io/docs/concepts/data_model/).
* Prometheus сервис раз в определённое время вызывает эндпоинт, чтобы получить значение метрик и сохранить их. Этот процесс называют скрапингом (scraping — соскабливание).
* Далее метрики из Prometheus сервиса можно запрашивать в агрегированном виде. Например, совместно с Prometheus часто используют систему Grafana — сервис с веб-интерфейсом, который отображает значения метрик в виде графиков и диаграмм.
* Также Prometheus сервер можно настроить таким образом, чтобы при достижении определёнными метриками пороговых значений он генерировал алёрты. При помощи сервиса [AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/) можно легко настроить интеграцию Prometheus алёртов с такими сервисами, как PagerDuty.

В этой главе мы не будем рассматривать установку и настройку Prometheus сервера, а лишь ограничимся созданием эндпоинта `/metrics`, который отдаёт метрики в том формате, который ожидает Prometheus сервер.

## Крэйт metrics

Для работы с Prometheus метриками имеется библиотека [prometheus](https://crates.io/crates/prometheus), которая содержит:

* типы, представляющие все виды Prometheus метрик
* функциональность для форматирования значения метрик в формат, подходящий для скрапинга Prometheus сервером.

Однако работать с API библиотеки [prometheus](https://crates.io/crates/prometheus) напрямую не очень удобно, поэтому обычно используют связку библиотек:

* [metrics](https://crates.io/crates/metrics) — фасад, предоставляющий удобные макросы для работы с метриками
* [metrics-prometheus](https://crates.io/crates/prometheus) — реализация фасада, которая оборачивает крэйт [prometheus](https://crates.io/crates/prometheus)

> [!NOTE]
> Существует также другой крэйт-реализация фасада metrics для Prometheus — [**metrics-exporter-prometheus**](https://crates.io/crates/metrics-exporter-prometheus). Он имеет встроенный HTTP сервер, поэтому может оказаться более удобным для приложений, которые не экспортируют свой HTTP API. Но для наших примеров, в этой главе, мы будем использовать крэйт `metrics-prometheus` так как он проще, и имеет более очевидный API.

Для работы с метриками нам нужно будет добавить в `Cargo.toml` следующие зависимости:

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"

metrics = "0.24"
metrics-prometheus = "0.11"
prometheus = "0.14"
```

## Метрика Counter

Первая метрика, с которой мы разберёмся — [Counter](https://prometheus.io/docs/concepts/metric_types/#counter): числовой счётчик, который подразумевает либо увеличение своего значения, либо рестарт.

Для работы с метрикой Counter библиотека metrics предоставляет удобный макрос [counter](https://docs.rs/metrics/latest/metrics/macro.counter.html), который используется примерно следующим образом:

```rust,noplayground
metrics::counter!("имя_метрики").increment(1);
```

Этот вызов:

* создаёт Counter метрику с заданным именем, если она еще не была зарегистрирована, и инициализирует её нулём
* инкрементирует значение метрики

Рассмотрим простейший пример:

```rust,noplayground
fn main() {
    // Создаём глобальное хранилище метрик.
    let recorder = metrics_prometheus::install();

    // Объявляем, что если у нас будет метрика с именем first_counter, то 
    // её описанием должна быть строка "Some description"
    metrics::describe_counter!("first_counter", "Some description");

    // Создаём и инкрементируем Сounter метрику с именем first_counter
    metrics::counter!("first_counter").increment(1);
    // Увеличиваем my_counter на 2
    metrics::counter!("first_counter").increment(2);

    // Создаём и инкрементируем Сounter метрику с именем second_counter
    metrics::counter!("second_counter").increment(1);

    // Формируем такое текстовое представление значений метрик,
    // которое ожидается Prometheus сервером
    let report: String = prometheus::TextEncoder::new()
        .encode_to_string(&recorder.registry().gather())
        .unwrap();
    
    println!("{report}");
}
```

Эта программа напечатает:

```properties
# HELP first_counter Some description
# TYPE first_counter counter
first_counter 3
# HELP second_counter second_counter
# TYPE second_counter counter
second_counter 1
```

Именно такой текстовый формат использует Prometheus сервис, когда скрапит метрики с приложений путём вызова `/metrics` эндпоинта. Этот формат подразумевает, что для каждой метрики задано:

* `# HELP имя_метрики описание` — текстовое описание метрики.
* `# TYPE имя_метрики тип` — тип метрики: `counter` / `gauge` / `histogram`
* `имя_метрики значение` — текущее значение метрики

***

Теперь давайте рассмотрим, как использовать метрики совместно с Axum сервером. Для примера, модифицируем наш пример Hello-сервера, добавив в него Counter метрику, которая инкрементируется при каждом вызове `/hello`.

```rust,noplayground
use std::sync::Arc;
use axum::{Router, extract::State, routing::get};
use metrics_prometheus::Recorder;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    // Инициализируем объект для сбора метрик. Этот объект также используется для 
    // форматирования значений метрик, запрашиваемых Prometheus сервером.
    let recorder = metrics_prometheus::install();

    let state = AppState { recorder };

    let app = Router::new()
        .route("/hello", get(hello))
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    // Инкрементируем счётчик с именем hello_calls
    metrics::counter!("hello_calls").increment(1);
    "Hello!"
}

// Предполагается, что этот эндпоинт вызывается Prometheus скрапером.
async fn get_metrics(state: State<Arc<AppState>>) -> String {
    let report: String = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Теперь если мы запустим сервер и сразу перейдём по адресу [http://localhost:8080/metrics](http://localhost:8080/metrics), то получим пустой ответ. Так происходит потому, что метрика _hello\_calls_ не была инициализирована никаким значением. Но если мы сначала перейдём на [http://localhost:8080/hello](http://localhost:8080/hello), а потом снова на [http://localhost:8080/metrics](http://localhost:8080/metrics), то получим следующее:

```properties
# HELP hello_calls hello_calls
# TYPE hello_calls counter
hello_calls 1
```

Мы также можем сразу инициализировать метрику, например нулём. Сделать это можно при помощи метода [absolute](https://docs.rs/metrics/latest/metrics/struct.Counter.html#method.absolute):

```rust,noplayground
let recorder = metrics_prometheus::install();
// Сразу после создания объекта prometheus метрик, инициализируем метрику нулём
metrics::counter!("hello_calls").absolute(0);
```

## Измерения

К значениям метрик можно добавлять произвольные атрибуты в формате ключ=>значение, которые называются измерениями (dimension).

```rust,noplayground
metrics::counter!("имя_метрики", "измерение1" => "знач1", "измерение2" => "знач2")
    .increment(1);
```

Обычно измерения используются для задания дополнительной информации к значению метрики.

В качестве примера сделаем метрику, подсчитывающую количество запросов к каждому эндпоинту. Для этого мы создадим метрику `number_of_calls` с измерением `"path"`, которое будет хранить URL путь запроса. И для удобства мы поместим подсчёт этой метрики в отдельный мидлваре.

```rust,noplayground
use std::sync::Arc;
use axum::{Router, extract::{Request, State}, middleware::{Next, from_fn}};
use axum::{response::Response, routing::get};
use metrics_prometheus::Recorder;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    let recorder = metrics_prometheus::install();

    let state = AppState { recorder };

    let app = Router::new()
        .merge(
            // Помещаем во вложенный роутеры те эндпоинты, для которых
            // должно вызываться наше мидлваре с метрикой.
            Router::new()
                .route("/endpoint-1", get(endpoint_1))
                .route("/endpoint-2", get(endpoint_2))
                .layer(from_fn(metrics_middleware))
        )
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn metrics_middleware(request: Request, next: Next) -> Response {
    let path = request.uri().path().to_string();
    metrics::counter!("number_of_calls", "path" => path).increment(1);
    let response = next.run(request).await;
    response
}

async fn endpoint_1() -> &'static str {
    "Endpoint 1"
}

async fn endpoint_2() -> &'static str {
    "Endpoint 2"
}

async fn get_metrics(state: State<Arc<AppState>>) -> String {
    let report = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Запустим сервер, сделаем по запросу к эндпоинтам [http://localhost:8080/endpoint-1](http://localhost:8080/endpoint-1) и [http://localhost:8080/endpoint-2](http://localhost:8080/endpoint-2), после чего запросим метрики с [http://localhost:8080/metrics](http://localhost:8080/metrics).

Мы должны увидеть следующее:

```
# HELP number_of_calls number_of_calls
# TYPE number_of_calls counter
number_of_calls{path="/endpoint-1"} 1
number_of_calls{path="/endpoint-2"} 1
```

## Gauge

Метрика [Gauge](https://prometheus.io/docs/concepts/metric_types/#gauge), в отличие от Counter, подразумевает не только увеличение, но и уменьшение значения. Эта метрика часто используется для того, чтобы отображать текущее состояние некой величины. Например: количество сообщений, находящихся в обработке, нагрузка на процессор, количество свободного места на диске, количество активных соединений к БД и т.д.

Рассмотрим простой пример, который демонстрирует возможности Gauge:

```rust,noplayground
fn main() {
    let recorder = metrics_prometheus::install();

    // Установка абсолютного значения для метрики
    metrics::gauge!("my_gauge").set(5);
    // Инкрементирование значения
    metrics::gauge!("my_gauge").increment(3);
    // Декрементирование
    metrics::gauge!("my_gauge").decrement(1);

    let report = prometheus::TextEncoder::new()
        .encode_to_string(&recorder.registry().gather())
        .unwrap();

    println!("{report}");
}
```

Вывод программы:

```properties
# HELP my_gauge my_gauge
# TYPE my_gauge gauge
my_gauge 7
```

## Histogram

Метрика [Histogram](https://prometheus.io/docs/concepts/metric_types/#histogram) используется для подсчёта частотного распределения значений.

Гистограмма содержит внутри себя несколько счётчиков, каждый из которых связан с неким пороговым значением. Эти счётчики называются бакетами (bucket). По умолчанию Histogram содержит бакеты с такими пороговыми значениями:\
0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, ∞.

Когда мы записываем в метрику очередное значение, то для всех бакетов происходит проверка: является ли значение меньшим, чем пороговое значение бакета. Если является, то счётчик для этого бакета инкрементируется.

Таким образом, мы получаем, сколько записанных значений оказалось меньше, чем 0.005, сколько оказалось меньше, чем 0.01 и т.д.

![](img/prometheus-histogram.svg)

Метрика Histogram особенно удобна для того, чтобы следить за временем ответа для эндпоинтов, временем выполнения запросов в базу данных и т.д.

Для примера напишем мидлваре, который использует Histogram для замера времени обработки запроса. Для того чтобы имитировать разное время выполнения обработчика запроса, мы будем использовать библиотеку [rand](https://crates.io/crates/rand), поэтому добавьте `rand = "0.9"` в `Cargo.toml`.

```rust,noplayground
use std::{sync::Arc, time::Duration};
use axum::{Router, extract::{Request, State}, middleware::{Next, from_fn}};
use axum::{response::Response, routing::get};
use metrics_prometheus::Recorder;
use tokio::time::Instant;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    let recorder = metrics_prometheus::install();

    let state = AppState { recorder };

    let app = Router::new()
        .merge(
            Router::new()
                .route("/endpoint-1", get(endpoint_1))
                .layer(from_fn(metrics_middleware))
        )
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn metrics_middleware(request: Request, next: Next) -> Response {
    let path = request.uri().path().to_string();
    let start = Instant::now(); // Начинаем замер времени выполнения
    let response = next.run(request).await;
    let time_in_seconds = start.elapsed().as_secs_f64(); // Фиксируем время выполнения
    metrics::histogram!("call_duration", "path" => path).record(time_in_seconds);
    response
}

async fn endpoint_1() -> &'static str {
    // Имитируем задержку продолжительностью от 0 до 1000 миллисекунд
    tokio::time::sleep(Duration::from_millis(rand::random_range(0..1000))).await;
    "Endpoint 1"
}

async fn get_metrics(state: State<Arc<AppState>>) -> String {
    let report = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Итак, запустим наш сервер.

Сперва перейдём в браузере на [http://localhost:8080/endpoint-1](http://localhost:8080/endpoint-1) и 9 раз обновим страницу, чтобы в сумме иметь 10 обращений к эндпоинту.

Далее перейдём на [http://localhost:8080/metrics](http://localhost:8080/metrics), чтобы получить метрики:

```
# HELP call_duration call_duration
# TYPE call_duration histogram
call_duration_bucket{path="/endpoint-1",le="0.005"} 0
call_duration_bucket{path="/endpoint-1",le="0.01"} 0
call_duration_bucket{path="/endpoint-1",le="0.025"} 1
call_duration_bucket{path="/endpoint-1",le="0.05"} 2
call_duration_bucket{path="/endpoint-1",le="0.1"} 3
call_duration_bucket{path="/endpoint-1",le="0.25"} 3
call_duration_bucket{path="/endpoint-1",le="0.5"} 7
call_duration_bucket{path="/endpoint-1",le="1"} 10
call_duration_bucket{path="/endpoint-1",le="2.5"} 10
call_duration_bucket{path="/endpoint-1",le="5"} 10
call_duration_bucket{path="/endpoint-1",le="10"} 10
call_duration_bucket{path="/endpoint-1",le="+Inf"} 10
call_duration_sum{path="/endpoint-1"} 4.274135234
call_duration_count{path="/endpoint-1"} 10
```

Нетрудно заметить, что так как вызов `sleep` в нашем эндпоинте засыпает на интервал от 0 до 1000 миллисекунд (то есть не дольше 1 секунды), а мы замеряем время выполнения обработчика в секундах, то все бакеты в гистограмме после бакета с пороговым значением 1.0 не имеют смысла: мы всё равно не засыпаем дольше, чем на секунду.

Для подобных ситуаций, когда мы заранее знаем, в каких пределах будут находиться значения, имеет смысл вручную задать желаемый набор бакетов. Это можно сделать при помощи типа [HistogramOpts](https://docs.rs/prometheus/latest/prometheus/struct.HistogramOpts.html):

```rust,noplayground
let custom_buckets = vec![0.1, 0.25, 0.5, 1.0, 5.0];

let opts = HistogramOpts::new("имя_гистограммы", "Описание метрики")
    .buckets(custom_buckets);
```

Рассмотрим простой пример, где мы конфигурируем Histogram метрику, указывая, что нас интересуют бакеты для пороговых значений: 0.1, 0.25, 0.5, 0.9 и 1.0.

```rust,noplayground
use std::time::Duration;
use prometheus::{Histogram, HistogramOpts};

fn main() {
    let recorder = metrics_prometheus::install();
    // Интересующие нас пороговые значения для бакетов
    let custom_buckets = vec![0.1, 0.25, 0.5, 0.9, 1.0];
    // Задаём конфигурацию (описание и бакеты) для Histogram с именем call_duration
    let opts = HistogramOpts::new("call_duration", "My description")
        .buckets(custom_buckets);
    // Регистрируем конфигурацию для Histogram с именем call_duration
    let histogram = Histogram::with_opts(opts).unwrap();
    recorder.register_metric(histogram);

    for _ in 0 .. 10 {
        // Эмулируем ожидание в интервале от 0 до 1000 миллисекунд
        let latency = Duration::from_millis(rand::random_range(0..1000));
        // Записываем в метрику очередное значение
        metrics::histogram!("call_duration").record(latency.as_secs_f64());
    }

    let report = prometheus::TextEncoder::new()
        .encode_to_string(&recorder.registry().gather())
        .unwrap();

    println!("{report}");
}
```

Вывод программы:

```
# HELP call_duration My description
# TYPE call_duration histogram
call_duration_bucket{le="0.1"} 2
call_duration_bucket{le="0.25"} 5
call_duration_bucket{le="0.5"} 6
call_duration_bucket{le="0.9"} 6
call_duration_bucket{le="1"} 10
call_duration_bucket{le="+Inf"} 10
call_duration_sum 4.932
call_duration_count 10
```

Как видите, в этой Histogram уже нет "бессмысленных" бакетов.

## Метрики процесса

Часто бывает полезно иметь не только метрики, заполняемые непосредственно приложением, но и такие метрики процесса (программы), как потребление CPU и оперативной памяти, количество запущенных потоков, количество открытых файловых дескрипторов и т.д.

Экосистема Rust предлагает множество крэйтов, которые помогают получить метрики хоста и процесса. Мы будем использовать библиотеку [metrics-process](https://crates.io/crates/metrics-process), которая умеет сразу и получать метрики процесса, и записывать их в соответствующие Prometheus метрики.

Для начала добавим `metrics-process` в `Cargo.toml`:

```toml
metrics-process = "2"
```

Библиотека предоставляет тип [metrics\_process::Collector](https://docs.rs/metrics-process/latest/metrics_process/struct.Collector.html), который позволяет записать метрики процесса просто путём вызова:

```rust,noplayground
// Получить объект коллектора
let collector = Collector::default();
// Инициализировать описание метрик текстовками по умолчанию (опционально)
collector.describe();
// Произвести сбор метрик процесса и "втолкнуть" значения в Prometheus метрики
collector.collect();
```

Модифицируем наш самый первый пример из этой главы, добавив сбор метрик процесса:

```rust,noplayground
use std::sync::Arc;
use axum::{Router, extract::State, routing::get};
use metrics_process::Collector;
use metrics_prometheus::Recorder;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    let recorder = metrics_prometheus::install();
    let state = AppState { recorder };
    let app = Router::new()
        .route("/hello", get(hello))
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    metrics::counter!("hello_calls").increment(1);
    "Hello!"
}

async fn get_metrics(state: State<Arc<AppState>>) -> String {
    // Обновляем метрики процесса
    let collector = Collector::default();
    collector.describe();
    collector.collect();

    let report: String = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Запустим сервер и перейдём на [http://localhost:8080/metrics](http://localhost:8080/metrics). Мы должны увидеть что-то похожее на:

```ini
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 0
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1048576
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 16
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 6397952
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1767575577
# HELP process_threads Number of OS threads in the process.
# TYPE process_threads gauge
process_threads 17
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 1116295168
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes 0
```

