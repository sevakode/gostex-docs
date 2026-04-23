# Статусы транзакций

> Источник: `OrderStatus.php`, `CallbackController.php`, Postman-коллекция AGGREpay API.

---

## Поля статуса

В системе два отдельных поля:

| Поле | Тип | Описание |
|------|-----|----------|
| `status` | `true` / `false` | Оплачено или нет. `true` = деньги получены |
| `state` | строка | Текущий этап жизненного цикла |
| `reason` | строка / null | Под-статус — уточняет причину (см. ниже) |

**Правило:** `status: true` всегда означает `state: finished` — для payin и payout.

---

## Статусы (state) — payin

| state | status | Описание |
|-------|--------|----------|
| `created` | `false` | Платёж создан, трейдер ещё не принял |
| `pending` | `false` | Трейдер принял, ожидаем перевода |
| `in_check` | `false` | Клиент нажал «Оплатил» — идёт ручная проверка |
| `dispute` | `false` | Открыт диспут |
| `finished` | `true` | Платёж подтверждён ✅ |
| `canceled` | `false` | Платёж отменён ❌ |
| `expired` | `false` | Истёк срок ожидания |
| `failed` | `false` | Техническая ошибка провайдера |

**Активные** (платёж ещё в работе): `created`, `pending`, `in_check`, `dispute`

### Жизненный цикл

```
created → pending → in_check → finished  (status: true)
                             ↘ canceled
                   ↘ dispute → finished
                             ↘ canceled
         ↘ expired
         ↘ failed
```

---

## Статусы (state) — payout

Логика payout отличается: выплата сразу создаётся с `status: true` и меняется на `false` только при отмене.

| state | status | Описание |
|-------|--------|----------|
| `pending` | `true` | Выплата создана, ожидает обработки |
| `in_check` | `false` | На проверке |
| `finished` | `true` | Выплата успешно отправлена ✅ |
| `canceled` | `false` | Выплата отменена ❌ |
| `expired` | `false` | Истёк срок обработки |

### Настройка: отправка коллбэка при отмене выплаты

У мерчанта есть настройка **«Отправлять коллбэк при отмене выплаты»** (`send_cancel_payout_callback`). По умолчанию отключена (`false`).

| Значение | Поведение системы |
|--------|-------------------|
| `false` (по умолчанию) | При получении кб `canceled` от провайдера: **статус выплаты не меняется**, мерчанту коллбэк не шлёт. Статус инвойса обновляется на `canceled`. В лог системы записывается уведомление: `Recieved callback with status canceled, payment status is not changed` |
| `true` | Статус выплаты меняется на `canceled`, мерчанту отправляется коллбэк с `state: canceled` |

> ℹ️ **`Recieved callback with status canceled, payment status is not changed`** — это **не ошибка**. Информационное сообщение: выплата получила кб «отмена» от провайдера, но у мерчанта выключена отправка коллбэка при отмене. Мерчант не узнает об отмене провайдера — выплата остаётся в прежнем статусе.

---

## Под-статусы (reason)

Поле `reason` уточняет причину текущего `state`. Может быть задано системой автоматически или оператором вручную.

### При `canceled`

| reason | Кто задаёт | Описание |
|--------|-----------|----------|
| `fake check` | Трейдер / Оператор | Клиент прислал поддельный чек |
| `no transfer made` | Трейдер / Оператор | Клиент не совершил перевод |
| `incorrect details` | Система / Оператор | Неверные реквизиты |
| `Canceled by Twix` | Система (Twix) | Отменено провайдером Twix |

### При `in_check`

| reason | Кто задаёт | Описание |
|--------|-----------|----------|
| `client clicked "I paid" button` | Система | Клиент нажал кнопку на странице оплаты |
| `taken for payout` | Система | Взято трейдером для выплаты |

### При `dispute`

| reason | Кто задаёт | Описание |
|--------|-----------|----------|
| `dispute created by merchant` | Мерчант (API / кабинет) | |
| `dispute created by client` | Клиент (Flow) | |
| `assigned to executor` | Система | Назначен исполнитель |
| `support assigned after 30 minutes` | Система | Автоназначение саппорта |
| `manager assigned after 60 minutes` | Система | Автоназначение менеджера |
| `request for video/statement from merchant` | Оператор | Запрошено подтверждение |
| `request for video/statement from executor` | Оператор | Запрошено подтверждение от трейдера |
| `Dispute opened by Twix` | Система (Twix) | |

### При `finished`

| reason | Кто задаёт | Описание |
|--------|-----------|----------|
| `success` | Система | Стандартное завершение |
| `success with recalculation` | Система | Завершён с перерасчётом суммы |
| `forced success` | Оператор | Принудительно закрыт как успешный |
| `forced success with recalculation` | Оператор | Принудительно + перерасчёт |

---

## Коллбэк мерчанту

AGGREpay отправляет POST-запрос на `callbackUri` мерчанта при каждом изменении статуса платежа. Ответ мерчанта игнорируется — система ожидает только HTTP 200. При недоступности эндпоинта возможна повторная отправка.

### Подпись запроса

Заголовок `Signature` содержит HMAC-подпись тела запроса с использованием `secret_key` мерчанта:

```
Signature: <hmac_sha256(secret_key, json_body)>
```

Если `secret_key` не задан — заголовок присутствует, но содержит пустую строку.

### Тело запроса

```json
{
  "datetime": 1755690497,
  "id": "order-id",
  "orderId": "external-id",
  "address": "4111111111111111",
  "created_at": "2025-08-20T11:47:48.000000Z",
  "status": true,
  "state": "finished",
  "type": "payin",
  "amount": "5500.00",
  "init_amount": "5500.00",
  "amount_commission_less": 5445.00,
  "currency": "RUB",
  "rate": 91.93,
  "userId": "u-4095"
}
```

### Описание полей

| Поле | Тип | Описание |
|------|-----|----------|
| `datetime` | int | Unix timestamp момента отправки |
| `id` | string | Внутренний ID платежа (наш `order_id`) |
| `orderId` | string | Внешний ID мерчанта (`external_id`) |
| `status` | bool | `true` = оплачено/завершено успешно |
| `state` | string | Текстовый статус: `finished`, `canceled`, `expired`, `in_check` |
| `type` | string | `payin` или `payout` |
| `amount` | string | Сумма транзакции (с учётом возможного перерасчёта) |
| `init_amount` | string | Изначальная сумма при создании |
| `amount_commission_less` | float | Сумма за вычетом комиссии (фактически зачисляется мерчанту) |
| `currency` | string | Валюта (`RUB`, `KZT`, etc.) |
| `rate` | float | Курс USDT на момент транзакции |
| `address` | string | Реквизиты (номер карты, телефон и т.д.) |
| `userId` | string | ID пользователя на стороне мерчанта (если передавался) |
| `file_url` | string | URL чека/скриншота (если есть) |
| `message` | string | Комментарий к инвойсу (если есть) |

> **Важно:** поля с пустым или `null` значением из тела автоматически вырезаются. Мерчант должен обрабатывать отсутствие необязательных полей.

> `status: true` + `state: finished` = оплачено. Ориентироваться нужно на оба поля вместе.

### Поведение при выплатах (payout)

По умолчанию коллбэк при **отмене** выплаты мерчанту **не отправляется**. Управляется флагом `send_cancel_payout_callback` на уровне мерчанта:

| Флаг | Поведение |
|------|-----------|
| `false` (по умолчанию) | Коллбэк при `canceled` не отправляется. Статус выплаты не меняется. |
| `true` | Коллбэк отправляется при любом изменении статуса, включая отмену. |

Сообщение в логах `Recieved callback with status canceled, payment status is not changed` — **не ошибка**, штатное поведение при `send_cancel_payout_callback = false`.

Отправляется POST на `callbackUri` при изменении статуса. Подписывается заголовком `Signature`.

### Тело запроса

```json
{
  "datetime": 1755690497,
  "id": "order-id",
  "orderId": "external-id",
  "address": "4111111111111111",
  "created_at": "2025-08-20T11:47:48.000000Z",
  "status": true,
  "state": "finished",
  "type": "payin",
  "amount": "5500.00",
  "init_amount": "5500.00",
  "amount_commission_less": "5445.00",
  "currency": "RUB",
  "rate": 91.93,
  "userId": "u-4095"
}
```

> `status: true` + `state: finished` = оплачено. Ориентироваться нужно на оба поля вместе.

---

## Где смотреть статус

- **API** → `POST /api/v1/payment/status` с `paymentId`
- **Админка** → Транзакции → поиск по ID
- **Grafana** → дашборд Payments → фильтр по `payment_id`
