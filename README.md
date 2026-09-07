# Авито.Кухня

MVP веб-приложения для заказа еды из ресторанов.

## Возможности

- Просмотр списка заведений
- Просмотр меню заведения (с автоматической синхронизацией)
- Создание заказа с фиксацией цен
- Получение информации о заказе
- Отмена заказа
- Обновление статуса заказа рестораном (webhook)

## Архитектура

```
┌──────────┐     HTTP      ┌─────────────┐     HTTP     ┌──────────────────┐
│  Client  │ ────────────> │ Kitchen API │ ──────────> │ Restaurant       │
│          │ <──────────── │  (FastAPI)  │ <────────── │ Service (FastAPI) │
└──────────┘               └──────┬──────┘              └──────────────────┘
                                  │ SQL
                                  v
                           ┌──────────────┐
                           │  PostgreSQL  │
                           └──────────────┘
```

- **Kitchen API** (порт 8000) — основной сервис. Владеет заказами и реестром ресторанов.
- **Restaurant Service** (порт 8001) — пример заведения. Source of truth для меню.
- **PostgreSQL** — БД Kitchen API.

### Владение данными

| Данные | Владелец | Пояснение |
|--------|----------|-----------|
| restaurants | Kitchen API | Реестр заведений |
| menu_items | Kitchen API (кэш) | Source of truth — Restaurant Service |
| orders, order_items | Kitchen API | Полный владелец |
| Меню ресторана | Restaurant Service | Hardcoded in-memory данные |

### Синхронизация меню (Lazy Pull)

При запросе меню клиентом Kitchen API:
1. Запрашивает актуальное меню у Restaurant Service
2. При успехе — обновляет кэш (upsert) и возвращает свежие данные
3. При недоступности — возвращает кэш с пометкой `cached: true`
4. Если нет ни ресторана, ни кэша — 503

## Схема БД

### restaurants
| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID (PK) | |
| name | VARCHAR(255) | Название |
| description | TEXT | Описание |
| address | VARCHAR(500) | Адрес |
| is_active | BOOLEAN | Работает ли |
| webhook_url | VARCHAR(500) | URL для связи |
| external_id | VARCHAR(255) UNIQUE | ID во внешней системе |

### menu_items
| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID (PK) | |
| restaurant_id | UUID (FK → restaurants) | |
| external_id | VARCHAR(255) | ID блюда в ресторане |
| name | VARCHAR(255) | Название |
| price | INTEGER | Цена в копейках |
| is_available | BOOLEAN | В наличии |
| UNIQUE | (restaurant_id, external_id) | |

### orders
| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID (PK) | |
| restaurant_id | UUID (FK → restaurants) | |
| status | VARCHAR(20) | PENDING/CONFIRMED/... |
| total_price | INTEGER | Сумма в копейках |
| customer_name | VARCHAR(255) | Имя клиента |
| customer_phone | VARCHAR(50) | Телефон |

### order_items
| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID (PK) | |
| order_id | UUID (FK → orders) | |
| menu_item_id | UUID (FK → menu_items) | |
| quantity | INTEGER | Количество |
| price_at_order | INTEGER | Snapshot цены |
| name_at_order | VARCHAR(255) | Snapshot названия |

### Почему snapshot цены?

Если ресторан изменит цену после оформления заказа, сумма заказа не должна измениться. `price_at_order` фиксирует цену на момент заказа.

## State Machine заказа

```
PENDING → CONFIRMED → PREPARING → READY → COMPLETED
   ├──→ REJECTED
   └──→ CANCELLED (клиентом)
```

## API

### Клиентское API

| Метод | Путь | Описание |
|-------|------|----------|
| GET | /api/v1/restaurants | Список заведений |
| GET | /api/v1/restaurants/{id}/menu | Меню заведения |
| POST | /api/v1/orders | Создать заказ |
| GET | /api/v1/orders/{id} | Детали заказа |
| POST | /api/v1/orders/{id}/cancel | Отменить заказ |

### Webhook API (для ресторана)

| Метод | Путь | Описание |
|-------|------|----------|
| POST | /api/v1/webhooks/orders/{id}/status | Обновить статус заказа |

### Restaurant Service API

| Метод | Путь | Описание |
|-------|------|----------|
| GET | /api/menu | Меню ресторана |
| POST | /api/orders | Принять заказ |
| POST | /api/orders/{id}/status | Обновить статус |
| GET | /api/orders | Список заказов |

## Запуск проекта

```bash
docker compose up --build
```

После запуска:
- Kitchen API: http://localhost:8000
- Swagger UI: http://localhost:8000/docs
- Restaurant Service: http://localhost:8001
- Restaurant Swagger: http://localhost:8001/docs

## Запуск тестов

```bash
docker compose exec kitchen-api pytest -v
```

## Запуск линтера

```bash
docker compose exec kitchen-api ruff check .
docker compose exec kitchen-api ruff format --check .
```

## Примеры запросов

### Получить рестораны
```bash
curl http://localhost:8000/api/v1/restaurants
```

### Получить меню
```bash
curl http://localhost:8000/api/v1/restaurants/{id}/menu
```

### Создать заказ
```bash
curl -X POST http://localhost:8000/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "restaurant_id": "ID_РЕСТОРАНА",
    "customer_name": "Иван",
    "customer_phone": "+79991234567",
    "items": [
      {"menu_item_id": "ID_БУРГЕРА", "quantity": 2}
    ]
  }'
```

### Подтвердить заказ (от лица ресторана)
```bash
curl -X POST http://localhost:8001/api/orders/rest-order-1/status \
  -H "Content-Type: application/json" \
  -d '{"status": "CONFIRMED"}'
```

## Архитектурные решения

| Решение | Обоснование |
|---------|-------------|
| PostgreSQL | Надёжная СУБД, поддержка транзакций, UUID, JSON |
| FastAPI | Async, автогенерация OpenAPI, валидация через Pydantic |
| SQLAlchemy 2.x (async) | Современный ORM с async поддержкой |
| UUID для PK | Безопасность, генерация на клиенте, масштабирование |
| Деньги в копейках | Избегаем проблем с FLOAT (0.1 + 0.2 ≠ 0.3) |
| Lazy Pull для меню | Баланс простоты и актуальности данных |
| HTTP вместо Kafka | Достаточно для MVP, просто в реализации |
| String вместо ENUM для статуса | Гибкость при добавлении новых статусов |

## Ограничения MVP

- Нет авторизации/аутентификации
- Нет проверки цены перед созданием заказа (цена берётся из кэша)
- Меню обновляется только при запросе клиента (Lazy Pull)
- Синхронная отправка заказа в ресторан
- Restaurant Service хранит данные in-memory
- Нет пагинации
- Один демо-ресторан

## Возможные улучшения

- JWT авторизация
- Очередь сообщений (RabbitMQ) для надёжной доставки заказов
- Push-уведомления от ресторана при изменении меню
- Пагинация и поиск
- Сверка цен перед созданием заказа
- Система регистрации ресторанов
- Кэширование в Redis
- Расширенная state machine с политиками отмены
