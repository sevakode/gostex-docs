# Deployment — Развёртывание платформы (НЕАКТУАЛЬНО)

> Руководство по развёртыванию сервисов PSP Platform

---

## 1. Обзор инфраструктуры

Каждый White-label (ВЛ) разворачивается как изолированный набор сервисов:

```
ВЛ = Aggregator × 2 + Trade + Flow + Admin Panel + Merchant Panel + PostgreSQL + Patroni + Redis
```

Shared-сервисы (один инстанс на все ВЛ):
- Rate Service
- OpenSearch
- Grafana
- S3 / MinIO

---

## 2. Docker

Все сервисы контейнеризированы. Каждый репозиторий содержит `Dockerfile` и `docker-compose.yml`.

### Aggregator

```bash
cd aggregator
docker-compose up -d
# Миграции
docker exec aggregator php artisan migrate
# Очереди
docker exec aggregator php artisan horizon
```

### Trade

```bash
cd trade
docker-compose up -d
# Миграции
docker exec trade php yii migrate
```

### Flow

```bash
cd flow
docker build -t flow \
  --build-arg VITE_LARAVEL_APP_HOST=https://api.domain.com/api/v2/orders \
  --build-arg VITE_STYLE=default .
```

### Frontend (Admin + Merchant Panel)

```bash
cd frontend
# Admin Panel
npm run build -w admin-panel
# Merchant Panel
npm run build -w merchant-panel
```

---

## 3. CI/CD

Все проекты используют GitLab CI (`.gitlab-ci.yml`). Пайплайн:
1. **Build** — сборка Docker-образа
2. **Test** — запуск тестов (при наличии)
3. **Deploy** — деплой на целевой сервер

---

## 4. Зависимости при развёртывании

```
PostgreSQL + Patroni  ← требуется первым
         ↓
Redis                 ← требуется вторым
         ↓
Rate Service          ← автономный, запускать в любое время
         ↓
Aggregator            ← требует PostgreSQL, Redis, Rate Service
         ↓
Trade                 ← требует Aggregator, PostgreSQL, Redis, Rate Service
         ↓
Flow                  ← требует Aggregator API
Admin Panel           ← требует Aggregator API
Merchant Panel        ← требует Aggregator API
         ↓
Support Service       ← опционально, требует Aggregator
```

---

## 5. Переменные окружения

### Aggregator (ключевые)

| Переменная | Описание |
|------------|----------|
| `DB_HOST`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL |
| `REDIS_HOST`, `REDIS_PORT` | Redis |
| `RATE_SERVICE_URL` | URL Rate Service |
| `SANCTUM_STATEFUL_DOMAINS` | Домены для Sanctum (админка, кабинет) |
| `S3_*` | Конфигурация S3 хранилища |

### Trade (ключевые)

| Переменная | Описание |
|------------|----------|
| `DB_DSN` | PostgreSQL DSN |
| `REDIS_HOSTNAME` | Redis |
| `AGGREGATOR_URL` | URL Aggregator API |
| `AGGREGATOR_TOKEN` | Токен для callback в Aggregator |
| `TELEGRAM_BOT_TOKEN` | Токен Telegram бота для трейдеров |

### Flow

| Переменная | Описание |
|------------|----------|
| `VITE_LARAVEL_APP_HOST` | URL Aggregator API |
| `VITE_STYLE` | Бренд (white-label стиль) |

---

## 6. Health Checks

| Сервис | Endpoint | Ожидаемый ответ |
|--------|----------|-----------------|
| Aggregator | `/api/health` | 200 OK |
| Trade | `/site/health` | 200 OK |
| Rate Service | `/health` | 200 OK |
| Support Service | `/docs` | Swagger UI |
| PostgreSQL | port 5432 | TCP connect |
| Redis | port 6379 | PONG |
| OpenSearch | `/_cluster/health` | green/yellow |

---

## 7. Мониторинг

| Инструмент | Назначение |
|------------|------------|
| **Grafana** | Дашборды, метрики, алерты |
| **OpenSearch** | Логи всех сервисов |
| **Filebeat** | Доставка логов в OpenSearch |
| **Prometheus** | Сбор метрик |

---

## 8. Бэкапы

| Что | Как |
|-----|-----|
| PostgreSQL | pg_dump / Patroni snapshots |
| Redis | RDB snapshots |
| S3 | Версионирование объектов |
| Конфиги | Git (encrypted .env) |
