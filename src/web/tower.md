# Tower

Настало время поговорить о библиотеке [Tower](https://crates.io/crates/tower). Эта библиотека предоставляет инструментарий для создания клиентов и серверов из отдельных переиспользуемых блоков.

Первое, что нам необходимо сделать — добавить в `Cargo.toml` зависимости [tower](https://crates.io/crates/tower) и [tower_http](https://crates.io/crates/tower-http).

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"
tower = { version = "0.5", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "fs"] }

tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Базовый тип, который предоставляет Tower — трэйт [Service](https://docs.rs/tower/latest/tower/trait.Service.html), являющийся абстракцией над асинхронной функцией.

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // Работает по принципу poll в трэйте Future:
    // * Возвращает Poll::Ready(Ok(())), если сервис готов к обработке запросов,
    //   и можно вызывать call
    // * Возвращает Poll::Pending, если не готов. Экзекьютор будет оповещён
    //   о готовности через Context
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    // Используется для непосредственного вызова функциональности сервиса
    fn call(&mut self, req: Request) -> Self::Future;
}
```

`Service` — абстракция над асинхронной функцией, которая принимает в качестве аргумента объект некоторого генерик-типа `Request` и возвращает либо некий генерик `Response`, либо ошибку.

Можно рассматривать сервис как `async (Request) => Result<Response, Error>`.

Такая абстракция — как раз то, что нужно для построения сетевого сервера. Например:

* Функции-обработчики запросов, которые мы регистрируем в роутере, по своей сути являются асинхронными функциями, которые принимают HTTP Request и возвращают HTTP Response.
* Мидлваре, с которыми мы познакомились в прошлой главе, также, по сути, являются асинхронными функциями, принимающими HTTP Request и возвращающими HTTP Response.

Хорошо, мы убедились, что такие сущности, как обработчики запросов и мидлваре, можно абстрагировать через трэйт `Service`. Но что это нам даёт?

Дело в том, что если различные обработчики абстрагированы до <ins>единого</ins> интерфейса, то они превращаются в универсальные строительные блоки, из которых можно строить цепочки для обработки запросов. То есть можно создать универсальные реализации для таких вещей, как retry, back-pressure, фильтрация запросов, CORS, компрессия, таймауты и т.д. И именно это и представляет из себя библиотека tower: трэйт `Service` и множество готовых универсальных строительных блоков, построенных на его основе.

Для нас всё это особенно интересно по причине того, что Axum позволяет использовать реализации `Service` и в качестве обработчиков запросов, и в качестве мидлваре.

## Обработчик запроса как Service

До этого момента мы создавали эндпоинты путём регистрации функций-обработчиков в роутере при помощи функции [route](https://docs.rs/axum/latest/axum/struct.Router.html#method.route). При этом роутер предоставляет еще один метод — [route_service](https://docs.rs/axum/latest/axum/struct.Router.html#method.route_service), который регистрирует `Service` в качестве обработчика:

```rust
pub fn route_service<T>(self, path: &str, service: T) -> Self
where
    T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static,
    T::Response: IntoResponse,
    T::Future: Send + 'static,
```

`where` блок гласит, что для того чтобы тип выступал в роли обработчика запроса, он должен иметь такую реализацию трэйта `Service`, в которой:

* В качестве типа входного аргумента используется `axum::extract::Request`.
* Метод `call` возвращает объект типа, реализующего `IntoResponse`, завёрнутый только в `Ok`, не в `Err`. При возникновении ошибки можно возвращать объект `Ok(Response)` с соответствующим HTTP кодом, но это должен быть именно `Ok`.

То есть:

```rust,noplayground
impl Service<Request> for МойСервис {
  type Response = Response;
  type Error = Infallible;
  type Future = Pin<Box<dyn Future<Output=Result<Self::Response,Self::Error>>+Send>>;

  fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    ...
  }

  fn call(&mut self, req: Request) -> Self::Future {
    ...
  }
}
```

> [!TIP]
> [std::convert::Infallible](https://doc.rust-lang.org/std/convert/enum.Infallible.html) — тип ошибки, объявленный как пустое перечисление, что делает невозможным создание объекта этого типа.
> Он используется в ситуациях, когда метод, объявленный в трэйте, возвращает `Result`, но от некоторых реализаций этого трэйта требуется, чтобы они никогда не возвращали ошибку.

За основу для примера возьмём наш традиционный hello сервер, но на этот раз в качестве обработчика запроса будет выступать не функция, а объект, чей тип реализует трэйт `Service`.

```rust,noplayground
use axum::{Router, extract::Request, response::{Response,IntoResponse}};
use axum::http::{Method, StatusCode};
use std::{future::Future, pin::Pin, task::{Context, Poll}, convert::Infallible};
use tower::Service;

#[derive(Clone)]
struct HelloService {
    greeting: String,
}

impl Service<Request> for HelloService {
    type Response = Response;
    type Error = Infallible;
    type Future = Pin<Box<dyn Future<Output=Result<Self::Response,Self::Error>>+Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: Request) -> Self::Future {
        if req.method() == Method::GET {
            let greeting = self.greeting.clone();
            Box::pin(async move { Ok(greeting.into_response()) })
        } else {
            Box::pin(async move { Ok(StatusCode::METHOD_NOT_ALLOWED.into_response()) })
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route_service("/hello", HelloService { greeting: "Hello!".to_string() });
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Запустим сервер и перейдём на [http://localhost:8080/hello](http://localhost:8080/hello). Мы должны увидеть всё то же "Hello!".

Нетрудно заметить, что создавать обработчик запроса из типа, реализующего `Service`, более трудозатратно, чем создавать обработчик из функции. Однако такой подход предоставляет больше гибкости, так как позволяет нашему объекту-обработчику иметь внутреннее состояние.

## Мидлваре как Service

Как мы знаем из прошлой главы, мидлваре регистрируется в роутере при помощи метода [layer](https://docs.rs/axum/latest/axum/struct.Router.html#method.layer), который мы наконец-то готовы разобрать подробнее. Он имеет следующую сигнатуру:

```rust,noplayground
pub fn layer<L>(self, layer: L) -> Router<S>
where
    L: Layer<Route> + Clone + Send + Sync + 'static,
    L::Service: Service<Request> + Clone + Send + Sync + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static
```

Здесь [Layer](https://docs.rs/tower-layer/latest/tower_layer/trait.Layer.html) — трэйт для типов, которые возвращают объект, реализующий трэйт `Service`. Можно сказать, что `Layer` — это фабрика для `Service`. Сам трэйт `Layer` объявлен так:

```rust,noplayground
pub trait Layer<S> {
    type Service;

    fn layer(&self, inner: S) -> Self::Service;
}
```

Метод `layer` в качестве аргумента `inner` принимает объект, который является либо следующим мидлваре в цепочке, либо обработчиком запроса. Метод возвращает объект мидлваре, который будет встроен в цепочку обработки перед звеном, переданным в аргумент `inner`.

В качестве примера перепишем в виде сервиса наш мидлваре из прошлой главы: тот, который логирует время выполнения обработчика запроса.

```rust,noplayground
use std::{pin::Pin, task::{Context, Poll}};
use axum::{Router, extract::Request, response::Response, routing::get};
use tokio::time::Instant;
use tower::{Layer, Service};

// Мидлваре, который замеряет время выполнения запроса.
#[derive(Clone)]
struct ExecTimeLogService<S> {
    // Следующий мидлваре/обработчик запроса
    next_handler: S,
}

impl<S> Service<Request> for ExecTimeLogService<S>
where
    S: Service<Request, Response = Response> + Send + 'static,
    S::Future: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = Pin<Box<dyn Future<Output=Result<Self::Response,Self::Error>>+Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.next_handler.poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> Self::Future {
        let start = Instant::now();
        let future = self.next_handler.call(request);
        Box::pin(async move {
            let response = future.await?;
            tracing::info!("Request took: {} micros", start.elapsed().as_micros());
            Ok(response)
        })
    }
}

// Реализация Layer, которая отвечает за встраивание нашего мидлваре
// в цепочку обработки запроса
#[derive(Clone)]
struct ExecTimeLogLayer;

impl<S> Layer<S> for ExecTimeLogLayer {
    type Service = ExecTimeLogService<S>;

    // Axum вызывает этот метод, чтобы получить объект мидлваре для дальнейшего
    // встраивания его в цепочку обработки запроса.
    // inner - это следующий в цепочке мидлваре или обработчик запроса.
    fn layer(&self, inner: S) -> Self::Service {
        ExecTimeLogService { next_handler: inner }
    }
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();
    let app = Router::new()
        .route("/hello", get(hello))
        .layer(ExecTimeLogLayer);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

Запустим наше приложение и перейдём на [http://localhost:8080/hello](http://localhost:8080/hello)

```
$ cargo run

   Compiling test_axum v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 2.00s
     Running `target/debug/test_axum`
2025-12-05T17:32:48.512545Z  INFO test_axum: Request took: 82 micros
```

Как и в случае с обработчиками запросов, создание мидлваре путём реализации трэйта `Service` даёт больше гибкости, так как позволяет объекту мидлваре иметь состояние.

## Стандартные сервисы

В прошлой главе мы сказали, что для Axum существует ряд стандартных мидлваре (и сервисов), основанных на инфраструктуре Tower. Теперь мы готовы с ними познакомиться.

### сервис ServeFile

Сервис [ServeFile](https://docs.rs/tower-http/latest/tower_http/services/struct.ServeFile.html) позволяет отдавать файл в качестве ответа на запрос.

Для примера сделаем эндпоинт, который возвращает HTML файл.

Сначала создадим в корне проекта файл `index.html`:

```html
<html>
    <body>
        <h1>Hello</h1>
    </body>
</html>
```

Теперь `src/main.rs`:

```rust,noplayground
use axum::Router;
use tower_http::services::ServeFile;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route_service("/index", ServeFile::new("index.html"));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

После запуска сервера при переходе на [http://localhost:8080/index](http://localhost:8080/index) мы должны увидеть нашу страницу из `index.html`.

### Мидлваре CorsLayer

Если мы пишем сервер с таким API, который предполагается вызывать из браузера, то мы неизбежно столкнёмся с CORS (Cross-Origin Resource Sharing) проблемой: если домен, с которого загружается сам сайт, отличается от домена, на котором располагается API сервер, то браузер в целях безопасности не позволит делать вызовы.

Для того чтобы браузер разрешил странице, загруженной с домена X, выполнять запросы на API, которое находится на домене Y, необходимо, чтобы API сервер Y разрешил вызывать себя со страниц, загруженных с домена X. Конечно, мы можем сами написать мидлваре, который будет проставлять необходимые `Access-Control-Allow-Origin` заголовки, но в этом нет нужды, так как  существует готовый мидлваре, делающий тоже самое — [CorsLayer](https://docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html).

В примере ниже мы разрешаем Cross-origin вызовы наших эндпоинтов, если вызовы осуществляются из страниц, загруженных с http://mydomain.com или http://api.mydomain.com.

```rust,noplayground
use axum::{Router, routing::get};
use tower_http::cors::CorsLayer;

#[tokio::main]
async fn main() {
    let allowed_origins = [
        "http://mydomain.com".parse().unwrap(),
        "http://api.mydomain.com".parse().unwrap(),
    ];
    let app = Router::new()
        .route_service("/api/hello", get(hello))
        .layer(CorsLayer::new().allow_origin(allowed_origins));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

## Другие сервисы

К этому моменту вам должно быть понятно, как использовать сервисы и мидлваре, основанные на tower. Напоследок перечислим некоторые наиболее часто используемые из них.

* сервис [ServeDir](https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html) — подобен вышерассмотренному `ServeFile`, но возвращает не конкретный файл, а запрошенные файлы из указанной директории.
* сервис [Redirect](https://docs.rs/tower-http/latest/tower_http/services/struct.Redirect.html) — позволяет переадресовать запрос на другой URL
* мидлваре [CompressionLayer](https://docs.rs/tower-http/latest/tower_http/compression/struct.CompressionLayer.html) — сжимает ответ на запрос при помощи указанного кодека (gzip, deflate, zstd и т.д.)
* мидлваре [RequestDecompressionLayer](https://docs.rs/tower-http/latest/tower_http/decompression/struct.RequestDecompressionLayer.html) — автоматически декодирует тело запроса, сжатое кодеком
* мидлваре [NormalizePathLayer](https://docs.rs/tower-http/latest/tower_http/normalize_path/struct.NormalizePathLayer.html) — убирает ненужные знаки `/` в конце URL пути
* мидлваре [RequestBodyLimitLayer](https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html) — отвергает с 413-м кодом те запросы, чьё тело превышает заданный размер
* мидлваре [TimeoutLayer](https://docs.rs/tower-http/latest/tower_http/timeout/struct.TimeoutLayer.html) — позволяет установить максимальное время выполнения эндпоинта, после истечения которого будет возвращён указанный HTTP код
* мидлваре [TraceLayer](https://docs.rs/tower-http/latest/tower_http/trace/struct.TraceLayer.html) — для всех запросов логирует момент получения запроса, и время его выполнения
* мидлваре [InFlightRequestsLayer](https://docs.rs/tower-http/latest/tower_http/metrics/struct.InFlightRequestsLayer.html) — считает метрику "количество запросов, находящихся в обработке"
