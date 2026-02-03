# Конфигурация приложения

В этой главе мы рассмотрим, как при помощи библиотеки [config](https://crates.io/crates/config) считывать конфигурации приложения из текстовых файлов.

> [!NOTE]
> В интернете вы можете встретить множество примеров конфигурации при помощи библиотеки [dotenv](https://crates.io/crates/dotenv), которая когда-то была популярна, но уже давно заброшена. У этой библиотеки был немного более свежий форк [dotenvy](https://crates.io/crates/dotenvy), однако он также заброшен.

## Чтение конфиг файла

Для начала добавим крэйт _config_ в `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
config = "0.15"
serde = { version = "1", features = ["derive"] }
```

Предположим, мы создаём бэкенд приложение, которое работает с реляционной базой данных. В конфигурационный файл мы вынесем:

* настройки подключения к БД
* порт, который будет слушать наше бэкенд приложение

В качестве формата для конфигурационного файла мы возьмём TOML, который является наиболее популярным в мире Rust. Однако библиотека _config_ также поддерживает JSON, Yaml, INI, RON и Corn.

Итак, создадим файл `config/application.toml`, в который поместим конфигурации:

```toml
[db]
host = "localhost"
port = 5432
login = "postgres"
password = "1111"

[server]
listen_port = 3000
```

Теперь напишем программу, которая считывает этот конфигурационный файл в виде объекта структуры.

```rust
use config::{Config, File};
use serde::Deserialize;

// Корневой тип для конфигурации
#[derive(Debug, Deserialize, Clone)]
struct AppConfig {
    db: DbConfig,
    server: ServerConfig,
}

// Тип для полей из секции [db]
#[derive(Debug, Deserialize, Clone)]
struct  DbConfig {
    host: String,
    port: u32,
    login: String,
    password: String,
}

// Тип для полей из секции [server]
#[derive(Debug, Deserialize, Clone)]
struct ServerConfig {
    listen_port: u32,
}

fn main() {
    // Считываем конфиг файл
    let cfg  = Config::builder()
        .add_source(File::with_name("config/application.toml"))
        .build()
        .unwrap();

    // Конвертируем конфигурацию в объект структуры
    let app_config: AppConfig = cfg.try_deserialize().unwrap();

    println!("{app_config:?}");
}
```

Вывод программы:

```
AppConfig {
  db: DbConfig { host: "localhost", port: 5432, login: "postgres", password: "1111" },
  server: ServerConfig { listen_port: 3000 }
}
```

Как видите, для создания объекта структуры из конфигурации библиотека _config_ опирается на крэйт _serde_, поэтому все возможности serde также доступны и для десериализации конфига.

## Многослойный конфиг

Если вы разрабатываете бэкенд приложение, то, скорее всего, у вас будет несколько конфигураций для запуска на разных окружениях (environment), таких как:

* локальный компьютер разработчика
* продакшен
* пре-продакшен
* и т.д.

При этом какая-то часть конфигурации у вас будет одинаковая для всех окружений, а какая-то часть будет специфична для каждого окружения. Например, слушать порт 8080 сервер будет на всех окружениях, а настройки подключения к БД везде будут разными.

Для таких ситуаций удобно разбивать конфигурацию на минимум два слоя:

* Первый слой представлен одним файлом (назовём его `default.toml`) и содержит значения параметров по умолчанию.
* Второй — специфичен для окружения (`prod.toml`, `stg.toml` и т.д.) и хранит только значения, которые отличаются от тех, что заданы в `default.toml`.

Расширим наш предыдущий пример. Вместо одного файла `application.toml` у нас будет

* файл с конфигурациями по умолчанию — `default.toml`
* конфигурация для окружения разработчика — `dev.toml`
* конфигурация для продакшена — `prod.toml`.

Файл `config/default.toml`:

```toml
[db]
host = "localhost"
port = 5432
login = "postgres"
password = "1111"

[server]
listen_port = 3000
```

Файл `config/dev.toml`:

```toml
[db]
host = "dev-db.my.com"
port = 5432
login = "postgres"
password = "1111"
```

Файл `config/prod.toml`:

```toml
[db]
host = "prod-db.my.com"
port = 5432
login = "prod_user"
password = "prod_passwd"

[server]
listen_port = 8080
```

Теперь `main.rs`. В нашем примере мы рассчитываем, что название окружения будет передаваться через переменную окружения `PROFILE`.

```rust
use config::{Config, File};
use serde::Deserialize;

#[derive(Debug, Deserialize, Clone)]
struct AppConfig { db: DbConfig, server: ServerConfig }

#[derive(Debug, Deserialize, Clone)]
struct  DbConfig { host: String, port: u32, login: String, password: String }

#[derive(Debug, Deserialize, Clone)]
struct ServerConfig { listen_port: u32 }

fn main() {
    // Получаем имя окружения
    let profile = std::env::var("PROFILE").unwrap_or_else(|_| "dev".into());

    // Загружаем сначала значения из default.toml, а потом из файла соответствующего
    // окружению. При этом значения с совпадающими именами будут перезатираться.
    let cfg  = Config::builder()
        .add_source(File::with_name("config/default.toml"))
        .add_source(File::with_name(&format!("config/{profile}.toml")))
        .build()
        .unwrap();

    let app_config: AppConfig = cfg.try_deserialize().unwrap();

    println!("{app_config:?}");
}
```

Вывод программы:

```
$ export PROFILE=dev
$ cargo run
AppConfig {
  db: DbConfig { host: "dev-db.my.com", port: 5432, login: "postgres", password: "1111" },
  server: ServerConfig { listen_port: 3000 }
}
```

## Перезапись переменными окружения

Иногда удобно иметь возможность перезаписать значение какого-то параметра без изменения конфигурационного файла, а при помощи переменной окружения.

В примере ниже мы добавляем возможность перезаписывать значения параметров конфигурации при помощи переменных окружения с префиксом "APP__".

```rust
fn main() {
    let profile = std::env::var("PROFILE").unwrap_or_else(|_| "dev".into());

    let cfg  = Config::builder()
        .add_source(File::with_name("config/default.toml"))
        .add_source(File::with_name(&format!("config/{profile}.toml")))
        .add_source( // перезапись
             config::Environment::with_prefix("app").separator("__")
         )
        .build()
        .unwrap();

    let app_config: AppConfig = cfg.try_deserialize().unwrap();

    println!("{app_config:?}");
}
```

Например, если мы хотим перезаписать значение параметра `host` из секции `[db]`, то мы сможем сделать это при помощи переменной окружения с именем `APP__db__host`.

Запуск программы:

```
$ export APP__db__host=XXX
$ cargo run
AppConfig {
  db: DbConfig { host: "XXX", port: 5432, login: "postgres", password: "1111" },
  server: ServerConfig { listen_port: 3000 }
}
```

