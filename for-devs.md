# 👨‍💻 Быстрый старт для разработчиков

Найди задачу — перейди к нужному разделу.

---

## 🚀 Онбординг

| Задача | Куда смотреть |
|--------|---------------|
| Понять как устроена платформа | [Архитектура](architecture.md) |
| Поднять окружение | [Деплой](deployment.md) |
| Разобраться в потоке платежа | [Flow](flow.md) + [Архитектура](architecture.md#потоки-платежей) |

## 💳 Платежи и API

| Задача | Куда смотреть |
|--------|---------------|
| API мерчанта (создание инвойса, статус) | [Агрегатор API](aggregator.md) |
| Статусы, state machine | [Статусы транзакций](transaction-statuses.md) |
| Callback мерчанту — формат, подпись | [Коллбэк](transaction-statuses.md#коллбэк-мерчанту) |
| Платёжная страница (Flow) | [Flow](flow.md) |

## 🔌 Шлюзы и провайдеры

| Задача | Куда смотреть |
|--------|---------------|
| Написать новый шлюз (PHP) | [Агрегатор → Gateway dev](aggregator.md) |
| Список существующих шлюзов | [gateways.csv](gateways.csv) |
| Конструктор шлюзов (без кода) | [Gateway Builder](dev-in-progress/gateway-builder.md) |

## 💱 Курсы валют

| Задача | Куда смотреть |
|--------|---------------|
| Rate API | [Rate Service](rate.md) |
| Эндпоинты `/rate/relevant`, `PATCH /rate` | [Rate API](rate.md#api) |

## 🛠 Сервисы

| Задача | Куда смотреть |
|--------|---------------|
| Support Service API (диспуты) | [Support Service](support-service.md) |
| Frontend (новая админка) | [Frontend](frontend.md) |
| Трейдерская платформа | [InternalTrade](trade.md) |
| Агент кабинет API | [Agent Cabinet API](dev-in-progress/agent-cabinet-api.md) |

## 📐 В разработке

| Задача | Куда смотреть |
|--------|---------------|
| Кабинет мерчанта | [Merchant Cabinet](dev-in-progress/merchant-cabinet.md) |
| Gateway Builder | [Gateway Builder](dev-in-progress/gateway-builder.md) |

---

!!! tip "Термины непонятны?"
    Смотри [Глоссарий](ai/glossary.md) — там все PSP-термины с объяснением.

!!! warning "Устаревшая дока"
    Некоторые разделы могут быть неактуальны. Актуальную информацию уточняй у Рашида или смотри код.
