## 4. Интерфейсы взаимодействия с окружением (API)

### 4.1. Внешние интерфейсы, используемые системой

#### 4.1.1. Платёжная система (YooKassa)

* **Протокол:** HTTPS, REST
* **Аутентификация:** OAuth2 Bearer (заголовок `Authorization: Bearer {token}`)
* **Формат:** JSON

| Операция                  | Метод | URL                                                | Назначение                                                                      |
| ------------------------- | ----- | -------------------------------------------------- | ------------------------------------------------------------------------------- |
| Создание платежа          | POST  | `https://api.yookassa.ru/v3/payments`              | Инициировать платёж по `order_id`, `amount`, `currency`, `payment_method_token` |
| Получение статуса платежа | GET   | `https://api.yookassa.ru/v3/payments/{payment_id}` | Узнать статус (`succeeded`/`pending`/`canceled`)                                |

**Пример запроса**:

```http
POST /v3/payments HTTP/1.1
Host: api.yookassa.ru
Authorization: Bearer AQAB…xyz
Content-Type: application/json

{
  "amount": { "value": "123.45", "currency": "RUB" },
  "confirmation": { "type": "embed", "return_url": "https://api.bargo.example.com/v1/orders/42" },
  "metadata": { "order_id": "42" }
}
```

**Пример успешного ответа**:

```json
{
  "id": "21b0ef94-000f-5000-9000-1c1c00000031",
  "status": "succeeded",
  "paid": true,
  "amount": { "value": "123.45", "currency": "RUB" },
  "metadata": { "order_id": "42" }
}
```

**Ошибки и QoS**:

* `400 Bad Request` – валидация JSON
* `401 Unauthorized` – неверный токен
* `429 Too Many Requests` – >10 req/s (rate limit)
* `5xx` – retry ×3 с эксп. бэкофф
* Тайм-аут: 5 s

---

#### 4.1.2. Аналитическая BI-система (Metabase)

* **Протокол:** HTTPS, REST или напрямую SQL (JDBC)
* **Аутентификация:** токен (`X-Metabase-Session`) или LDAP/Basic Auth
* **Форматы:** JSON для REST, CSV для экспорта результатов

| Операция                | Метод | URL                                                    | Описание                                                   |
| ----------------------- | ----- | ------------------------------------------------------ | ---------------------------------------------------------- |
| Авторизация             | POST  | `https://bi.example.com/api/session`                   | Получить session-токен                                     |
| Список дашбордов        | GET   | `https://bi.example.com/api/dashboard`                 | Мета-данные доступных дашбордов                            |
| Детали дашборда         | GET   | `https://bi.example.com/api/dashboard/{id}`            | Карточки (charts) внутри дашборда                          |
| Данные карточки в JSON  | GET   | `https://bi.example.com/api/card/{card_id}/query/json` | Выполнить SQL-запрос карточки, вернуть JSON                |
| Данные карточки в CSV   | GET   | `https://bi.example.com/api/card/{card_id}/query/csv`  | Выполнить SQL-запрос карточки, вернуть CSV                 |
| Произвольный SQL-запрос | POST  | `https://bi.example.com/api/dataset`                   | Тело `{ "database": <id>, "native": { "query": "SQL…" } }` |

**Пример получения токена**:

```http
POST /api/session HTTP/1.1
Host: bi.example.com
Content-Type: application/json

{ "username": "admin", "password": "••••••" }
```

**Пример списка дашбордов**:

```http
GET /api/dashboard HTTP/1.1
Host: bi.example.com
X-Metabase-Session: A1B2C3D4…
Accept: application/json
```

**Ошибки и QoS**:

* `401 Unauthorized` – неверный/истёкший токен
* `403 Forbidden` – недостаточно прав
* `404 Not Found` – не найден `dashboard_id` или `card_id`
* `429 Too Many Requests` – >30 req/min
* TLS 1.2+, не логировать сырые данные

---

### 4.2. Публичные API «BarGo API»

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
            name:   { type: string }
            table:  { type: integer }
        items:
          type: array
          items: { $ref: '#/components/schemas/OrderItem' }
        payment_token:
          type: string
    OrderItem:
      type: object
      required: [ drink_id, quantity ]
      properties:
        drink_id: { type: integer }
        quantity: { type: integer }
    OrderResponse:
      type: object
      properties:
        order_id:       { type: integer }
        status:
          type: string
          enum: [ pending_payment, paid, cooking, ready, canceled ]
        payment_status:
          type: string
          enum: [ approved, declined, pending ]
```

#### 4.2.1. Эндпойнты

```yaml
paths:
  /orders:
    post:
      summary: Создать новый заказ
      tags: [orders-adapter]
      security: [ { BearerAuth: [] } ]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/OrderRequest' }
      responses:
        '201': { description: Заказ создан, оплата в процессе, content: { application/json: { schema: { $ref: '#/components/schemas/OrderResponse' } } } }
        '400': { description: Некорректный запрос }
        '402': { description: Оплата отклонена }
        '500': { description: Внутренняя ошибка }

  /orders/{order_id}:
    get:
      summary: Получить статус заказа
      tags: [orders-adapter]
      security: [ { BearerAuth: [] } ]
      parameters:
        - in: path
          name: order_id
          schema: { type: integer }
          required: true
      responses:
        '200': { description: Информация о заказе, content: { application/json: { schema: { $ref: '#/components/schemas/OrderResponse' } } } }
        '404': { description: Заказ не найден }

  /recipes:
    get:
      summary: Список напитков
      tags: [recipes-adapter]
      security: [ { BearerAuth: [] } ]
      responses:
        '200': 
          description: Массив напитков
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id:   { type: integer }
                    name: { type: string }

  /recipes/{drink_id}:
    get:
      summary: Детальный рецепт напитка
      tags: [recipes-adapter]
      security: [ { BearerAuth: [] } ]
      parameters:
        - in: path
          name: drink_id
          schema: { type: integer }
          required: true
      responses:
        '200':
          description: Полный рецепт
          content:
            application/json:
              schema:
                type: object
                properties:
                  ingredients:
                    type: array
                    items:
                      type: object
                      properties:
                        name:   { type: string }
                        amount: { type: string }
        '404': { description: Напиток не найден }
```

#### 4.2.2. Протокол взаимодействия и QoS

1. Клиент (Web-интерфейс) → `POST /orders` + JWT → **orders-adapter**
2. **orders-adapter** → **payment-adapter** → YooKassa
3. При `approved` → сохранение → возврат `201`; при `declined` → `402`
4. Клиент опрашивает `GET /orders/{order_id}` для обновления UI

* **Тайм-ауты:** 5 s (платёж), 2 s (рецепты)
* **Retry:** до 3 попыток (эксп. бэкофф)
* **Rate-limit:** 50 req/s (public), 10 req/s (`payment-adapter`)
* **SLAs:** P95 <200 ms, 99.9 % uptime

#### 4.2.3. Безопасность

* Все соединения TLS 1.3
* JWT (TTL 5 min) + refresh
* RBAC:

  * **officiant** — `orders-adapter` (POST/GET)
  * **bartender** — `recipes-adapter` (GET) + обновление статуса
  * **admin** — все эндпойнты + `bi-adapter` + CRUD через `manage-admin-comp` (если добавлен)
* Чувствительные поля (`payment_token`) не логируются

---

### 4.3. Примеры использования

```bash
# 1. Создание заказа
curl -X POST https://api.bargo.example.com/v1/orders \
  -H "Authorization: Bearer {jwt}" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": { "name": "Иван", "table": 12 },
    "items": [{ "drink_id": 5, "quantity": 2 }],
    "payment_token": "tok_1K..."
  }'
# 201 Created
```

```bash
# 2. Проверка статуса
curl https://api.bargo.example.com/v1/orders/101 \
  -H "Authorization: Bearer {jwt}"
# 200 OK
# {
#   "order_id": 101,
#   "status": "paid",
#   "payment_status": "approved"
# }
```

```bash
# 3. Получить дашборды Metabase
curl https://bi.example.com/api/dashboard \
  -H "X-Metabase-Session: {session_id}"
```

```bash
# 4. Запрос данных карточки в CSV
curl https://bi.example.com/api/card/12/query/csv \
  -H "X-Metabase-Session: {session_id}"
```
