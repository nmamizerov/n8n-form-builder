# Postgres-нода (запись в БД)

Тип: `n8n-nodes-base.postgres` (typeVersion ~`2.x`). В шаблоне используется для записи
валидной заявки в таблицу.

## Credential

Менеджер один раз создаёт в n8n **Postgres credential** (host, port, database, user,
password, SSL). Нода ссылается на неё; секреты в параметры не вписываем.

## Insert строки

```json
{
  "id": "postgres-1",
  "name": "Postgres",
  "type": "n8n-nodes-base.postgres",
  "typeVersion": 2.5,
  "position": [900, 400],
  "parameters": {
    "operation": "insert",
    "schema": "public",
    "table": "leads",
    "columns": "name,email,phone,received_at",
    "options": {}
  }
}
```

При `operation: "insert"` n8n берёт значения колонок из входного item. Поэтому Code-нода
должна положить в `data` поля с именами, совпадающими с колонками таблицы (`name`, `email`,
`phone`, `received_at`), либо настрой явный маппинг колонок в ноде.

## Что уточнить у менеджера

- Имя таблицы и схема (`public.leads`?).
- Список колонок и их соответствие полям формы.
- Существует ли таблица. Если нет — дай менеджеру готовый SQL `CREATE TABLE`, например:

```sql
CREATE TABLE IF NOT EXISTS leads (
  id          bigserial PRIMARY KEY,
  name        text,
  email       text,
  phone       text,
  received_at timestamptz DEFAULT now()
);
```

## Замечания

- Современные версии ноды используют режим маппинга колонок (`columns` как объект/маппинг),
  а не строку. Точную форму параметров (`mappingMode`, `columns`, `matchingColumns`) сверь
  через `n8n-mcp:get_node` (`n8n-nodes-base.postgres`, mode `full`).
- На ошибку записи (нет таблицы/нарушение типа) выполнение упадёт — в v1 это допустимо,
  алерт по невалидным данным уже покрывает ветка Telegram.
