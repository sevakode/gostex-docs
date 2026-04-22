# AGGREpay Aggregator

> **Стек:** Laravel 12 / PHP 8.4 + PostgreSQL + Redis
> **Репозиторий:** `aggregator/`
> **Последнее обновление:** 2026-04-15 (автосинк)

---

## 1. Обзор

AGGREpay Aggregator — ядро платёжной платформы. Принимает запросы от мерчантов, маршрутизирует через платёжных провайдеров (шлюзы), управляет каскадами, балансами, трейдерами, диспутами.

**Ключевые цифры:**
- Большое количество платёжных шлюзов в `app/Gateway/` (список в `docs/gateways.csv`)
- Поддерживает payin, payout, withdrawal, conversion
- Многоарендная архитектура (WL = отдельный инстанс)

---

## 2. Архитектура

```
Мерчант API → Aggregator → Каскад провайдеров → Платёжный шлюз
                  │
                  ├── Trade (реквизиты трейдеров)
                  ├── Flow / gostex-pay (платёжная страница)
                  ├── Support Service (диспуты)
                  └── Rate Service (курсы валют)
```

---

## 2.5. Политика интеграции новых провайдеров (шлюзов)

Все новые шлюзы разрабатываются и интегрируются нашей командой. Код от внешних команд не принимается.

**Условия:**
- 1 шлюз в месяц — бесплатно
- Дополнительные шлюзы — платно по тарифной лесенке:

| Уровень | Кол-во интеграций | Цена за шлюз |
|---------|-------------------|--------------|
| Базовый | 1–2 | $420 |
| Стартовый | 3–4 | $370 |
| Партнёрский | 5–6 | $320 |
| Продвинутый | 7–9 | $270 |
| Профессиональный | 10+ | $230 |
| Пакет 10 шт. | единовременно | $1800 |

Ставка применяется ко всем последующим интеграциям после достижения уровня.

**Исключения:** White-labels Протокол, Агрипей и Ковенант — интеграции в приоритетном порядке.

**Запросы на интеграцию** направлять через менеджера.

### Компоненты

| Директория | Назначение |
|------------|------------|
| `app/Gateway/` | 213+ интеграций с провайдерами |
| `app/Gateway/Universal/` | Gateway Builder — JSON-конфигурируемые шлюзы |
| `app/ReverseGateway/` | Reverse интеграции (входящие от провайдеров) |
| `app/Models/` | Payment, Invoice, Merchant, Provider, Method, CascadeAttempt и др. |
| `app/Services/` | Gateway, SignatureService, ProfitCalculator и др. |
| `app/Jobs/` | Фоновые задачи (каскад, уведомления и др.) |
| `app/Console/` | Artisan-команды и cron |

---

## 3. API

### 3.1 Merchant API (`/api/v2/`)

Публичный API для мерчантов. Аутентификация: `api_key` + `signature` (HMAC).

| Endpoint | Method | Описание |
|----------|--------|----------|
| `/api/v2/payment/create` | POST | Создать payin |
| `/api/v2/payout/create` | POST | Создать payout |
| `/api/v2/payment/status` | GET | Статус платежа по `order_id` |
| `/api/v2/orders/{id}` | GET | Данные заказа (для Flow) |
| `/api/v2/orders/{id}/methods` | GET | Доступные методы |
| `/api/v2/orders/{id}/banks` | GET | Список банков |
| `/api/v2/orders/{id}/payments` | POST | Создать инвойс (выбор метода) |
| `/api/v2/orders/{id}/status` | PATCH | Обновить статус (чек, OTP) |
| `POST /api/callback/{method_code}` | POST | Callback от провайдера |

### 3.2 Admin API (Trade Token)

Внутренний API для Admin Panel и Trade. Защищён `VerifyTradeToken` middleware.

**Merchants:**
- CRUD мерчантов, переключение, управление методами
- `POST /api/merchants/create` — создание мерчанта (генерирует api_key, secret_key, credentials)
- `POST /api/merchants/{id}/switch` — включить/выключить мерчанта

**Providers / Gateways:**
- `GET/POST /api/gateways/` — список, создание провайдеров

> **Примечание (2026-04-15):** IP whitelist middleware (`VerifyGatewayWhitelistIp`) на `/callback/*` маршрутах временно отключён — все callback-запросы принимаются без IP-фильтрации.
- `GET /api/gateways/{gateway}/balance` — баланс провайдера
- `POST /api/gateways/balances` — балансы всех провайдеров
- `POST /api/gateways/broadcast` — рассылка сообщений провайдерам

**Methods:**
- CRUD методов, типов методов, whitelist
- `GET /api/methods`, `POST /api/methods/offers`

**Counterparties:**
- `GET /api/counterparties/` — список, статистика
- `PATCH /api/counterparties/{id}` — обновление
- `POST /api/counterparties/{id}/block|unblock|unbind|rebind`

**Settings:**
- `GET/PUT /api/admin/settings` — настройки приложения
- `POST /api/admin/settings/telegram/webhook` — установка Telegram webhook
- `POST /api/admin/clear-banks-cache` — сброс кэша банков

**Transactions:**
- `GET /api/merchants/transactions` — транзакции
- `POST /api/merchants/export` — экспорт в Excel/CSV

### 3.3 Health Check

| Endpoint | Описание |
|----------|----------|
| `GET /api/health` | Общий статус |
| `GET /api/health/liveness` | Liveness probe (k8s) |
| `GET /api/health/readiness` | Readiness probe |
| `GET /api/health/jobs` | Статус фоновых задач |

---

## 4. Merchant Panel API (`/merchants/`)

Кабинет мерчанта (legacy web-routes). Аутентификация: `merchant` middleware.

| Endpoint | Описание |
|----------|----------|
| `GET /` | Dashboard |
| `GET /components/info` | Данные мерчанта |
| `GET /components/balance` | Баланс |
| `GET /components/methods` | Доступные методы |
| `POST /components/payout` | Создать payout |
| `POST /components/withdrawal` | Вывод средств |
| `POST /components/convert` | Конвертация |
| `POST /components/checkout` | Checkout |
| `POST /merchants/manual-payin` | Ручной payin |
| `POST /components/resend-callbacks` | Повторная отправка callbacks |
| `POST /components/check-status` | Проверка статуса |
| `POST /components/cancel-dispute` | Отмена диспута |

---

## 5. Жизненный цикл платежа (PayIn)

### Отмена выплаты (Payout Cancellation) — исправление двойной отмены (hotfix/v1.23.69, INB-518)

Исправлено поведение при отмене выплаты через API мерчанта:

**Проблема:** При вызове ручки отмены выплаты агрегатор отменял выплату на своей стороне, даже если:
- у мерчанта установлен флаг «не отменять и не отправлять callback»;
- провайдер вернул неуспешный ответ на отмену.

**Исправление в `PaymentController`:**
- Запрос к провайдеру на отмену вынесен с пометкой `TODO: вынести из транзакционности` (внешний HTTP-запрос не должен выполняться внутри транзакции БД).
- Логика отмены теперь учитывает настройки мерчанта и статус ответа провайдера перед изменением статуса выплаты.

> Jira: [INB-518] — двойной payout при неуспешной отмене у провайдера.  
> Коммит: `hotfix/v1.23.69`.

1. Мерчант → `POST /api/v2/payment/create`
2. Aggregator валидирует запрос и подпись
3. Каскад: выбор провайдера (приоритет → cascade_percent → лимиты → доступность)
4. Создаётся Invoice, возвращается `url` для Flow
5. Плательщик видит форму оплаты, оплачивает
6. Провайдер → `POST /api/callback/{method_code}` → Aggregator обновляет статус
7. Aggregator → callback на `callback_uri` мерчанта

### Каскадинг

При неудаче провайдера `CreateCascadeAttemptJob` автоматически переключается на следующего. Хранится в `CascadeAttempt`.

---

## 6. Gateway Builder

Система создания шлюзов через JSON-конфигурацию без PHP-кода.

```
app/Gateway/Universal/
├── UniversalGateway.php      — реализация
├── TemplateEngine.php        — подстановка {{переменных}}
├── SignatureGenerator.php    — 13 алгоритмов подписи
└── ResponseMapper.php        — маппинг ответов (dot notation)
```

**Возможности:**
- Payin и payout без деплоя кода
- 13 алгоритмов подписи (HMAC, SHA, RSA, JWT, MD5, Base64 и др.)
- Multi-step запросы (OAuth2, токен + создание)
- Импорт/экспорт шаблонов (JSON)
- Встроенный API-тестер
- AI-ассистент для генерации конфигов
- 6 пресетов

**Admin API:**
- `GET/POST/PATCH/DELETE /api/gateway-templates/` — CRUD шаблонов
- `POST /api/gateway-tester/` — тестирование (create_invoice, create_payout, get_status, callback)
- `POST /api/ai-assistant/` — AI-чат (streaming SSE)

> **Статус:** В разработке (ветка dev)

---

## 7. Cron-задачи

| Команда | Расписание | Описание |
|---------|-----------|----------|
| `CheckPaymentCron` | каждые 5 мин | Проверка зависших платежей |
| `ExpiredPaymentCron` | каждую минуту | Отмена просроченных payins |
| `ExpiredPayoutCron` | каждую минуту | Отмена просроченных payouts |
| `AutoWithdrawalCommand` | каждую минуту | Автовывод средств |
| `UpdateCurrencyRatesCommand` | каждую минуту | Обновление курсов валют |
| *(ещё 3 daily команды)* | ежедневно | — |

---

## 8. Аутентификация

### Merchant API
- `api_key` — публичный ключ мерчанта
- `signature` — HMAC-подпись тела запроса с `secret_key`
- Проверяется через `VerifySignature` middleware
- IP-whitelist: `VerifyMerchantWhiteListIp` middleware

### Admin / Trade API
- `VerifyTradeToken` middleware — Bearer-токен из Trade

---

## 9. Тесты

```
tests/
├── Feature/        — API endpoints
├── Integration/    — взаимодействие с замоканными API
├── Real/           — тесты с реальными API провайдеров
├── Unit/           — юнит-тесты сервисов
└── Stubs/          — моки и заглушки
```

Запуск: `php artisan test` или `./vendor/bin/phpunit`

---

## 10. Провайдеры — последние обновления

### GostexEcom / GostexEcomGatewable (добавлен 2026-04-15)

Новый шлюз для **quasi-ecom** платежей (двухэтапный flow с карточными данными).

**Flow:**
1. `POST /api/v2/payments` — создаёт payment на удалённом агрегаторе (без method)
2. `POST /api/v2/payments/{orderId}/payments` — отправляет карточные данные + method, получает инвойс

Карточные данные (`number`, `cvv`, `expiry`, `holder`) передаются мерчантом через `optionalParams['card']` → `injectParams`.

**Конфигурация метода:**
| Параметр | Обязательный | Описание |
|----------|-------------|----------|
| `currency` | ✅ | Валюта (напр. `RUB`) |
| `methodType` | ✅ | Тип метода |
| `senderBank` | — | Банк отправителя |
| `assetOrBank` | — | Актив или банк |

**Конфигурация шлюза:** `dropdown` (sandbox/live), `baseUrl`, `token`, `secretKey`, `merchantId`.

### FinViaGatewableV2 (добавлен 2026-04-15)

Новая версия шлюза FinVia (`app/Gateway/FinVia/FinViaGatewableV2.php`). Реализует `Gatewable`, `GatewayBalance`, `StatusChangable`, `BreakerRules`.

**Конфигурация:** `baseUrl`, `apiKey`, `secretKey`. Метод: опциональный `methodCode`.

---

## 11. Настройки приложения

Ключевые параметры в `app_settings` (через Admin API):

| Параметр | Описание |
|----------|----------|
| `app.sup_service` | Включить Support Service |
| `app.support_service_url` | URL Support Service |
| `app.support_service_token` | Токен аутентификации Support Service |
| `app.flow_url` | URL Flow (платёжная страница) |
| `app.trade_url` | URL Trade (реквизиты) |
| `app.rate_service_url` | URL Rate Service |

---

## 12. Callback маршруты

| Маршрут | Методы | Описание |
|---------|--------|----------|
| `/callback/check/upload` | POST | Загрузка чека |
| `/callback/check/delete` | POST | Удаление чека |
| `/callback/{method_code}` | GET, POST | Callback от провайдера |
| `/callback/{method_code}/{transaction_id}` | GET, POST | Callback с ID транзакции |

> **Изменение 2026-04-15:** Middleware `VerifyGatewayWhitelistIp` закомментирован на `/callback/` группе. Маршруты открыты для всех IP.

### EuphoriaV2Gatewable
Добавлен новый шлюз **EuphoriaV2Gatewable** в агрегатор GostexPay.

- **Тип:** Gateway (Gatewable)
- **Версия:** V2
- **Статус:** Активен
- **Добавлен:** Sofia Dieva

Шлюз `EuphoriaV2Gatewable` является второй версией интеграции Euphoria и подключается через стандартный интерфейс агрегатора. Поддерживает все базовые операции, определённые в протоколе Gatewable: инициализация платежа, подтверждение транзакции, возврат средств.

**Пример подключения:**
```python
from aggregator.gateways import EuphoriaV2Gatewable

gateway = EuphoriaV2Gatewable(config={
    "api_key": "<YOUR_API_KEY>",
    "endpoint": "https://api.euphoria.example.com/v2"
})
aggregator.register_gateway(gateway)
```

> **Примечание:** При миграции с EuphoriaV1 убедитесь, что конфигурационные параметры обновлены в соответствии со спецификацией V2.

### Исправление агрегатора (aggregator fix)
> **Обновление:** Внесено исправление в модуль агрегатора.

- **Автор:** ani
- **Источник:** GitLab
- **Приоритет:** Important

В модуль агрегатора было внесено исправление (fix). Рекомендуется проверить актуальность интеграционных сценариев, связанных с агрегатором, и убедиться, что поведение системы соответствует ожидаемому. Подробности изменения уточняйте у ответственного разработчика (ani) или в истории коммитов репозитория.

---

## 13. Провайдеры — обновления 2026-04-16

### NimboGatewable (интегрирован 2026-04-16)

Добавлен полный шлюз **Nimbo** (`app/Gateway/Nimbo/NimboGatewable.php`, 620 строк).

- **Тип:** Gateway (Gatewable)
- **Автор:** Hayko
- **Статус:** Активен
- **Особенности:** Полная реализация, крупнейший из добавленных сегодня шлюзов

### WellbitGatewable (добавлен 2026-04-16)

Добавлен шлюз **Wellbit** через стандартный интерфейс Gatewable.

- **Тип:** Gateway (Gatewable)
- **Статус:** Активен
- Задача Jira: [INB-627] Залить шлюз Wellbit

### FincoreGatewable + KredosGatewable (добавлены 2026-04-16)

Добавлены два новых шлюза с ASGatewable-обёртками:

- **FincoreGatewable** — Fincore payment gateway
- **KredosGatewable** / **KredosASGatewable** — Kredos integration
- **Тип:** Gateway + ASGatewable wrapper
- **Статус:** Активны

### InfinityV2Gatewable (добавлен 2026-04-16)

Вторая версия интеграции Infinity (`app/Gateway/Infinity/InfinityV2Gatewable.php`).

- Добавлен `client_id` в запросы
- Добавлены required fields для корректной инициализации платежей
- Расширяет `InfinityGatewable`

### Обновления существующих шлюзов (2026-04-16)

| Шлюз | Изменение |
|------|-----------|
| **Ikra / IkraGatewable** | Добавлена поддержка `crossBorderRequisiteType` и `crossBorderCurrency` |
| **Finaxis** | Добавлено поле `utr` (UTR — Unique Transaction Reference) |
| **GostexEcom** | Обновлён endpoint API для payment requests |
| **KZT rate** | Исправлен курс KZT |
