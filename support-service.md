# Support Service v2 — Сервис поддержки

> **Стек:** FastAPI + SQLAlchemy + Telegram (aiogram)
> **Порт:** 8000
> **Тип:** Опциональный (требует Aggregator)
> **Последнее обновление:** 2026-04-15 (автосинк)

---

## 1. Обзор

Support Service v2 — микросервис для управления диспутами и коммуникации с трейдерами через Telegram. Интегрируется с Aggregator для получения событий по транзакциям.

---

## 2. API

| Endpoint | Method | Описание |
|----------|--------|----------|
| `/disputes/{order_uuid}/status` | POST | Обновить статус диспута |
| `/disputes/{order_uuid}/actions` | GET | История действий по заказу |
| `/disputes/{order_uuid}/actions` | POST | Создать действие по диспуту |
| `/disputes/{order_uuid}/payment-success` | POST | Подтвердить успешную оплату |
| `/messages/send` | POST | Отправить сообщение в Telegram |
| `/messages/reply` | POST | Ответить на сообщение в Telegram |
| `/webhook/tg` | POST | Telegram webhook (aiogram) |
| `/docs` | GET | Swagger UI |

---

## 3. Статусы диспута

**State:** `created` → `active` → `closed`

**Status:**
- `no_payment` — оплата не поступила
- `fake_check` — поддельный чек
- `wrong_requirements` — неверные реквизиты
- `request_proof` — запрос подтверждения
- `request_proof_check` — запрос чека
- `request_proof_statement` — запрос выписки
- `request_proof_video` — запрос видео
- `unknown`

---

## 4. Архитектура

```
Aggregator ──webhook──► Support Service ──Telegram API──► Чаты саппорта
                              │
                              ▼
                         PostgreSQL
                      (диспуты, действия)
```

---

## 5. Интеграция с Aggregator

Включается в настройках Aggregator:
- `app.sup_service` = true
- `app.support_service_url` = URL API
- `app.support_service_token` = токен аутентификации

---

## 6. Поведение Telegram бота

### Stale Update Middleware
С апрельского релиза (2026-04-15) бот игнорирует устаревшие Telegram-обновления старше **2 часов** (`IgnoreStaleTelegramUpdateMiddleware`). Это устраняет проблему обработки старых сообщений после перезапуска бота.

### UUID regex
Исправлен regexp для извлечения UUID/ссылок из текста сообщений (commit `b0c97e5`).

### Обработка новых сообщений (new_message)
- Лог-превью текста расширен с 100 до 300 символов
- При нахождении активного диспута по UUID (без reply) `counterparty_side` автоматически устанавливается в `COMPLAINANT`
- Создание диспута использует `message_id` вместо `thread_id` как внешний ключ
- Добавлен класс `FileDownloadError` для явной обработки ошибок загрузки файлов

---

## 7. CI/CD ветки

| Среда | Ветка (до 2026-04-14) | Ветка (после) |
|-------|----------------------|---------------|
| Test build | `develop` | `dev` |
| Prod build | `prod/all` | `main` |

---

## 8. Контакт

Разработчик: Viktor P (`@gostex_viktor_p`)

