[← Оглавление](README.md) · [§3 transport / intake](03-in-intake-envelope.md) · [§§1–2](01-general-and-flows.md)

> **§3.3** — договоры.

### 3.3 Договоры

**Модель:** договор 1:1 связан с клиентом (`login` = внешний код договора / контрагента в 1С, см. §1.3). Номер договора отдельным полем **не передаётся** — `login` сам является этим номером. Юридический адрес контрагента, нужный для документов и UI, хранится **только в договоре** (в `contract.`*), а не в карточке компании/кабинета.

#### `contract.create` — создать договор (только данные, без файла скана)

> **Атрибуты компании в payload договора не передаются.** `organization_name` задаётся на записи пользователя (**§3.1** `user.create` / `user.update`); сайт берёт его при отображении договора по `login`. Здесь — только реквизиты самого договора и юр. адрес контрагента.

```json
// Запрос от 1С
{
  "event_type": "contract.create",
  "payload": {
    "login":         "01/02/03",        // REQUIRED — он же номер договора
    "contract_id":   "cnt-001",         // REQUIRED — внешний id в 1С (стабильный ключ договора)
    "contract_date": "2026-01-01",      // REQUIRED
    "valid_from":    "2026-01-01",
    "valid_until":   "2027-01-01",
    "address": {                        // optional — юр. адрес контрагента (см. §1.3)
      "public": "вул. Хрещатик 1, Київ",
      "exact":  "вул. Хрещатик, буд. 1, оф. 5, м. Київ, 01001"
    }
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
    "login":         "01/02/03",        // REQUIRED
    "contract_id":   "cnt-001",         // REQUIRED
    "contract_date": "2026-02-01",
    "valid_from":    "2026-02-01",
    "valid_until":   "2028-01-01",
    "address": {
      "public": "вул. Хрещатик 2, Київ",
      "exact":  "вул. Хрещатик, буд. 2, оф. 12, м. Київ, 01001"
    }
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

> В `payload` перечисляются только **изменившиеся** поля + обязательные ключи `login`, `contract_id`. Семантика патча — общая (см. §1.3): отсутствующее поле не трогается, `null` — сброс.

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


