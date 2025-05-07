# 4. Интерфейсы взаимодействия с окружением (API)

## 4.1 Внешние интерфейсы, используемые системой

### 4.1.1 Платёжная система (YooKassa)

**Протокол**: HTTPS, REST
**Аутентификация**: OAuth 2.0 Bearer (заголовок `Authorization: Bearer {token}`)
**Формат**: JSON

#### Таблица операций

| Операция                  | Метод | URL                                                                                  | Описание                                                         |
|---------------------------|-------|--------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Создание платежа          | POST  | [https://api.yookassa.ru/v3/payments](https://api.yookassa.ru/v3/payments)           | Инициировать платёж по `order_id`, `amount`, `currency`, `token` |
| Получение статуса платежа | GET   | [https://api.yookassa.ru/v3/payments/{id}](https://api.yookassa.ru/v3/payments/{id}) | Получить статус: `succeeded`/`pending`/`canceled`                |

#### Пример запроса (создание платежа)

```
POST /v3/payments HTTP/1.1
Host: api.yookassa.ru
Authorization: Bearer AQAB…xyz
Content-Type: application/json

{
  "amount": { "value": "123.45", "currency": "RUB" },
  "confirmation": {
    "type": "embed",
    "return_url": "https://bar.example.com/orders/42"
  },
  "metadata": { "order_id": "42" }
}
```

#### Пример ответа

```json
{
  "id": "21b0ef94-000f-5000-9000-1c1c00000031",
  "status": "succeeded",
  "paid": true,
  "amount": {
    "value": "123.45",
    "currency": "RUB"
  },
  "metadata": {
    "order_id": "42"
  }
}
```

#### Обработка ошибок

* **400 Bad Request** — неверный формат данных (JSON-валидация).
* **401 Unauthorized** — неверный или просроченный токен.
* **429 Too Many Requests** — превышен лимит (10 запросов/сек).
* **5xx** — временные сбои (рекомендуется retry до 3 раз с экспоненциальным бэкоффом).

---

### 4.1.2 HR-сервис (BambooHR-like REST API)

**Протокол**: HTTPS, REST
**Аутентификация**: OAuth 2.0 Bearer или API-ключ в заголовке `X-API-Key`
**Формат**: JSON

#### Таблица операций

| Операция                 | Метод | URL                                                                                          | Описание                                  |
|--------------------------|-------|----------------------------------------------------------------------------------------------|-------------------------------------------|
| Список сотрудников       | GET   | [https://hr.example.com/api/v1/employees](https://hr.example.com/api/v1/employees)           | Получить все активные профили             |
| Детали одного сотрудника | GET   | [https://hr.example.com/api/v1/employees/{id}](https://hr.example.com/api/v1/employees/{id}) | Расписание и смены конкретного сотрудника |

#### Пример запроса

```
GET /api/v1/employees HTTP/1.1
Host: hr.example.com
Authorization: Bearer eyJ…ABC
Accept: application/json
```

#### Пример ответа

```json
[
  {
    "id": 17,
    "full_name": "Иванов Иван Иванович",
    "position": "Бармен",
    "shifts": [
      {
        "date": "2025-05-10",
        "start": "18:00",
        "end": "02:00"
      }
    ],
    "contact": {
      "email": "ivanov@example.com",
      "phone": "+7 900 123-45-67"
    }
  }
  …
]
```

#### Обработка ошибок

* **401 Unauthorized** — неверный токен.
* **403 Forbidden** — недостаточно прав доступа.
* **404 Not Found** — сотрудник не найден.

---

## 4.2 Публичные API, предоставляемые нашей системой

Мы специфицируем RESTful API по стандарту **OpenAPI 3.0**.

### 4.2.1 Компоненты OpenAPI

```yaml
openapi: 3.0.3
info:
  title: BarGo — Order & Recipe API
  version: 1.0.0
servers:
  - url: https://api.bargo.example.com/v1
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    OrderRequest:
      type: object
      required: [ customer, items, payment_token ]
      properties:
        customer:
          type: object
          properties:
            name:
              type: string
            table:
              type: integer
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        payment_token:
          type: string
          description: Token from YooKassa UI

    OrderItem:
      type: object
      required: [ drink_id, quantity ]
      properties:
        drink_id:
          type: integer
        quantity:
          type: integer

    OrderResponse:
      type: object
      properties:
        order_id:
          type: integer
        status:
          type: string
          enum: [ pending_payment, paid, cooking, ready, canceled ]
        payment_status:
          type: string
          enum: [ approved, declined, pending ]
security:
  - BearerAuth: [ ]
```

---

### 4.2.2 Эндпойнты

| Операция               | Метод | Путь                  | Описание                                           |
|------------------------|-------|-----------------------|----------------------------------------------------|
| Создать заказ          | POST  | `/orders`             | Создаёт заказ, инициирует оплату                   |
| Получить статус заказа | GET   | `/orders/{order_id}`  | Возвращает `OrderResponse`                         |
| Список напитков        | GET   | `/recipes`            | Возвращает массив `{id, name}`                     |
| Детали рецепта напитка | GET   | `/recipes/{drink_id}` | Возвращает полный рецепт (ингредиенты и пропорции) |

---

### 4.2.3 Протокол взаимодействия

1. Клиент отправляет `POST /orders` с JSON-телом и заголовком
   `Authorization: Bearer {JWT}`.
2. Сервер валидирует токен, парсит запрос и вызывает компонент **PaymentAdapter**.
3. **PaymentAdapter**:

    * `POST https://api.yookassa.ru/v3/payments`
    * Ждёт ответа; при `succeeded` продолжает, при `declined` отдаёт 402.
4. Сервер сохраняет заказ, возвращает `201 Created`.
5. Клиент опрашивает `GET /orders/{order_id}` для обновления UI.

---

### 4.2.4 Обработка ошибок и требования QoS

* **Коды ответов**:

    * 4xx – валидация, авторизация
    * 5xx – внутренние ошибки (retryable)
* **Тайм-ауты**:

    * 5 s на внешний вызов платежа
    * 2 s на чтение рецептов
* **Retry**: до 3 попыток по экспоненциальному бэкоффу
* **Rate limiting**:

    * 50 req/s на публичных эндпойнтах
    * 10 req/s на PaymentAdapter
* **SLA**: 99.9 % uptime, P95 latency < 200 ms

---

### 4.2.5 Безопасность

* TLS 1.3 для всех соединений.
* JWT с TTL 5 мин и refresh-токенами.
* RBAC (роль «официант» только создаёт и читает заказы; «бармен» – читает рецепты и обновляет статус приготовления;
  «админ» – полный доступ).
* Не логировать чувствительные данные (`payment_token`, `card_data`).

---

## 4.3 Примеры использования

**1. Создание заказа**

```bash
curl -X POST https://api.bargo.example.com/v1/orders \
  -H "Authorization: Bearer eyJ…" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": { "name": "Иван", "table": 12 },
    "items": [{ "drink_id": 5, "quantity": 2 }],
    "payment_token": "tok_1Kxyz…"
  }'
```

**Ответ (201 Created)**

```json
{
  "order_id": 101,
  "status": "pending_payment",
  "payment_status": "pending"
}
```

---

**2. Проверка статуса**

```bash
curl https://api.bargo.example.com/v1/orders/101 \
  -H "Authorization: Bearer eyJ…"
```

**Ответ (200 OK)**

```json
{
  "order_id": 101,
  "status": "paid",
  "payment_status": "approved"
}
```

---
