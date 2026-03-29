---

# `api-contracts-v3.md`

```markdown
## 1. Общие правила

### 1.1 Base URLs

| Сторона | Mock URL | Production URL |
|---|---|---|
| VirtOffice Site (intake) | `https://mock.virt-off.com.ua/api/v2` | `TBD` |
| API 1С (OUT с сайта) | `https://mock.1c.virtoffice.ua/api/v2` | `TBD` |


### 1.2 Аутентификация

Все запросы в обе стороны требуют заголовок:

```http
X-API-Key: {api_key}
```


| Ключ      | Значение                   | Где хранится              |
| --------- | -------------------------- | ------------------------- |
| Site → 1С | `mock-site-to-1c-key-2026` | `settings.ONE_C_API_KEY`  |
| 1С → Site | `mock-1c-to-site-key-2026` | `settings.INTAKE_API_KEY` |


**Переход с легаси:** до полного отключения старых эндпоинтов допускается **dual-read**: тот же ключ в заголовке `Authorization` (сырое значение, без префикса `Bearer`) **или** `X-API-Key`. Новые интеграции используют только `X-API-Key`.

Ответ при неверном ключе:

```json
HTTP 401
{
  "error": "Invalid or missing API key"
}
```

### 1.3 Формат запросов и ответов

- Все тела запросов и ответов: `Content-Type: application/json`
- **Дата-время** (момент во времени): только `ISO 8601` в UTC, с суффиксом `Z` → `"2026-03-20T10:00:00Z"`.
- **Календарная дата** (без времени суток): только `YYYY-MM-DD` → `"2026-03-20"` (договоры, документы, `issue_date`, диапазоны в §4.7).
- Кодировка: `UTF-8`
- Пустые поля: не включать в тело (опускать, не слать `null`)

**Именование кода контрагента в 1С:** единое поле `**login`** — внешний код договора / контрагента (не email и не учётная запись корп. пользователя). В контексте **компании в корп. кабинете** используется `**login`** (то же по смыслу, явная привязка к сущности «компания»). В телах ошибок для этого кода всегда то же имя, что в запросе (`login` или `login`).

### 1.4 Идемпотентность


| Тип события               | Поле идемпотентности                                                                                      |
| ------------------------- | --------------------------------------------------------------------------------------------------------- |
| Пользователь              | `login` / `email`                                                                                         |
| Письмо                    | `message_id`                                                                                              |
| Документ                  | `document_id`                                                                                             |
| Договор                   | `contract_id`                                                                                             |
| Корп. кабинет             | `cabinet_id`                                                                                              |
| Скан договора             | `contract_id + login`                                                                                     |
| OUT: ТТН в 1С (после НП)  | `message_id` — один номер ТТН на письмо; повтор с теми же `message_id` + `ttn_number` — без дубля в учёте |
| OUT: статус доставки в 1С | `ttn_number + status` (нормализованный код статуса)                                                       |
| OUT: запрос скана письма  | `request_id` (UUID с сайта) **или** дедупликация по `message_id` при отсутствии `request_id`              |
| IN: Укрпочта required     | `login` (+ опционально хеш `template_url` для смены файла)                                        |


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


| Код   | Значение                            |
| ----- | ----------------------------------- |
| `200` | Успешно обновлено                   |
| `201` | Успешно создано                     |
| `400` | Ошибка валидации входных данных     |
| `401` | Неверный или отсутствующий API ключ |
| `404` | Объект не найден                    |
| `409` | Конфликт — объект уже существует    |
| `500` | Внутренняя ошибка сервера           |


### 1.7 Миграция с легаси (PHP)

Текущий прод: отдельные скрипты вместо единого `POST /intake/`, без оболочки `event_type` + `payload`.


| Легаси                                                                                                              | Целевой контракт v3                                              |
| ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Ключ в заголовке `Authorization` (сырое значение)                                                                   | `X-API-Key` (или dual-read, см. §1.2)                            |
| `POST users.php`, `PUT users.php`, `POST messages.php`, `PUT messages.php`, `POST documents.php`, …                 | `POST .../intake/` с `event_type`                                |
| Проверка пользователя: `GET check_user.php?login=`                                                                  | `GET /users/{login}/` (OUT к 1С в контракте; в легаси — к сайту) |
| Список организаций: `POST organizationList.php` (form `login`), ответ `{ status, data }`, запрос к 1С по Basic Auth | `GET /organizations/{login}/` к API 1С                           |


**Поля пользователя:** при создании пользователя легаси ожидает `delivery_link` (ссылка на форму доставки в письмах). В новой реализации сайт общается напрямую с API Нова Пошта и отсылает в 1с информацию о запросе доставки и изменения ее статуса.

**Письма:** в легаси отсутствие пользователя по коду клиента возвращалось как **500**; в v3 — `404` с телом `{ "error": "Client not found", "login": "..." }` (см. `mail.create`).

---

## 2. User Flows — корпоративный кабинет

### Флоу 1 — менеджер регистрирует через 1С

```
Менеджер в 1С создаёт пользователя
        ↓
1С → Site:  user.create (event)
        ↓
Site: создаёт пользователя, шлёт magic-link на email
        ↓
Пользователь переходит по ссылке → попадает в кабинет
```

### Флоу 2 — пользователь регистрируется на сайте сам

```
Шаг 1: Самостоятельная регистрация
─────────────────────────────────────────────────────
Пользователь заполняет форму на сайте:
  email (REQUIRED), ФИО, телефон, пароль
        ↓
Site → 1С:  corporate.user.registered  ← уведомляем 1С о новом юзере
        ↓
Пользователь получает подтверждение email

Шаг 2: Приглашение в компанию
─────────────────────────────────────────────────────
Вариант А: мастер приглашает через интерфейс сайта
  Мастер вводит email → Site проверяет:
    ├── пользователь существует →
    │     добавляет в компанию сразу
    │     Site → 1С: corporate.user.linked
    │
    └── пользователь НЕ существует →
          Site шлёт magic-link приглашение на email
          пользователь регистрируется по ссылке →
          автоматически добавляется в компанию
          Site → 1С: corporate.user.registered
                      + corporate.user.linked

Вариант Б: менеджер приглашает через 1С
  1С → Site:  corporate.user.invite (event)
        ↓
  Site проверяет:
    ├── пользователь существует → добавляет в компанию
    └── пользователь НЕ существует → шлёт magic-link
```

---

## 3. IN: 1С → Site (входящие события)

### Endpoint

```http
POST https://mock.virt-off.com.ua/api/v2/intake/
X-API-Key: {api_key}
Content-Type: application/json
```

### Структура запроса

Обязательные поля верхнего уровня: `event_type`, `payload`. Дополнительно (для трассировки и идемпотентности на транспортном уровне):


| Поле          | Обяз. | Описание                                                                                                              |
| ------------- | ----- | --------------------------------------------------------------------------------------------------------------------- |
| `event_type`  | да    | Тип события                                                                                                           |
| `payload`     | да    | Тело события                                                                                                          |
| `event_id`    | нет   | UUID события от 1С; при повторной доставке с тем же `event_id` сайт отвечает успехом без повторного побочного эффекта |
| `occurred_at` | нет   | Время возникновения в 1С, ISO 8601                                                                                    |


```json
{
  "event_type": "mail.create",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "occurred_at": "2026-03-20T10:00:00Z",
  "payload": {}
}
```

---

### 3.1 Пользователи

#### `user.create` — создать пользователя (обычный кабинет)

```json
// Запрос от 1С
{
  "event_type": "user.create",
  "payload": {
    "login":               "01/02/03",        // REQUIRED — номер договора
    "password":            "hashedpwd",       // REQUIRED
    "name":                "ООО Ромашка",     // REQUIRED
    "phone_sms":           "+380991234567",
    "email":               "info@romashka.ua",
    "secondary_email":     "office@romashka.ua"
  }
}

// Ответ — создан
HTTP 201
{ "message": "Record successfully created.", "created": true }

// Ответ — пользователь с таким login уже есть (идемпотентный повтор)
HTTP 200
{ "message": "Already exists.", "created": false }

// Ответ — ошибка валидации
HTTP 400
{
  "error": "Missing required fields",
  "fields": ["login", "password"]
}
```

---

#### `user.update` — обновить пользователя

```json
// Запрос от 1С
{
  "event_type": "user.update",
  "payload": {
    "login":               "01/02/03",    // REQUIRED
    "name":                "ООО Ромашка 2",
    "password":            "newhashedpwd",
    "phone_sms":           "+380991234568",
    "email":               "new@romashka.ua",
    "secondary_email":     "new1@romashka.ua"
  }
}

// Ответ — успех
HTTP 200
{ "message": "Record successfully updated." }
```

---

### 3.2 Корпоративный кабинет

#### `corporate.cabinet.create` — создать корпоративный кабинет

```json
// Запрос от 1С
{
  "event_type": "corporate.cabinet.create",
  "payload": {
    "cabinet_id": "cab-001",         // REQUIRED
    "master_email":  "ceo@company.ua",  // REQUIRED
    "master_name":   "Иваненко Иван Иванович",
    "master_phone":  "+380991234567",
    "companies": [                       // REQUIRED — минимум одна
      {
        "login":             "01/02/03",
        "name":              "ООО Ромашка",
        "organization_name": "ООО Ромашка Груп",
        "scan_rule":         "all"       // "all" | "on_request" | "gov_only"
      }
    ]
  }
}

// Ответ — успех
HTTP 201
{
  "message":              "Cabinet created.",
  "cabinet_external_id":  "cab-001",
  "site_cabinet_id":      42,
  "invite_sent_to":       "ceo@company.ua"
}
```

---

#### `corporate.user.invite` — пригласить сотрудника через 1С

```json
// Запрос от 1С
{
  "event_type": "corporate.user.invite",
  "payload": {
    "cabinet_id": "cab-001",                // REQUIRED
    "email":         "accountant@company.ua",  // REQUIRED
    "role":          "Бухгалтер",              // REQUIRED
    "login": "01/02/03",               // REQUIRED
    "name":          "Пётр Петренко"
  }
}

// Ответ — успех
HTTP 201
{
  "message":     "Invite sent.",
  "invite_token": "uuid-xxx",
  "expires_at":  "2026-04-20T10:00:00Z"
}
```

---

#### `corporate.cabinet.update` — обновить данные корпоративного кабинета

Передача актуальных данных кабинета из 1С (реквизиты мастера, состав компаний и т.д.). **Отдельное** событие от `corporate.cabinet.create`.

```json
// Запрос от 1С
{
  "event_type": "corporate.cabinet.update",
  "payload": {
    "cabinet_id":    "cab-001",              // REQUIRED
    "master_email":  "ceo@company.ua",      // optional — передать, если изменилось
    "master_name":   "Иваненко Иван Иванович",
    "master_phone":  "+380991234567",
    "companies": [
      {
        "login":             "01/02/03",
        "name":              "ООО Ромашка",
        "organization_name": "ООО Ромашка Груп",
        "scan_rule":         "all"
      }
    ]
  }
}

// Ответ — успех
HTTP 200
{ "message": "Cabinet updated." }

// Ответ — кабинет не найден
HTTP 404
{
  "error": "Cabinet not found",
  "cabinet_id": "cab-001"
}
```

> Семантика списка `companies`: передаётся **полный актуальный набор** компаний, привязанных к кабинету в 1С (замена списка на сайте), либо согласованный с 1С вариант частичного патча — зафиксировать при внедрении.

---

#### `company.scan_rule.update` — обновить правило сканирования (из 1С)

```json
// Запрос от 1С
{
  "event_type": "company.scan_rule.update",
  "payload": {
    "login": "01/02/03",  // REQUIRED
    "scan_rule":     "gov_only"   // REQUIRED — "all"|"on_request"|"gov_only"
  }
}

// Ответ — успех
HTTP 200
{ "message": "Scan rule updated." }

// Ответ — компания не найдена
HTTP 404
{
  "error":        "Company not found",
  "login": "01/02/03"
}
```

---

### 3.3 Договоры

#### `contract.create` — создать договор (только данные, без файла скана)

```json
// Запрос от 1С
{
  "event_type": "contract.create",
  "payload": {
    "login":     "01/02/03",         // REQUIRED
    "contract_id":       "cnt-001",          // REQUIRED — внешний id в 1С
    "contract_number":   "ДГВ-2026-001",     // REQUIRED
    "contract_date":     "2026-01-01",       // REQUIRED
    "valid_from":        "2026-01-01",
    "valid_until":       "2027-01-01",
    "organization_name": "ООО Ромашка",
    "legal_address":     "ул. Крещатик 1, Киев"
  }
}

// Ответ — создан
HTTP 201
{ "message": "Contract created." }

// Ответ — договор с таким contract_id уже есть
HTTP 409
{
  "error": "Contract already exists",
  "contract_id": "cnt-001"
}
```

---

#### `contract.update` — обновить данные договора (без файла скана)

```json
// Запрос от 1С
{
  "event_type": "contract.update",
  "payload": {
    "login":     "01/02/03",         // REQUIRED
    "contract_id":       "cnt-001",          // REQUIRED
    "contract_number":   "ДГВ-2026-001-v2",
    "contract_date":     "2026-02-01",
    "valid_from":        "2026-02-01",
    "valid_until":       "2028-01-01",
    "organization_name": "ООО Ромашка Груп",
    "legal_address":     "ул. Крещатик 2, Киев"
  }
}

// Ответ — успех
HTTP 200
{ "message": "Contract updated." }

// Ответ — не найден
HTTP 404
{
  "error": "Contract not found",
  "contract_id": "cnt-001"
}
```

> В `payload` перечисляются только **изменившиеся** поля + обязательные ключи `login`, `contract_id` (или полный снимок полей — согласовать с 1С).

---

#### `contract.scan.upload` — загрузить / заменить скан договора (файл)

Отдельно от создания и обновления реквизитов. Повтор с новым `file_url` — новая версия скана.

```json
// Запрос от 1С
{
  "event_type": "contract.scan.upload",
  "payload": {
    "login": "01/02/03",                                              // REQUIRED
    "contract_id":   "cnt-001",                                               // REQUIRED
    "file_url":      "https://storage.virtoffice.ua/contracts/cnt-001.pdf"   // REQUIRED                                                    
  }
}

// Ответ — принято
HTTP 201
{ "message": "Contract scan uploaded." }

// Ответ — замена существующего скана
HTTP 200
{ "message": "Contract scan updated." }

// Ответ — компания или договор не найдены
HTTP 404
{
  "error": "Company or contract not found",
  "login": "01/02/03",
  "contract_id": "cnt-001"
}
```

---

### 3.4 Письма

#### `mail.create` — новое письмо

```json
// Запрос от 1С
{
  "event_type": "mail.create",
  "payload": {
    "message_id":      "msg-001",                                          // REQUIRED
    "login":           "01/02/03",                                         // REQUIRED — код клиента в 1С
    "scan_date":       "2026-03-20T10:00:00Z",                             // REQUIRED
    "attachment_link": "https://storage.virtoffice.ua/scans/msg-001.pdf",  // REQUIRED
    "issue_date":      "2026-03-18"
  }
}

// Ответ — успех
HTTP 201
{ "message": "Record successfully created." }

// Ответ — контрагент (клиент по login) не найден на сайте
HTTP 404
{
  "error": "Client not found",
  "login": "01/02/03"
}
```

---

#### `mail.update` — обновить письмо

```json
// Запрос от 1С
{
  "event_type": "mail.update",
  "payload": {
    "message_id":      "msg-001",          // REQUIRED
    "scan_date":       "2026-03-21T10:00:00Z",
    "attachment_link": "https://storage.virtoffice.ua/scans/msg-001-v2.pdf",
    "issue_date":      "2026-03-19"
  }
}

// Ответ — успех
HTTP 200
{ "message": "Record successfully updated." }
```

---

#### `mail.status.update` — обновить статус письма

```json
// Запрос от 1С
{
  "event_type": "mail.status.update",
  "payload": {
    "message_id":  "msg-001",               // REQUIRED
    "status":      "issued",                // REQUIRED
    "issued_to":   "Директор Иваненко",
    "issued_date": "2026-03-22T14:30:00Z"
  }
}

// Статусы (базовый enum; см. матрицу ниже и ТЗ §4.1.1):
// "received"  — получено на склад / новое поступление (ТЗ: Новое)
// "scanned"   — отсканировано
// "pending"   — ожидает выдачи / этап до НП (см. матрицу: «Заказана доставка»)
// "issued"    — выдано клиенту (ТЗ: Выдано в офисе)
// "delivery"  — отправлено НП (ТЗ: Отправлено НП)
// "disposed"  — утилизировано (ТЗ: Утилизировано) — расширение enum, согласовать с 1С
// "archived"  — архивировано (ТЗ: Архивировано) — расширение enum, согласовать с 1С

// Ответ — успех
HTTP 200
{ "message": "Status updated." }
```

---

#### `mail.scan.ready` — скан письма готов

```json
// Запрос от 1С
{
  "event_type": "mail.scan.ready",
  "payload": {
    "message_id":    "msg-001",                                               // REQUIRED
    "scan_file_url": "https://storage.virtoffice.ua/scans/msg-001-final.pdf"  // REQUIRED
  }
}

// Ответ — успех + уведомление клиенту
HTTP 200
{ "message": "Scan saved. Notification sent." }
```

#### Матрица статусов письма: ТЗ ↔ API ↔ источник

Ниже — соответствие **ТЗ** (п. 4.1.1 «статус письма», п. 4.1.2 «статус сканирования») полям и событиям контракта. Значения `status` в `mail.status.update`.

**Статус письма** 


| Статус в ТЗ               | `status` в `mail.status.update`         | Кто задаёт истину  | События / примечание                                                                                                                                                      |
| ------------------------- | --------------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Новое (поступление из 1С) | `received`                              | 1С                 | Первичное поступление: `mail.create`; при необходимости явное `mail.status.update`                                                                                        |
| Просмотрено               | *(внутреннее состояние сайта)*          | Сайт               | Действие пользователя в кабинете — **не** входящее событие 1С IN                                                                                                          |
| Заказана доставка         | `pending` (ожидает выдачи / этап до НП) | Сайт → 1С OUT      | После `POST /delivery/ttn/` и смены состояния; дубль ТТН запрещён (ТЗ п. 4.3)                                                                                             |
| Отправлено НП             | `delivery`                              | НП → сайт → 1С     | После события НП сайт обновляет письмо и шлёт в 1С **OUT** `POST /delivery/status/` (см. §4.10)                                                                           |
| Доставлено НП             | *(не дублировать в IN)*                 | API НП → сайт → 1С | Истина — **OUT** `POST /delivery/status/` (`status`, например `delivered`); опционально сайт обновляет внутренний статус письма без отдельного `mail.status.update` от 1С |
| Выдано в офисе            | `issued`                                | 1С                 | `mail.status.update` + `issued_to` / `issued_date`                                                                                                                        |
| Утилизировано             | `disposed`                              | 1С                 | Согласовать с 1С перед релизом                                                                                                                                            |
| Архивировано              | `archived`                              | 1С                 | Согласовать с 1С перед релизом                                                                                                                                            |


---

### 3.5 Документы

#### `document.create` — создать документ

```json
// Запрос от 1С
{
  "event_type": "document.create",
  "payload": {
    "document_id":       "doc-001",       // REQUIRED
    "document_number":   "АКТ-2026-001",  // REQUIRED
    "document_date":     "2026-03-20",    // REQUIRED
    "document_name":     "Акт сверки",    // REQUIRED
    "login":             "01/02/03",       // REQUIRED — код контрагента в 1С
    "contract_number":   "ДГВ-2026-001",
    "contract_date":      "2026-01-01",
    "organization_name":  "ООО Ромашка",
    "doc_link":           "https://storage.virtoffice.ua/docs/doc-001.pdf"
  }
}

// Ответ — успех
HTTP 201
{ "message": "Record successfully created." }

// Ответ — контрагент не найден
HTTP 404
{
  "error": "Counterparty not found",
  "login": "01/02/03"
}
```

### 3.6 Укрпошта — шаблон договора и уведомления

Сценарий: **1С** инициирует требование передать клиенту **шаблон договора** (файл с хранилища 1С / файлового сервера). Сайт показывает **закреплённое уведомление** в кабинете компании и даёт скачать шаблон по ссылке. Когда в 1С зафиксировано получение документов или требование отменено, **1С** шлёт событие **снятия** уведомления.

Привязка по `login`: на компанию одновременно одно активное требование; повтор `company.ukrposhta_docs.required` с тем же `login` обновляет шаблон/ссылку.

---

#### `company.ukrposhta_docs.required` — показать уведомление и выдать ссылку на шаблон

```json
// Запрос от 1С
{
  "event_type": "company.ukrposhta_docs.required",
  "payload": {
    "login": "01/02/03",                                              // REQUIRED
    "template_url":  "https://storage.virtoffice.ua/templates/contract.docx"
  }
}

// Ответ — создано закреплённое уведомление
HTTP 201
{ "message": "Ukrposhta docs requirement created.", "login": "01/02/03" }

// Ответ — обновление существующего требования для той же компании
HTTP 200
{ "message": "Ukrposhta docs requirement updated.", "login": "01/02/03" }

// Ответ — компания не найдена
HTTP 404
{
  "error": "Company not found",
  "login": "01/02/03"
}
```

---

#### `company.ukrposhta_docs.resolved` — снять уведомление (триггер из 1С)

Вызывается, когда в 1С **больше не нужно** показывать требование (документы получены, сценарий отменён и т.д.).

```json
// Запрос от 1С
{
  "event_type": "company.ukrposhta_docs.resolved",
  "payload": {
    "login": "01/02/03"
  }
}

// Ответ — уведомление снято
HTTP 200
{ "message": "Ukrposhta docs requirement cleared.", "login": "01/02/03" }

// Ответ — для компании нет активного требования
HTTP 404
{
  "error": "Ukrposhta requirement not found",
  "login": "01/02/03"
}
```

---

## 4. OUT: Site → 1С (исходящие запросы)

---

### 4.1 Проверка существования пользователя

```http
GET https://mock.1c.virtoffice.ua/api/v2/users/{login}/
X-API-Key: {api_key}
```

```json
// Ответ — найден
HTTP 200
{
  "exists":              true,
  "email":             "info@romashka.ua",
  "phone_sms":          "+380991234567",
  "secondary_email":   "info1@romashka.ua"
}

// Ответ — не найден
HTTP 200
{ "exists": false }
```

---

### 4.2 Синхронизация профиля пользователя

```http
POST https://mock.1c.virtoffice.ua/api/v2/users/sync/
X-API-Key: {api_key}
Content-Type: application/json

{
  "login":               "01/02/03",        // REQUIRED
  "email":               "info@romashka.ua",
  "f_phone":             "+380991234567",
  "secondary_email":     "info2@romashka.ua"
}
```

```json
// Ответ — успех
HTTP 200
{ "message": "User synced." }
```

---

### 4.3 Регистрация корпоративного пользователя на сайте

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/registered/
X-API-Key: {api_key}
Content-Type: application/json

{
  "email":  "user@company.ua",    // REQUIRED
  "name":   "Пётр Петренко",     // REQUIRED
  "phone":  "+380991234567"       // REQUIRED
}
```

```json
// Ответ — 1С принял
HTTP 201
{ "message": "Registration received." }
```

> Этот запрос шлётся сразу после самостоятельной регистрации,
> до привязки к компании. 1С получает информацию о новом юзере.

---

### 4.4 Привязка пользователя к компании

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/linked/
X-API-Key: {api_key}
Content-Type: application/json

{
  "email":         "user@company.ua",  // REQUIRED
  "login": "01/02/03",         // REQUIRED
  "cabinet_id":    "cab-001"           // REQUIRED
}
```

```json
// Ответ — успех
HTTP 201
{ "message": "User linked to company." }
```

> Шлётся в двух случаях:
>
> 1. Мастер добавил существующего пользователя через интерфейс сайта
> 2. Новый пользователь прошёл регистрацию по magic-link приглашению

---

### 4.5 Обновление правила сканирования (из кабинета)

```http
POST https://mock.1c.virtoffice.ua/api/v2/companies/scan-rule/
X-API-Key: {api_key}
Content-Type: application/json

{
  "login": "01/02/03",  // REQUIRED
  "scan_rule":     "on_request" // REQUIRED — "all"|"on_request"|"gov_only"
}
```

```json
// Ответ — успех
HTTP 200
{ "message": "Scan rule updated." }
```

> Триггер: мастер или сотрудник с правами изменил scan_rule в кабинете.
> Идемпотентность: если значение не изменилось — 1С игнорирует.

---

### 4.6 Запрос сканирования письма (сайт → 1С)

Триггер: клиент нажал «Заказать скан» в кабинете. Сайт фиксирует состояние и передаёт заявку в 1С для учёта и передачи в сервис-отдел.

```http
POST https://mock.1c.virtoffice.ua/api/v2/mails/scan-request/
X-API-Key: {api_key}
Content-Type: application/json
```


| Поле           | Обяз. | Описание                                                            |
| -------------- | ----- | ------------------------------------------------------------------- |
| `message_id`   | да    | Идентификатор письма (как в `mail.create`)                          |
| `login`        | да    | Код клиента в 1С (как в `mail.create`)                              |
| `request_id`   | нет   | UUID заявки на стороне сайта; **рекомендуется** для идемпотентности |
| `requested_at` | нет   | Время действия пользователя, ISO 8601                               |


**Идемпотентность:** повтор с тем же `request_id` → **200**, без второй заявки. Без `request_id` — дедупликация по `message_id` (второй запрос → **200** «уже зарегистрировано») либо **409** — зафиксировать с командой 1С.

```json
{
  "message_id": "msg-001",
  "login": "01/02/03",
  "request_id": "660e8400-e29b-41d4-a716-446655440001",
  "requested_at": "2026-03-20T12:00:00Z"
}
```

```json
// Ответ — заявка принята
HTTP 201
{
  "message": "Scan request accepted.",
  "request_id": "660e8400-e29b-41d4-a716-446655440001"
}

// Ответ — идемпотентный повтор
HTTP 200
{
  "message": "Scan request already registered.",
  "duplicate": true
}

// Ответ — письмо не найдено
HTTP 404
{
  "error": "Message not found",
  "message_id": "msg-001"
}

// Ответ — операция недопустима для текущего статуса
HTTP 409
{
  "error": "Scan request not allowed for current mail state",
  "message_id": "msg-001"
}
```

> После **201** на сайте выставляется статус сканирования «Скан заказан» (ТЗ §4.1.2); готовый файл — через IN `mail.scan.ready` / `mail.update`.

---

### 4.7 Запрос акта сверки

```http
POST https://mock.1c.virtoffice.ua/api/v2/documents/reconciliation/
X-API-Key: {api_key}
Content-Type: application/json

{
  "organization": "ООО Ромашка",
  "login":          "01/02/03",
  "date_from":      "2026-01-01",
  "date_to":        "2026-03-31"
}
```

Поля `date_from` / `date_to` — календарные даты диапазона, формат `YYYY-MM-DD` (как частный случай ISO 8601). Единый формат с остальным контрактом; **не** использовать отдельную схему `01_01_2026`.

```json
// Ответ — успех
HTTP 201
{
  "document_number":   "АКТ-2026-001",
  "document_date":     "2026-03-31",
  "document_name":     "Акт сверки",
  "login":             "01/02/03",
  "document_id":       "doc-001",
  "contract_number":   "ДГВ-2026-001",
  "contract_date":     "2026-01-01",
  "organization_name": "ООО Ромашка",
  "doc_link":          "https://storage.virtoffice.ua/docs/doc-001.pdf"
}
```

---

### 4.8 Список организаций клиента

```http
GET https://mock.1c.virtoffice.ua/api/v2/organizations/{login}/
X-API-Key: {api_key}
```

```json
// Ответ — успех
HTTP 200
{
  "organization_list": [
    {
      "organization_id":   "org-001",
      "organization_name": "ООО Ромашка",
      "contract_number":   "ДГВ-2026-001",
      "scan_rule":         "all"
    },
    {
      "organization_id":   "org-002",
      "organization_name": "ООО Ромашка Логистика",
      "contract_number":   "ДГВ-2026-002",
      "scan_rule":         "on_request"
    }
  ]
}
```

---

### 4.9 Уведомление 1С о новом корпоративном сотруднике (через 1С флоу)

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/
X-API-Key: {api_key}
Content-Type: application/json

{
  "cabinet_id": "cab-001",              // REQUIRED
  "email":         "accountant@company.ua", // REQUIRED
  "name":          "Пётр Петренко",
  "phone":         "+380991234567",
  "role":          "Бухгалтер",
  "login": "01/02/03"
}
```

```json
// Ответ — успех
HTTP 201
{ "message": "User registered in 1C." }
```

---

### 4.10 Доставка «Новая почта»: API НП на сайте, уведомление 1С

**Flow:**

1. Клиент оформляет доставку в кабинете.
2. **Сайт** вызывает **официальное API Новой почты** (справочники города/отделения, создание ЕН/ТТН и т.д.) — `city_ref`, `warehouse_ref` и другие поля используются **в запросах к НП**, не дублируются в контракте с 1С.
3. После **успешного** ответа НП (номер ТТН получен) сайт **асинхронно** уведомляет **1С**: `POST /delivery/ttn/` — чтобы в учёте были ТТН и связь с письмом/контрагентом.
4. Дальнейшие **изменения статуса** отправления сайт получает от НП (webhook, polling или смешанная схема — реализация) и при каждой существенной смене **уведомляет 1С**: `POST /delivery/status/`.

**1С не вызывает** API НП в этом сценарии; источник статусов доставки — **НП → сайт → 1С** (OUT).

#### 4.10.1 Уведомление 1С о созданной ТТН

Вызывается **после** успешного создания отправления в API НП.

```http
POST https://mock.1c.virtoffice.ua/api/v2/delivery/ttn/
X-API-Key: {api_key}
Content-Type: application/json

{
  "message_id":      "msg-001",                                    // REQUIRED — связь с письмом
  "login":           "01/02/03",                                   // REQUIRED
  "recipient_name":  "Иваненко Иван",                              // REQUIRED
  "recipient_phone": "+380991234567",                              // REQUIRED
  "ttn_number":      "20450000000001"                              // REQUIRED — из ответа НП
}
```

```json
// Ответ — успех (1С приняла событие)
HTTP 201
{
  "message": "TTN registered in 1C."
}
```

#### 4.10.2 Уведомление 1С об изменении статуса доставки

Вызывается сайтом, когда из API НП известен новый статус отправления (на каждое существенное изменение или с дедупликацией по `ttn_number` + `status` — согласовать с 1С).

```http
POST https://mock.1c.virtoffice.ua/api/v2/delivery/status/
X-API-Key: {api_key}
Content-Type: application/json

{
  "ttn_number":    "20450000000001",     // REQUIRED
  "status":        "delivered",          // REQUIRED — нормализованный код (согласовать с 1С; может соответствовать маппингу статусов НП)
  "message_id":    "msg-001",            // optional — если 1С ведёт связь письмо ↔ ТТН
  "login":         "01/02/03",           // optional
  "delivered_at":  "2026-03-22T14:00:00Z" // optional — в зависимости от статуса
}
```

```json
// Ответ — успех
HTTP 200
{ "message": "Delivery status synced to 1C." }
```

## 5. Сводная таблица всех событий

### IN: 1С → Site

См. также оболочку `POST /intake/` (§3): опциональные `event_id`, `occurred_at` для трассировки и транспортной идемпотентности.


| event_type                        | Описание                                                                 | Поле идемпотентности          | Уведомление                                 |
| --------------------------------- | ------------------------------------------------------------------------ | ----------------------------- | ------------------------------------------- |
| `user.create`                     | Создать пользователя в обычном (не корпоративном) кабинете               | `login`                       | Magic-link email                            |
| `user.update`                     | Обновить данные пользователя                                             | `login`                       | Нет                                         |
| `corporate.cabinet.create`        | Создать корпоративный кабинет компании                                   | `cabinet_id`                  | Invite email мастеру                        |
| `corporate.cabinet.update`        | Обновить данные корпоративного кабинета                                  | `cabinet_id`                  | Нет                                         |
| `corporate.user.invite`           | Пригласить сотрудника в компанию через 1С                                | `email + cabinet_id`          | Invite email                                |
| `company.scan_rule.update`        | Обновить правило сканирования писем (синхронизация из 1С)                | `login`               | Нет                                         |
| `contract.create`                 | Создать карточку договора (только данные, без файла скана)               | `contract_id`                 | Нет                                         |
| `contract.update`                 | Обновить данные договора (без файла скана)                               | `contract_id`                 | Нет                                         |
| `contract.scan.upload`            | Загрузить или заменить файл скана договора                               | `contract_id + login` | Настроенные каналы нотификаций              |
| `company.ukrposhta_docs.required` | Показать закреплённое уведомление и выдать ссылку на шаблон для Укрпочты | `login`               | Закреплённое уведомление + ссылка на шаблон |
| `company.ukrposhta_docs.resolved` | Снять уведомление о документах Укрпочты (триггер из 1С)                  | `login`               | Снять закрепление                           |
| `mail.create`                     | Новое письмо в кабинете                                                  | `message_id`                  | Настроенные каналы нотификаций              |
| `mail.update`                     | Обновить данные письма                                                   | `message_id`                  | Нет                                         |
| `mail.status.update`              | Обновить статус письма                                                   | `message_id + status`         | Уведомление на сайте                        |
| `mail.scan.ready`                 | Скан письма готов к просмотру                                            | `message_id`                  | Настроенные каналы нотификаций              |
| `document.create`                 | Создать документ в кабинете                                              | `document_id`                 | Настроенные каналы нотификаций              |


### OUT: Site → 1С


| Действие                     | Метод | URL                            | Триггер                                |
| ---------------------------- | ----- | ------------------------------ | -------------------------------------- |
| Проверка пользователя        | GET   | `/users/{login}/`              | Регистрация, вход                      |
| Синхронизация профиля        | POST  | `/users/sync/`                 | Celery Beat                            |
| Регистрация корп. юзера      | POST  | `/corporate/users/registered/` | Самостоятельная регистрация            |
| Привязка юзера к компании    | POST  | `/corporate/users/linked/`     | Добавление в компанию                  |
| Обновление scan_rule         | POST  | `/companies/scan-rule/`        | Клиент изменил в кабинете              |
| Заказ скана письма (ТЗ §4.2) | POST  | `/mails/scan-request/`         | Клиент нажал «Заказать скан»           |
| Запрос акта сверки           | POST  | `/documents/reconciliation/`   | Клиент в кабинете                      |
| Список организаций           | GET   | `/organizations/{login}/`      | Открытие кабинета                      |
| Уведомить о сотруднике       | POST  | `/corporate/users/`            | Флоу 1 через 1С                        |
| ТТН создано (после API НП)   | POST  | `/delivery/ttn/`               | После успешного вызова API Новой почты |
| Статус доставки (НП → сайт)  | POST  | `/delivery/status/`            | Изменение статуса ТТН со стороны НП    |


