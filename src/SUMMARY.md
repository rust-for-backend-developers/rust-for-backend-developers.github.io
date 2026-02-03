# Summary

* [Предисловие](preface.md)

# Установка

* [Установка Rust](setup/setup-rust.md)
* [Установка на Windows](setup/install-on-windows.md)
* [Установка на Linux](setup/install-on-linux.md)
* [Среда разработки](setup/ide.md)

# Основы языка

* [Первый взгляд](rust-basics/first-look.md)
* [Безопасный Rust](rust-basics/safe-rust.md)
* [Переменные](rust-basics/variables.md)
* [Примитивные типы данных](rust-basics/primitive-types.md)
* [Печать на консоль](rust-basics/console-output.md)
* [Скоупы](rust-basics/scopes.md)
* [Ссылки](rust-basics/references.md)
* [Массивы](rust-basics/arrays.md)
* [Вектор](rust-basics/vector.md)
* [Слайсы](rust-basics/slices.md)
* [Строки](rust-basics/strings.md)
* [Условный оператор if](rust-basics/operator-if.md)
* [Циклы](rust-basics/loops.md)
* [Функции](rust-basics/functions.md)
* [Кортежи](rust-basics/tuples.md)
* [Владение](rust-basics/ownership.md)
* [Лайфтаймы](rust-basics/lifetimes.md)
* [Декларативные макросы](rust-basics/declarative-macro.md)
* [Указатели](rust-basics/pointers.md)
* [Структуры](rust-basics/structs.md)
* [Модули](rust-basics/modules.md)
* [Трэйты](rust-basics/traits.md)
* [Авто-реализация трэйтов](rust-basics/auto-derive.md)
* [Деструктурирование](rust-basics/destructuring-assignment.md)
* [Паттерн матчинг](rust-basics/pattern-matching.md)
* [Анонимные функции](rust-basics/anonymous-functions.md)
* [Генерики](rust-basics/generics.md)
* [Перечисления](rust-basics/enums.md)
* [Option](rust-basics/option.md)
* [Result](rust-basics/result.md)
* [Итераторы](rust-basics/iterators.md)
* [Умные указатели](rust-basics/smart-pointers.md)

# Rust проект

* [Cargo](project/cargo.md)
* [Зависимости](project/dependencies.md)
* [Создание библиотеки](project/making-lib.md)
* [Несколько исполняемых файлов](project/multiple-bin.md)
* [Пакет, крэйт, модуль](project/package-crate-module.md)
* [Workspace проект](project/workspace-project.md)
* [Тестирование](project/testing.md)

# Погружаемся в Rust

* [Основные трейты](dive-deeper/common-traits.md)
* [Any](dive-deeper/any.md)
* [Коллекции](dive-deeper/collections.md)
* [Ввод/вывод](dive-deeper/io.md)
* [Файловая система](dive-deeper/file-system.md)
* [Newtype паттерн](dive-deeper/newtype-pattern.md)
* [Паника](dive-deeper/panic.md)
* [Многопоточность](dive-deeper/multithreading.md)
* [Глобальные данные](dive-deeper/global-data.md)
* [Обработка ошибок](dive-deeper/error-handling.md)
* [Сериализация](dive-deeper/serialization.md)
* [Дата и время](dive-deeper/date-time.md)
* [Логирование](dive-deeper/logging.md)
* [Конфигурация приложения](dive-deeper/config.md)

# Асинхронность

* [Потоки и ввод/вывод](async/threads-and-io.md)
* [Асинхронность в Rust](async/async-in-rust.md)
* [Tokio](async/tokio.md)
* [async\_trait](async/async_trait.md)

# Бекенд на Rust

* [Обзор экосистемы](web/backend-ecosystem.md)
* [Axum - основы](web/axum-basics.md)
* [Состояние бекенда](web/backend-state.md)
* [Роутинг](web/routing.md)
* [Экстракторы](web/extractors.md)
* [Мидлваре](web/middleware.md)
* [Tower](web/tower.md)
* [Завершение работы сервера](web/graceful_shutdown.md)
* [Тестирование эндпоинтов](web/endpoints-testing.md)
* [SQLx](web/sqlx.md)
* [Версионирование структуры БД](web/db-migrate.md)
* [Тест контейнеры](web/testcontainers.md)
* [SQLx макросы](web/sqlx-macros.md)
* [Axum и SQLx](web/axum-and-sqlx.md)
* [Структура бекенда](web/backend-structure.md)
* [Документирование API](web/api-doc.md)
* [Prometheus метрики](web/prometheus-metrics.md)
* [Кросс сборка](web/cross-compilation.md)
