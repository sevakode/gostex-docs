# InternalTrade - Платформа для P2P транзакций

Платформа для управления P2P транзакциями с поддержкой автоматической обработки SMS/Push уведомлений от банков, распределения заявок между трейдерами и интеграции с merchant системами.

## 📚 API Документация

### Merchant API (WebhookController)
- `POST /webhook/set-send-deal` - Создание заявки на пополнение (Payin)
- `POST /webhook/check-requisites` - Проверка доступности реквизитов
- `POST /webhook/set-payout-deal` - Создание заявки на вывод (Payout)
- `GET /webhook/get-status-deal` - Получение статуса заявки
- `POST /webhook/set-status-deal` - Изменение статуса заявки
- `POST /webhook/get-available-cards` - Список доступных карт
- `GET /webhook/get-available-banks` - Список доступных банков
- `POST /webhook/set-sms-{bank}` - SMS/Push вебхук от банков

### Trader API ⭐ (ОБНОВЛЕНО)
- `GET /api/orders` - Получение заказов с фильтрами
  - **Новое**: Фильтрация по статусам (`status=1,2,3` или `all`)
  - **Новое**: Фильтрация по типам (`type=export,import` или `all`)
  - **Новое**: Пагинация (`limit` и `offset`)
  - **Новое**: Полная информация по каждой сделке (все поля)
- `GET /api/devices` - **НОВЫЙ** Получение устройств трейдера
- `GET /api/device-banks` - **НОВЫЙ** Получение банков для устройства
- `GET /api/device-qr` - **НОВЫЙ** Получение QR данных для привязки устройства к банку
- `GET /api/statuses` - **НОВЫЙ** Получение списка всех статусов

### Wallet API
- `POST /wallet/get-admin-wallet` - Получение адреса кошелька USDT
- `POST /wallet/create-manual-deposit` - Создание депозита USDT
- `POST /wallet/create-withdrawal` - Создание заявки на вывод USDT

## 🚀 Быстрый старт

### Установка и инициализация

```bash
# Миграции БД
php yii migrate
php yii migrate --migrationPath=@yii/rbac/migrations/
php yii rbac-seeder/init

# Заполнение методов оплаты
php yii methods-seeder/seed

# Заполнение method_id для существующих реквизитов
php yii fill-method-id/index
```

### Запуск воркера очереди

Для запуска обработки заданий в очереди используйте команду:

```bash
php yii queue/listen
```

Для автоматического перезапуска в случае сбоев можно использовать Supervisor.

### Автоматический запуск с Supervisor

Рекомендуется использовать [Supervisor](https://www.yiiframework.com/extension/yiisoft/yii2-queue/doc/guide/2.0/ru/worker) для постоянной работы воркера и автоматического перезапуска в случае сбоев.

## 🎯 Примеры использования Trader API

### 1. Получение заказов с фильтрами

```bash
# Получить только открытые заявки (статус 1)
GET /api/orders?token=YOUR_TOKEN&status=1

# Получить заявки в работе и споры (статусы 2,3)
GET /api/orders?token=YOUR_TOKEN&status=2,3

# Получить только payin заявки (type=export)
GET /api/orders?token=YOUR_TOKEN&type=export

# Получить только payout заявки (type=import)
GET /api/orders?token=YOUR_TOKEN&type=import

# Комбинированные фильтры с пагинацией.
GET /api/orders?token=YOUR_TOKEN&status=1,2,3&type=export,import&limit=50&offset=0&date_from=01.11.2024&date_to=27.11.2024
```

**Ответ**:
```json
{
  "success": true,
  "data": [
    {
      "id": 123,
      "order_id": "ORDER-001",
      "transaction_id": "xyz123",
      "type": "export",
      "amount": -1000,
      "dispute_amount": null,
      "currency": "RUB",
      "rate": 92.5,
      "commission": 2.5,
      "status": 1,
      "status_name": "Created",
      "status_code": "pending",
      "file_url": null,
      "comment": null,
      "created_at": 1732701600,
      "updated_at": 1732701600,
      "created_at_formatted": "2024-11-27 10:00:00",
      "updated_at_formatted": "2024-11-27 10:00:00",
      "bank": "Сбербанк",
      "bank_slug": "sber",
      "requisite_number": "5559********1234",
      "requisite_recipient": "Иван Иванов"
    }
  ],
  "pagination": {
    "total": 150,
    "limit": 50,
    "offset": 0,
    "pages": 3
  }
}
```

### 2. Получение устройств

```bash
GET /api/devices?token=YOUR_TOKEN
```

**Ответ**:
```json
{
  "success": true,
  "data": [
    {
      "id": 5,
      "number": "samsung-s20-001",
      "created_at": 1732701600,
      "updated_at": 1732701600,
      "created_at_formatted": "2024-11-27 10:00:00",
      "updated_at_formatted": "2024-11-27 10:00:00"
    }
  ]
}
```

### 3. Получение банков для устройства

После выбора устройства получаем список банков, которые привязаны к этому устройству через реквизиты:

```bash
GET /api/device-banks?token=YOUR_TOKEN&device_id=5
```

**Ответ**:
```json
{
  "success": true,
  "device": {
    "id": 5,
    "number": "samsung-s20-001"
  },
  "banks": [
    {
      "id": 1,
      "name": "Сбербанк",
      "slug": "sber",
      "currency": "RUB"
    },
    {
      "id": 2,
      "name": "Тинькофф",
      "slug": "tinkoff",
      "currency": "RUB"
    }
  ]
}
```

### 4. Получение QR данных для SMS Forwarder

После выбора банка получаем QR данные для настройки SMS Forwarder:

```bash
GET /api/device-qr?token=YOUR_TOKEN&device_id=5&bank_id=1
```

**Ответ**:
```json
{
  "success": true,
  "data": {
    "from": "Sberbank",
    "type": "sms",
    "token": "YOUR_AUTH_KEY",
    "number": "samsung-s20-001",
    "url": "https://test-trade.gost-fintech.com/webhook/set-sms-sber"
  },
  "device": {
    "id": 5,
    "number": "samsung-s20-001"
  },
  "bank": {
    "id": 1,
    "name": "Сбербанк",
    "slug": "sber"
  },
  "parsing_type": "sms",
  "sender_name": "Sberbank"
}
```

**Логика работы**:
1. Трейдер выбирает устройство (`GET /api/devices`)
2. Получает список банков для этого устройства (`GET /api/device-banks?device_id=5`)
3. Выбирает банк и получает QR данные (`GET /api/device-qr?device_id=5&bank_id=1`)
4. Сканирует QR в приложении SMS Forwarder
5. SMS от этого банка будут автоматически отправляться на платформу

### 5. Получение списка статусов

```bash
GET /api/statuses
```

**Ответ**:
```json
{
  "success": true,
  "data": [
    {
      "code": 1,
      "name": "Created",
      "name_ru": "Создан",
      "api_code": "pending"
    },
    {
      "code": 2,
      "name": "In Progress",
      "name_ru": "В работе",
      "api_code": "in_check"
    },
    {
      "code": 3,
      "name": "Dispute",
      "name_ru": "Спор",
      "api_code": "dispute"
    },
    {
      "code": 4,
      "name": "Expired",
      "name_ru": "Истек срок",
      "api_code": "expired"
    },
    {
      "code": 5,
      "name": "Approved",
      "name_ru": "Одобрен",
      "api_code": "success"
    },
    {
      "code": 6,
      "name": "Cancelled",
      "name_ru": "Отменен",
      "api_code": "cancel"
    }
  ]
}
```

## 📊 Фильтры API

### Статусы (status)
- `1` - Created (Создан)
- `2` - In Progress (В работе)
- `3` - Dispute (Спор)
- `4` - Expired (Истек срок)
- `5` - Approved (Одобрен)
- `6` - Cancelled (Отменен)
- `all` - Все статусы

**Примеры**:
- `status=1` - только созданные
- `status=1,2,3` - созданные, в работе и споры
- `status=all` - все статусы

### Типы сделок (type)
- `export` - Payin (клиент → трейдер)
- `import` - Payout (трейдер → клиент)
- `deposit` - Пополнение USDT
- `withdrawal` - Вывод USDT
- `conversion` - Конверсия валют
- `all` - Все типы

**Примеры**:
- `type=export` - только payin
- `type=export,import` - payin и payout
- `type=all` - все типы

### Пагинация
- `limit` - Количество записей (по умолчанию: 100)
- `offset` - Смещение (по умолчанию: 0)

**Примеры**:
- `limit=50&offset=0` - первые 50 записей
- `limit=50&offset=50` - следующие 50 записей

## 🧪 Тестирование

Импортируйте Postman коллекцию для тестирования API:
```
InternalTrade_Webhook_API.postman_collection.json
```

**Переменные коллекции**:
- `baseUrl` - https://test-trade.gost-fintech.com
- `merchant_code` - Ваш код мерчанта
- `trader_token` - Токен трейдера для API
- `device_id` - ID устройства трейдера
- `bank_id` - ID банка для QR

## ⚙️ Конфигурация

### Переменные окружения (.env)

```bash
# Platform
PLATFORM=ASGARD

# Database
DB_DSN=pgsql:host=localhost;dbname=trade
DB_USERNAME=user
DB_PASSWORD=password

# USDT Wallets
USDT_TRC20_WALLET=TYourTRC20Address...
USDT_BEP20_WALLET=0xYourBEP20Address...

# Blockchain API Keys
BSCSCAN_API_KEY=your_bscscan_key
TRONSCAN_API_KEY=your_tronscan_key

# Aggregator
AGREPAY_BASE_URL=https://your-aggregator.com
AGREPAY_AUTH_TOKEN=your_token

# Telegram Logging
TELEGRAM_LOGGER=bot_token
TELEGRAM_LOGGER_ID=chat_id

# Payout Settings
PAYOUT_AUTO_ASSIGN=true
PAYOUT_COOLDOWN_MINUTES=5
```

## 🛠️ Технологии

- **PHP 8.1+**
- **Yii2 Framework**
- **PostgreSQL**
- **Redis** (для кеширования и очередей)
- **Telegram Bot API**
- **OpenTelemetry** (трейсинг)
- **Google2FA** (2FA)
- **BscScan/TronScan API**

---

**Версия**: 1.1  
**Дата обновления**: 27.11.2024

## Добавить в env агрегатора API_TOKEN_TRADE, в трейдер добавить API_TOKEN_AGREPAY с одинаковыми токенами

---

## PayoutStatusCallbackJob — масштабная доработка (2026-04-16)

В течение дня проведена масштабная итерация над `PayoutStatusCallbackJob` и связанными компонентами. Суммарно ~15 hotfix-коммитов (v1.24.9 → v1.24.23).

### Изменения

| Компонент | Изменение |
|-----------|-----------|
| **PayoutStatusCallbackJob** | Увеличены `maxRetries` и `retryIntervalSec` |
| **PayoutStatusCallbackJob** | Интеграция OpenTelemetry (трейсинг для всех операций) |
| **PayoutStatusCallbackJob** | Улучшена обработка ошибок, структурированное логирование |
| **PayoutStatusCallbackJob** | Добавлена обработка несовпадения transaction ID |
| **PayoutStatusCallbackJob** | Включён/отключён `cancelDeal` (итерационно отлаживалось) |
| **WebhookController** | Добавлена конфигурация agrepay |
| **WebhookController** | Временно отключалось/включалось queuing PayoutStatusCallbackJob |
| **WebhookController** | Добавлен компонент `OtelComponent` для трейсинга |
| **BaseMultiService** | Улучшена обработка ошибок |

### Итог

Job обработки статусов выплат переведён на OpenTelemetry, стал более устойчивым к ошибкам и несоответствиям данных. Retry-логика стала агрессивнее (1 сек интервал вместо 2).
