# AGGREpay Aggregator

> **Стек:** Laravel 12 / PHP 8.4 + PostgreSQL + Redis
> **Репозиторий:** `aggregator/`
> **Последнее обновление:** 2026-04-15

---

## 1. Обзор

AGGREpay Aggregator — ядро платёжной платформы. Принимает запросы от мерчантов, маршрутизирует через платёжных провайдеров (шлюзы), управляет каскадами, балансами, трейдерами, диспутами.

**Ключевые цифры:**
- 213+ платёжных шлюзов в `app/Gateway/`
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
- Дополнительные шлюзы — платно согласно тарифной сетке (уточняйте у менеджера)

**Исключения:** White-labels Протокол, Агрипей и Ковенант — интеграции в приоритетном порядке.

**Запросы на интеграцию** направлять через руководителя технического отдела.

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

## 10. Настройки приложения

Ключевые параметры в `app_settings` (через Admin API):

| Параметр | Описание |
|----------|----------|
| `app.sup_service` | Включить Support Service |
| `app.support_service_url` | URL Support Service |
| `app.support_service_token` | Токен аутентификации Support Service |
| `app.flow_url` | URL Flow (платёжная страница) |
| `app.trade_url` | URL Trade (реквизиты) |
| `app.rate_service_url` | URL Rate Service |
