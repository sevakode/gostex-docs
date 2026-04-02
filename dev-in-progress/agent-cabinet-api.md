# Agent Cabinet API

API кабинета агента. Только чтение — агент не может создавать, редактировать или удалять данные.

**Base URL:** `/api/v2/agent`
**Auth:** Bearer token (Sanctum, guard `agent`)
**Content-Type:** `application/json`

---

## Аутентификация

### POST `/auth/login`

Публичный эндпоинт. Поддерживает 2FA (Google Authenticator).

**Request:**

```json
{
  "email": "agent@example.com",
  "password": "secret",
  "otp": "123456"           // обязательно, если включён 2FA
}
```

**Response 200** (успех):

```json
{
  "token": "1|abc123...",
  "agent": {
    "id": 1,
    "name": "Agent Name",
    "email": "agent@example.com",
    "has_2fa": true
  },
  "merchants": [
    { "id": 5, "name": "MerchantOne", "code": "M001" }
  ]
}
```

**Response 200** (требуется 2FA — otp не передан):

```json
{
  "requires_2fa": true,
  "message": "Требуется код 2FA"
}
```

**Response 422** (ошибка):

```json
{
  "message": "The given data was invalid.",
  "errors": { "email": ["Неверные учётные данные."] }
}
```

---

### POST `/auth/logout`

Удаляет текущий токен.

**Response 200:**

```json
{ "message": "Logged out" }
```

---

### GET `/auth/me`

Текущий агент + его мерчанты и провайдеры.

**Response 200:**

```json
{
  "agent": {
    "id": 1,
    "name": "Agent Name",
    "email": "agent@example.com",
    "has_2fa": true,
    "last_login_at": "2026-03-31T10:00:00+00:00"
  },
  "merchants": [
    { "id": 5, "name": "MerchantOne", "code": "M001" }
  ],
  "providers": [
    { "id": 12, "name": "ProviderX" }
  ]
}
```

---

## Профиль

### GET `/profile`

**Response 200:**

```json
{
  "data": {
    "id": 1,
    "name": "Agent Name",
    "email": "agent@example.com",
    "has_2fa": false,
    "payout_details": {
      "method": "usdt_trc20",
      "network": "TRC20",
      "walletAddress": "T...",
      "currency": "USDT"
    },
    "last_login_at": "2026-03-31T10:00:00+00:00",
    "created_at": "2026-01-15T12:00:00+00:00"
  }
}
```

`payout_details` — объект или `null`. Структура задаётся при создании агента в админке.

---

## Дашборд

### GET `/dashboard/summary`

Сводка по всем мерчантам агента.

**Query:**

| Параметр | Тип    | Обязательный | Описание        |
|----------|--------|:------------:|-----------------|
| `from`   | date   | нет          | Начало периода  |
| `to`     | date   | нет          | Конец периода   |

**Response 200:**

```json
{
  "data": {
    "total_volume": 1500000.00,
    "total_commission": 15000.00,
    "merchant_count": 3,
    "payment_count": 1240,
    "conversion_rate": 78.50
  }
}
```

| Поле               | Тип   | Описание                                                |
|--------------------|-------|---------------------------------------------------------|
| `total_volume`     | float | Сумма finished-платежей, ₽                              |
| `total_commission` | float | Заработок агента (volume × rate по каждому мерчанту)     |
| `merchant_count`   | int   | Количество назначенных мерчантов                        |
| `payment_count`    | int   | Количество finished-платежей                            |
| `conversion_rate`  | float | finished / total × 100, %                               |

---

### GET `/dashboard/daily-volume`

Объём по дням (для графика).

**Query:**

| Параметр | Тип    | Обязательный | Описание                        |
|----------|--------|:------------:|---------------------------------|
| `from`   | date   | **да**       | Начало периода                  |
| `to`     | date   | **да**       | Конец периода (≥ from)          |

**Response 200:**

```json
{
  "data": [
    { "date": "2026-03-01", "volume": 52000.00 },
    { "date": "2026-03-02", "volume": 48000.00 }
  ]
}
```

---

### GET `/dashboard/provider-stats`

Статистика по провайдерам агента.

**Response 200:**

```json
{
  "data": [
    {
      "provider_id": 12,
      "total": 500,
      "success": 420,
      "failed": 80,
      "payout_conversion": 84.00,
      "total_earned": 4200.00
    }
  ]
}
```

| Поле                | Тип   | Описание                          |
|---------------------|-------|-----------------------------------|
| `provider_id`       | int   | ID провайдера                     |
| `total`             | int   | Всего инвойсов                    |
| `success`           | int   | finished-платежи                  |
| `failed`            | int   | failed + expired + canceled       |
| `payout_conversion` | float | success / total × 100, %         |
| `total_earned`      | float | Заработок агента от провайдера    |

---

## Мерчанты

### GET `/merchants`

Список мерчантов агента.

**Query:**

| Параметр   | Тип    | Обязательный | Описание                          |
|------------|--------|:------------:|-----------------------------------|
| `search`   | string | нет          | Поиск по name / code (ilike)      |
| `status`   | string | нет          | Фильтр по статусу мерчанта       |
| `per_page` | int    | нет          | Записей на страницу (default: 20) |
| `page`     | int    | нет          | Номер страницы                    |

**Response 200** (пагинация Laravel):

```json
{
  "data": [
    { "id": 5, "name": "MerchantOne", "status": "active", "code": "M001" }
  ],
  "links": { "first": "...", "last": "...", "prev": null, "next": "..." },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 3,
    "per_page": 20,
    "to": 20,
    "total": 55
  }
}
```

---

### GET `/merchants/{merchant}`

Детали мерчанта с финансовой статистикой.

> Middleware `EnsureAgentMerchant` проверяет, что мерчант назначен агенту.

**Response 200:**

```json
{
  "data": {
    "id": 5,
    "name": "MerchantOne",
    "status": "active",
    "code": "M001",
    "commission_payin": 1.50,
    "commission_payout": 2.00,
    "deposit_volume": 1200000.00,
    "average_check": 5000.00,
    "payment_conversion": 78.50,
    "payout_volume": 300000.00,
    "chargeback_rate": 0.50,
    "refund_rate": 1.20,
    "daily_volume": [
      { "date": "2026-03-01", "volume": 52000.00 }
    ]
  }
}
```

| Поле                 | Тип   | Описание                                      |
|----------------------|-------|-----------------------------------------------|
| `commission_payin`   | float | Ставка комиссии агента на payin, %            |
| `commission_payout`  | float | Ставка комиссии агента на payout, %           |
| `deposit_volume`     | float | Общий объём finished-платежей, ₽              |
| `average_check`      | float | Средний чек finished-платежей, ₽              |
| `payment_conversion` | float | finished / total × 100, %                     |
| `payout_volume`      | float | Объём finished-payouts, ₽                     |
| `chargeback_rate`    | float | dispute / total × 100, %                      |
| `refund_rate`        | float | canceled / total × 100, %                     |
| `daily_volume`       | array | Объём по дням за последние 30 дней            |

---

### GET `/merchants/{merchant}/payments`

Платежи мерчанта.

**Query:**

| Параметр    | Тип    | Обязательный | Описание                                     |
|-------------|--------|:------------:|----------------------------------------------|
| `type`      | string | нет          | payin, payout, withdrawal, converted, deposit |
| `status`    | string | нет          | created, pending, finished, canceled, etc.    |
| `date_from` | date   | нет          | Начало периода                               |
| `date_to`   | date   | нет          | Конец периода                                |
| `per_page`  | int    | нет          | default: 20                                  |
| `page`      | int    | нет          | Номер страницы                               |

**Response 200:** стандартная пагинация с объектами Payment (все поля модели).

---

### GET `/merchants/{merchant}/balance`

Балансы мерчанта.

**Response 200:**

```json
{
  "data": [
    {
      "base_currency": "RUB",
      "quote_currency": "USDT",
      "amount": 150000.00
    }
  ]
}
```

---

## Провайдеры

### GET `/providers`

Список провайдеров агента.

**Query:**

| Параметр   | Тип | Обязательный | Описание  |
|------------|-----|:------------:|-----------|
| `per_page` | int | нет          | default: 20 |
| `page`     | int | нет          | Страница  |

**Response 200** (пагинация):

```json
{
  "data": [
    { "id": 12, "name": "ProviderX", "status": "active", "type": "acquiring" }
  ],
  "links": { ... },
  "meta": { ... }
}
```

---

### GET `/providers/{provider}`

Детали провайдера.

> Проверяет доступ: если провайдер не назначен агенту → `403`.

**Response 200:**

```json
{
  "data": {
    "id": 12,
    "name": "ProviderX",
    "status": "active",
    "type": "acquiring"
  }
}
```

---

### GET `/providers/stats`

То же что `GET /dashboard/provider-stats` — статистика по всем провайдерам агента.

---

## Комиссии

### GET `/commissions`

Список комиссий агента (рассчитывается на лету из finished-платежей).

**Query:**

| Параметр      | Тип    | Обязательный | Описание           |
|---------------|--------|:------------:|--------------------|
| `date_from`   | date   | нет          | Начало периода     |
| `date_to`     | date   | нет          | Конец периода      |
| `merchant_id` | int    | нет          | Фильтр по мерчанту |
| `type`        | string | нет          | payin / payout     |
| `per_page`    | int    | нет          | default: 20        |
| `page`        | int    | нет          | Страница           |

**Response 200** (пагинация):

```json
{
  "data": [
    {
      "id": 1042,
      "date": "2026-03-30",
      "merchant_id": 5,
      "merchant_name": "MerchantOne",
      "provider_id": 12,
      "type": "payin",
      "payment_amount": 10000.00,
      "currency": "RUB",
      "rate": 1.50,
      "amount": 150.00,
      "status": "approved"
    }
  ],
  "links": { ... },
  "meta": { ... }
}
```

| Поле             | Тип    | Описание                            |
|------------------|--------|-------------------------------------|
| `id`             | int    | ID платежа (Payment.id)             |
| `date`           | string | Дата платежа (YYYY-MM-DD)           |
| `merchant_id`    | int    | ID мерчанта                         |
| `merchant_name`  | string | Имя мерчанта                        |
| `provider_id`    | int?   | ID провайдера (из Payment)          |
| `type`           | string | payin / payout                      |
| `payment_amount` | float  | Сумма платежа                       |
| `currency`       | string | Валюта платежа                      |
| `rate`           | float  | Ставка комиссии агента, %           |
| `amount`         | float  | Сумма комиссии (payment × rate / 100) |
| `status`         | string | Всегда `approved` (finished-платежи) |

**Формула:** `amount = payment_amount × rate / 100`

---

### GET `/commissions/summary`

Сводка по комиссиям.

**Query:**

| Параметр | Тип  | Обязательный | Описание       |
|----------|------|:------------:|----------------|
| `from`   | date | нет          | Начало периода |
| `to`     | date | нет          | Конец периода  |

**Response 200:**

```json
{
  "data": {
    "total_earned": 15000.00,
    "by_merchant": [
      { "merchant_id": 5, "merchant_name": "MerchantOne", "earned": 9000.00 },
      { "merchant_id": 8, "merchant_name": "MerchantTwo", "earned": 6000.00 }
    ],
    "by_type": {
      "payin": 12000.00,
      "payout": 3000.00
    }
  }
}
```

---

## Выплаты

### GET `/payouts`

История выплат агенту.

**Query:**

| Параметр    | Тип    | Обязательный | Описание                                     |
|-------------|--------|:------------:|----------------------------------------------|
| `status`    | string | нет          | pending, processing, completed, blocked       |
| `date_from` | date   | нет          | Начало периода                               |
| `date_to`   | date   | нет          | Конец периода                                |
| `per_page`  | int    | нет          | default: 20                                  |
| `page`      | int    | нет          | Страница                                     |

**Response 200** (пагинация, модель AgentPayout):

```json
{
  "data": [
    {
      "id": 1,
      "agent_id": 1,
      "date": "2026-03-15",
      "amount": 5000.00,
      "currency": "USDT",
      "method": "usdt_trc20",
      "status": "completed",
      "reference": "txhash123...",
      "block_reason": null,
      "created_at": "2026-03-15T10:00:00+00:00",
      "updated_at": "2026-03-15T10:30:00+00:00"
    }
  ],
  "links": { ... },
  "meta": { ... }
}
```

**Статусы выплат:**

| Статус       | Описание                          |
|--------------|-----------------------------------|
| `pending`    | Ожидает обработки                 |
| `processing` | В процессе                        |
| `completed`  | Выполнена (reference заполнен)    |
| `blocked`    | Заблокирована (block_reason)      |

---

### GET `/payouts/balance`

Баланс выплат.

**Response 200:**

```json
{
  "data": {
    "total_earned": 15000.00,
    "total_paid": 8000.00,
    "total_pending": 2000.00,
    "available": 5000.00
  }
}
```

| Поле            | Тип   | Описание                                        |
|-----------------|-------|-------------------------------------------------|
| `total_earned`  | float | Сумма всех комиссий (finished-платежи)          |
| `total_paid`    | float | Сумма completed-выплат                          |
| `total_pending` | float | Сумма pending + processing выплат               |
| `available`     | float | total_earned − total_paid − total_pending       |

---

## Уведомления (Alerts)

### GET `/alerts`

Список уведомлений агента.

**Query:**

| Параметр   | Тип     | Обязательный | Описание                    |
|------------|---------|:------------:|-----------------------------|
| `is_read`  | 0 / 1   | нет          | Фильтр: 0 = непрочитанные  |
| `type`     | string  | нет          | Тип уведомления             |
| `per_page` | int     | нет          | default: 20                 |
| `page`     | int     | нет          | Страница                    |

**Response 200** (пагинация, модель AgentAlert с relations):

```json
{
  "data": [
    {
      "id": 1,
      "agent_id": 1,
      "type": "provider_down",
      "message": "Провайдер ProviderX недоступен",
      "merchant_id": 5,
      "merchant": { "id": 5, "name": "MerchantOne" },
      "provider_id": 12,
      "provider": { "id": 12, "name": "ProviderX" },
      "is_read": false,
      "created_at": "2026-03-31T10:00:00+00:00",
      "updated_at": "2026-03-31T10:00:00+00:00"
    }
  ],
  "links": { ... },
  "meta": { ... }
}
```

`merchant` и `provider` — вложенные объекты (`{id, name}`) или `null`.

---

### POST `/alerts/{alertId}/read`

Пометить уведомление прочитанным.

> Единственная write-операция агента. Обновляет только флаг `is_read`.

**Response 200:**

```json
{ "message": "Alert marked as read" }
```

---

### POST `/alerts/read-all`

Пометить все непрочитанные уведомления прочитанными.

**Response 200:**

```json
{ "message": "Marked 5 alerts as read" }
```

---

## Ошибки

Все ошибки возвращаются в едином формате.

| Код | Описание                                      |
|-----|-----------------------------------------------|
| 401 | `{"message": "Unauthenticated"}`              |
| 403 | `{"message": "Account is deactivated"}`       |
| 403 | `{"message": "Merchant not accessible"}`      |
| 403 | `{"message": "Provider not accessible"}`      |
| 404 | `{"message": "Merchant not found"}`           |
| 422 | `{"message": "...", "errors": {"field": [...]}}`|

---

## Безопасность

1. **Только чтение** — нет POST/PUT/DELETE для бизнес-данных (единственный POST — `alerts/read`)
2. **Guard `agent`** — отдельный Sanctum guard, изолирован от `admin` и `cabinet`
3. **EnsureAgentAccess** — проверяет `is_active`, блокирует деактивированных
4. **EnsureAgentMerchant** — агент видит только свои мерчанты (проверка через pivot-таблицу `agent_merchant`)
5. **Провайдеры** — агент видит только назначенные провайдеры (проверка через `agent_provider`)
6. **2FA** — Google Authenticator, обязателен если настроен (кроме `dev`-окружения)
7. **Комиссии** — рассчитываются query-time из платежей, агент не может влиять на ставки

---

## Сводная таблица эндпоинтов

| Метод | Эндпоинт                          | Описание                     |
|-------|------------------------------------|------------------------------|
| POST  | `/auth/login`                      | Авторизация + 2FA            |
| POST  | `/auth/logout`                     | Выход                        |
| GET   | `/auth/me`                         | Текущий агент                |
| GET   | `/profile`                         | Профиль с реквизитами        |
| GET   | `/dashboard/summary`               | Сводка (объём, комиссия)     |
| GET   | `/dashboard/daily-volume`          | Объём по дням                |
| GET   | `/dashboard/provider-stats`        | Стата по провайдерам         |
| GET   | `/merchants`                       | Список мерчантов             |
| GET   | `/merchants/{id}`                  | Детали мерчанта              |
| GET   | `/merchants/{id}/payments`         | Платежи мерчанта             |
| GET   | `/merchants/{id}/balance`          | Балансы мерчанта             |
| GET   | `/providers`                       | Список провайдеров           |
| GET   | `/providers/{id}`                  | Детали провайдера            |
| GET   | `/providers/stats`                 | Стата по провайдерам         |
| GET   | `/commissions`                     | Список комиссий              |
| GET   | `/commissions/summary`             | Сводка комиссий              |
| GET   | `/payouts`                         | Список выплат                |
| GET   | `/payouts/balance`                 | Баланс выплат                |
| GET   | `/alerts`                          | Уведомления                  |
| POST  | `/alerts/{id}/read`                | Прочитать уведомление        |
| POST  | `/alerts/read-all`                 | Прочитать все                |
