[← Оглавление](README.md) · [§3 transport / intake](03-in-intake-envelope.md) · [§§1–2](01-general-and-flows.md)

> **§3.1–3.2** и связанные **OUT §4.1–§4.3, §4.9–§4.10** — пользователи и корпоративный кабинет (IN и Site → 1С в одном файле).

### 3.1 Пользователи

**Модель.** Запись пользователя по `login` — это и есть **индивидуальный кабинет компании** на сайте (один `login` — одна компания в учёте). Поля `name` (краткое отображаемое имя), `organization_name` (полное название организации) и `scan_rule` (правило сканирования писем) **не привязаны к корпоративному кабинету**: они живут на этой записи и одинаково используются, когда компания только в индивидуальном режиме и когда она **дополнительно** привязана к корп. кабинету через `corporate.company.attach` / `transfer`. Корп. кабинет задаёт только членство и видимость между пользователями (`cabinet_id`, access-снимки); атрибуты компании правятся через `user.create` / `user.update` (1С → Site) и исходящий `POST /users/sync/` (Site → 1С), см. §4.2.

#### `user.create` — создать пользователя (обычный кабинет)

Пароли **не передаются** между 1С и сайтом: первичный вход — через magic-link, который сайт генерирует сам и отправляет на `email` после успешного `user.create`. В 1С нет представления о пароле пользователя сайта.

```json
// Запрос от 1С
{
  "event_type": "user.create",
  "payload": {
    "login":               "01/02/03",        // REQUIRED — номер договора / код клиента
    "name":                "ООО Ромашка",     // REQUIRED — краткое имя в кабинете
    "organization_name":   "ООО Ромашка Груп", // optional — полное название; без поля или null → не задано
    "scan_rule":           "all",             // optional — "all" | "on_request" | "gov_only"; без поля или null → правило **не установлено** (первичную установку с сайта см. §4.2)
    "email":               "info@romashka.ua", // REQUIRED — нужен для magic-link
    "phone":               "+380991234567",
    "secondary_email":     "office@romashka.ua"
  }
}

// Ответ — создан (сайт отправит magic-link на email автоматически)
HTTP 201
{ "message": "Record successfully created.", "created": true }

// Ответ — пользователь с таким login уже есть (идемпотентный повтор)
HTTP 200
{ "message": "Already exists.", "created": false, "idempotent": true }

// Ответ — ошибка валидации
HTTP 400
{
  "error":   "missing_required_fields",
  "message": "Missing required fields",
  "fields":  ["login", "email"]
}
```

---

#### `user.update` — обновить пользователя

Семантика полей — по §1.3: отсутствующее поле не трогается, `null` — сброс. Пароли не передаются (сброс пароля — операция на стороне сайта через magic-link).

Здесь же передаются изменения `organization_name` и `scan_rule` с стороны 1С (менеджер); отдельного события только под правило сканирования нет — всё в одном `user.update`. После применения изменения `scan_rule` сайт генерирует внутреннее уведомление `scan_rule.changed_by_manager` (§3.6), если новое значение отличается от предыдущего.

Поле `changed_at` (ISO 8601 UTC) — **REQUIRED**: момент фиксации изменения в 1С; ключ **last-write-wins** относительно исходящего `POST /users/sync/` (§4.2) для того же `login`. Если входящее `changed_at` старее уже применённого на сайте — ответ `200` с `idempotent_stale: true`, локальное состояние не меняется.

```json
// Запрос от 1С
{
  "event_type": "user.update",
  "payload": {
    "login":             "01/02/03",              // REQUIRED
    "changed_at":        "2026-04-27T12:00:00Z",   // REQUIRED — LWW vs §4.2
    "name":              "ООО Ромашка 2",
    "organization_name": "ООО Ромашка Холдинг",
    "scan_rule":         "gov_only",
    "phone":             "+380991234568",
    "email":             "new@romashka.ua",
    "secondary_email":   "new1@romashka.ua"
  }
}

// Ответ — успех
HTTP 200
{ "message": "Record successfully updated." }

// Ответ — вход устарел (LWW)
HTTP 200
{ "message": "Stale user update ignored.", "idempotent_stale": true }

// Ответ — пользователь не найден
HTTP 404
{
  "error":   "user_not_found",
  "message": "User not found",
  "login":   "01/02/03"
}
```

---

## OUT — Site → 1С (пользователь и карточка компании по `login`)

### 4.1 Проверка существования пользователя

```http
GET https://mock.1c.virtoffice.ua/api/v2/users/{login}/
X-API-Key: {api_key}
```

**Ответ — найден (HTTP 200):**

```json
{
  "exists": true,
  "email": "info@romashka.ua",
  "phone": "+380991234567",
  "secondary_email": "info1@romashka.ua"
}
```

**Ответ — не найден (HTTP 200):**

```json
{ "exists": false }
```

---

### 4.2 Синхронизация пользователя / компании (Site → 1С)

Единый эндпоинт для правок из кабинета, которые должны отразиться в карточке контрагента в 1С:

- контактные поля (`email`, `phone`, `secondary_email`);
- атрибуты записи компании по `login` (`name`, `organization_name`, `scan_rule`) — та же модель, что в §3.1 (`user.create` / `user.update`).

Семантика тел запроса — **PATCH по §1.3**: отсутствующий ключ не трогается, `null` — явный сброс.

Поле **`changed_at`** (ISO 8601 UTC) — **обязательно**: момент фиксации набора изменений на сайте. Общий ключ **last-write-wins** относительно входящего **`user.update`** (§3.1) для того же `login`: более поздний `changed_at` побеждает; если этот запрос **старее** уже принятого в 1С изменения — ответ **`200`** с **`idempotent_stale: true`**, состояние в 1С не меняется.

Поле **`changed_by`** (email пользователя-инициатора на сайте) — **обязательно**, если в теле передано поле **`scan_rule`**; для остальных сценариев — optional (1С может записать историю с источником `"site"`).

```http
POST https://mock.1c.virtoffice.ua/api/v2/users/sync/
X-API-Key: {api_key}
Content-Type: application/json
```

```jsonc
{
  "login": "01/02/03",
  "changed_at": "2026-04-27T13:00:00Z",
  "changed_by": "user@company.ua",
  "email": "info@romashka.ua",
  "phone": "+380991234567",
  "secondary_email": "info2@romashka.ua",
  "name": "ООО Ромашка",
  "organization_name": "ООО Ромашка Груп",
  "scan_rule": "on_request"
}
```

Поля: `login`, `changed_at` — REQUIRED; `changed_by` — REQUIRED, если передан `scan_rule`, иначе optional; остальные — по необходимости (PATCH по §1.3).

**Ответ — успех (HTTP 200, данные приняты в 1С):**

```json
{ "message": "User synced." }
```

**Ответ — устаревший вызов (HTTP 200, LWW):**

```json
{ "message": "Stale user sync ignored.", "idempotent_stale": true }
```

**Ответ — контрагент / пользователь не найден (HTTP 404):**

```json
{
  "error": "user_not_found",
  "message": "User not found",
  "login": "01/02/03"
}
```

**Ответ — первичная установка `scan_rule` с сайта запрещена (HTTP 409):**

```json
{
  "error": "scan_rule_not_initialized",
  "message": "Scan rule has never been set by manager; site cannot initialize it. Send request via office@virt-of.com.ua",
  "login": "01/02/03"
}
```

**Правило `scan_rule`:**

- Пока **`scan_rule`** не задано (`null`, в т.ч. после **`user.create`** без правила), селектор в кабинете **недоступен**. UI собирает контактные данные и сайт шлёт письмо на **`office@virt-of.com.ua`** (SMTP **вне этого контракта**). После того как менеджер завёл правило в 1С, оно приходит на сайт через **`user.update`** с полем **`scan_rule`** и тем же механизмом LWW по **`changed_at`**.
- После того как правило **установлено**, клиент может менять его **сам** — через этот эндпоинт (поля **`scan_rule`**, **`changed_at`**, **`changed_by`**).
- Самостоятельное изменение клиентом **не** порождает **`notification.push`** другим членам кабинета (локальный UI / бейдж — по усмотрению сайта). Изменение менеджером через **`user.update`** может породить внутренний код **`scan_rule.changed_by_manager`** (§3.6).

**История в 1С:** при успешном применении фиксируется источник **`"site"`** и, когда передано, **`changed_by`**; изменения из 1С через **`user.update`** — источник **`"1c"`**.

**Жизненный цикл (кратко):**

```
1С → Site: user.create (опц. scan_rule сразу)
при корп. кабинете:
1С → Site: corporate.company.attach (только cabinet_id + login + attached_at)

Если scan_rule всё ещё null:
  клиент → форма → SMTP office@... (вне контракта)
  менеджер → в 1С заводит правило → user.update (scan_rule, changed_at) → Site

Дальше правки с любой стороны:
  Site → 1С: POST /users/sync/   или   1С → Site: user.update
  конфликты — LWW по changed_at (§1.4)
```

---


### 3.2 Корпоративный кабинет

#### Принципы устройства корпоративного кабинета

Контракт исходит из следующей модели (она формирует ожидания и для §3.2, и для §3.6, и для §4.3 / §4.9):

**Сущности:**

- **Cabinet** — корпоративный кабинет, идентифицируется `cabinet_id`. Внутри кабинета — набор компаний (юр. лиц), каждая со своим `login`.
- **Company** — юр. лицо в учёте, идентифицируется **`login`**. Поля **`name`**, **`organization_name`** и **`scan_rule`** хранятся на записи пользователя (**§3.1**, `user.create` / `user.update`), а не на корп. кабинете. Принадлежность к корп. кабинету — отдельная связь (`corporate.company.attach` / `transfer` / `detach`); все остальные сущности контракта (договор, прайс, письма, документы, уведомления) ссылаются на компанию через `login`.
- **User** — глобальная сущность, идентифицируется по `email` (уникален в системе). Один и тот же `email` может состоять в **нескольких** кабинетах одновременно (например, штатный бухгалтер обслуживает компании двух разных корпоративных клиентов).
- **Membership** — связь `(cabinet_id × email)`. Это и есть единица идемпотентности приглашения / привязки. У одного `email` может быть N membership'ов; разные кабинеты — независимы.

**Роли:**

- В каждом кабинете **ровно один мастер** (`master_email` в `corporate.cabinet.create` / `corporate.cabinet.update`). Передача мастерства — через `corporate.cabinet.update` с новым `master_email`: бывший мастер **полностью отвязывается** от кабинета (его membership удаляется, access-список — тоже). Если бывшего мастера нужно оставить в кабинете в роли сотрудника, его потом отдельно приглашают через `corporate.user.invite`.
- Все остальные membership'ы — **обычные сотрудники** (`member`). Мастер кабинета всегда имеет полный доступ ко всем компаниям своего кабинета.
- **Роль закреплена за membership-ем, не за пользователем.** Один и тот же `email` может одновременно быть мастером в одних кабинетах и сотрудником (`member`) в других — это два независимых membership-я с разными ролями. Контракт не накладывает на это ограничений.

**Доступы сотрудников к компаниям внутри кабинета (двусторонняя синхронизация):**

- **Двусторонняя связь.** Видимость компаний для membership-я редактируется обеими сторонами: на сайте — мастером в админке, в 1С — менеджером в карточке сотрудника. Каждое изменение шлётся встречной стороне:
  - **Site → 1С:** `POST /corporate/users/access/` (§4.9);
  - **1С → Site:** IN-событие `corporate.user.access.update` (см. §3.2 ниже).
  Обе стороны хранят локальный список и применяют входящие снимки по правилу **last-write-wins по `synced_at`**: если входящий снимок старше уже применённого, он игнорируется (`200 idempotent_stale`). Возможна короткая eventual-consistency-расхождение между правкой и отражением её на встречной стороне.
- **Master не синхронизируется отдельно.** Мастер по определению имеет доступ ко **всем** компаниям своего кабинета — обе стороны выводят это из `master_email` без отдельной записи. Ни сайт, ни 1С не шлют access-snapshot для текущего мастера; попытка отправить → `409 master_access_immutable`.
- **Стартовый список — пустой (0 компаний).** При создании membership-я начальная видимость **не подразумевается дефолтом «все компании»**. Конкретный стартовый список задаётся явно тем, кто инициирует приглашение:
  1. **1С** — через optional поле `company_logins[]` в `corporate.user.invite` (см. ниже);
  2. **Сайт** — мастер выбирает компании в диалоге приглашения, и сайт сразу после `/corporate/users/sync/` (§4.3) шлёт `POST /corporate/users/access/` (§4.9) с этим списком.
  Если ни одна сторона начальный список не задала, membership создаётся с пустым списком — сотрудник видит UI кабинета, но без компаний внутри, пока кто-то из сторон не пришлёт первый снимок.
- **Каскадов между сторонами нет.** Каждая сторона:
  - на собственные изменения списков → шлёт встречной стороне снимок;
  - при выходе компании из кабинета (через `corporate.company.transfer` или `corporate.company.detach`) автоматически удаляет этот `login` из всех своих локальных access-списков **локально, без обращения к встречной стороне** (обе стороны делают то же самое, конечная картина согласована);
  - при добавлении компании в кабинет (через `corporate.company.attach` или `corporate.company.transfer`) — никаких автоматических действий с access-списками (default = 0; новый `login` появится в списке только после явного снимка от мастера сайта или менеджера 1С).
- **Передача мастерства.** При `corporate.cabinet.update` с новым `master_email` бывший мастер **полностью отвязывается** от кабинета: его membership и access-список удаляются на обеих сторонах. Если он должен остаться в кабинете в роли сотрудника — отдельный invite через `corporate.user.invite` со стартовым `company_logins[]`. См. также §3.2 «Смена мастера кабинета».
- **Fan-out уведомлений** (§3.6) сайт рассчитывает по **локальной** копии access-списков (после применения LWW). Если IN-снимок ещё в пути — fan-out временно идёт по предыдущей версии; это допустимая eventual consistency.

**Что значит «1С приглашает сотрудника»:**

- В `corporate.user.invite` 1С передаёт email + (опционально) **стартовый список компаний** в поле `company_logins[]`. Если поле передано — обе стороны после обработки события согласованы по этому списку, и сайт **не** делает обратного `POST /corporate/users/access/` сразу после invite.
- Если `company_logins[]` не передан или пустой — membership создаётся с пустым списком; сотрудник видит UI кабинета, но без компаний. Любая из сторон позже может прислать снимок, чтобы открыть видимость.
- Дальнейшие правки видимости — синхронизируются в обе стороны через access-snapshot: OUT `POST /corporate/users/access/` (§4.9) и IN `corporate.user.access.update` (см. §3.2 ниже).

**Идентификация в событиях:**

- Membership-events (`corporate.user.invite`, OUT `/corporate/users/sync/`) идентифицируются парой `cabinet_id + email`.
- Cabinet-events (`corporate.cabinet.create`, `corporate.cabinet.update`) — по `cabinet_id`.
- Company-events (договор, прайс, скан, письма, документы, уведомления) — всегда по `login`. Контекст кабинета восстанавливает сайт самостоятельно, исходя из того, в какой кабинет входит этот `login`.

---

#### `corporate.cabinet.create` — создать корпоративный кабинет

Создаёт **только** кабинет с реквизитами мастера. Компании в кабинет добавляются **отдельными** событиями `corporate.company.attach` (см. §3.2 ниже) — это согласуется с принципом «одно событие = одна операция». Для каждого такого `login` на сайте к этому моменту уже должна существовать запись пользователя (`user.create` в §3.1); затем 1С отправляет `corporate.company.attach` для каждой компании, которую нужно поместить в этот кабинет.

```json
// Запрос от 1С
{
  "event_type": "corporate.cabinet.create",
  "payload": {
    "cabinet_id":   "cab-001",                     // REQUIRED
    "master_email": "ceo@company.ua",              // REQUIRED
    "master_name":  "Иваненко Иван Иванович",      // optional
    "master_phone": "+380991234567"                // optional
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

// Ответ — кабинет с этим cabinet_id уже создан (идемпотентный повтор)
HTTP 200
{ "message": "Cabinet already exists.", "cabinet_external_id": "cab-001", "idempotent": true }

// Ответ — устаревший вызов с попыткой передать атрибуты компаний прямо в cabinet.create
HTTP 400
{
  "error":   "companies_field_not_allowed",
  "message": "Use corporate.company.attach to add companies to the cabinet after creation",
  "fields":  ["companies"]
}
```

> **Жизненный цикл:** до прихода первого `corporate.company.attach` кабинет существует, но «пустой» — мастер по invite-ссылке зайдёт и увидит UI-каркас без компаний. Это нормальное промежуточное состояние; первые `attach` обычно следуют сразу после `create`.

---

#### `corporate.user.invite` — пригласить сотрудника через 1С

Создаёт membership `(cabinet_id, email)` и отправляет invite-email на регистрацию. Стартовая видимость компаний задаётся **явно** в поле `company_logins[]` (см. «Принципы устройства корпоративного кабинета» в начале §3.2) Если поле не передано или пустое, membership создаётся с пустым access-списком.

```json
// Запрос от 1С
{
  "event_type": "corporate.user.invite",
  "payload": {
    "cabinet_id":     "cab-001",                       // REQUIRED — кабинет, в который приглашаем
    "email":          "accountant@company.ua",         // REQUIRED — может уже состоять в других кабинетах; контракт это допускает
    "name":           "Пётр Петренко",                 // optional — отображаемое имя; сайт может подменить, если пользователь уже зарегистрирован
    "phone":          "+380991112233",                 // optional
    "company_logins": ["01/02/03", "01/02/04"]         // optional — стартовая видимость компаний кабинета; пусто/нет → пустой список (см. §3.2)
  }
}

// Ответ — приглашение создано
HTTP 201
{
  "message":      "Invite sent.",
  "invite_token": "uuid-xxx",
  "expires_at":   "2026-04-20T10:00:00Z"
}

// Ответ — пользователь уже состоит в этом кабинете (idempotent)
HTTP 200
{
  "message":    "User already in cabinet, invite skipped.",
  "idempotent": true
}
```

**Особые случаи:**

- Если `email` уже зарегистрирован на сайте (как обычный пользователь либо как член другого кабинета) — **новый аккаунт не создаётся**, добавляется membership к существующему пользователю. Invite-email при этом всё равно отправляется (как уведомление о новом доступе) и содержит ссылку на нужный кабинет.
- Если `email` уже состоит ровно в этом же `cabinet_id` (повтор события) — `200 idempotent`, повторное письмо не шлём; новый `company_logins[]`, если он отличается от уже хранящегося, **применяется** так же, как если бы пришёл `corporate.user.access.update` (см. §3.2 ниже): полная замена списка с проверкой LWW.
- Поля `name` / `phone` для уже зарегистрированного пользователя **не перезаписывают** его профиль (профиль — собственность пользователя). Они используются только для отображения в кабинете-приглашающем, если пользователь сам ничего не заполнял.
- `company_logins[]` валидируется: каждый `login` обязан быть актуальной компанией этого `cabinet_id`. Иначе → `400 invalid_company_logins` (см. §4.9). После успешной обработки сайт **не** делает обратного `POST /corporate/users/access/` — стартовый список уже синхронизирован.
- Если `company_logins[]` не передан или пустой — membership создаётся с пустым списком. Любая из сторон позже может прислать снимок (через §4.9 / `corporate.user.access.update`), чтобы открыть видимость.

---

#### `corporate.cabinet.update` — обновить реквизиты кабинета и/или мастера

Только реквизиты **самого кабинета** и **мастера**. Управление списком компаний кабинета — через специализированные события: `corporate.company.attach` (привязать существующий `login`), `corporate.company.transfer`, `corporate.company.detach`. Атрибуты компании (`name`, `organization_name`, `scan_rule`) — только **§3.1** (`user.create` / `user.update`) и **§4.2** (`POST /users/sync/`).

```json
// Запрос от 1С
{
  "event_type": "corporate.cabinet.update",
  "payload": {
    "cabinet_id":    "cab-001",                         // REQUIRED
    "master_email":  "ceo@company.ua",                  // optional — передать, если изменилось (передача мастерства)
    "master_name":   "Иваненко Иван Иванович",          // optional
    "master_phone":  "+380991234567"                    // optional
  }
}

// Ответ — успех
HTTP 200
{ "message": "Cabinet updated." }

// Ответ — нет ни одного изменяемого поля (по сути no-op)
HTTP 200
{ "message": "Cabinet update is no-op.", "idempotent": true }

// Ответ — кабинет не найден
HTTP 404
{
  "error":      "cabinet_not_found",
  "message":    "Cabinet not found",
  "cabinet_id": "cab-001"
}

// Ответ — устаревший вызов с попыткой управлять компаниями кабинета
HTTP 400
{
  "error":   "companies_field_not_allowed",
  "message": "Use corporate.company.attach / .transfer / .detach for cabinet membership; user attributes via §3.1 / §4.2",
  "fields":  ["companies"]
}
```

> Семантика частичного PATCH-а — по §1.3: отсутствие поля → не трогать, `null` → сбросить. Поле `companies[]` в payload **не допускается** (отдаём `400`, см. выше).

**Смена мастера кабинета:**

- Изменение `master_email` в этом событии — единственный способ передать мастерство.
- **Бывший мастер полностью отвязывается** от кабинета: его membership удаляется, access-список (если 1С его держала) удаляется. Сотрудником кабинета он автоматически не становится. Если он должен в этом кабинете остаться в роли сотрудника, его потом отдельно приглашают через `corporate.user.invite` со стартовым `company_logins[]`.
- Если новый `master_email` ещё не зарегистрирован на сайте — сайт отправляет ему invite-email (по логике `corporate.user.invite`). До завершения регистрации новый мастер не может управлять кабинетом; бывший мастер уже потерял доступ — на этот переходный период менеджер 1С может вернуть мастерство обратно отдельным `corporate.cabinet.update`.
- Если в кабинете и так был только мастер, смена `master_email` на нового — то же поведение: новый получает invite, старый отвязывается.
- **Access-список бывшего мастера** между сторонами не синхронизируется (он удалён вместе с membership-ем). Для **нового** мастера access-список не ведётся (см. §3.2 «Принципы устройства»).

---

#### `corporate.company.attach` — привязать компанию (`login`) к существующему кабинету (1С → Site)

Только **членство в корп. кабинете**: сайт отмечает, что данный `login` входит в `cabinet_id`. Поля **`name`**, **`organization_name`** и **`scan_rule`** сюда **не передаются** — они живут на записи пользователя (**§3.1**, `user.create` / `user.update`) и уходят в 1С через **`POST /users/sync/`** (**§4.2**).

Запись с этим `login` на сайте должна уже существовать (как правило после `user.create` для нового клиента). Если пользователя нет → `404 user_not_found` (сначала создайте пользователя, затем привязывайте к кабинету).

При создании корп. кабинета 1С шлёт `corporate.cabinet.create`, затем по одному `corporate.company.attach` для каждого `login`, который должен войти в этот кабинет. Если компания уже в **другом** кабинете — `409 company_in_other_cabinet`, нужен `corporate.company.transfer`. Переход из индивидуального режима «без компании в другом кабинете» — обычный `attach` после `user.create`.

```json
// Запрос от 1С
{
  "event_type": "corporate.company.attach",
  "payload": {
    "cabinet_id":   "cab-001",                     // REQUIRED — кабинет
    "login":        "01/02/03",                    // REQUIRED — код контрагента в 1С (он же номер договора, см. §3.3)
    "attached_at":  "2026-04-27T12:00:00Z"        // REQUIRED — момент привязки в 1С
  }
}

// Ответ — компания привязана
HTTP 201
{ "message": "Company attached.", "created": true }

// Ответ — компания уже в этом кабинете (повтор события)
HTTP 200
{ "message": "Company already in cabinet.", "created": false, "idempotent": true }

// Ответ — кабинет не найден
HTTP 404
{ "error": "cabinet_not_found", "cabinet_id": "cab-001" }

// Ответ — пользователя с таким login ещё нет на сайте
HTTP 404
{
  "error":   "user_not_found",
  "message": "Create user via user.create before attaching to a corporate cabinet",
  "login":   "01/02/03"
}

// Ответ — компания уже привязана к другому кабинету (используйте transfer)
HTTP 409
{
  "error":              "company_in_other_cabinet",
  "message":            "Company is already attached to a different cabinet; use corporate.company.transfer",
  "login":              "01/02/03",
  "current_cabinet_id": "cab-099"
}
```

**Семантика и правила:**

- **Идемпотентность:** пара `cabinet_id` + `login`. Повтор для уже привязанной к **тому же** кабинету компании → `200 idempotent`, без побочных эффектов.
- `attach` не создаёт пользователя и не задаёт атрибуты компании — только связь с кабинетом.
- **Каскадных действий с access-списками нет.** По общему правилу «стартовая видимость = 0 компаний» новый `login` в кабинете не появляется ни у одного сотрудника автоматически. Видимость задаёт мастер сайта (через §4.9) или менеджер 1С (через `corporate.user.access.update`).
- **Связанные сущности.** Договор / прайс / письма / документы / уведомления для этого `login` приходят отдельными событиями (`contract.create`, `pricing.upsert`, `mail.create`, `document.create`, `notification.push`); `corporate.company.attach` сам по себе их не создаёт.

---

#### `corporate.user.access.update` — обновить видимость компаний у сотрудника (1С → Site)

IN-копия OUT-эндпоинта §4.9. Менеджер 1С изменил список компаний, видимых сотрудником в кабинете → 1С шлёт снимок на сайт. Полная замена списка по `(cabinet_id, email)`. Связь двусторонняя — детальная семантика (LWW по `synced_at`, master-исключение, валидация `company_logins`) описана в §3.2 «Принципы устройства корпоративного кабинета» и §4.9.

```json
// Запрос от 1С
{
  "event_type": "corporate.user.access.update",
  "payload": {
    "cabinet_id":     "cab-001",                         // REQUIRED
    "email":          "accountant@company.ua",           // REQUIRED — сотрудник; для текущего мастера событие не применяется (см. ниже 409)
    "company_logins": ["01/02/03", "01/02/04"],          // REQUIRED — полный текущий список (snapshot); пустой массив = доступа нет ни к одной компании
    "synced_at":      "2026-04-27T12:00:00Z"             // REQUIRED — момент изменения в 1С (UTC); ключ LWW
  }
}

// Ответ — снимок применён (или идемпотентный повтор того же содержимого)
HTTP 200
{ "message": "Access snapshot applied." }

// Ответ — входящий снимок старше уже применённого (LWW)
HTTP 200
{ "message": "Stale snapshot ignored.", "idempotent_stale": true }

// Ответ — кабинет / membership не найдены
HTTP 404
{
  "error":      "membership_not_found",
  "message":    "User is not a member of this cabinet on site",
  "cabinet_id": "cab-001",
  "email":      "accountant@company.ua"
}

// Ответ — попытка задать список для текущего мастера кабинета
HTTP 409
{
  "error":      "master_access_immutable",
  "message":    "Master always has full cabinet access; access snapshot is not applicable",
  "cabinet_id": "cab-001",
  "email":      "ceo@company.ua"
}

// Ответ — login из списка не входит в этот кабинет
HTTP 400
{
  "error":          "invalid_company_logins",
  "message":        "Some logins are not part of this cabinet",
  "invalid_logins": ["01/99/99"]
}
```

**Семантика:** идентична OUT-эндпоинту §4.9 в обратном направлении — snapshot, идемпотентность по `(cabinet_id, email)`, LWW по `synced_at`. После применения сайт пересчитывает fan-out уведомлений (§3.6) для затронутого сотрудника по обновлённому списку.

---

#### `corporate.company.transfer` — перевод компании между кабинетами (1С → Site)

Атомарно переносит `login` из кабинета `from_cabinet_id` в кабинет `to_cabinet_id`. Все связанные сущности (договор, прайс, скан, письма, документы, уведомления) остаются привязанными к `login` и никак не трогаются.

**Видимость компаний (`access`) этим событием не задаётся:** после перевода ни у одного сотрудника целевого кабинета компания автоматически не появляется в списке (правило «стартовая видимость = 0», как при `attach`). Удаление из access-списков выполняется только для **старого** кабинета (каскад при выходе компании). Открыть видимость в `to_cabinet_id` — отдельно через `corporate.user.access.update` или OUT `POST /corporate/users/access/` (§4.9).

```json
// Запрос от 1С
{
  "event_type": "corporate.company.transfer",
  "payload": {
    "login":            "01/02/03",                      // REQUIRED — переносимая компания
    "from_cabinet_id":  "cab-001",                       // REQUIRED — текущий кабинет (для контроля)
    "to_cabinet_id":    "cab-002",                       // REQUIRED — целевой кабинет
    "transferred_at":   "2026-04-27T12:00:00Z"           // REQUIRED — момент перевода в 1С
  }
}

// Ответ — перевод применён (или идемпотентный повтор)
HTTP 200
{ "message": "Company transferred." }

// Ответ — кабинет не найден (любой из from / to)
HTTP 404
{
  "error":      "cabinet_not_found",
  "message":    "Cabinet not found",
  "cabinet_id": "cab-001"
}

// Ответ — login не находится в from_cabinet_id (мог быть уже перенесён или находится в другом кабинете)
HTTP 404
{
  "error":           "company_not_in_cabinet",
  "message":         "Company is not in the source cabinet",
  "login":           "01/02/03",
  "from_cabinet_id": "cab-001"
}

// Ответ — login уже находится в to_cabinet_id (идемпотентный повтор перевода)
HTTP 200
{ "message": "Company already in target cabinet.", "idempotent": true }
```

**Семантика на сайте:**

- Локально удаляем `login` из `from_cabinet_id` и каскадно из всех access-списков сотрудников этого кабинета (см. §3.2 «Принципы устройства» — обе стороны делают это симметрично).
- Локально добавляем `login` в состав компаний целевого кабинета **без** правок access-списков в `to_cabinet_id`: сотрудники нового кабинета этот `login` не получают в видимость, пока кто-то не пришлёт снимок доступа отдельно (§4.9 / `corporate.user.access.update`).
- Идемпотентность: ключ `(login, to_cabinet_id)`. Повтор события (login уже в `to_cabinet_id`) → `200 idempotent`, побочных эффектов нет.
- Атомарность: на сайте обработка события — одна транзакция; либо оба эффекта (удаление из исходного кабинета с каскадом access, добавление в целевой кабинет) проходят, либо ни один.

---

#### `corporate.company.detach` — отвязка компании от кабинета в индивидуальный режим (1С → Site)

Компания выводится из корп. кабинета и становится **самостоятельной**: у неё появляется индивидуальный пользователь с собственным паролем (по обычному флоу личного кабинета). Связанные сущности (договор, прайс, письма, документы) остаются привязанными к `login` и переходят в индивидуальный кабинет вместе с компанией.

```json
// Запрос от 1С
{
  "event_type": "corporate.company.detach",
  "payload": {
    "login":           "01/02/03",                       // REQUIRED — отвязываемая компания
    "from_cabinet_id": "cab-001",                        // REQUIRED — текущий кабинет (для контроля)
    "owner_email":     "owner@company.ua",               // REQUIRED — email, на который сайт пошлёт magic-link установки пароля
    "owner_name":      "Иваненко Иван",                  // optional — для письма
    "owner_phone":     "+380991234567",                  // optional
    "detached_at":     "2026-04-27T12:00:00Z"            // REQUIRED — момент отвязки в 1С
  }
}

// Ответ — компания отвязана; magic-link установки пароля отправлен на owner_email
HTTP 200
{ "message": "Company detached; password setup link sent.", "owner_email": "owner@company.ua" }

// Ответ — кабинет не найден
HTTP 404
{ "error": "cabinet_not_found", "cabinet_id": "cab-001" }

// Ответ — login не в from_cabinet_id
HTTP 404
{
  "error":           "company_not_in_cabinet",
  "message":         "Company is not in the source cabinet",
  "login":           "01/02/03",
  "from_cabinet_id": "cab-001"
}

// Ответ — компания уже в индивидуальном режиме (idempotent)
HTTP 200
{ "message": "Company already detached.", "idempotent": true }
```

**Семантика на сайте:**

- Локально удаляем `login` из `from_cabinet_id` и каскадно из всех access-списков сотрудников этого кабинета.
- Создаём индивидуального пользователя для `login` с `owner_email` (эквивалент `user.create`, см. §3.1) — без передачи пароля; пароль клиент задаёт сам.
- Шлём на `owner_email` **magic-link установки пароля**: одноразовая ссылка, переход по которой открывает форму установки нового пароля для входа в личный кабинет компании. После установки клиент логинится по обычной паре (`login` или `email` + пароль) — флоу обычного индивидуального кабинета.
- Если на сайте уже существует индивидуальный пользователь для `login` (например, компания когда-то была независимой), сайт **не** пересоздаёт его, но **обязательно** шлёт повторный magic-link на новый `owner_email` (1С — источник актуального контактного email-а; могла поменяться).
- Идемпотентность: ключ `login`. Повтор события без изменений → `200 idempotent`, новый magic-link **не** шлётся (защита от спама). Если в повторе изменился `owner_email` — magic-link отправляется на новый email.
- Связанные сущности (договор, прайс, скан, письма, документы) при отвязке **не трогаются** — они привязаны к `login` и автоматически становятся видны в индивидуальном кабинете.

---

## OUT — Site → 1С (корпоративный кабинет: регистрация, доступ, удаление membership)

### 4.3 Синхронизация корпоративного пользователя (регистрация и/или привязка к компании)

Каждый вызов описывает, какие переходы состояния произошли в текущем запросе: `registered` (юзер впервые появился на сайте) и/или `linked` (юзер привязан к компании). В сценарии регистрации по magic-link приглашению оба флага — `true` в одном вызове.

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/sync/
X-API-Key: {api_key}
Content-Type: application/json
```

```jsonc
{
  "email": "user@company.ua",
  "registered": true,
  "linked": true,
  "name": "Пётр Петренко",
  "phone": "+380991234567",
  "cabinet_id": "cab-001"
}
```

Поля: `email`, `registered`, `linked` — REQUIRED; `name` и `phone` — REQUIRED при `registered: true`; `cabinet_id` — REQUIRED при `linked: true`.

**Ответ — успех (HTTP 201):**

```json
{ "message": "User synced." }
```

**Ответ — идемпотентный повтор (HTTP 200):**

```json
{ "message": "User already synced.", "idempotent": true }
```

**Допустимые комбинации флагов:**


| `registered` | `linked` | Сценарий                                                                            |
| ------------ | -------- | ----------------------------------------------------------------------------------- |
| `true`       | `false`  | Самостоятельная регистрация без привязки к компании (шаг 1 Флоу 2, Вариант А)       |
| `false`      | `true`   | Мастер добавил существующего пользователя в компанию                                |
| `true`       | `true`   | Новый пользователь прошёл регистрацию по magic-link приглашению (оба события сразу) |
| `false`      | `false`  | — (недопустимо, `400`)                                                              |


---

### 4.9 Синхронизация доступа корпоративного сотрудника к компаниям кабинета (Site → 1С)

Передаёт в 1С **снимок** текущего списка компаний, к которым у сотрудника есть доступ в UI сайта. Связь **двусторонняя** — IN-копия события (1С → Site) описана в §3.2 как `corporate.user.access.update`. Обе стороны хранят локальный список и применяют входящие снимки по правилу **last-write-wins по `synced_at`** (см. §3.2 «Принципы устройства корпоративного кабинета»).

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/access/
X-API-Key: {api_key}
Content-Type: application/json
```

```jsonc
{
  "cabinet_id": "cab-001",
  "email": "accountant@company.ua",
  "company_logins": ["01/02/03", "01/02/04"],
  "synced_at": "2026-04-27T11:00:00Z"
}
```

Поля — REQUIRED; `company_logins` — полный текущий список (пустой массив = нет доступа ни к одной компании). Для текущего мастера эндпоинт не вызывается.

**Ответ — снимок применён или идемпотентный повтор того же содержимого (HTTP 200):**

```json
{ "message": "Access snapshot applied." }
```

**Ответ — входящий снимок старше уже применённого в 1С, LWW (HTTP 200):**

```json
{ "message": "Stale snapshot ignored.", "idempotent_stale": true }
```

**Ответ — кабинет не найден (HTTP 404):**

```json
{
  "error": "cabinet_not_found",
  "message": "Cabinet not found",
  "cabinet_id": "cab-001"
}
```

**Ответ — сотрудник не состоит в этом кабинете в 1С (HTTP 404):**

```json
{
  "error": "membership_not_found",
  "message": "User is not a member of this cabinet in 1C",
  "cabinet_id": "cab-001",
  "email": "accountant@company.ua"
}
```

**Ответ — попытка задать список для текущего мастера кабинета (HTTP 409):**

```json
{
  "error": "master_access_immutable",
  "message": "Master always has full cabinet access; access snapshot is not applicable",
  "cabinet_id": "cab-001",
  "email": "ceo@company.ua"
}
```

**Ответ — в `company_logins` есть `login`, не входящий в этот `cabinet_id` (HTTP 400):**

```json
{
  "error": "invalid_company_logins",
  "message": "Some logins are not part of this cabinet",
  "invalid_logins": ["01/99/99"]
}
```

**Семантика и правила:**

- **Snapshot, не diff.** Тело — декларация текущего состояния на сайте; 1С полностью **заменяет** хранящийся список для пары `(cabinet_id, email)`.
- **LWW по `synced_at`.** Если 1С уже применила более поздний снимок (например, в результате правки менеджером в 1С), ответ — `200 idempotent_stale`, локальное состояние не меняется. Это защищает от out-of-order при одновременных правках с обеих сторон.
- **Идемпотентность повтора:** повтор с тем же `synced_at` и тем же содержимым → `200 message: "Access snapshot applied."`, без побочных эффектов и без повторных уведомлений менеджеру в 1С (если 1С их формирует).
- **Master-исключение.** Сайт **не** вызывает эндпоинт для текущего `master_email`. `409 master_access_immutable` — защита от багов на сайте.
- **Каскад при выходе компании из кабинета.** При обработке `corporate.company.transfer` или `corporate.company.detach` (см. §3.2) обе стороны **локально** выкидывают этот `login` из всех своих access-списков. Дополнительный access-snapshot для этого слать не нужно. (Старый механизм через `corporate.cabinet.update` с `_deleted: true` для company больше не используется — см. §3.2.)
- **Когда сайт обязан вызвать:**
  1. Сразу после успешного создания membership-я через сайт (`/corporate/users/sync/` с `linked: true`) — со списком, который мастер выбрал в диалоге приглашения. Если мастер ничего не выбрал — пустой массив (см. §3.2 «Стартовый список — пустой»).
  2. После каждой правки списка мастером в UI сайта.
- **Когда сайт НЕ должен вызывать:**
  1. После прихода `corporate.user.invite` от 1С (если в нём передан `company_logins[]`, обе стороны уже синхронизированы — повторный обратный snapshot был бы дублем).
  2. После прихода `corporate.user.access.update` от 1С (это применение входящего снимка; обратной отсылки не нужно).
  3. После применения смены мастера через `corporate.cabinet.update` — у бывшего мастера membership удалён вместе с access-списком, у нового мастера access не ведётся (см. §3.2 «Смена мастера кабинета»).
- **404 как транзиентная ошибка.** В норме `404 cabinet_not_found` / `membership_not_found` означает, что 1С ещё не успела применить более раннее событие (например, `corporate.user.invite` или `corporate.cabinet.create`). Сайт повторяет вызов с экспоненциальным backoff; если после нескольких часов сущности так и нет — алерт по правилам §1.5.
- **Удаление сотрудника из кабинета.** Если мастер на сайте удаляет membership через UI — сайт уведомляет 1С отдельным OUT-эндпоинтом `POST /corporate/users/membership/removed/` (см. §4.10). Связанный access-список очищается локально вместе с membership-ем — отдельного access-snapshot не нужно.

**Итог двусторонней связи:**

```
Сайт правит видимость → POST /corporate/users/access/ → 1С применяет (LWW по synced_at)
                                                              ↓
                                                  карточка сотрудника в 1С обновлена

1С правит видимость → corporate.user.access.update → Сайт применяет (LWW по synced_at)
                                                              ↓
                                                 пересчёт fan-out уведомлений (§3.6)
```

---

### 4.10 Удаление сотрудника из корпоративного кабинета (Site → 1С)

Триггер — мастер кабинета на сайте удалил сотрудника через UI. Сайт удаляет membership-я и его access-список локально, после чего уведомляет 1С, чтобы менеджер видел в карточке кабинета актуальный состав сотрудников.

```http
POST https://mock.1c.virtoffice.ua/api/v2/corporate/users/membership/removed/
X-API-Key: {api_key}
Content-Type: application/json
```

```jsonc
{
  "cabinet_id": "cab-001",
  "email": "accountant@company.ua",
  "removed_at": "2026-04-27T13:00:00Z",
  "removed_by": "ceo@company.ua"
}
```

Поля `cabinet_id`, `email`, `removed_at` — REQUIRED; `removed_by` — optional.

**Ответ — membership удалён в 1С или его не было (HTTP 200, идемпотентный успех):**

```json
{ "message": "Membership removed." }
```

**Ответ — кабинет не найден (HTTP 404):**

```json
{
  "error": "cabinet_not_found",
  "message": "Cabinet not found",
  "cabinet_id": "cab-001"
}
```

**Ответ — попытка удалить мастера кабинета через этот эндпоинт (HTTP 409):**

```json
{
  "error": "master_cannot_be_removed",
  "message": "Master cannot be removed via membership endpoint; use master transfer in cabinet.update",
  "cabinet_id": "cab-001",
  "email": "ceo@company.ua"
}
```

**Семантика и правила:**

- **Идемпотентность.** Ключ — `(cabinet_id, email)`. Если membership-я в 1С нет (никогда не было или уже удалён) — `200`, без ошибки. Это сознательная ассиметрия: цель эндпоинта — привести 1С в согласованное состояние «membership-я нет», а не сообщить об удалении факта существования.
- **Master нельзя удалить через этот эндпоинт.** Передача мастерства идёт через `corporate.cabinet.update` с новым `master_email` (см. §3.2). При попытке передать `email` текущего мастера → `409 master_cannot_be_removed`.
- **Каскадных событий не нужно.** Сайт уже удалил membership и access-список локально; 1С при обработке делает то же самое в своей карточке кабинета. Дополнительный `POST /corporate/users/access/` для удалённого membership-я **не** нужен (более того — он бы вернул `404 membership_not_found`).
- **Уведомления удалённому сотруднику.** Сайт может опционально отправить email удалённому пользователю «вы удалены из кабинета X»; контракт этого не требует — это UX-опция. На membership-ы сотрудника в **других** кабинетах удаление никак не влияет.
- **Обратная связь не нужна.** 1С после применения никаких IN-событий обратно не шлёт — для сайта факт удаления уже совершён локально, ответ `200` достаточен.

