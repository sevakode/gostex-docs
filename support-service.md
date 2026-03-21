# Support Service — Сервис поддержки

> **Стек:** FastAPI + SQLAlchemy + Telegram
> **Порт:** 8000
> **Тип:** Опциональный (требует Aggregator)

---

## 1. Обзор

Support Service — микросервис для управления тикетами, диспутами и коммуникации с трейдерами через Telegram. Интегрируется с Aggregator для получения событий по транзакциям.

---

## 2. API

| Endpoint | Method | Описание |
|----------|--------|----------|
| `/webhook/action` | POST | Входящие действия из Aggregator |
| `/orders/{order_id}/messages` | GET | История сообщений по заказу |
| `/disputes` | GET/POST | Управление диспутами |
| `/docs` | GET | Swagger UI (health check) |

---

## 3. Функции

### Тикеты
- Создание тикета при изменении статуса транзакции
- История сообщений по заказу
- Привязка к мерчанту и провайдеру

### Диспуты
- Создание диспута при статусе «Спор»
- Управление статусами диспута
- Уведомления в Telegram

### Telegram-интеграция
- Отправка уведомлений в чаты саппорта
- Отправка алертов о критических событиях
- Inline-кнопки для быстрых действий

---

## 4. Архитектура

```
Aggregator ──webhook──► Support Service ──Telegram API──► Чаты саппорта
                              │
                              ▼
                         PostgreSQL
                      (тикеты, диспуты)
```

---

## 5. Интеграция с Aggregator

Включается в настройках Aggregator:
- `app.sup_service` = true
- `app.support_service_url` = URL API
- `app.support_service_token` = токен аутентификации

При изменении статуса транзакции Aggregator отправляет webhook в Support Service.

---

## 6. Логи

| Файл | Описание |
|------|----------|
| `webhook.log` | Входящие вебхуки |
| stdout | Docker logs |
