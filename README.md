# Платформа анализа кэш-эффективности C-кода

Микросервисная система для статического и динамического анализа cache-friendly паттернов
в исходниках на C: загрузка файла → статический разбор паттернов → симуляция кэша
(L1+L2) → агрегированные метрики и admin-панель.

- **Конфиг симулятора кэша**: только **`.json`** (валидный JSON); до **10** файлов на обычного пользователя (`GET` / `POST` / `DELETE /api/v1/analysis/cache-configs`), выбор обязателен перед загрузкой/анализом; путь пробрасывается в событии Kafka `start_cache`.

## Запуск

```bash
cd infra
cp .env.example .env
make up
```

После сборки доступны:

| URL                                     | Сервис                            |
|-----------------------------------------|-----------------------------------|
| http://localhost:8080/                  | Веб-приложение (Frontend)         |
| http://localhost:8080/health            | Health gateway (nginx)            |
| http://localhost:19001/                 | MinIO Console                     |
| http://localhost:18123/                 | ClickHouse HTTP                   |
| http://localhost:15432/                 | PostgreSQL                        |
| http://localhost:19092/                 | Kafka (host listener)             |

Учётка администратора по умолчанию: `admin@system.local` / `admin`.

## Состав

| Сервис                     | Технологии                                  | Назначение                                                                |
|----------------------------|---------------------------------------------|---------------------------------------------------------------------------|
| `core-api`         | Go 1.22, Gin, PostgreSQL                    | Регистрация / логин / JWT, проекты, admin                                 |
| `analysis-api`     | Go 1.24, Gin, Postgres, MinIO, Kafka, CH    | Оркестрация пайплайна анализа, агрегация метрик, admin-эндпоинты          |
| `static-analysis-worker`   | Go 1.22 + wine + `cmd.exe`                  | Статический анализ `.c` → паттерны → ClickHouse `static_patterns`         |
| `cache-analysis-worker` | Go 1.22 + wine + `CacheSim.exe`             | Кэш-симуляция `.c` → JSON-артефакт с L1/L2/Arrays                         |
| `frontend`         | Vue 3 + Vite + Pinia + Tailwind v4 + Monaco | UI пользователя/админа на русском с переключателем тёмной/светлой темы    |
| `infra`            | Docker Compose + Nginx                      | Точка входа и единый запуск всего стека                                   |
| `docs`              | VitePress                                   | Документация архитектуры и контрактов                                     |

## Учётные данные по умолчанию

| Кому             | Логин                       | Пароль                |
|------------------|-----------------------------|-----------------------|
| Администратор    | `admin@system.local`        | `admin`               |
| Postgres         | `diplom`                    | `diplom_secret`       |
| MinIO            | `minioadmin`                | `minioadmin123`       |
| ClickHouse       | `default`                   | `clickhouse_secret`   |
| Redis            | —                           | `redis_secret`        |

Все значения переопределяются через `infra/.env`.

## Команды Makefile

| Команда         | Действие                                                          |
|-----------------|-------------------------------------------------------------------|
| `make up`       | Поднять весь стек (init-БД + сервисы) с автосборкой               |
| `make down`     | Остановить контейнеры (тома сохранить)                            |
| `make restart`  | Пересоздать сервисы без потери данных                             |
| `make rebuild`  | Пересборка без кэша                                               |
| `make ps`       | Статус контейнеров                                                |
| `make logs`     | Следить за логами                                                 |
| `make clean`    | down + удалить тома (всё стирается)                               |
| `make status`   | Проверить health                                                  |

## Production (Yandex Cloud + Kubernetes)

Инфраструктура: **[infra/MANUAL.md](infra/MANUAL.md)** (полный мануал), **[infra/DEPLOY-K8S.md](infra/DEPLOY-K8S.md)** (команды).

## Документация

```bash
cd docs
docker compose up -d docs   # http://localhost:8088
```

Локально без Docker: `npm install && npm run docs:dev`.

## Тестовый прогон

1. Откройте http://localhost:8080/, войдите как `admin@system.local` / `admin`.
2. Создайте проект на странице «Проекты».
3. Загрузите файл `infra/samples/loop.c` и дождитесь статуса «Готово».
4. Метрики появятся по клику на «Метрики».

## Замечание по платформам

Воркеры запускаются в контейнерах `linux/amd64` (Windows x86_64 PE-бинарники
`cmd.exe` и `CacheSim.exe` работают через `wine`). На обычных linux/amd64-хостах
система работает из коробки. На Apple Silicon контейнер запускается через
эмуляцию Docker Desktop / orbstack, и совместимость `wine` зависит от настроек
Rosetta — возможны ошибки уровня страничной памяти при запуске бинарников.
