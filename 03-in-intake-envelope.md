[← Оглавление](README.md)

> **§3 (начало)** — endpoint intake и оболочка запроса.


## 3. IN: 1С → Site (входящие события)

### Endpoint

```http
POST https://mock.virt-off.com.ua/api/v2/intake/
X-API-Key: {api_key}
Content-Type: application/json
```

### Структура запроса

Обязательные поля верхнего уровня — `event_type` и `payload`. Дополнительных технических полей на уровне envelope нет: идемпотентность строится **только** на доменных ключах внутри `payload` (см. §1.4).


| Поле         | Обяз. | Описание     |
| ------------ | ----- | ------------ |
| `event_type` | да    | Тип события  |
| `payload`    | да    | Тело события |


```json
{
  "event_type": "mail.create",
  "payload": {}
}
```

---

