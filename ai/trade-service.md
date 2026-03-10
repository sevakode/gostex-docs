# 🔄 Trade Service - Детальная документация

> **Тип:** P2P процессинг  
> **Стек:** Yii2/PHP  
> **Зависимость:** Требует Aggregator

---

## 1. Обзор

Trade — это внутренний P2P сервис для обработки платежей через трейдеров (физических лиц с банковскими картами).

### 1.1 Ключевые функции

| Функция | Описание |
|---------|----------|
| Управление трейдерами | Регистрация, верификация, балансы |
| Управление реквизитами | Карты, счета, лимиты |
| Распределение заявок | Автоматический выбор трейдера |
| SMS парсинг | Автоматическое подтверждение по SMS |
| Telegram Bot | Уведомления и управление для трейдеров |

### 1.2 Архитектура

```
trade/
├── backend/           # Админка
│   ├── controllers/   # 15 контроллеров
│   ├── models/        # Формы и поиск
│   ├── views/         # Админ-интерфейс
│   └── services/      # Бизнес-логика
├── frontend/          # API трейдеров
│   ├── controllers/
│   │   ├── WebhookController.php   # API от Aggregator
│   │   ├── ApiController.php       # API трейдеров
│   │   └── SiteController.php      # UI трейдеров
│   └── services/
│       └── cards/     # Логика работы с картами
├── common/
│   └── models/        # 24 ActiveRecord модели
└── console/
    └── controllers/   # Cron jobs
```

---

## 2. Модели данных

### 2.1 Deals (Заявки)

Основная сущность — заявка на payin или payout.

```php
class Deals extends ActiveRecord {
    // Идентификаторы
    int $id;                    // Внутренний ID
    string $deal_id;            // ID от Aggregator (order_id)
    string $user_id_api;        // userId от мерчанта
    
    // Связи
    int $requisite_id;          // Реквизит трейдера
    int $method_id;             // Метод оплаты
    int $bank_id;               // Банк
    
    // Финансы
    float $amount;              // Сумма в фиате
    float $rate;                // Курс
    float $commission;          // Комиссия трейдера
    
    // Статусы
    int $status;                // 1=открыто, 2=закрыто, 3=отменено, 4=спор
    int $is_payed;              // Флаг оплаты
    int $is_payout;             // 0=payin, 1=payout
    
    // Файлы
    string $check_image;        // Чек от плательщика
    string $payout_check;       // Чек от трейдера (payout)
    
    // Данные плательщика
    string $payer_card;         // Карта плательщика
    string $payer_name;         // ФИО плательщика
    
    // Timestamps
    datetime $created_at;
    datetime $closed_at;
    datetime $expired_at;
}
```

**Статусы заявок:**

| Status | Название | Описание |
|--------|----------|----------|
| 1 | Открыто | Ожидает оплаты |
| 2 | Закрыто | Успешно завершено |
| 3 | Отменено | Отменено/истекло |
| 4 | Спор | Диспут/апелляция |
| 5 | На проверке | Ручная проверка |

### 2.2 Requisites (Реквизиты)

```php
class Requisites extends ActiveRecord {
    int $id;
    int $user_id;               // Трейдер-владелец
    int $bank_id;               // Банк
    
    // Данные карты/счёта
    string $number;             // Номер карты/телефона
    string $recipient;          // ФИО получателя
    
    // Лимиты
    float $limit;               // Дневной лимит
    float $limit_one;           // Лимит на одну операцию
    float $current_turnover;    // Текущий оборот за день
    
    // Настройки
    int $archive;               // 0=активный, 1=архив
    bool $intrabank;            // Внутрибанковский режим (bank-to-bank)
    bool $is_payout;            // Доступен для payout
    
    // Timestamps
    datetime $last_used_at;     // Последнее использование
}
```

### 2.3 User (Трейдеры)

```php
class User extends ActiveRecord implements IdentityInterface {
    int $id;
    string $username;
    string $email;
    string $password_hash;
    string $auth_key;           // Token для API
    
    // Финансы
    float $balance;             // Баланс USDT
    float $frozen_balance;      // Замороженный баланс
    
    // Настройки
    bool $is_active;            // Активен
    bool $is_verified;          // Верифицирован
    int $max_deals;             // Макс. одновременных заявок
    
    // Telegram
    string $telegram_id;
    string $telegram_username;
}
```

### 2.4 Bank (Банки)

```php
class Bank extends ActiveRecord {
    int $id;
    string $name;               // "Сбербанк"
    string $slug;               // "sber"
    string $color;              // "#00A859"
    string $icon;               // URL иконки
    bool $is_active;
}
```

### 2.5 Device (Устройства SMS)

```php
class Device extends ActiveRecord {
    int $id;
    int $user_id;               // Трейдер
    string $device_id;          // UUID устройства
    string $name;               // "iPhone 12"
    bool $is_active;
    datetime $last_seen_at;
}
```

### 2.6 DeviceMessage (SMS сообщения)

```php
class DeviceMessage extends ActiveRecord {
    int $id;
    int $device_id;
    string $sender;             // "900" (Сбер), "Tinkoff"
    string $body;               // Текст SMS
    float $parsed_amount;       // Распознанная сумма
    string $parsed_card;        // Распознанные последние цифры карты
    int $deal_id;               // Связанная заявка (если найдена)
    datetime $received_at;
}
```

---

## 3. Webhook API

### 3.1 Создание Payin заявки

**Endpoint:** `POST /webhook/set-send-deal`

**Request:**
```json
{
    "token": "trader_auth_token",
    "deal_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 5000,
    "method": "sbp",
    "user_id_api": "user_123",
    "user_ip": "188.11.55.10",
    "sender_bank": "tbank",
    "callback_url": "https://aggregator.domain.com/callback/trade"
}
```

**Response (Success):**
```json
{
    "status": true,
    "transaction_id": "trade_txn_123",
    "requisite": {
        "number": "+79001234567",
        "recipient": "Иван П.",
        "bank": "Тинькофф",
        "message": "Перевод другу"
    }
}
```

**Response (No Cards):**
```json
{
    "status": false,
    "error": "No available cards"
}
```

### 3.2 Создание Payout заявки

**Endpoint:** `POST /webhook/set-payout-deal`

**Request:**
```json
{
    "token": "trader_auth_token",
    "deal_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 10000,
    "method": "sbp",
    "recipient": {
        "card_number": "4276380012345678",
        "holder_name": "IVAN IVANOV",
        "bank_name": "sber"
    },
    "callback_url": "https://aggregator.domain.com/callback/trade"
}
```

### 3.3 Получение статуса

**Endpoint:** `POST /webhook/get-status-deal`

**Request:**
```json
{
    "token": "trader_auth_token",
    "deal_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Response:**
```json
{
    "status": true,
    "deal": {
        "id": 12345,
        "deal_id": "550e8400-e29b-41d4-a716-446655440000",
        "status": 2,
        "status_name": "Закрыто",
        "amount": 5000,
        "closed_at": "2024-12-17T10:05:00Z"
    }
}
```

### 3.4 Проверка доступности реквизитов

**Endpoint:** `POST /webhook/check-requisites`

**Request:**
```json
{
    "token": "trader_auth_token",
    "method": "sbp",
    "amount": 5000
}
```

**Response:**
```json
{
    "status": true,
    "available_count": 15,
    "min_amount": 1000,
    "max_amount": 100000
}
```

### 3.5 SMS Webhook

**Endpoint:** `POST /webhook/set-sms-{bank}`

Где `{bank}` = sber, tbank, vtb, alfa, etc.

**Request:**
```json
{
    "token": "device_auth_token",
    "device_id": "device_uuid",
    "sender": "900",
    "body": "Перевод 5000р от Иван П. Баланс: 15000р",
    "received_at": "2024-12-17T10:05:00Z"
}
```

---

## 4. SMS Парсинг

### 4.1 Поддерживаемые банки

| Банк | Sender | Формат |
|------|--------|--------|
| Сбербанк | 900 | "Перевод {amount}р от {name}" |
| Тинькофф | Tinkoff | "Пополнение +{amount} ₽" |
| ВТБ | VTB | "Зачисление {amount} RUB" |
| Альфа | Alfabank | "Поступление {amount}" |
| Райффайзен | RAIFFEISEN | "+{amount} RUB на счёт" |

### 4.2 Алгоритм парсинга

```
1. Получить SMS от устройства трейдера
2. Определить банк по sender
3. Применить regex для извлечения суммы
4. Найти открытую заявку с такой суммой у этого трейдера
5. Если найдена → автоматически закрыть заявку
6. Отправить callback в Aggregator
```

### 4.3 Matching правила

```php
// Точное совпадение суммы
$deal = Deals::find()
    ->where(['requisite_id' => $requisiteIds])
    ->andWhere(['amount' => $parsedAmount])
    ->andWhere(['status' => 1]) // Открыто
    ->orderBy(['created_at' => SORT_ASC])
    ->one();

// Если точного нет — ищем с допуском ±1₽
if (!$deal) {
    $deal = Deals::find()
        ->where(['requisite_id' => $requisiteIds])
        ->andWhere(['between', 'amount', $parsedAmount - 1, $parsedAmount + 1])
        ->andWhere(['status' => 1])
        ->one();
}
```

---

## 5. Распределение заявок

### 5.1 Алгоритм выбора реквизита

```php
class DealCardAllocatorService {
    public function findCard(string $method, float $amount, array $options): ?Requisites {
        // 1. Получить банки для метода
        $bankIds = Methods::getBankIdsByMethodName($method);
        
        // 2. Найти активные реквизиты
        $query = Requisites::find()
            ->where(['archive' => 0])
            ->andWhere(['bank_id' => $bankIds])
            ->andWhere(['>=', 'limit_one', $amount])
            ->andWhere('limit - current_turnover >= :amount', [':amount' => $amount]);
        
        // 3. Исключить занятые (есть открытая заявка)
        $query->andWhere(['not in', 'id', $this->getLockedCards()]);
        
        // 4. Фильтр по sender_bank (если указан)
        if ($senderBank = $options['sender_bank'] ?? null) {
            $query->andWhere(['bank_id' => Bank::findBySlug($senderBank)->id]);
        }
        
        // 5. Сортировка по приоритету
        $query->orderBy([
            'last_used_at' => SORT_ASC,  // Давно не использовался
            'current_turnover' => SORT_ASC,  // Меньше оборот
        ]);
        
        return $query->one();
    }
}
```

### 5.2 Блокировка карт

```php
class CardLockerService {
    private $redis;
    private $lockTtl = 900; // 15 минут
    
    public function lock(int $requisiteId, string $dealId): bool {
        $key = "card_lock:{$requisiteId}";
        return $this->redis->set($key, $dealId, ['NX', 'EX' => $this->lockTtl]);
    }
    
    public function unlock(int $requisiteId): void {
        $key = "card_lock:{$requisiteId}";
        $this->redis->del($key);
    }
    
    public function isLocked(int $requisiteId): bool {
        return $this->redis->exists("card_lock:{$requisiteId}");
    }
}
```

---

## 6. Telegram Bot

### 6.1 Команды бота

| Команда | Описание |
|---------|----------|
| `/start` | Начало работы, привязка аккаунта |
| `/status` | Текущий статус (баланс, активные заявки) |
| `/on` | Включить приём заявок |
| `/off` | Выключить приём заявок |
| `/cards` | Список карт |
| `/history` | История заявок |

### 6.2 Уведомления

**Новая заявка (payin):**
```
🆕 Новая заявка #12345

💰 Сумма: 5,000 ₽
🏦 Банк: Сбербанк
📱 Реквизит: **** 1234

⏱ Время на оплату: 15 минут

[Подтвердить] [Отменить]
```

**Новая заявка (payout):**
```
📤 Выплата #12346

💰 Сумма: 10,000 ₽
🏦 Банк: Тинькофф
💳 Карта: 4276 **** **** 5678
👤 Получатель: IVAN IVANOV

[Выполнено] [Проблема]
```

### 6.3 Inline кнопки

| Кнопка | Действие |
|--------|----------|
| ✅ Подтвердить | Закрыть заявку как успешную |
| ❌ Отменить | Отменить заявку |
| 🔄 Продлить | Продлить время на 5 минут |
| 📷 Загрузить чек | Запросить чек у плательщика |

---

## 7. Админка Trade

### 7.1 Разделы админки

| Раздел | URL | Описание |
|--------|-----|----------|
| Заявки | `/site/index` | Список всех заявок |
| Трейдеры | `/admin/index` | Управление трейдерами |
| Реквизиты | `/requisite/index` | Карты и счета |
| Устройства | `/device/index` | SMS Forwarder устройства |
| Банки | `/bank/index` | Справочник банков |
| Методы | `/methods/index` | Методы оплаты |
| Настройки | `/settings/index` | Системные настройки |
| Логи | `/logs/index` | Системные логи |

### 7.2 Роли и права

| Роль | Права |
|------|-------|
| admin | Полный доступ |
| manager | Заявки, трейдеры (только просмотр) |
| support | Заявки (только просмотр) |

---

## 8. Интеграция с Aggregator

### 8.1 Gateway класс

Trade интегрируется в Aggregator через `app/Gateway/InternalTrade/Pspru.php`:

```php
class Pspru extends BaseGateway implements Gatewable {
    
    public function createInvoice(Payment $payment, Method $method, array $params): Invoice {
        $response = Http::post($this->baseUrl . '/webhook/set-send-deal', [
            'token' => $this->config['token'],
            'deal_id' => $payment->order_id,
            'amount' => $payment->amount,
            'method' => $method->code,
            'user_id_api' => $params['userId'] ?? null,
            'sender_bank' => $params['senderBank'] ?? null,
        ]);
        
        if (!$response->json('status')) {
            throw new GatewayException($response->json('error'));
        }
        
        return new Invoice([
            'transaction_id' => $response->json('transaction_id'),
            'address' => $response->json('requisite.number'),
            'bank' => $response->json('requisite.bank'),
            'recipient' => $response->json('requisite.recipient'),
        ]);
    }
    
    public function processCallback(Request $request): array {
        $data = $request->all();
        
        return [
            'transaction_id' => $data['transaction_id'],
            'status' => $data['status'] === 'success' ? 'finished' : 'failed',
            'amount' => $data['amount'],
        ];
    }
}
```

### 8.2 Callback формат

Trade отправляет callback в Aggregator:

```json
{
    "transaction_id": "trade_txn_123",
    "deal_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "success",
    "amount": 5000,
    "payer_card": "4276****1234",
    "payer_name": "Иван П.",
    "closed_at": "2024-12-17T10:05:00Z"
}
```

---

## 9. Troubleshooting

### 9.1 Частые проблемы

**Проблема:** "No available cards"

**Причины:**
1. Все карты заняты (есть открытые заявки)
2. Превышены лимиты
3. Нет карт для указанного банка
4. Все трейдеры offline

**Решение:**
```sql
-- Проверить доступные карты
SELECT r.*, 
       (r.limit - r.current_turnover) as available_limit,
       (SELECT COUNT(*) FROM deals d WHERE d.requisite_id = r.id AND d.status = 1) as open_deals
FROM requisites r
WHERE r.archive = 0
  AND r.bank_id IN (SELECT bank_id FROM methods_bank WHERE method_id = ?)
```

**Проблема:** SMS не парсится

**Причины:**
1. Неизвестный формат SMS
2. Устройство offline
3. Нет открытой заявки с такой суммой

**Решение:**
- Проверить логи парсинга в `/logs/index`
- Добавить новый шаблон в SMS парсер

---

## 10. Метрики и мониторинг

### 10.1 Ключевые метрики

| Метрика | Описание | Норма |
|---------|----------|-------|
| `trade.deals.open` | Открытые заявки | < 100 |
| `trade.deals.avg_close_time` | Среднее время закрытия | < 5 мин |
| `trade.requisites.available` | Доступные карты | > 10 |
| `trade.sms.parsed` | SMS распознано | > 95% |
| `trade.traders.online` | Трейдеров онлайн | > 5 |

### 10.2 Алерты

| Алерт | Условие | Действие |
|-------|---------|----------|
| LowCards | available < 5 | Уведомить админа |
| HighOpenDeals | open > 50 | Проверить трейдеров |
| LongDeals | open > 30 min | Эскалация |
| DeviceOffline | device offline > 1h | Связаться с трейдером |


