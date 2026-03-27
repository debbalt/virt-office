# `api-contracts.md`

```markdown
# API Контракты: VirtOffice ↔ 1С
**Версия:** 2.0  
**Дата:** 2025  
**Статус:** DRAFT — требует подтверждения от 1С разработчика

---

## 1. Общие правила

### 1.1 Base URLs

| Сторона | Mock URL | Production URL |
|---|---|---|
| VirtOffice Site | `https://mock.virtoffice.ua/api/v2` | `TBD` |
| 1С API | `https://mock.1c.virtoffice.ua/api/v2` | `TBD` |

### 1.2 Аутентификация

Все запросы в обе стороны требуют header:

```http
X-API-Key: {api_key}
```

| Ключ | Значение | Где хранится |
|---|---|---|
| Site → 1С | `mock-site-to-1c-key-2025` | `settings.ONE_C_API_KEY` |
| 1С → Site | `mock-1c-to-site-key-2025` | `settings.INTAKE_API_KEY` |

Ответ при неверном ключе:
```json
HTTP 401
{
  "error": "Invalid or missing API key"
}
```

### 1.3 Формат запросов и ответов

- Все тела запросов и ответов: `Content-Type: application/json`
- Все даты: `ISO 8601` → `"2025-03-20T10:00:00Z"` (UTC)
- Кодировка: `UTF-8`
- Пустые поля: не включать в тело (опускать, не слать `null`)

### 1.4 Идемпотентность

Идемпотентность обеспечивается по бизнес-полю из 1С:

| Тип события | Поле идемпотентности |
|---|---|
| Пользователь | `login` |
| Письмо | `message_id` |
| Документ | `document_1cid` |
| Договор | `contract_id` |
| Корп. кабинет | `cabinet_1c_id` |

Если запрос с тем же ID уже обработан успешно — сайт возвращает
`200` с результатом предыдущей обработки, не создаёт дубль.

### 1.5 Retry логика

**1С → Site** (если сайт вернул 5xx):

```
Попытка 1:  сразу
Попытка 2:  через 1 минуту
Попытка 3:  через 5 минут
Попытка 4:  через 30 минут
Попытка 5:  через 2 часа
После 5 попыток → алерт разработчику, задача в очередь на ручной разбор
```

**Site → 1С** (если 1С вернул 5xx):

```
Celery retry:
  max_retries = 5
  countdown:   60 → 300 → 1800 → 7200 → 86400 (секунды)
  После 5 попыток → статус OutgoingEvent = "failed", алерт
```

### 1.6 Коды ответов

| Код | Значение |
|---|---|
| `200` | Успешно обновлено |
| `201` | Успешно создано |
| `400` | Ошибка валидации входных данных |
| `401` | Неверный или отсутствующий API ключ |
| `404` | Объект не найден |
| `409` | Конфликт — объект уже существует |
| `500` | Внутренняя ошибка сервера |

---

## 2. IN: 1С → Site (входящие события)

Единая точка входа для всех событий от 1С.

### Endpoint

```http
POST https://mock.virtoffice.ua/api/v2/intake/
X-API-Key: {api_key}
Content-Type: application/json
```

### Структура запроса

```json
{
  "event_type": "letter.create",
  "payload": { }
}
```

> `event_type` — определяет маршрутизацию внутри IntakeDispatcher.  
> `payload` — тело события, уникальное для каждого типа (см. ниже).

---

### 2.1 Пользователи

#### `user.create` — создать пользователя (обычный кабинет)

```json
// Запрос от 1С
{
  "event_type": "user.create",
  "payload": {
    "login":                "12345",        // REQUIRED — номер договора
    "password":             "hashedpwd",    // REQUIRED
    "name":                 "ТОВ Ромашка", // REQUIRED
    "phone_sms":            "+380991234567",
    "f_email":              "info@romashka.ua",
    "email_notifications":  true,
    "delivery_link":        "https://..."
  }
}

// Ответ сайта — успех
HTTP 201
{
  "message": "Record successfully created."
}

// Ответ сайта — уже существует (идемпотентность)
HTTP 200
{
  "message": "Already exists."
}

// Ответ сайта — ошибка валидации
HTTP 400
{
  "error": "Missing required fields",
  "fields": ["login", "password"]
}

// Ответ сайта — пользователь не найден (для update)
HTTP 404
{
  "error": "User not found",
  "login": "12345"
}
```

---

#### `user.update` — обновить пользователя

```json
// Запрос от 1С
{
  "event_type": "user.update",
  "payload": {
    "login": "12345",           // REQUIRED — идентификатор
    // Минимум одно поле из опциональных:
    "name":                 "ТОВ Ромашка 2",
    "password":             "newhashedpwd",
    "phone_sms":            "+380991234568",
    "f_email":              "new@romashka.ua",
    "email_notifications":  false,
    "delivery_link":        "https://..."
  }
}

// Ответ сайта — успех
HTTP 200
{
  "message": "Record successfully updated."
}
```

---

### 2.2 Корпоративный кабинет

#### `cabinet.corporate.create` — создать корпоративный кабинет

```json
// Запрос от 1С
{
  "event_type": "cabinet.corporate.create",
  "payload": {
    "cabinet_1c_id":  "cab-001",          // REQUIRED — ID из 1С
    "master_email":   "ceo@company.ua",   // REQUIRED
    "master_name":    "Іваненко Іван Іванович",
    "master_phone":   "+380991234567",
    "companies": [                         // REQUIRED — минимум одна
      {
        "login":             "12345",
        "name":              "ТОВ Ромашка",
        "organization_name": "ТОВ Ромашка Груп"
      }
    ]
  }
}

// Ответ сайта — успех
HTTP 201
{
  "message": "Cabinet created.",
  "cabinet_id": 42,
  "invite_sent_to": "ceo@company.ua"
}
```

---

#### `corporate.user.invite` — пригласить сотрудника через 1С

```json
// Запрос от 1С
{
  "event_type": "corporate.user.invite",
  "payload": {
    "cabinet_1c_id":  "cab-001",           // REQUIRED
    "email":          "accountant@company.ua", // REQUIRED
    "role":           "Бухгалтер",         // REQUIRED
    "company_login":  "12345",             // REQUIRED — к какой компании
    "name":           "Петренко Петро"
  }
}

// Ответ сайта — успех
HTTP 201
{
  "message": "Invite sent.",
  "invite_token": "uuid-xxx",
  "expires_at": "2025-04-20T10:00:00Z"
}
```

---

### 2.3 Договора

#### `contract.upsert` — создать или обновить договор

```json
// Запрос от 1С
{
  "event_type": "contract.upsert",
  "payload": {
    "contract_id":        "cnt-001",      // REQUIRED — ID из 1С
    "login":              "12345",        // REQUIRED — логин компании
    "service_type":       "storage",      // REQUIRED
    "start_date":         "2025-01-01",
    "end_date":           "2026-01-01",
    "organization_name":  "ТОВ Ромашка",
    "address":            "вул. Хрещатик 1, Київ"
  }
}

// Ответ сайта — создан
HTTP 201
{
  "message": "Contract created."
}

// Ответ сайта — обновлён
HTTP 200
{
  "message": "Contract updated."
}
```

---

### 2.4 Листи (Письма)

#### `letter.create` — новое письмо

```json
// Запрос от 1С
{
  "event_type": "letter.create",
  "payload": {
    "message_id":       "msg-001",          // REQUIRED — ID из 1С
    "user_login":       "12345",            // REQUIRED
    "scan_date":        "2025-03-20T10:00:00Z", // REQUIRED
    "attachment_link":  "https://storage.virtoffice.ua/scans/msg-001.pdf", // REQUIRED
    "issued_from":      "ДПС України",
    "issue_date":       "2025-03-18",
    "contact_person":   "Коваленко О.М."
  }
}

// Ответ сайта — успех
HTTP 201
{
  "message": "Record successfully created."
}

// Ответ сайта — компания не найдена
HTTP 404
{
  "error": "User not found",
  "login": "12345"
}
```

---

#### `letter.update` — обновить письмо

```json
// Запрос от 1С
{
  "event_type": "letter.update",
  "payload": {
    "message_id":      "msg-001",     // REQUIRED — ключ поиска
    // Минимум одно поле из опциональных:
    "scan_date":       "2025-03-21T10:00:00Z",
    "attachment_link": "https://storage.virtoffice.ua/scans/msg-001-v2.pdf",
    "issued_from":     "ДПС України (оновлено)",
    "issue_date":      "2025-03-19",
    "contact_person":  "Мельник В.О."
  }
}

// Ответ сайта — успех
HTTP 200
{
  "message": "Record successfully updated."
}
```

---

#### `letter.status.update` — обновить статус письма

```json
// Запрос від 1С
{
  "event_type": "letter.status.update",
  "payload": {
    "message_id":  "msg-001",     // REQUIRED
    "status":      "issued",      // REQUIRED — см. список статусів нижче
    "issued_to":   "Директор Іваненко",
    "issued_date": "2025-03-22T14:30:00Z"
  }
}

// Статуси листів
// "received"  — отримано на склад
// "scanned"   — відскановано
// "pending"   — очікує видачі
// "issued"    — видано клієнту
// "delivery"  — відправлено НП

// Ответ сайта — успех
HTTP 200
{
  "message": "Status updated."
}
```

---

#### `letter.scan.ready` — скан письма готов

```json
// Запрос от 1С
{
  "event_type": "letter.scan.ready",
  "payload": {
    "message_id":      "msg-001",   // REQUIRED
    "scan_file_url":   "https://storage.virtoffice.ua/scans/msg-001-final.pdf" // REQUIRED
  }
}

// Ответ сайта — успех (+ триггерит уведомление клиенту)
HTTP 200
{
  "message": "Scan saved. Notification sent."
}
```

---

### 2.5 Документы

#### `document.create` — создать документ

```json
// Запрос от 1С
{
  "event_type": "document.create",
  "payload": {
    "document_1cid":     "doc-001",       // REQUIRED
    "document_number":   "АКТ-2025-001",  // REQUIRED
    "document_date":     "2025-03-20",    // REQUIRED
    "document_name":     "Акт звірки",    // REQUIRED
    "counterparty_login": "12345",        // REQUIRED
    "contract_number":   "ДГВ-2025-001",
    "contract_date":     "2025-01-01",
    "organization_name": "ТОВ Ромашка",
    "doc_link":          "https://storage.virtoffice.ua/docs/doc-001.pdf"
  }
}

// Ответ сайта — успех
HTTP 201
{
  "message": "Record successfully created."
}

// Ответ сайта — компания не найдена
HTTP 404
{
  "error": "Counterparty not found",
  "login": "12345"
}
```

---

#### `document.update` — обновить документ

```json
// Запрос от 1С
{
  "event_type": "document.update",
  "payload": {
    "document_1cid": "doc-001",         // REQUIRED — ключ поиска
    // Минимум одно поле из опциональных:
    "document_number":   "АКТ-2025-001-v2",
    "document_date":     "2025-03-21",
    "document_name":     "Акт звірки (оновлено)",
    "contract_number":   "ДГВ-2025-002",
    "contract_date":     "2025-02-01",
    "organization_name": "ТОВ Ромашка Груп",
    "doc_link":          "https://storage.virtoffice.ua/docs/doc-001-v2.pdf"
  }
}

// Ответ сайта — успех
HTTP 200
{
  "message": "Record successfully updated."
}
```

---

#### `document.delete` — удалить документ

```json
// Запрос от 1С
{
  "event_type": "document.delete",
  "payload": {
    "document_1cid": "doc-001"    // REQUIRED
  }
}

// Ответ сайта — успех
HTTP 200
{
  "message": "Record successfully deleted."
}

// Ответ сайта — не найден
HTTP 404
{
  "error": "Document not found",
  "document_1cid": "doc-001"
}
```

---

### 2.6 Доставка

#### `delivery.status.update` — обновить статус доставки НП

```json
// Запрос от 1С
{
  "event_type": "delivery.status.update",
  "payload": {
    "ttn_number":    "20450000000001",   // REQUIRED — ТТН Нова Пошта
    "status":        "delivered",        // REQUIRED — см. список ниже
    "delivered_at":  "2025-03-22T14:00:00Z"
  }
}

// Статусы доставки
// "created"    — ТТН создана
// "in_transit" — в пути
// "arrived"    — прибыла в отделение
// "delivered"  — получена
// "returned"   — возврат

// Ответ сайта — успех (+ WebSocket push клиенту)
HTTP 200
{
  "message": "Delivery status updated."
}
```

---

## 3. OUT: Site → 1С (исходящие запросы)

Все запросы выполняются через `OneCAdapter` асинхронно через Celery.

### 3.1 Проверка существования пользователя

```http
GET https://mock.1c.virtoffice.ua/api/v2/users/{login}/
X-API-Key: {api_key}
```

```json
// Ответ 1С — найден
HTTP 200
{
  "exists":               true,
  "f_email":              "info@romashka.ua",
  "phone_sms":            "+380991234567",
  "email_notifications":  true
}

// Ответ 1С — не найден
HTTP 200
{
  "exists": false
}
```

---

### 3.2 Синхронизация профиля пользователя

```http
POST https://mock.1c.virtoffice.ua/api/v2/users/sync/
X-API-Key: {api_key}
Content-Type: application/json

{
  "login":                "12345",          // REQUIRED
  "f_email":              "info@romashka.ua",
  "f_phone":              "+380991234567",
  "email_notifications":  true
}
```

```json
// Ответ 1С — успех
HTTP 200
{
  "message": "User synced."
}
```

---

### 3.3 Запрос акта сверки (Reconciliation Act)

```http
POST https://mock.1c.virtoffice.ua/api/v2/documents/reconciliation/
X-API-Key: {api_key}
Content-Type: application/json

{
  "organization":      "ТОВ Ромашка",      // REQUIRED
  "counterparty_id":   "12345",            // REQUIRED (/ → _)
  "startDate":         "01_01_2025",       // REQUIRED формат d_m_Y
  "endDate":           "31_03_2025"        // REQUIRED формат d_m_Y
}
```

```json
// Ответ 1С — успех (документ создан)
HTTP 201
{
  "document_number":    "АКТ-2025-001",
  "document_date":      "2025-03-31",
  "document_name":      "Акт звірки",
  "counterparty_login": "12345",
  "document_1cid":      "doc-001",
  "contract_number":    "ДГВ-2025-001",
  "contract_date":      "2025-01-01",
  "organization_name":  "ТОВ Ромашка",
  "doc_link":           "https://storage.virtoffice.ua/docs/doc-001.pdf"
}
```

---

### 3.4 Список организаций клиента

```http
GET https://mock.1c.virtoffice.ua/api/v2/organizations/{login}/
X-API-Key: {api_key}
```

```json
// Ответ 1С — успех
HTTP 200
{
  "OrganizationList": [
    {
      "organization_id":   "org-001",
      "organization_name": "ТОВ Ромашка",
      "contract_number":   "ДГВ-2025-001"
    },
    {
      "organization_id":   "org-002",
      "organization_name": "ТОВ Ромашка Логістика",
      "contract_number":   "ДГВ-2025-002"
    }
  ]
}
```

---

### 3.5 Уведомление 1С о новом корпоративном пользователе

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/
X-API-Key: {api_key}
Content-Type: application/json

{
  "cabinet_1c_id":  "cab-001",             // REQUIRED
  "email":          "accountant@company.ua", // REQUIRED
  "name":           "Петренко Петро",
  "phone":          "+380991234567",
  "role":           "Бухгалтер",
  "company_login":  "12345"
}
```

```json
// Ответ 1С — успех
HTTP 201
{
  "message": "User registered in 1C."
}
```

---

### 3.6 Запрос создания ТТН Нова Пошта

```http
POST https://mock.1c.virtoffice.ua/api/v2/delivery/ttn/
X-API-Key: {api_key}
Content-Type: application/json

{
  "message_id":       "msg-001",           // REQUIRED — ID письма
  "login":            "12345",             // REQUIRED
  "recipient_name":   "Іваненко Іван",     // REQUIRED
  "recipient_phone":  "+380991234567",     // REQUIRED
  "city_ref":         "db5c88d5-391c-11dd-90d9-001a92567626", // REQUIRED — ref НП
  "warehouse_ref":    "1ec09d88-e1c2-11e3-8c4a-0050568002cf"  // REQUIRED — ref НП
}
```

```json
// Ответ 1С — успех
HTTP 201
{
  "ttn_number":   "20450000000001",
  "ttn_link":     "https://novaposhta.ua/tracking/20450000000001"
}
```

---

## 4. Сводная таблица всех событий

### IN: 1С → Site

| event_type | Метод | Поле идемпотентности | Триггерит уведомление |
|---|---|---|---|
| `user.create` | POST /intake/ | `login` | Magic-link email |
| `user.update` | POST /intake/ | `login` | Нет |
| `cabinet.corporate.create` | POST /intake/ | `cabinet_1c_id` | Invite email мастеру |
| `corporate.user.invite` | POST /intake/ | `email + cabinet_1c_id` | Invite email |
| `contract.upsert` | POST /intake/ | `contract_id` | Нет |
| `letter.create` | POST /intake/ | `message_id` | Email + WebSocket |
| `letter.update` | POST /intake/ | `message_id` | Нет |
| `letter.status.update` | POST /intake/ | `message_id + status` | Email + WebSocket |
| `letter.scan.ready` | POST /intake/ | `message_id` | Email + WebSocket |
| `document.create` | POST /intake/ | `document_1cid` | Email |
| `document.update` | POST /intake/ | `document_1cid` | Нет |
| `document.delete` | POST /intake/ | `document_1cid` | Нет |
| `delivery.status.update` | POST /intake/ | `ttn_number + status` | Email + WebSocket |

### OUT: Site → 1С

| Действие | Метод | URL | Триггер |
|---|---|---|---|
| Проверка пользователя | GET | `/users/{login}/` | Регистрация, вход |
| Синхронизация профиля | POST | `/users/sync/` | Cron (Celery Beat) |
| Запрос акта сверки | POST | `/documents/reconciliation/` | Клиент в кабинете |
| Список организаций | GET | `/organizations/{login}/` | Открытие кабинета |
| Уведомить о новом сотруднике | POST | `/corporate/users/` | Приглашение принято |
| Создание ТТН | POST | `/delivery/ttn/` | Клиент заказал доставку |

---

## 5. Открытые вопросы

> Эти пункты требуют подтверждения от 1С разработчика перед production деплоем.

```
[ ] 1. Подтвердить production URL 1С API
[ ] 2. Подтвердить финальные значения API ключей
[ ] 3. Подтвердить список статусов письма (letter.status)
[ ] 4. Подтвердить формат counterparty_id (/ → _ замена — сохраняется?)
[ ] 5. Подтвердить retry логику на стороне 1С (кол-во попыток, интервалы)
[ ] 6. Добавить поле event_id от 1С (UUID) — возможно в будущей версии
[ ] 7. Подтвердить: 1С шлёт delivery.status.update или НП напрямую?
[ ] 8. Подтвердить формат дат в reconciliation запросе (d_m_Y актуален?)
```

---

## 6. История изменений

| Версия | Дата | Изменение |
|---|---|---|
| 2.0 | 2025-03 | Первая версия. Единая точка входа /intake/. Двусторонняя X-API-Key аутентификация. |

```

---

Сохрани как `api-contracts.md` в папку `02_Архитектура/` и скинь 1С разработчику на согласование раздел **5. Открытые вопросы** — это минимум что нужно закрыть до начала кодинга.
