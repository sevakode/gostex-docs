# Flow — Платёжная страница (legacy)

> ⚠️ **Это старый Flow (`flow/`, React 18).** Продолжает дорабатываться, но новая документация не ведётся.
> **Новый Flow:** `gostex-pay/` — документация будет писаться с нуля.

> **Стек:** React 18 + TypeScript + Vite + Tailwind CSS
> **Репозиторий:** `flow/`
> **Последнее обновление:** 2026-04-15

---

## Содержание

1. [Обзор](#1-обзор)
2. [Архитектура](#2-архитектура)
3. [Деплой и настройка](#3-деплой-и-настройка)
4. [Переменные окружения](#4-переменные-окружения)
5. [Кэшбокс: method_code → UI](#5-кэшбокс-method_code--ui)
6. [Жизненный цикл платежа](#6-жизненный-цикл-платежа)
7. [API Endpoints](#7-api-endpoints)
8. [Брендирование](#8-брендирование)
9. [Локализация](#9-локализация)
10. [WebSocket и обновления](#10-websocket-и-обновления)
11. [Deeplinks](#11-deeplinks)
12. [Компоненты](#12-компоненты)
13. [Настройка мерчанта](#13-настройка-мерчанта)
14. [Troubleshooting](#14-troubleshooting)

---

## 1. Обзор

Flow — SPA-приложение, которое показывает плательщику форму оплаты. Мерчант перенаправляет клиента на URL с ID заказа, клиент видит реквизиты, оплачивает, загружает чек.

| Возможность | Описание |
|-------------|----------|
| Мульти-метод | Выбор способа оплаты (СБП, C2C, NSPK, крипто и др.) |
| Deeplinks | Автоматическое открытие банковских приложений |
| White-label | 32+ готовых бренда (логотип, цвета, стиль) |
| Мультиязычность | 8 языков: ru, en, es, tr, ko, kz, az, uz |
| Загрузка чеков | Фото/скриншот оплаты |
| OTP | Поддержка 3DS / одноразовых кодов |
| Диспуты | Форма апелляции с email + файлом |
| Real-time | WebSocket (Soketi/Pusher) + HTTP polling как fallback |

---

## 2. Архитектура

Каждый WL разворачивает свой экземпляр Flow. Flow **не имеет** собственного бэкенда — все данные через REST API агрегатора.

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
```

---

## 3. Деплой и настройка

### Docker (рекомендуемый)

> **Важно:** Все `VITE_*` переменные подставляются **на этапе сборки** (build-time). При изменении любого параметра нужна **пересборка образа**.

```bash
docker build -t flow \
  --build-arg VITE_LARAVEL_APP_HOST=https://api.your-wl.com/api/v2/orders \
  --build-arg VITE_STYLE=aggrepay \
  --build-arg VITE_SHOW_LOGO=show \
  --build-arg VITE_SHOW_RECIPIENT=show \
  --build-arg VITE_SHOW_CHECK=show \
  --build-arg VITE_SHOW_TROUBLE_COMPONENT=show \
  .

docker run -d -p 8080:80 --name flow flow
```

### Nginx (SPA-роутинг)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /healthz {
        return 200 'ok';
    }

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

## 4. Переменные окружения

| Переменная | Обязательная | Описание | Пример |
|---|---|---|---|
| `VITE_LARAVEL_APP_HOST` | ✅ | Базовый URL API агрегатора | `https://api.wl.com/api/v2/orders` |
| `VITE_STYLE` | ✅ | Название бренда/стиля | `default`, `aggrepay`, `asgard` |
| `VITE_SHOW_LOGO` | — | Показать логотип мерчанта | `show` / `hide` |
| `VITE_SHOW_RECIPIENT` | — | Показать имя получателя | `show` / `hide` |
| `VITE_SHOW_CHECK` | — | Кнопка загрузки чека | `show` / `hide` |
| `VITE_SHOW_TROUBLE_COMPONENT` | — | Блок «Проблемы с оплатой?» | `show` / `hide` |
| `VITE_SHOW_VIDEO_INSTRUCTION` | — | Кнопка видеоинструкции | `show` / `hide` |
| `VITE_PUSHER_APP_KEY` | — | Ключ WebSocket | `app-key` |
| `VITE_PUSHER_HOST` | — | Хост WebSocket-сервера | `soketi.wl.com` |
| `VITE_PUSHER_PORT` | — | Порт WebSocket | `443` |
| `VITE_PUSHER_SCHEME` | — | Протокол | `https` |
| `VITE_PUSHER_CLUSTER` | — | Кластер Pusher | `mt1` |
| `VITE_GA_ID` | — | Google Analytics ID | `G-XXXXXXXXXX` |
| `VITE_YM_ID` | — | Yandex.Metrika ID | `12345678` |

**Минимум для запуска:**
```env
VITE_LARAVEL_APP_HOST=https://api.your-wl.com/api/v2/orders
VITE_STYLE=default
```

---

## 5. Кэшбокс: method_code → UI

Flow определяет тип формы по `method_code` из API-ответа.

| method_code | Название | Валюта | UI | Что видит плательщик |
|---|---|---|---|---|
| `sbp` | СБП | RUB | MainForm | Номер телефона + QR + deeplink |
| `c2c` | Card-to-Card | RUB | MainForm | Номер карты + имя получателя |
| `nspk` | НСПК QR | RUB | Redirect | Автоматический редирект на QR СБП |
| `spay` | Sberbank Pay | RUB | MainForm | Реквизиты + deeplink Сбер |
| `tpay` | Tinkoff Pay | RUB | MainForm | Реквизиты + deeplink T-Bank |
| `vpay` | VTB Pay | RUB | MainForm | Реквизиты + deeplink ВТБ |
| `apay` | Alfa Pay | RUB | MainForm | Реквизиты + deeplink Альфа |
| `ecom` | E-commerce | RUB | CardForm + OTP | Форма ввода карты → 3DS код |
| `card_ru` | Карта РФ | RUB | CardTransfer / MainForm | Зависит от `senderBank` |
| `crypto` | Крипто | USDT/BTC | MainForm | Адрес кошелька + сумма |
| `c2cuz` | P2P Узбекистан | UZS | MainForm | Номер карты Humo/Uzcard |
| `c2ckz` | P2P Казахстан | KZT | MainForm | Номер карты |
| `c2caz` / `q_ecom` | P2P/Ecom AZ | AZN | CardForm | Форма ввода карты |
| `humo` | HUMO | UZS | MainForm | Реквизиты HUMO |
| `uzcard` | UZCARD | UZS | MainForm | Реквизиты UZCARD |
| `upi` | UPI | INR | MainForm + UTR | QR/VPA + ввод UTR |
| `m2tbank` | Трансгран T-Bank | RUB | CardTransfer | Deeplink T-Bank + инструкция |
| `m2sber` | Трансгран Сбер | RUB | CardTransfer | Deeplink Сбер + инструкция |
| `m2vtb` | Трансгран ВТБ | RUB | CardTransfer | Deeplink ВТБ + инструкция |
| `abh` | Нерезидент | Разные | MainForm | Реквизиты карты |

**Правило m2\*:** если `methodCode` начинается с `m2` — это трансграничный перевод, показывается CardTransfer.

**Правило card_ru:** дополнительно проверяется `senderBank` для выбора deeplink (tbank/sber/vtb/alfa) или стандартного MainForm.

---

## 6. Жизненный цикл платежа

```
1. Мерчант → POST /api/v2/payments → Aggregator создаёт Payment
2. Ответ: { url: "{flow_url}/{order_id}" }
3. Мерчант редиректит плательщика
4. Flow: GET /{orderId} → orderData
5a. Метод определён → показать реквизиты
5b. Метод не определён → PaymentMethodSelector → POST /{orderId}/payments
6. Плательщик оплачивает
7. (Опц.) Загружает чек → PATCH /{orderId}/status
8. Нажимает «Я оплатил» → status: "in_check"
9. Flow ждёт (WebSocket / Polling)
10. Провайдер → callback → Aggregator обновляет статус
11. finished → успех → редирект на success_uri
    canceled/expired/failed → ошибка → редирект на fail_uri
```

### Статусы заказа

| Статус | Что видит плательщик |
|--------|----------------------|
| `created` | Список методов оплаты |
| `pending` | Реквизиты + таймер + «Я оплатил» |
| `in_check` | Спиннер / ожидание |
| `finished` | Зелёный экран → редирект |
| `canceled` | Экран отмены → редирект |
| `expired` | «Время истекло» → редирект |
| `failed` | Красный экран → редирект |
| `dispute` | Форма апелляции (email + файл) |

---

## 7. API Endpoints

Base URL: `VITE_LARAVEL_APP_HOST` (напр. `https://api.wl.com/api/v2/orders`)

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/{orderId}` | Данные заказа (сумма, реквизиты, статус, метод) |
| GET | `/{orderId}/methods` | Доступные методы оплаты |
| GET | `/{orderId}/banks` | Список банков для метода |
| POST | `/{orderId}/payments` | Создать инвойс (выбрать метод) |
| PATCH | `/{orderId}/status` | Обновить статус (подтвердить, OTP, чек) |

---

## 8. Брендирование

Стиль задаётся через `VITE_STYLE` (build-time). Каждый стиль определяет цвета, фон, акценты, скругления.

**Доступные стили (32):** `1go`, `aggrepay`, `arkada`, `asgard`, `cat`, `clickconnect`, `daddy`, `default`, `drip`, `fresh`, `gama`, `gizbo`, `gtx`, `irwin`, `izzi`, `jet`, `kent`, `legzo`, `lex`, `mers`, `monro`, `pipay`, `pspcatalog`, `pulsar`, `r7`, `roxcasino`, `sol`, `starda`, `tappypay`, `tiktak`, `vires`, `volna`

Логотип загружается из `logo_url` в данных заказа. Поддерживается тёмная/светлая тема (авто по `prefers-color-scheme` + ручное переключение).

---

## 9. Локализация

8 языков: `ru`, `en`, `es`, `tr`, `ko`, `kz`, `az`, `uz`

Язык определяется по настройкам браузера. Переключатель языков встроен в интерфейс. Файлы: `/public/locales/{lng}/translation.json`.

---

## 10. WebSocket и обновления

**WebSocket (рекомендуется):** Soketi/Pusher, канал `payments.{orderId}`, событие `.PaymentUpdated`. Требует `VITE_PUSHER_*`.

**HTTP Polling (fallback):** если WebSocket недоступен — `GET /{orderId}` каждые 3 секунды. Останавливается на финальном статусе.

> Всегда настраивайте WebSocket — polling создаёт нагрузку и задержку до 3 сек.

---

## 11. Deeplinks

Открывают банковское приложение с предзаполненными данными. Работают только на мобильных.

| Банк | Метод |
|------|-------|
| Сбербанк | СБП по номеру телефона |
| T-Bank (Тинькофф) | СБП по номеру телефона |
| ВТБ | СБП по номеру телефона |
| Альфа-Банк | СБП по номеру телефона |
| Mercado Pago | Латинская Америка |

Банк определяется из поля `senderBank` в данных заказа.

---

## 12. Компоненты

```
src/components/
├── MainForm/               # Главная логика — решает что показывать
├── PaymentMethodSelector/  # Выбор метода оплаты
├── PaymentDetails/         # Реквизиты
├── Details/                # Карта / телефон / QR
├── DeepLinkButton/         # Кнопка «Открыть в приложении»
├── ImageUploader/          # Загрузка чека
├── CardForm/               # Ввод данных карты (ECOM 3DS)
├── OtpForm/                # Ввод OTP
├── Timer/                  # Таймер до истечения
├── CancelPopup/            # Подтверждение отмены
├── TroubleComponent/       # «Возникли трудности?»
├── SupportSection/         # Форма апелляции
├── TransactionStatus/      # Финальный статус
├── LanguageSwitcher/       # Переключатель языка
└── ThemeSwitcher/          # Переключатель темы
```

---

## 13. Настройка мерчанта

В Admin Panel обязательно настроить:

1. **flow_url** — URL где развёрнут Flow (напр. `https://pay.your-wl.com`). Агрегатор формирует `{flow_url}/{order_id}`.
2. **callback_uri** — URL для webhook при смене статуса.
3. **success_uri** — редирект после успешной оплаты.
4. **fail_uri** — редирект при неуспехе.
5. **Methods** — подключить методы с `cascade_percent > 0`, лимитами, комиссией.
6. **Providers** — хотя бы один активный провайдер с валидными credentials.

---

## 14. Troubleshooting

### Белый экран / не загружается
- Проверьте `VITE_LARAVEL_APP_HOST`
- Проверьте CORS: API должен разрешать запросы с домена Flow
- Проверьте Nginx: `try_files $uri $uri/ /index.html`
- `GET /healthz` → должен вернуть `200 ok`

### Ошибки при создании платежа

| Код | Причина | Решение |
|-----|---------|---------|
| `40003` | Метод не подключён | Admin Panel → Merchant → Methods |
| `40002` | Нет свободных реквизитов | Проверить трейдеров/реквизиты в Trade |
| `40004` | Недостаточно средств (payout) | Пополнить баланс мерчанта |
| `40403` | Нет доступных методов | Настроить `cascade_percent > 0` |
| `40901` | Дубликат orderId | Мерчант повторно отправляет тот же orderId |

### Статус не обновляется
- Проверьте `VITE_PUSHER_*` переменные
- Если нет WebSocket — polling работает каждые 3 сек
- Callback провайдера: `{aggregator}/api/callback/{method_code}`

### Неправильная форма оплаты
- Проверьте `method.code` в ответе `GET /{orderId}`
- method_code определяет тип кэшбокса (см. раздел 5)

### Стили не применяются
- `VITE_STYLE` — build-time, нужна пересборка Docker-образа
- Очистите кэш браузера (статика кэшируется 30 дней)

### Deeplinks не работают
- Только на мобильных устройствах
- Банковское приложение должно быть установлено
- Проверьте `senderBank` в данных заказа
