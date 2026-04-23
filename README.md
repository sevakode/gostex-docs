# Документация платформы Gostex / AGGREpay

Документация платёжной платформы: архитектура, сервисы, API, операционные регламенты.

## Быстрые ссылки

### Для разработчиков

- **Быстрый старт**: [`for-devs.md`](./for-devs.md)
- **Архитектура**: [`architecture.md`](./architecture.md)
- **Aggregator (ядро)**: [`aggregator.md`](./aggregator.md)
- **Frontend (админка / кабинет мерчанта)**: [`frontend.md`](./frontend.md)
- **Flow (платёжная страница, legacy)**: [`flow.md`](./flow.md)
- **Trade / InternalTrade (P2P)**: [`trade.md`](./trade.md)
- **Rate Service (курсы)**: [`rate.md`](./rate.md)
- **Support Service (тикеты/диспуты)**: [`support-service.md`](./support-service.md)
- **Статусы транзакций**: [`transaction-statuses.md`](./transaction-statuses.md)

### Для саппорта

- **Быстрый старт**: [`for-support.md`](./for-support.md)
- **F.A.Q.**: [`faq.md`](./faq.md)
- **Клиенты и White-labels**: [`clients.md`](./clients.md)
- **Автоматика SMS**: [`automation-sms.md`](./automation-sms.md)
- **Трейдерская админ-панель**: [`ops/trader-admin-guide.md`](./ops/trader-admin-guide.md)
- **Админ-панель агрегатора (legacy)**: [`agradmin.md`](./agradmin.md)

### Справочники

- **Глоссарий**: [`ai/glossary.md`](./ai/glossary.md)
- **AI Knowledge Base**: [`ai/README.md`](./ai/README.md)
- **Список провайдеров**: [`gateways.csv`](./gateways.csv)
- **API Reference (Postman)**: [Postman Documentation](https://documenter.getpostman.com/view/13931884/2sAYQdipUu) · [`postman/collections/`](./postman/collections/)

### В разработке

- **Gateway Builder**: [`dev-in-progress/gateway-builder.md`](./dev-in-progress/gateway-builder.md)
- **Кабинет мерчанта**: [`dev-in-progress/merchant-cabinet.md`](./dev-in-progress/merchant-cabinet.md)
- **Agent Cabinet API**: [`dev-in-progress/agent-cabinet-api.md`](./dev-in-progress/agent-cabinet-api.md)

## Структура каталога

```
.
├── README.md
├── for-devs.md             # Быстрый старт разработчика
├── for-support.md          # Быстрый старт саппорта
├── architecture.md         # Архитектура платформы
├── aggregator.md           # Aggregator (Laravel 12 / PHP 8.4)
├── agradmin.md             # Legacy админ-панель (Yii2)
├── automation-sms.md       # SMS-автоматика
├── clients.md              # White-label клиенты
├── faq.md                  # F.A.Q.
├── flow.md                 # Flow (legacy платёжная страница)
├── frontend.md             # Новая админ-панель / кабинет мерчанта
├── gateways.csv            # Список провайдеров
├── rate.md                 # Rate Service
├── support-service.md      # Support Service
├── trade.md                # InternalTrade (P2P)
├── transaction-statuses.md # Статусы транзакций
├── ai/
│   ├── README.md
│   └── glossary.md
├── dev-in-progress/
│   ├── agent-cabinet-api.md
│   ├── gateway-builder.md
│   └── merchant-cabinet.md
├── ops/
│   └── trader-admin-guide.md
└── postman/
    └── collections/
```

## Репозитории

| Репозиторий | Описание | Стек |
|-------------|----------|------|
| `aggregator/` | Core API, каскады, провайдеры | Laravel 12 / PHP 8.4 |
| `trade/` | P2P процессинг, трейдеры, SMS | Yii2 / PHP |
| `flow/` | Платёжная страница (legacy) | React 18 / Vite |
| `flow-new/` | Новая платёжная страница | React 19 / Ant Design |
| `front/` | Админ-панель + кабинет мерчанта | React 19 / TanStack / Vite |
| `agradmin/` | Админ-панель (legacy) | Yii2 / PHP |
| `rate/` | Сервис курсов валют | FastAPI / Python |
| `support-service/` | Сервис тикетов и диспутов | FastAPI / Python |
