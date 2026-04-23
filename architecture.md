# PSP Platform Architecture

> **Связанная документация:**
>
> - [🔄 Trade Service](./ai/trade-service.md) — P2P процессинг
> - [📡 API Reference (Postman)](https://documenter.getpostman.com/view/13931884/2sAYQdipUu) — API документация
> - [📖 Glossary](./ai/glossary.md) — Глоссарий терминов

---

## 🏗️ Обзор сервисов


| Сервис                  | Стек                | Порт   | Описание                                                                         |
| ----------------------- | ------------------- | ------ | -------------------------------------------------------------------------------- |
| **Aggregator**          | Laravel 12/PHP 8.4  | 80/443 | Core API для мерчантов, cascade к провайдерам                                    |
| **Admin Panel**         | React 19/TypeScript | 80/443 | Новая админ-панель агрегатора (SPA, взаимодействует с Aggregator через REST API) |
| **Agradmin** *(legacy)* | Yii2/PHP            | 80/443 | Старая админ-панель (заменяется на Admin Panel)                                  |
| **Trade**               | Yii2/PHP            | 80/443 | P2P процессинг, управление трейдерами и реквизитами                              |
| **Flow**                | React/TypeScript    | 80/443 | Платёжный UI для плательщиков                                                    |
| **Merchant Panel**      | React 19/TypeScript | 80/443 | Кабинет мерчанта (SPA)                                                           |
| **Rate Service**        | FastAPI/Python      | 8080   | Курсы валют из бирж (Binance, Bybit, etc.)                                       |
| **Support Service**     | FastAPI/Python      | 8000   | Тикеты, диспуты, Telegram интеграция                                             |
| **Bank Detection**      | —                   | —      | Определение банка по номеру карты (BIN)                                          |


---

## 📊 Архитектурная диаграмма

```mermaid
flowchart TB
    subgraph Internet["🌐 Internet"]
        Merchants["🏪 Мерчанты<br/>(API интеграция)"]
        Payers["👤 Плательщики<br/>(Web UI)"]
        Admins["⚙️ Админы<br/>(Web UI)"]
        Traders["📱 Трейдеры<br/>(Telegram Bot)"]
    end

    subgraph LoadBalancer["⚖️ Load Balancer / nginx"]
        LB["nginx / HAProxy"]
    end

    subgraph CoreServices["🔷 Core Services"]
        subgraph AggregatorCluster["Aggregator (Laravel)"]
            direction TB
            AGG1["Aggregator #1"]
            AGG2["Aggregator #2"]
            AGG_Workers["Horizon Workers"]
        end
        
        subgraph TradeCluster["Trade (Yii2)"]
            direction TB
            TRADE["Trade API"]
            TRADE_Workers["Queue Workers"]
            TRADE_TG["Telegram Bot"]
        end

        Flow["Flow<br/>(React SPA)"]
    end

    subgraph SharedServices["📡 Shared Services"]
        Rate["Rate Service<br/>(FastAPI)"]
        Support["Support Service<br/>(FastAPI)"]
        BankDetect["Bank Detection"]
    end

    subgraph Providers["🔌 Провайдеры"]
        TradeProvider["Trade<br/>(внутренний)"]
        ExtProvider1["Provider 1"]
        ExtProvider2["Provider 2"]
        ExtProviderN["Provider N..."]
    end

    subgraph Storage["💾 Data Layer"]
        subgraph PostgreSQL["PostgreSQL + Patroni"]
            PG_Primary["Primary"]
            PG_Replica["Replica"]
        end
        Redis["Redis<br/>(Cache, Queue, Sessions)"]
        S3["S3 / MinIO<br/>(Files, Checks)"]
    end

    subgraph Observability["📊 Observability"]
        OpenSearch["OpenSearch"]
        Grafana["Grafana"]
        Filebeat["Filebeat"]
    end

    subgraph ExternalAPIs["🌍 External APIs"]
        Binance["Binance P2P"]
        Bybit["Bybit P2P"]
        Banks["Bank APIs"]
        TelegramAPI["Telegram API"]
    end

    %% Internet → Load Balancer
    Merchants -->|"API v1/v2"| LB
    Payers -->|"HTTPS"| LB
    Admins -->|"HTTPS"| LB
    Traders -->|"Telegram"| TRADE_TG

    %% Load Balancer → Services
    LB --> AGG1 & AGG2
    LB --> TRADE
    LB --> Flow

    %% Aggregator → Providers (Cascade)
    AGG1 & AGG2 -->|"Cascade"| TradeProvider
    AGG1 & AGG2 -->|"Cascade"| ExtProvider1
    AGG1 & AGG2 -->|"Cascade"| ExtProvider2
    AGG1 & AGG2 -->|"Cascade"| ExtProviderN
    
    TradeProvider -.->|"="| TRADE

    %% Core → Shared Services
    AGG1 & AGG2 --> Rate
    TRADE --> Rate
    AGG1 & AGG2 --> Support
    AGG1 & AGG2 --> BankDetect

    %% Flow → Aggregator
    Flow -->|"API"| AGG1

    %% Shared Services → External
    Rate --> Binance & Bybit
    TRADE_TG --> TelegramAPI
    Support --> TelegramAPI

    %% Storage connections
    AGG1 & AGG2 --> PG_Primary
    TRADE --> PG_Primary
    PG_Primary -.->|"Streaming Replication"| PG_Replica

    AGG1 & AGG2 --> Redis
    TRADE --> Redis

    AGG1 & AGG2 --> S3
    TRADE --> S3

    %% Workers
    AGG_Workers --> Redis
    TRADE_Workers --> Redis

    %% Observability
    AGG1 & AGG2 -->|"JSON Logs"| Filebeat
    TRADE -->|"JSON Logs"| Filebeat
    Filebeat --> OpenSearch
    OpenSearch --> Grafana

    %% Styling
    classDef core fill:#4F46E5,stroke:#312E81,color:#fff
    classDef shared fill:#0891B2,stroke:#164E63,color:#fff
    classDef storage fill:#059669,stroke:#064E3B,color:#fff
    classDef obs fill:#D97706,stroke:#78350F,color:#fff

    class AGG1,AGG2,TRADE,Flow core
    class Rate,Support,BankDetect shared
    class PG_Primary,PG_Replica,Redis,S3 storage
    class OpenSearch,Grafana,Filebeat obs
```



---

## 📦 Детализация сервисов

### 1. Aggregator (Laravel)

**Назначение:** Центральный API для мерчантов, бэкенд админки, cascade к провайдерам.

```
aggregator/
├── app/
│   ├── Gateway/           # Провайдеры (полный список в docs/gateways.csv)
│   │   └── Universal/     # Gateway Builder (UniversalGateway, TemplateEngine, SignatureGenerator)
│   ├── Http/Controllers/
│   │   ├── Api/           # Merchant API v1/v2
│   │   └── Merchants/     # Merchant cabinet
│   ├── Jobs/              # Async tasks (callbacks, checks)
│   ├── Services/          # Business logic
│   └── Models/            # Eloquent models
├── routes/
│   ├── api.php            # /api/* routes
│   └── web.php            # Callbacks, admin
└── config/
    └── gateway.php        # Provider configurations
```

**API Endpoints:**


| Endpoint                   | Method | Описание                 |
| -------------------------- | ------ | ------------------------ |
| `/api/v2/payments`         | POST   | Создание payin           |
| `/api/v2/payouts`          | POST   | Создание payout          |
| `/api/v2/status`           | POST   | Статус платежа           |
| `/api/v2/balance`          | GET    | Баланс мерчанта          |
| `/api/callback/{provider}` | POST   | Callbacks от провайдеров |


**Зависимости:**

- PostgreSQL (основная БД)
- Redis (cache, queue, sessions)
- Rate Service (курсы)
- S3 (чеки, файлы)

---

### 2. Trade (Yii2)

**Назначение:** P2P процессинг — управление трейдерами, реквизитами, автоматическая обработка SMS/Push.

```
trade/
├── backend/
│   ├── controllers/
│   │   ├── WebhookController.php  # Merchant API
│   │   └── ApiController.php      # Trader API
│   ├── models/
│   │   ├── Order.php              # Заявки
│   │   ├── Requisite.php          # Реквизиты
│   │   └── Trader.php             # Трейдеры
│   └── services/
│       ├── SmsParser/             # Парсинг SMS по банкам
│       └── OrderDistributor.php   # Распределение заявок
├── frontend/                       # Trader UI (Yii2 Views)
└── console/
    └── controllers/               # Cron jobs
```

**API Endpoints (WebhookController):**


| Endpoint                    | Описание                        |
| --------------------------- | ------------------------------- |
| `/webhook/set-send-deal`    | Создание payin заявки           |
| `/webhook/set-payout-deal`  | Создание payout заявки          |
| `/webhook/get-status-deal`  | Статус заявки                   |
| `/webhook/set-sms-{bank}`   | SMS webhook от банков           |
| `/webhook/check-requisites` | Проверка доступности реквизитов |


**Trader API:**


| Endpoint         | Описание               |
| ---------------- | ---------------------- |
| `/api/orders`    | Список заявок трейдера |
| `/api/devices`   | Устройства трейдера    |
| `/api/device-qr` | QR для SMS Forwarder   |


**Особенности:**

- Не продаётся без Aggregator
- Нет мерчантов — работает только как провайдер
- Telegram Bot для трейдеров
- Автоматический парсинг SMS (Sber, Tinkoff, etc.)

---

### 3. Flow (React)

**Назначение:** Платёжный UI для плательщиков.

```
flow/
├── src/
│   ├── components/
│   │   ├── MainForm/          # Основная форма
│   │   ├── PaymentMethodSelector/  # Выбор метода
│   │   ├── Details/           # Реквизиты для оплаты
│   │   ├── ImageUploader/     # Загрузка чека
│   │   └── Timer/             # Таймер оплаты
│   ├── assets/brand/          # Брендинг (30+ брендов)
│   ├── services/api.ts        # API клиент
│   └── store/                 # Redux state
└── public/
```

**Роуты:**


| Route       | Описание                     |
| ----------- | ---------------------------- |
| `/:orderId` | Страница оплаты              |
| `/`         | Тестовое создание транзакции |


**Брендинг:** 30+ white-label брендов (logo-light.svg, logo-dark.svg, favicon.ico)

---

### 4. Frontend (React 19 Monorepo)

**Назначение:** Новая админ-панель и кабинет мерчанта. Монорепозиторий с shared пакетами.

**Репозиторий:** `highpay/frontend`

```
frontend/
├── admin-panel/           # Админ-панель агрегатора (SPA)
│   └── src/
│       ├── pages/         # ~29 страниц (Dashboard, Merchants, Methods, Gateway Builder, etc.)
│       ├── components/
│       │   ├── merchants/        # Табы карточки мерчанта (General, Methods, Balance, etc.)
│       │   ├── gateway-builder/  # AI Chat, API Tester, Monaco Editor, Import, Presets
│       │   ├── roles/            # Матрица прав (RBAC)
│       │   └── ui/               # UI-компоненты
│       ├── clients/       # White-label конфиги (aggrepay, asgard, default)
│       └── i18n/          # ru, en, ko
├── merchant-panel/        # Кабинет мерчанта (SPA)
│   └── src/
│       ├── pages/         # Payments, Balance, Reports, Settings, etc.
│       ├── components/
│       │   └── modals/    # Payin, Payout, Withdrawal, Convert, Insurance
│       └── stores/        # Zustand (auth)
├── packages/
│   ├── lib/               # Shared: API client, types, utils, hooks
│   └── ui/                # Shared UI: badge, button, input, modal, sidebar, etc.
└── package.json           # npm workspaces
```

**Стек:** React 19, TypeScript 5.9, TanStack Query + Table, Zustand, Tailwind CSS 4, Vite 6

**Статус:** В разработке. Заменяет agradmin (Yii2) и старый merchant cabinet.

**API:** Использует REST API Aggregator (Sanctum auth, RBAC permissions).

---

### 5. Rate Service (FastAPI)

**Назначение:** Агрегация курсов с бирж для всех сервисов.

```
rate/
├── main.py              # FastAPI app
├── exchanges.py         # Binance, Bybit, HTX adapters
├── tasks.py             # Background polling
├── config.py            # Methods configuration
└── authentication.py    # Basic auth
```

**API:**


| Endpoint               | Описание                |
| ---------------------- | ----------------------- |
| `GET /rate`            | Получить курс           |
| `PATCH /rate`          | Установить курс вручную |
| `GET /methods`         | Список методов          |
| `POST /update_methods` | Обновить конфигурацию   |


**Поддерживаемые валюты:**
RUB, UZS, KZT, KGS, AZN, TJS, TRY, GEL, INR, ARS, EUR

**Биржи:** Binance, Bybit, HTX, Grinex, Rapira

---

### 6. Support Service (FastAPI)

**Назначение:** Система тикетов и диспутов с Telegram интеграцией.

```
support-service/
├── app/
│   ├── routers/
│   │   ├── webhook.py      # Incoming webhooks
│   │   ├── orders.py       # Order actions
│   │   ├── messages.py     # Message history
│   │   └── disputes.py     # Dispute management
│   ├── services/
│   │   ├── telegram.py     # Telegram client
│   │   └── alerts.py       # Alert notifications
│   └── models.py           # SQLAlchemy models
└── main.py
```

**API:**


| Endpoint                      | Описание               |
| ----------------------------- | ---------------------- |
| `/webhook/action`             | Действия из Aggregator |
| `/orders/{order_id}/messages` | История сообщений      |
| `/disputes`                   | Управление диспутами   |


---

---

## 🔄 Потоки данных

### Payin Flow (Aggregator → Trade)

```mermaid
sequenceDiagram
    participant M as Merchant
    participant A as Aggregator
    participant T as Trade
    participant Tr as Trader
    participant P as Payer

    M->>A: POST /api/v2/payments
    A->>A: Cascade selection
    A->>T: POST /webhook/set-send-deal
    T->>T: Find available requisite
    T->>Tr: Telegram notification
    T-->>A: {requisites, transaction_id}
    A-->>M: {payment_url, order_id}
    
    M->>P: Redirect to Flow
    P->>A: GET /api/v2/orders/{id}
    A-->>P: Payment details
    
    P->>P: Pay via bank
    Tr->>T: SMS received (auto-parsed)
    T->>A: POST /callback/trade
    A->>M: POST callback_uri
```



### Payout Flow (Aggregator → Trade)

```mermaid
sequenceDiagram
    participant M as Merchant
    participant A as Aggregator
    participant T as Trade
    participant Tr as Trader

    M->>A: POST /api/v2/payouts
    A->>T: POST /webhook/set-payout-deal
    T->>T: Assign to trader
    T->>Tr: Telegram notification
    Tr->>Tr: Make bank transfer
    Tr->>T: Upload check
    T->>A: POST /callback/trade (success)
    A->>M: POST callback_uri
```



---

## 🏢 Multi-tenant Architecture

```mermaid
flowchart TB
    subgraph WL1["ВЛ #1 (Brand A)"]
        WL1_AGG["Aggregator x2"]
        WL1_TRADE["Trade"]
        WL1_FLOW["Flow"]
        WL1_PG["PostgreSQL + Patroni"]
        WL1_REDIS["Redis"]
    end

    subgraph WL2["ВЛ #2 (Brand B)"]
        WL2_AGG["Aggregator x2"]
        WL2_TRADE["Trade"]
        WL2_FLOW["Flow"]
        WL2_PG["PostgreSQL + Patroni"]
        WL2_REDIS["Redis"]
    end

    subgraph Shared["🌐 Shared Infrastructure"]
        Rate["Rate Service"]
        OpenSearch["OpenSearch Cluster"]
        Grafana["Grafana"]
        S3["S3 / MinIO"]
    end

    WL1_AGG & WL2_AGG --> Rate
    WL1_TRADE & WL2_TRADE --> Rate

    WL1_AGG & WL1_TRADE -->|"Logs"| OpenSearch
    WL2_AGG & WL2_TRADE -->|"Logs"| OpenSearch

    OpenSearch --> Grafana
    
    WL1_AGG & WL2_AGG --> S3
    WL1_TRADE & WL2_TRADE --> S3
```



**Изолировано для каждого ВЛ:**

- Aggregator (2 реплики)
- Trade
- Flow (с брендингом)
- PostgreSQL + Patroni
- Redis

**Shared между всеми ВЛ:**

- Rate Service (один инстанс)
- OpenSearch (разные индексы по ВЛ)
- Grafana
- S3/MinIO

---

## 📋 Зависимости между сервисами

```mermaid
graph LR
    subgraph Required["Обязательные"]
        AGG["Aggregator"]
        RATE["Rate Service"]
        PG["PostgreSQL"]
        REDIS["Redis"]
    end
    
    subgraph Optional["Опциональные"]
        TRADE["Trade"]
        SUPPORT["Support Service"]
        BANK["Bank Detection"]
        FLOW["Flow"]
    end
    
    subgraph Standalone["Автономные"]
        RATE2["Rate Service"]
    end

    AGG -->|"required"| RATE
    AGG -->|"required"| PG
    AGG -->|"required"| REDIS
    
    AGG -.->|"optional"| TRADE
    AGG -.->|"optional"| SUPPORT
    AGG -.->|"optional"| BANK
    AGG -.->|"optional"| FLOW
    
    TRADE -->|"required"| AGG
    TRADE -->|"required"| RATE
```




| Сервис              | Может работать без                   |
| ------------------- | ------------------------------------ |
| **Aggregator**      | Trade, Support, Bank Detection, Flow |
| **Trade**           | — (требует Aggregator)               |
| **Rate Service**    | Всё (полностью автономный)           |
| **Support Service** | — (требует Aggregator)               |
| **Flow**            | — (требует Aggregator API)           |


---

## 🔧 Технологический стек


| Компонент                                        | Технология                           | Версия        |
| ------------------------------------------------ | ------------------------------------ | ------------- |
| **Aggregator**                                   | Laravel                              | 12 (PHP 8.4)  |
| **Trade**                                        | Yii2                                 | 2.0 (PHP 8.x) |
| **Flow**                                         | React + TypeScript + Vite            | 18+           |
| **Admin Panel**                                  | React 19 + TanStack + Zustand + Vite | —             |
| **Merchant Panel**                               | React 19 + TanStack + Zustand + Vite | —             |
| **Rate Service**                                 | FastAPI + Redis                      | 0.100+        |
| **Support Service**                              | FastAPI + SQLAlchemy                 | 0.100+        |
| **Agradmin** (legacy, заменяется на Admin Panel) | Yii2                                 | 2.0           |
| **Database**                                     | PostgreSQL + Patroni                 | 15+           |
| **Cache/Queue**                                  | Redis                                | 7+            |
| **Search/Logs**                                  | OpenSearch                           | 2.x           |
| **Metrics**                                      | Prometheus + Grafana                 | —             |
| **Log Shipping**                                 | Filebeat                             | 8.x           |
| **Storage**                                      | S3 / MinIO                           | —             |
| **Container**                                    | Docker                               | —             |


---

## 📡 Порты и эндпоинты


| Сервис          | Порт   | Health Check         |
| --------------- | ------ | -------------------- |
| Aggregator      | 80/443 | `/api/health`        |
| Trade           | 80/443 | `/site/health`       |
| Flow            | 80/443 | Static SPA           |
| Rate Service    | 8080   | `/health`            |
| Support Service | 8000   | `/docs` (Swagger UI) |
| PostgreSQL      | 5432   | —                    |
| Redis           | 6379   | —                    |
| OpenSearch      | 9200   | `/_cluster/health`   |
| Grafana         | 3000   | `/api/health`        |


---

## 🔐 Безопасность


| Уровень           | Механизм                            |
| ----------------- | ----------------------------------- |
| **API Auth**      | Bearer Token (Sanctum)              |
| **Signature**     | HMAC-SHA256 для платежей            |
| **IP Whitelist**  | Для merchant API                    |
| **2FA**           | Google Authenticator (Trade, Admin) |
| **Internal Auth** | Basic Auth (Rate, Support)          |
| **Secrets**       | Encrypted в .env                    |


