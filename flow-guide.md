# Руководство по настройке и использованию Flow (Платёжная страница)

> **Аудитория:** WL-операторы (владельцы вайтлейблов)
> **Версия:** Март 2026

---

## Содержание

1. [Что такое Flow](#1-что-такое-flow)
2. [Архитектура и место в системе](#2-архитектура-и-место-в-системе)
3. [Первоначальная настройка (деплой)](#3-первоначальная-настройка-деплой)
4. [Переменные окружения (env)](#4-переменные-окружения-env)
5. [Как работает кэшбокс (KB)](#5-как-работает-кэшбокс-kb)
6. [Определение KB по method_code](#6-определение-kb-по-method_code)
7. [Таблица method_code → UI](#7-таблица-method_code--ui)
8. [Брендирование и кастомизация](#8-брендирование-и-кастомизация)
9. [Локализация (языки)](#9-локализация-языки)
10. [WebSocket и обновления в реальном времени](#10-websocket-и-обновления-в-реальном-времени)
11. [Жизненный цикл платежа во Flow](#11-жизненный-цикл-платежа-во-flow)
12. [API-эндпоинты Flow](#12-api-эндпоинты-flow)
13. [Настройка мерчанта для работы с Flow](#13-настройка-мерчанта-для-работы-с-flow)
14. [Deeplinks (открытие банковских приложений)](#14-deeplinks-открытие-банковских-приложений)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Что такое Flow

**Flow** — это React SPA (Single Page Application), которая отображает платёжную форму для плательщика (payer). Когда мерчант создаёт платёж через API, агрегатор возвращает ссылку вида `{flow_url}/{order_id}` — именно её видит конечный пользователь.

Flow отвечает за:

- Отображение реквизитов для оплаты (номер карты, телефон, QR-код)
- Выбор способа оплаты (если мерчант подключил несколько методов)
- Загрузку чеков/квитанций
- Ввод OTP-кодов (для ecom/3DS)
- Отображение статуса платежа в реальном времени
- Редирект на `success_uri` / `fail_uri` после завершения

Существуют две версии: `flow/` (React 18 + Redux, текущая production) и `flow-new/` (React 19 + Ant Design + TanStack Query, новая версия).

---

## 2. Архитектура и место в системе

Каждый WL разворачивает свой экземпляр Flow:

```
WL = Aggregator ×2 + Trade + Flow + Admin Panel + Merchant Panel + PostgreSQL + Redis
```

```
┌──────────────┐     POST /api/v2/payments      ┌───────────────┐
│   Мерчант    │ ──────────────────────────────→ │  Aggregator   │
│   (сайт)     │ ←────────────────────────────── │  (Laravel)    │
│              │     { url: "flow.wl.com/abc" }  │               │
└──────┬───────┘                                 └───────┬───────┘
       │ redirect                                        │
       ▼                                                 │
┌──────────────┐     GET /api/v2/orders/{id}     ┌───────┴───────┐
│    Flow      │ ──────────────────────────────→ │  Aggregator   │
│   (React)    │ ←────────────────────────────── │  API          │
│              │     { orderData, methods }       │               │
└──────────────┘                                 └───────────────┘
       ↑ WebSocket / Polling
       │ (статус платежа)
```

Flow **не имеет** собственного бэкенда — все данные получает через REST API агрегатора.

---

## 3. Первоначальная настройка (деплой)

### Docker (рекомендуемый способ)

Flow поставляется как Docker-образ: Node.js для сборки → Nginx для раздачи.

```bash
# Сборка образа с нужными env (подставляются на этапе build)
docker build -t flow \
  --build-arg VITE_LARAVEL_APP_HOST=https://api.your-wl.com/api/v2/orders \
  --build-arg VITE_STYLE=aggrepay \
  --build-arg VITE_SHOW_LOGO=show \
  --build-arg VITE_SHOW_RECIPIENT=show \
  --build-arg VITE_SHOW_CHECK=show \
  --build-arg VITE_SHOW_TROUBLE_COMPONENT=show \
  .

# Запуск
docker run -d -p 8080:80 --name flow flow
```

> **Важно:** Все `VITE_*` переменные подставляются **на этапе сборки** (build-time), а не runtime. При изменении любого `VITE_*` параметра необходимо **пересобрать** образ.

### Nginx-конфигурация

Flow — это SPA, поэтому все маршруты должны вести на `index.html`:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Healthcheck для оркестратора
    location /healthz {
        return 200 'ok';
    }

    # Кэш для статики
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
}
```

### Локальная разработка

```bash
cd flow
npm install
npm run dev    # http://localhost:5173
```

---

## 4. Переменные окружения (env)

| Переменная | Обязательная | Описание | Пример |
|---|---|---|---|
| `VITE_LARAVEL_APP_HOST` | ✅ | Базовый URL API агрегатора | `https://api.wl.com/api/v2/orders` |
| `VITE_STYLE` | ✅ | Название стиля/бренда (определяет цвета и внешний вид) | `default`, `aggrepay`, `asgard`, `pipay` |
| `VITE_SHOW_LOGO` | — | Показать логотип мерчанта в шапке | `show` / `hide` |
| `VITE_SHOW_RECIPIENT` | — | Показать имя получателя | `show` / `hide` |
| `VITE_SHOW_CHECK` | — | Показать кнопку загрузки чека | `show` / `hide` |
| `VITE_SHOW_TROUBLE_COMPONENT` | — | Показать блок «Проблемы с оплатой?» | `show` / `hide` |
| `VITE_SHOW_VIDEO_INSTRUCTION` | — | Показать кнопку видеоинструкции | `show` / `hide` |
| `VITE_PUSHER_APP_KEY` | — | Ключ WebSocket (Soketi/Pusher) | `app-key` |
| `VITE_PUSHER_HOST` | — | Хост WebSocket-сервера | `soketi.wl.com` |
| `VITE_PUSHER_PORT` | — | Порт WebSocket | `443` |
| `VITE_PUSHER_SCHEME` | — | Протокол WebSocket | `https` |
| `VITE_PUSHER_CLUSTER` | — | Кластер Pusher | `mt1` |
| `VITE_GA_ID` | — | Google Analytics ID | `G-XXXXXXXXXX` |
| `VITE_YM_ID` | — | Yandex.Metrika ID | `12345678` |

### Минимальный набор для запуска

```env
VITE_LARAVEL_APP_HOST=https://api.your-wl.com/api/v2/orders
VITE_STYLE=default
```

Все остальные параметры опциональны и имеют разумные значения по умолчанию.

---

## 5. Как работает кэшбокс (KB)

**Кэшбокс (KB, cashbox)** — это конкретная визуальная форма оплаты, которую видит плательщик. Flow автоматически определяет, какой KB показать, на основе **method_code** из ответа API.

### Схема определения KB

```
Мерчант создаёт платёж (POST /api/v2/payments)
    │
    ▼
Агрегатор выполняет каскад:
  1. Берёт доступные методы мерчанта
  2. Фильтрует: is_hidden=false, cascade_percent>0, лимиты, валюта
  3. Сортирует: приоритет контрагента → cascade_percent → индекс
  4. Пробует каждый метод → первый успешный → Invoice
    │
    ▼
Ответ содержит method_code (sbp, c2c, ecom, crypto, ...)
    │
    ▼
Flow загружает заказ: GET /{orderId}
    │
    ▼
Из ответа извлекается methodCode:
  method.code → method.slug → method.key → method.identifier → method.name
    │
    ▼
На основе methodCode Flow решает, какой компонент показать:
  • nspk        → Автоматический редирект на redirectUrl
  • ecom        → Форма ввода карты + OTP (EnterCodePage)
  • upi         → Форма ввода UTR
  • m2tbank/m2sber/m2vtb → CardTransfer (трансграничный перевод)
  • card_ru + senderBank=alfa → CardTransfer (Альфа-Банк)
  • sbp/c2c/spay/... → MainForm с реквизитами + QR + deeplink
  • crypto      → MainForm с адресом кошелька
```

### Логика в коде (resolveMethodCode)

Flow извлекает method_code из API-ответа по цепочке приоритетов:

```typescript
const resolveMethodCode = (method: any): string | undefined => {
  return (
    method.code ??     // основной код метода
    method.slug ??     // slug (альтернативный идентификатор)
    method.key ??      // ключ
    method.identifier ?? // идентификатор
    method.name        // имя (fallback)
  );
};
```

Поддерживаются два формата API-ответа:

- **Формат с invoice:** `response.invoice.method.code` → methodCode
- **Плоский формат:** `response.method` (строка) → methodCode напрямую

---

## 6. Определение KB по method_code

После получения `methodCode` Flow определяет, какой тип кэшбокса показать. Логика распределения:

### Проверка на трансграничный перевод (m2*)

```typescript
const isTransborderMethod = (method?: string, methodCode?: string): boolean => {
  const value = (methodCode || method || '').toLowerCase();
  return value.startsWith('m2');  // m2tbank, m2sber, m2vtb
};
```

Если `methodCode` начинается с `m2` — это **трансграничный перевод**, и Flow показывает специальный UI (`CardTransfer`) с:

- Полями для выбора банка отправителя
- Deeplink-ами в мобильное приложение
- Инструкциями по переводу

### Проверка на card_ru + банк отправителя

Если `methodCode = card_ru` или `card`, Flow дополнительно проверяет `senderBank`:

- `tbank` / `tinkoff` → CardTransfer с deeplink T-Bank
- `sber` / `sberbank` → CardTransfer с deeplink Сбербанк
- `vtb` → CardTransfer с deeplink ВТБ
- `alfa` / `alfabank` → CardTransfer с deeplink Альфа-Банк
- Другой банк → Стандартная форма MainForm

### Проверка на ecom

Если `methodCode = ecom` и статус `in_check` → показывается `EnterCodePage` (ввод OTP-кода для 3DS).

### Проверка на nspk

Если `methodCode = nspk` и есть `redirectUrl` → автоматический редирект на URL QR-кода СБП.

### Все остальные методы

Показывается `MainForm` — универсальная форма с:

- Реквизитами (номер карты, телефон, счёт)
- QR-кодом (если доступен)
- Таймером истечения
- Кнопкой загрузки чека
- Кнопкой deeplink (если определён банк)

---

## 7. Таблица method_code → UI

| method_code | Название | Валюта | Тип KB | Что видит плательщик |
|---|---|---|---|---|
| `sbp` | СБП | RUB | MainForm | Номер телефона + QR-код + deeplink в банк |
| `c2c` | Card-to-Card | RUB | MainForm | Номер карты + имя получателя |
| `nspk` | НСПК QR | RUB | Redirect | Автоматический редирект на QR-ссылку СБП |
| `spay` | Sberbank Pay | RUB | MainForm | Реквизиты + deeplink Сбербанк |
| `tpay` | Tinkoff Pay | RUB | MainForm | Реквизиты + deeplink T-Bank |
| `vpay` | VTB Pay | RUB | MainForm | Реквизиты + deeplink ВТБ |
| `apay` | Alfa Pay | RUB | MainForm | Реквизиты + deeplink Альфа |
| `ecom` | E-commerce | RUB | CardForm + OTP | Форма ввода карты → 3DS код |
| `card_ru` | Карта РФ | RUB | CardTransfer / MainForm | Зависит от `senderBank` (deeplinks / реквизиты) |
| `crypto` | Криптовалюта | USDT/BTC/… | MainForm | Адрес кошелька + сумма в крипте |
| `c2cuz` | P2P Узбекистан | UZS | MainForm | Номер карты Humo/Uzcard |
| `c2ckz` | P2P Казахстан | KZT | MainForm | Номер карты |
| `c2caz` | P2P Азербайджан | AZN | MainForm + CardForm | Форма ввода карты |
| `q_ecom` | Ecom Азербайджан | AZN | CardForm | Форма ввода карты (вариант c2caz) |
| `humo` | HUMO | UZS | MainForm | Реквизиты HUMO |
| `uzcard` | UZCARD | UZS | MainForm | Реквизиты UZCARD |
| `upi` | UPI | INR | MainForm + UTR | QR/VPA + поле ввода UTR |
| `m2ctj` | Мобильные TJ | TJS | MainForm | Номер телефона |
| `m2ntj` | Мобильные TJ | TJS | MainForm | Номер телефона |
| `m2tbank` | Трансгран T-Bank | RUB | CardTransfer | Deeplink T-Bank + инструкция |
| `m2sber` | Трансгран Сбер | RUB | CardTransfer | Deeplink Сбербанк + инструкция |
| `m2vtb` | Трансгран ВТБ | RUB | CardTransfer | Deeplink ВТБ + инструкция |
| `abh` | Нерезидент | Разные | MainForm | Реквизиты карты |

---

## 8. Брендирование и кастомизация

### Выбор стиля

Стиль определяется переменной `VITE_STYLE`. Каждый стиль задаёт полный набор цветов и параметров оформления.

Доступные стили (32): `default`, `aggrepay`, `asgard`, `pipay`, `pulsar`, `sol`, `starda`, `kent`, `legzo`, `fresh`, `drip`, `gizbo`, `gtx`, `irwin`, `izzi`, `jet`, `cat`, `lex`, `mers`, `monro`, `volna`, `vires`, `tiktak`, `tappypay`, `arkada`, `clickconnect`, `daddy`, `gama`, `1go`, `r7`, `roxcasino`, `pspcatalog` и другие.

### Что настраивается в стиле

Каждый стиль определяет:

- **Фоны:** светлый и тёмный режим (`lightMainBackground`, `darkMainBackground`)
- **Текст:** основной цвет, второстепенный
- **Акцент:** основной цвет кнопок и ссылок (+ hover и active состояния)
- **Ошибки:** красный цвет для ошибок и предупреждений
- **Отступы:** `large` (32px), `medium` (24px), `primary` (16px), `small` (8px)
- **Скругления:** радиусы для карточек и кнопок

### Тёмная/светлая тема

Flow автоматически определяет системную тему пользователя через `prefers-color-scheme: dark` и переключает между светлой и тёмной палитрой. Плательщик также может переключить тему вручную.

### Логотип мерчанта

Логотип загружается из поля `logo_url` в данных заказа (задаётся в настройках мерчанта в Admin Panel). Отображение контролируется переменной `VITE_SHOW_LOGO`.

### Иконки методов оплаты

Иконки платёжных методов хранятся в `/public/assets/methods/` и подгружаются автоматически по названию метода.

---

## 9. Локализация (языки)

Flow поддерживает **8 языков**:

| Код | Язык |
|---|---|
| `ru` | Русский |
| `en` | English |
| `es` | Español |
| `tr` | Türkçe |
| `ko` | 한국어 |
| `kz` | Қазақша |
| `az` | Azərbaycan |
| `uz` | O'zbekcha |

Язык определяется автоматически по настройкам браузера плательщика. На платёжной странице есть переключатель языков (`LanguageSwitcher`).

Файлы переводов расположены в `/public/locales/{lng}/translation.json`.

---

## 10. WebSocket и обновления в реальном времени

Flow поддерживает два способа получения обновлений о статусе платежа:

### WebSocket (рекомендуемый)

Если настроены `VITE_PUSHER_*` переменные, Flow подключается к Soketi (Pusher-совместимый WebSocket-сервер):

- **Канал:** `payments.{orderId}`
- **Событие:** `.PaymentUpdated`

При получении события Flow мгновенно обновляет UI и показывает актуальный статус.

### HTTP Polling (fallback)

Если WebSocket недоступен или соединение разорвалось, Flow переключается на polling:

- `GET /{orderId}` каждые **3 секунды**
- Polling останавливается при достижении финального статуса (finished, canceled, expired, failed)

> **Рекомендация:** Всегда настраивайте WebSocket (Soketi). Polling создаёт дополнительную нагрузку на API и имеет задержку до 3 секунд.

---

## 11. Жизненный цикл платежа во Flow

```
1. Мерчант → POST /api/v2/payments → Aggregator создаёт Payment
   ↓
2. Ответ содержит url: "{flow_url}/{order_id}"
   ↓
3. Мерчант редиректит плательщика на этот URL
   ↓
4. Flow загружает: GET /{orderId} → получает orderData
   ↓
5a. Если метод определён → показать реквизиты (MainForm / CardTransfer / EnterCode)
5b. Если метод не определён → показать PaymentMethodSelector
   ↓
6. Плательщик выбирает метод → POST /{orderId}/payments → создаётся Invoice
   ↓
7. Показываются реквизиты: номер карты, телефон, QR-код, deeplink
   ↓
8. Плательщик оплачивает через свой банк
   ↓
9. (Опционально) Плательщик загружает чек → PATCH /{orderId}/status
   ↓
10. Плательщик нажимает «Я оплатил» → PATCH /{orderId}/status {status: "in_check"}
   ↓
11. Flow ожидает подтверждения (WebSocket / Polling)
   ↓
12. Провайдер присылает callback → Aggregator обновляет статус
   ↓
13. Flow получает обновление:
    • finished → Экран успеха → редирект на success_uri
    • canceled/expired/failed → Экран ошибки → редирект на fail_uri
    • dispute → Форма апелляции
```

### Статусы платежа

| Статус | Описание | Что видит плательщик |
|---|---|---|
| `created` | Платёж создан, метод не выбран | Список методов оплаты |
| `pending` | Ожидание оплаты | Реквизиты + таймер + кнопка «Я оплатил» |
| `in_check` | Проверка оплаты | Спиннер / страница ожидания |
| `finished` | Успешно завершён | Зелёный экран успеха → редирект |
| `canceled` | Отменён | Экран отмены → редирект |
| `expired` | Истёк срок оплаты | «Время истекло» → редирект |
| `failed` | Ошибка | Красный экран ошибки → редирект |
| `dispute` | Диспут / апелляция | Форма отправки жалобы (email + файл) |

---

## 12. API-эндпоинты Flow

Все запросы идут к `VITE_LARAVEL_APP_HOST` (например, `https://api.wl.com/api/v2/orders`).

| Метод | Эндпоинт | Описание |
|---|---|---|
| `GET` | `/{orderId}` | Получить данные заказа (сумма, реквизиты, статус, метод) |
| `GET` | `/{orderId}/methods` | Получить доступные методы оплаты |
| `GET` | `/{orderId}/banks` | Получить список банков для метода |
| `POST` | `/{orderId}/payments` | Создать инвойс для выбранного метода (body: `{method: "sbp"}`) |
| `PATCH` | `/{orderId}/status` | Обновить статус: подтвердить оплату, отправить OTP, загрузить чек |

### Структура ответа GET /{orderId}

```json
{
  "success": true,
  "data": {
    "order_id": "abc123",
    "external_id": "merchant-order-456",
    "amount": "1000.00",
    "currency": "RUB",
    "method": "sbp",
    "methodCode": "sbp",
    "number": "+7 999 ***-**-12",
    "name": "Иванов И.И.",
    "bankCode": "sber",
    "bankName": "Сбербанк",
    "state": "pending",
    "success_uri": "https://merchant.com/success",
    "fail_uri": "https://merchant.com/fail",
    "redirectUrl": null,
    "logo_url": "https://merchant.com/logo.png",
    "check_required": true,
    "senderBank": null,
    "senderBankName": null
  }
}
```

---

## 13. Настройка мерчанта для работы с Flow

Для того чтобы мерчант мог использовать Flow, в Admin Panel необходимо настроить:

### Обязательные поля мерчанта

1. **flow_url** — базовый URL, где развёрнут Flow для этого WL (например, `https://pay.your-wl.com`). Агрегатор формирует ссылку как `{flow_url}/{order_id}`.

2. **callback_uri** — URL для webhook-уведомлений о смене статуса платежа.

3. **success_uri** — URL для редиректа плательщика после успешной оплаты.

4. **fail_uri** — URL для редиректа при неуспешной оплате.

### Настройка методов

В разделе Merchant → Methods необходимо подключить нужные платёжные методы. Для каждого метода настраивается:

- **Комиссия** (commission %)
- **cascade_percent** — вес метода в каскаде (0 = отключён)
- **Лимиты** — минимальная и максимальная сумма платежа
- **Валюта**

### Настройка провайдеров

Для каждого метода должен быть подключён хотя бы один активный провайдер (Provider) с валидными credentials. Провайдер — это конкретный платёжный шлюз, обрабатывающий транзакции.

---

## 14. Deeplinks (открытие банковских приложений)

Flow автоматически генерирует deeplink-ссылки для открытия мобильных банковских приложений с предзаполненными данными:

| Банк | Поддержка | Метод |
|---|---|---|
| Сбербанк | ✅ | СБП по номеру телефона |
| T-Bank (Тинькофф) | ✅ | СБП по номеру телефона |
| ВТБ | ✅ | СБП по номеру телефона |
| Альфа-Банк | ✅ | СБП по номеру телефона |
| Mercado Pago | ✅ | Латинская Америка |

Deeplinks работают только на мобильных устройствах. На десктопе показываются стандартные реквизиты.

Банк отправителя определяется из поля `senderBank` в данных заказа (если провайдер передаёт эту информацию).

---

## 15. Troubleshooting

### Flow не загружается (белый экран)

- Проверьте, что `VITE_LARAVEL_APP_HOST` указывает на правильный URL API
- Проверьте CORS: API агрегатора должен разрешать запросы с домена Flow
- Проверьте healthcheck: `GET /healthz` должен возвращать `200 ok`
- Проверьте Nginx-конфиг: SPA-роутинг (`try_files $uri $uri/ /index.html`)

### Платёж не создаётся (ошибка при выборе метода)

| Код ошибки | Причина | Решение |
|---|---|---|
| `40003` | Метод не подключён у мерчанта | Подключить метод в Admin Panel → Merchant → Methods |
| `40002` | Нет свободных реквизитов | Проверить наличие активных трейдеров/реквизитов в Trade |
| `40004` | Недостаточно средств (для payout) | Пополнить баланс мерчанта |
| `40403` | Нет доступных методов | Настроить cascade_percent > 0 для методов |
| `40901` | Дупликат orderId | Мерчант отправляет повторный запрос с тем же orderId |

### Статус не обновляется

- Проверьте WebSocket: `VITE_PUSHER_*` переменные должны быть корректны
- Если WebSocket не настроен — Flow использует polling (3 сек); проверьте доступность API
- Проверьте callback URL провайдера — провайдер должен слать callback на `{aggregator}/api/callback/{method_code}`

### Неправильный KB / форма оплаты

- Убедитесь, что `method.code` корректно возвращается API агрегатора
- Проверьте, какой method_code приходит в ответе `GET /{orderId}`
- method_code определяет тип кэшбокса (см. раздел 7)

### Стили / брендирование не применяются

- `VITE_STYLE` подставляется на этапе **сборки** — нужна пересборка Docker-образа
- Проверьте, что стиль есть в списке доступных (раздел 8)
- Очистите кэш браузера (статика кэшируется 30 дней)

### Deeplinks не работают

- Deeplinks работают только на **мобильных устройствах**
- Банковское приложение должно быть установлено на телефоне плательщика
- Проверьте, что `senderBank` корректно определяется в данных заказа

---
