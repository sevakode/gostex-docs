# 📡 API Reference - Полная документация

> **Base URL:** `https://{domain}/api/v2`  
> **Формат:** JSON  
> **Кодировка:** UTF-8

---

## 1. Аутентификация

### 1.1 Headers

Все запросы к API должны содержать следующие заголовки:

```http
Authorization: Bearer {api_token}
Content-Type: application/json
Signature: {hmac_sha256_signature}
```

### 1.2 Генерация Signature

**PHP:**
```php
$body = json_encode($data);
$signature = hash_hmac('sha256', $body, $secret_key);
```

**Python:**
```python
import hmac
import hashlib
import json

def generate_signature(data: dict, secret_key: str) -> str:
    body = json.dumps(data, separators=(',', ':'))
    return hmac.new(
        secret_key.encode(), 
        body.encode(), 
        hashlib.sha256
    ).hexdigest()
```

**JavaScript/Node.js:**
```javascript
const crypto = require('crypto');

function generateSignature(data, secretKey) {
    const body = JSON.stringify(data);
    return crypto
        .createHmac('sha256', secretKey)
        .update(body)
        .digest('hex');
}
```

**C#:**
```csharp
using System.Security.Cryptography;
using System.Text;

public static string GenerateSignature(string body, string secretKey)
{
    byte[] bodyBytes = Encoding.UTF8.GetBytes(body);
    byte[] keyBytes = Encoding.UTF8.GetBytes(secretKey);
    
    using (HMACSHA256 hmac = new HMACSHA256(keyBytes))
    {
        byte[] hashBytes = hmac.ComputeHash(bodyBytes);
        return BitConverter.ToString(hashBytes)
            .Replace("-", "")
            .ToLowerInvariant();
    }
}
```

### 1.3 Верификация Callback Signature

```php
$callbackBody = file_get_contents('php://input');
$signature = $_SERVER['HTTP_SIGNATURE'] ?? '';
$expectedSignature = hash_hmac('sha256', $callbackBody, $secretKey);

if ($signature === $expectedSignature) {
    // Signature valid - process callback
} else {
    // Signature invalid - reject
    http_response_code(401);
    exit;
}
```

---

## 2. Payments API

### 2.1 Create Payment (Payin)

Создание платежа для приёма средств.

**Endpoint:** `POST /api/v2/payments`

**Request Body:**

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `orderId` | string | ✅ | Уникальный ID заказа (макс. 64 символа) |
| `merchantId` | string | ✅ | ID мерчанта |
| `amount` | number | ✅ | Сумма в валюте |
| `currency` | string | ❌ | Валюта (RUB, UZS, KZT...), по умолчанию RUB |
| `method` | string | ❌ | Метод оплаты (sbp, c2c, nspk...) |
| `callbackUri` | string | ❌ | URL для callback |
| `successUri` | string | ❌ | URL редиректа при успехе |
| `failUri` | string | ❌ | URL редиректа при ошибке |
| `payer` | object | ❌ | Информация о плательщике |

**Payer Object:**

| Поле | Тип | Описание |
|------|-----|----------|
| `userId` | string | ID пользователя в системе мерчанта |
| `userIp` | string | IP адрес пользователя |
| `customerName` | string | ФИО плательщика |
| `phone` | string | Телефон |
| `email` | string | Email |
| `senderBank` | string | Банк отправителя (tbank, sber, vtb...) |
| `fingerprint` | string | Fingerprint устройства |
| `type` | string | Тип пользователя: ftd, std, trusted |
| `payments` | object | История платежей: {successful: N, expired: M} |

**Request Example:**
```json
{
    "orderId": "order_12345",
    "merchantId": "7ngpyb75rsuw6i5kdksabwsp",
    "amount": 5000,
    "currency": "RUB",
    "method": "sbp",
    "callbackUri": "https://merchant.com/callback",
    "successUri": "https://merchant.com/success",
    "failUri": "https://merchant.com/fail",
    "payer": {
        "userId": "user_123",
        "userIp": "188.11.55.10",
        "customerName": "Иван Иванов",
        "phone": "+79001234567",
        "email": "ivan@example.com",
        "senderBank": "tbank",
        "type": "std",
        "payments": {
            "successful": 3,
            "expired": 0
        }
    }
}
```

**Response (Success - 200):**
```json
{
    "status": true,
    "result": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "orderId": "order_12345",
        "state": "pending",
        "method": "sbp",
        "amount": 5000,
        "init_amount": 5000,
        "currency": "RUB",
        "rate": 97.50,
        "address": "+79001234567",
        "bank": "Тинькофф",
        "recipient": "Иван П.",
        "message": "Перевод другу",
        "expired_at": "2024-12-17T10:15:00Z"
    },
    "url": "https://flow.domain.com/550e8400-e29b-41d4-a716-446655440000"
}
```

**Response Fields:**

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | string | UUID платежа |
| `orderId` | string | ID заказа от мерчанта |
| `state` | string | Статус платежа |
| `method` | string | Метод оплаты |
| `amount` | number | Итоговая сумма (может отличаться из-за уникализации) |
| `init_amount` | number | Изначальная сумма |
| `currency` | string | Валюта |
| `rate` | number | Курс обмена (null для неттинга) |
| `address` | string | Номер карты/телефона для перевода |
| `bank` | string | Название банка |
| `recipient` | string | ФИО получателя |
| `message` | string | Комментарий к переводу |
| `url` | string | URL для NSPK/ECOM (если применимо) |
| `expired_at` | string | Время истечения |

---

### 2.2 Create Payout

Создание выплаты.

**Endpoint:** `POST /api/v2/payouts`

**Request Body:**

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `orderId` | string | ✅ | Уникальный ID заказа |
| `merchantId` | string | ✅ | ID мерчанта |
| `amount` | number | ✅ | Сумма выплаты |
| `currency` | string | ❌ | Валюта |
| `method` | string | ✅ | Метод выплаты |
| `callbackUri` | string | ❌ | URL для callback |
| `recipient` | object | ✅ | Данные получателя |

**Recipient Object:**

| Поле | Тип | Описание |
|------|-----|----------|
| `cardNumber` | string | Номер карты |
| `holderName` | string | ФИО держателя карты |
| `bankName` | string | Банк (slug: sber, tbank, vtb...) |
| `phone` | string | Телефон (для СБП) |
| `accountNumber` | string | Номер счёта |

**Request Example:**
```json
{
    "orderId": "payout_12345",
    "merchantId": "7ngpyb75rsuw6i5kdksabwsp",
    "amount": 10000,
    "currency": "RUB",
    "method": "c2c",
    "callbackUri": "https://merchant.com/callback",
    "recipient": {
        "cardNumber": "4276380012345678",
        "holderName": "IVAN IVANOV",
        "bankName": "sber"
    }
}
```

**Response (Success - 200):**
```json
{
    "status": true,
    "result": {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "orderId": "payout_12345",
        "state": "pending",
        "method": "c2c",
        "amount": 10000,
        "currency": "RUB"
    }
}
```

---

### 2.3 Get Payment Status

Получение статуса платежа.

**Endpoint (GET):** `GET /api/v2/orders/{orderId}/status`

**Endpoint (POST):** `POST /api/v2/status`

**POST Request Body:**
```json
{
    "orderId": "order_12345",
    "merchantId": "7ngpyb75rsuw6i5kdksabwsp"
}
```

**Response:**
```json
{
    "status": true,
    "result": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "orderId": "order_12345",
        "state": "finished",
        "method": "sbp",
        "amount": 5000,
        "init_amount": 5000,
        "currency": "RUB",
        "rate": 97.50,
        "commission": 3.5,
        "created_at": "2024-12-17T10:00:00Z",
        "finished_at": "2024-12-17T10:05:00Z"
    }
}
```

---

### 2.4 Cancel Payment

Отмена платежа (только для pending статуса).

**Endpoint:** `POST /api/v2/orders/{orderId}/cancel`

**Response:**
```json
{
    "status": true,
    "result": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "state": "canceled"
    }
}
```

---

### 2.5 Create Dispute

Создание апелляции/диспута.

**Endpoint:** `POST /api/v2/orders/{orderId}/dispute`

**Request Body:**
```json
{
    "merchantId": "7ngpyb75rsuw6i5kdksabwsp",
    "file": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ..."
}
```

**Response:**
```json
{
    "status": true,
    "result": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "state": "dispute",
        "dispute_created_at": "2024-12-17T10:10:00Z"
    }
}
```

---

## 3. Balance API

### 3.1 Get Balance

**Endpoint:** `GET /api/v2/balance`

**Response:**
```json
{
    "status": true,
    "result": {
        "usdt": 15000.50,
        "rub": 1500000.00,
        "uzs": 0,
        "kzt": 0,
        "azn": 0
    }
}
```

---

## 4. Methods API

### 4.1 Get Available Methods

**Endpoint:** `GET /api/v2/methods`

**Response:**
```json
{
    "status": true,
    "result": [
        {
            "code": "sbp",
            "name": "СБП",
            "currency": "RUB",
            "min_amount": 1000,
            "max_amount": 100000
        },
        {
            "code": "c2c",
            "name": "Card to Card",
            "currency": "RUB",
            "min_amount": 1000,
            "max_amount": 150000
        },
        {
            "code": "nspk",
            "name": "НСПК QR",
            "currency": "RUB",
            "min_amount": 100,
            "max_amount": 60000
        }
    ]
}
```

### 4.2 Список методов

| Код | Название | Валюта | Описание |
|-----|----------|--------|----------|
| `sbp` | СБП | RUB | Система быстрых платежей |
| `c2c` | Card to Card | RUB | P2P перевод по номеру карты |
| `nspk` | НСПК QR | RUB | QR-код через СБП |
| `spay` | SberPay | RUB | Внутрибанковский Сбер |
| `tpay` | T-Pay | RUB | Внутрибанковский Тинькофф |
| `vpay` | VTB Pay | RUB | Внутрибанковский ВТБ |
| `apay` | Alfa Pay | RUB | Внутрибанковский Альфа |
| `ecom` | ECOM 3DS | RUB | Классический эквайринг |
| `c2cuz` | UZS P2P | UZS | P2P Узбекистан |
| `humo` | HUMO | UZS | Карта HUMO |
| `uzcard` | UZCARD | UZS | Карта UZCARD |
| `c2ckz` | KZT P2P | KZT | P2P Казахстан |
| `c2caz` | AZN P2P | AZN | P2P Азербайджан |
| `upi` | UPI | INR | Индия UPI |
| `crypto` | Crypto | USDT | Криптовалюты |
| `m2ctj` | Трансгран TJ | RUB | Трансграничный Таджикистан |
| `m2ntj` | Трансгран TJ Phone | RUB | Трансграничный по телефону |
| `abh` | Абхазия | RUB | Трансграничный Абхазия |

---

## 5. Callbacks

### 5.1 Формат Callback

Callback отправляется на `callbackUri` при изменении статуса платежа.

**HTTP Headers:**
```
Content-Type: application/json
Signature: {hmac_sha256_signature}
X-Request-Id: {request_uuid}
```

**Body:**
```json
{
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "orderId": "order_12345",
    "merchantId": "7ngpyb75rsuw6i5kdksabwsp",
    "state": "finished",
    "method": "sbp",
    "amount": 5000,
    "init_amount": 5000,
    "currency": "RUB",
    "rate": 97.50,
    "commission": 3.5,
    "address": "+79001234567",
    "bank": "Тинькофф",
    "recipient": "Иван П.",
    "payer_card": "4276****1234",
    "payer_name": "Петров С.",
    "created_at": "2024-12-17T10:00:00Z",
    "finished_at": "2024-12-17T10:05:00Z"
}
```

### 5.2 Callback States

| State | Финальный | Когда отправляется |
|-------|-----------|-------------------|
| `pending` | ❌ | Реквизиты выданы |
| `in_check` | ❌ | Началась проверка |
| `dispute` | ❌ | Создан диспут |
| `finished` | ✅ | Успешно оплачено |
| `expired` | ✅ | Истёк срок |
| `failed` | ✅ | Ошибка создания |
| `canceled` | ✅ | Отменено |

### 5.3 Ответ на Callback

Ожидается HTTP 200 OK. Любой другой код — retry.

**Политика retry:**
- 3 попытки с интервалом 30 секунд
- Если все попытки неудачны — логируется ошибка

---

## 6. Error Codes

### 6.1 Полный список ошибок

| HTTP | Code | Message | Описание | Решение |
|------|------|---------|----------|---------|
| 400 | 400 | Invalid amount | Некорректная сумма | Проверить min/max лимиты |
| 400 | 40001 | Invalid orderId | Некорректный orderId | Проверить уникальность и формат |
| 400 | 40002 | All requisites are busy | Нет свободных реквизитов | Повторить позже или изменить сумму |
| 400 | 40003 | Invalid payment method | Метод не найден | Проверить code метода |
| 400 | 40004 | Insufficient funds | Недостаточно средств | Пополнить баланс |
| 400 | 40008 | Request is being processed | Запрос обрабатывается | Подождать и повторить |
| 401 | 401 | Unauthorized | Неавторизован | Проверить token и signature |
| 403 | 403 | Access denied | IP не в whitelist | Добавить IP в whitelist |
| 403 | 40301 | User blocked | Пользователь заблокирован | Связаться с поддержкой |
| 403 | 40302 | Anti-spam protection | Слишком много запросов | Подождать 15 минут |
| 404 | 40401 | Merchant not found | Мерчант не найден | Проверить merchantId |
| 404 | 40402 | Payment method not found | Метод не найден | Проверить доступные методы |
| 404 | 40403 | No payment methods | Методы не подключены | Подключить методы |
| 404 | 40404 | Resource not found | Ресурс не найден | Проверить URL |
| 409 | 40901 | Duplicate payment | Дубликат платежа | Использовать уникальный orderId |
| 422 | 422 | Unprocessable Content | Невалидные данные | Проверить формат данных |
| 500 | 50001 | Internal server error | Внутренняя ошибка | Связаться с поддержкой |

### 6.2 Пример ошибки

```json
{
    "status": false,
    "error": {
        "code": 40002,
        "message": "Invoice creation failed",
        "details": "All requisites are busy or url is not available"
    }
}
```

---

## 7. Webhook IPs

Callbacks отправляются с следующих IP:

```
116.202.228.230
64.226.73.77
```

**Рекомендация:** Добавить эти IP в whitelist на стороне мерчанта.

---

## 8. Rate Limits

| Ресурс | Лимит | Период |
|--------|-------|--------|
| Create Payment | 100 | минута |
| Get Status | 300 | минута |
| Get Balance | 60 | минута |
| Create Payout | 50 | минута |

При превышении лимита возвращается HTTP 429 Too Many Requests.

---

## 9. Postman Collection

**URL:** https://documenter.getpostman.com/view/13931884/2sAYQdipUu

**Environment Variables:**
- `baseUrl`: URL API
- `merchantId`: ID мерчанта
- `apiToken`: Bearer token
- `secretKey`: Secret key для подписи

---

## 10. Примеры интеграции

### 10.1 PHP (Laravel)

```php
class PaymentService
{
    private string $baseUrl;
    private string $merchantId;
    private string $apiToken;
    private string $secretKey;

    public function createPayment(array $data): array
    {
        $body = [
            'orderId' => $data['order_id'],
            'merchantId' => $this->merchantId,
            'amount' => $data['amount'],
            'currency' => $data['currency'] ?? 'RUB',
            'method' => $data['method'],
            'callbackUri' => route('payment.callback'),
            'successUri' => route('payment.success'),
            'failUri' => route('payment.fail'),
            'payer' => [
                'userId' => $data['user_id'],
                'userIp' => request()->ip(),
            ],
        ];

        $response = Http::withHeaders([
            'Authorization' => 'Bearer ' . $this->apiToken,
            'Signature' => $this->generateSignature($body),
        ])->post($this->baseUrl . '/api/v2/payments', $body);

        return $response->json();
    }

    private function generateSignature(array $data): string
    {
        return hash_hmac('sha256', json_encode($data), $this->secretKey);
    }
}
```

### 10.2 Python (FastAPI)

```python
import httpx
import hmac
import hashlib
import json
from typing import Optional

class PaymentClient:
    def __init__(self, base_url: str, merchant_id: str, 
                 api_token: str, secret_key: str):
        self.base_url = base_url
        self.merchant_id = merchant_id
        self.api_token = api_token
        self.secret_key = secret_key

    def _sign(self, data: dict) -> str:
        body = json.dumps(data, separators=(',', ':'))
        return hmac.new(
            self.secret_key.encode(),
            body.encode(),
            hashlib.sha256
        ).hexdigest()

    async def create_payment(
        self,
        order_id: str,
        amount: float,
        method: str,
        user_id: Optional[str] = None
    ) -> dict:
        data = {
            'orderId': order_id,
            'merchantId': self.merchant_id,
            'amount': amount,
            'method': method,
            'payer': {'userId': user_id} if user_id else {}
        }

        async with httpx.AsyncClient() as client:
            response = await client.post(
                f'{self.base_url}/api/v2/payments',
                json=data,
                headers={
                    'Authorization': f'Bearer {self.api_token}',
                    'Signature': self._sign(data),
                    'Content-Type': 'application/json'
                }
            )
            return response.json()
```

### 10.3 JavaScript (Node.js)

```javascript
const crypto = require('crypto');
const axios = require('axios');

class PaymentClient {
    constructor(baseUrl, merchantId, apiToken, secretKey) {
        this.baseUrl = baseUrl;
        this.merchantId = merchantId;
        this.apiToken = apiToken;
        this.secretKey = secretKey;
    }

    sign(data) {
        const body = JSON.stringify(data);
        return crypto
            .createHmac('sha256', this.secretKey)
            .update(body)
            .digest('hex');
    }

    async createPayment(orderId, amount, method, userId = null) {
        const data = {
            orderId,
            merchantId: this.merchantId,
            amount,
            method,
            payer: userId ? { userId } : {}
        };

        const response = await axios.post(
            `${this.baseUrl}/api/v2/payments`,
            data,
            {
                headers: {
                    'Authorization': `Bearer ${this.apiToken}`,
                    'Signature': this.sign(data),
                    'Content-Type': 'application/json'
                }
            }
        );

        return response.data;
    }
}

module.exports = PaymentClient;
```


