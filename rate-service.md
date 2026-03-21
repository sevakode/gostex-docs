# Rate Service — Сервис курсов валют

> **Стек:** FastAPI + Redis
> **Порт:** 8080
> **Тип:** Shared (один инстанс на все ВЛ)

---

## 1. Обзор

Rate Service — микросервис агрегации обменных курсов с криптобирж и внешних источников. Обеспечивает все сервисы платформы актуальными курсами для конвертации фиат → USDT.

---

## 2. API

| Endpoint | Method | Описание |
|----------|--------|----------|
| `/rate` | GET | Получить текущий курс |
| `/rate` | PATCH | Установить курс вручную |
| `/methods` | GET | Список методов (валютных пар) |
| `/update_methods` | POST | Обновить конфигурацию методов |
| `/health` | GET | Health check |

**Аутентификация:** Basic Auth

### Примеры

```bash
# Получить курс RUB/USDT с Binance
curl "http://rate:8080/rate?currency=RUB&second_currency=USDT&bank_or_payment_systems=sber&source_exchange=binance&side=buy"

# Установить курс вручную
curl -X PATCH "http://rate:8080/rate?currency=RUB&second_currency=USDT&bank_or_payment_systems=sber&source_exchange=manual&side=buy&rate=97.5" \
  -H "Authorization: Basic {credentials}"
```

---

## 3. Источники курсов

| Источник | Тип | Описание |
|----------|-----|----------|
| **Binance** | Криптобиржа | P2P курсы, основной источник |
| **Bybit** | Криптобиржа | P2P курсы |
| **HTX** | Криптобиржа | P2P курсы |
| **Grinex** | Криптообменник | Российский, P2P курсы |
| **Rapira** | Криптообменник | Российский, P2P курсы |
| **manual** | Ручной | Установленный администратором |

---

## 4. Поддерживаемые валюты

RUB, UZS, KZT, KGS, AZN, TJS, TRY, GEL, INR, ARS, EUR

---

## 5. Архитектура

```
┌─────────────┐     ┌─────────────┐
│ Aggregator  │────►│ Rate Service │◄──── Background polling
│ Trade       │     │ (FastAPI)    │         │
└─────────────┘     └──────┬──────┘         ▼
                           │         ┌──────────────┐
                           │         │ Binance API  │
                           ▼         │ Bybit API    │
                      ┌─────────┐   │ HTX API      │
                      │  Redis  │   │ Grinex API   │
                      │ (cache) │   │ Rapira API   │
                      └─────────┘   └──────────────┘
```

- Background tasks опрашивают биржи с заданным интервалом
- Курсы кэшируются в Redis
- При запросе `/rate` — возвращается кэшированное значение
- Ручной курс (`manual`) имеет приоритет

---

## 6. Использование в платформе

| Где | Как |
|-----|-----|
| **Aggregator** | Конвертация суммы платежа → USDT для баланса мерчанта |
| **Trade** | Расчёт комиссий трейдеров, курс для офферов |
| **Admin Panel** | Отображение текущих курсов, ручное обновление |

Источник курса задаётся на уровне:
- Метода оплаты (`rate_source`)
- MethodTypeRate мерчанта (`rate_source_payin`, `rate_source_payout`)
- Pivot MerchantMethod (`rate_source`, `rate_source_payout`)

---

## 7. Troubleshooting

### Курс не обновляется

```bash
# Проверить Redis
redis-cli keys "binance:*"
redis-cli hgetall "binance:RUB:USDT:sber:buy"
```

Решения:
1. Проверить соединение с биржами
2. Перезапустить polling tasks
3. Установить курс вручную через `PATCH /rate`

### Неверный курс

- Проверить `source_exchange` в настройках метода/оффера
- Сверить с курсом на бирже
- При необходимости — установить вручную
