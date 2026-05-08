[← Оглавление](README.md) · [§3 transport / intake](03-in-intake-envelope.md) · [§§1–2](01-general-and-flows.md)

> **§3.5** и связанный **OUT §4.5** — документы и запрос акта сверки из кабинета.

### 3.5 Документы

#### `document.create` — создать документ

> **Атрибуты компании в payload документа не передаются.** `organization_name` задаётся на записи пользователя (**§3.1** `user.create` / `user.update`); сайт берёт его при отображении документа по `login`.

Для **счёта по заявке на продление договора** (§3.9 / OUT §4.8) 1С передаёт тот же `document.create`: PDF счёта (включая QR при необходимости — **внутри файла** по правилам 1С) — в `document_link`. Связь с пользовательской заявкой — поле **`extension_request_id`** (должно совпадать с `request_id`, который сайт отправил в `POST /contracts/extension/`). Пока поле задано, сайт считает документ счётом этого запроса и может показать его в интерфейсе вместо индикатора ожидания (см. §3.9). Дополнительные атрибуты продления — в **`extension_meta`** (опционально, рекомендуется при наличии `extension_request_id`).

```json
// Запрос от 1С
{
  "event_type": "document.create",
  "payload": {
    "document_id":     "doc-001",       // REQUIRED
    "document_number": "АКТ-2026-001",  // REQUIRED — номер самого документа (не договора)
    "document_date":   "2026-03-20",    // REQUIRED
    "document_name":   "Акт сверки",    // REQUIRED
    "login":           "01/02/03",      // REQUIRED — код контрагента/договора в 1С (связь с договором 1:1)
    "contract_date":   "2026-01-01",    // optional — дата договора, на который ссылается документ
    "document_link":    "https://storage.virtoffice.ua/docs/doc-001.pdf",
    "extension_request_id": "660e8400-e29b-41d4-a716-446655440002",  // optional — UUID заявки §4.8; для счёта на продление = request_id из OUT
    "extension_meta": {                  // optional — рекомендуется при extension_request_id (обновление виджета договора на сайте)
      "extended_months":   24,
      "new_valid_to_date": "2028-06-01",
      "amount":            18900.00,
      "currency":          "UAH"
    }
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

## OUT — Site → 1С (запрос акта сверки)

### 4.5 Запрос акта сверки

Запрос **асинхронный**: сайт фиксирует, что пользователь инициировал акт сверки, и сразу показывает в UI «Запрос получен, документ появится после формирования в 1С». Готовый акт возвращается обратно **отдельным** входящим событием `document.create` (§3.5), по которому сайт создаёт запись документа и, при необходимости, уведомление через §3.6. Синхронного ответа с телом документа в этом эндпоинте не предусмотрено.

```http
POST https://mock.1c.virtoffice.ua/api/v2/documents/reconciliation/
X-API-Key: {api_key}
Content-Type: application/json
```

```jsonc
{
  "login": "01/02/03",
  "date_from": "2026-01-01",
  "date_to": "2026-03-31",
  "request_id": "660e8400-e29b-41d4-a716-446655440002"
}
```

Поля `date_from` / `date_to` — календарные даты диапазона, формат `YYYY-MM-DD`.

**Ответ — заявка принята в работу (HTTP 202):**

```json
{
  "message": "Reconciliation request accepted.",
  "request_id": "660e8400-e29b-41d4-a716-446655440002"
}
```

**Ответ — идемпотентный повтор с тем же `request_id` (HTTP 200):**

```json
{
  "message": "Reconciliation request already accepted.",
  "idempotent": true
}
```

**Ответ — контрагент не найден в 1С (HTTP 404):**

```json
{
  "error": "counterparty_not_found",
  "message": "Counterparty not found in 1C",
  "login": "01/02/03"
}
```

> Идемпотентность — по `request_id`. 1С должна уметь отдать `202` повторно на тот же `request_id` без повторного формирования документа. Парной дедупликации «по окну дат» в контракте нет: если пользователь запросил те же даты ещё раз с новым `request_id`, это валидная повторная заявка.

---

