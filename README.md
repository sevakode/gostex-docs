# Документация проекта

Этот каталог содержит **каноническую** документацию по платформе (архитектура, API, торговый сервис, troubleshooting) и операционные материалы (инструкции/регламенты).

## Быстрые ссылки

- **Архитектура**: [`architecture.md`](./architecture.md)
- **Flow (платёжная страница)**: [`flow.md`](./flow.md)
- **Rate Service (курсы)**: [`rate-service.md`](./rate-service.md)
- **Support Service**: [`support-service.md`](./support-service.md)
- **Deployment**: [`deployment.md`](./deployment.md)
- **Админ-панель агрегатора (legacy)**: [`agradmin.md`](./agradmin.md)
- **Кабинет мерчанта**: [`merchant-cabinet.md`](./merchant-cabinet.md)
- **AI-документация**: [`ai/README.md`](./ai/README.md)
- **FAQ**: [`faq.md`](./faq.md)
- **Операционные материалы**: [`ops/README.md`](./ops/README.md)
- **Инструкция по трейдерской админ-панели**: [`ops/trader-admin-guide.md`](./ops/trader-admin-guide.md)
- **API Reference (Postman)**: [Postman Documentation](https://documenter.getpostman.com/view/13931884/2sAYQdipUu) | [`postman/collections/`](./postman/collections/)

## Спецификации (внутренние)

- **ТЗ новой админ-панели**: [`specs/admin-panel-spec.md`](./specs/admin-panel-spec.md)
- **Чеклист перехода**: [`specs/switch-checklist.md`](./specs/switch-checklist.md)

## Структура каталога

```
docs/
  README.md
  architecture.md          # Архитектура платформы, сервисы, стек
  flow.md                  # Flow — платёжная страница
  rate-service.md          # Rate Service — курсы валют
  support-service.md       # Support Service — тикеты и диспуты
  deployment.md            # Развёртывание платформы
  agradmin.md              # Админ-панель агрегатора (legacy Yii2)
  merchant-cabinet.md      # Кабинет мерчанта
  faq.md                   # Часто задаваемые вопросы
  table-of-contents.md     # Полное оглавление
  ai/
    README.md
    trade-service.md
    troubleshooting.md
    glossary.md
  ops/
    README.md
    tech-support.md
    trader-admin-guide.md
  specs/
    admin-panel-spec.md    # ТЗ для продукт-дизайнера
    switch-checklist.md    # Чеклист деплоя новой админки
  postman/
    collections/
      *.json
```

## Репозитории

| Репозиторий | Описание | Стек |
|-------------|----------|------|
| `highpay/aggregator` | Core API, каскады, провайдеры | Laravel 12 / PHP 8.4 |
| `highpay/trade` | P2P процессинг, трейдеры, SMS | Yii2 / PHP |
| `highpay/flow` | Платёжная страница | React 18 / TypeScript / Vite |
| `highpay/frontend` | Админ-панель + кабинет мерчанта | React 19 / TypeScript / Vite |
| `highpay/agradmin` | Админ-панель (legacy) | Yii2 / PHP |
| `gostex-io/docs` | Эта документация | Markdown |

---
**Последнее обновление:** 2026-03-21
