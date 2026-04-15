# Flow — Платёжная страница (старый)

> ⚠️ **Этот файл описывает старый Flow (`flow/`).** Он продолжает дорабатываться, но новая документация не ведётся.
> Новый Flow: `gostex-pay/` — документация будет писаться с нуля.

> **Стек:** React 18 + TypeScript + Vite + Tailwind CSS
> **Репозиторий:** `flow/`

---

## 1. Обзор

Flow — SPA-приложение, которое показывает плательщику форму оплаты. Мерчант перенаправляет клиента на URL с ID заказа, клиент видит реквизиты, оплачивает, загружает чек.

### Ключевые возможности

| Возможность | Описание |
|-------------|----------|
| Мульти-метод | Выбор способа оплаты (СБП, C2C, NSPK, крипто и др.) |
| Deeplinks | Автоматическое открытие банковских приложений (Сбер, Тинькофф, ВТБ, Альфа, Mercado Pago) |
| White-label | 35+ готовых брендов (логотип, цвета, стиль) |
| Мультиязычность | 8 языков: ru, en, es, tr, ko, kz, az, uz |
| Загрузка чеков | Фото/скриншот оплаты |
| OTP | Поддержка 3DS / одноразовых кодов |
| Диспуты | Форма апелляции с email + файлом |
| Real-time | Polling статуса + WebSocket (Laravel Echo / Pusher) |

---

## 2. Как работает

```
Плательщик → GET /{orderId}
                ↓
         Данные заказа загружены
                ↓
       Есть метод оплаты? ─── НЕТ → Выбор метода → POST /{orderId}/payments
                │
               ДА
                ↓
         Показ реквизитов
    (номер карты / телефон / QR / deeplink)
                ↓
         Плательщик оплачивает
                ↓
      Загружает чек (PATCH /{orderId}/status)
                ↓
         Polling каждые 30 сек
                ↓
         Финальный статус (success / failed / expired)
```

---

## 3. API Endpoints

Flow общается с Aggregator через REST API:

| Endpoint | Method | Описание |
|----------|--------|----------|
| `/{orderId}` | GET | Получить данные заказа |
| `/{orderId}/methods` | GET | Список доступных методов оплаты |
| `/{orderId}/payments` | POST | Создать инвойс (выбрать метод) |
| `/{orderId}/status` | PATCH | Обновить статус (загрузить чек, OTP, диспут) |

Base URL задаётся через `VITE_LARAVEL_APP_HOST`.

---

## 4. Deeplinks банков

Для мобильных устройств Flow генерирует deeplinks, открывающие банковское приложение с предзаполненными данными перевода:

| Банк | Файл | Поддержка |
|------|------|-----------|
| Сбербанк | `sberbank.ts` | СБП по номеру телефона |
| Тинькофф / Т-Банк | `tinkoff.ts` | СБП по номеру телефона |
| ВТБ | `vtb.ts` | СБП по номеру телефона |
| Альфа-Банк | `alfabank.ts` | СБП по номеру телефона |
| Mercado Pago | `mercadopago.ts` | Латам |

Deeplinks определяются автоматически по `senderBank` из данных заказа.

---

## 5. White-label брендинг

Каждый ВЛ имеет папку в `src/assets/brand/{name}/` с файлами:
- `logo-light.svg` — логотип для светлой темы
- `logo-dark.svg` — логотип для тёмной темы
- `favicon.ico`

Бренд выбирается через `VITE_STYLE`:

```bash
VITE_STYLE=aggrepay  # или default, starda, sol, kent, legzo, и др.
```

**Текущие бренды (35+):** 1go, aggrepay, arkada, asgard, cat, clickconnect, daddy, default, drip, fresh, gama, gizbo, gtx, irwin, izzi, jet, kent, legzo, lex, mers, monro, pipay, pspcatalog, pulsar, r7, roxcasino, sol, starda, tappypay, tiktak, vires, volna.

---

## 6. Компоненты

```
src/components/
├── MainForm/               # Главная логика — решает что показывать
├── PaymentMethodSelector/  # Выбор метода оплаты
├── PaymentDetails/         # Реквизиты для оплаты
├── Details/                # Блок деталей (карта, телефон, QR)
├── DeepLinkButton/         # Кнопка «Открыть в приложении»
├── ImageUploader/          # Загрузка чека
├── CardForm/               # Ввод данных карты (ECOM 3DS)
├── OtpForm/                # Ввод OTP
├── Timer/                  # Таймер до истечения заявки
├── CancelPopup/            # Подтверждение отмены
├── TroubleComponent/       # Блок «Возникли трудности?»
├── SupportSection/         # Форма апелляции (email + чек + причина)
├── TransactionStatus/      # Финальный статус
├── LanguageSwitcher/       # Переключатель языка
└── ThemeSwitcher/          # Переключатель темы (light/dark)
```

---

## 7. Статусы заказа

| Статус | UI |
|--------|----|
| `pending` | Форма оплаты / выбор метода |
| `in_check` | Лоадер «На проверке» |
| `finished` / `success` | Зелёный экран успеха |
| `failed` | Красный экран ошибки |
| `canceled` | Экран отмены |
| `expired` | Экран «Время истекло» |
| `dispute` | Форма апелляции |

---

## 8. Переменные окружения

```bash
# API
VITE_LARAVEL_APP_HOST=https://api.domain.com/api/v2/orders

# UI элементы
VITE_SHOW_LOGO=show              # Логотип
VITE_SHOW_RECIPIENT=show         # ФИО получателя
VITE_SHOW_CHECK=show             # Загрузка чека
VITE_SHOW_TROUBLE_COMPONENT=show # Блок «Трудности с оплатой?»
VITE_SHOW_VIDEO_INSTRUCTION=show # Кнопка видеоинструкции

# Брендинг
VITE_STYLE=default               # Стиль/бренд
```

---

## 9. Запуск

```bash
# Локально
pnpm install
pnpm dev

# Docker
docker build -t flow .
docker run -p 3000:80 \
  -e VITE_LARAVEL_APP_HOST=https://api.domain.com/api/v2/orders \
  -e VITE_STYLE=default \
  flow
```

---

## 10. Зависимости

- **Aggregator** — API для данных заказа
- Не требует Trade напрямую — данные приходят через Aggregator
- WebSocket (опционально): Laravel Echo + Pusher для real-time обновлений статуса
