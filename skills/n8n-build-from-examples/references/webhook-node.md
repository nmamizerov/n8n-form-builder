# Webhook-нода (приём данных формы)

Тип: `n8n-nodes-base.webhook`, typeVersion `2`.

Принимает POST-запрос от внешней формы (сайт, Tilda, лендинг). Данные приходят в `$json.body`.

## Ключевые параметры

| Параметр | Значение | Зачем |
|----------|----------|-------|
| `httpMethod` | `"POST"` | формы шлют POST |
| `path` | slug формы, напр. `"lead-acme"` | часть URL вебхука; делай уникальным на воркфлоу |
| `responseMode` | `"lastNode"` | ответ формирует последняя выполненная нода |
| `options` | `{}` | при необходимости — CORS, raw body и т.п. |

## Пример параметров ноды

```json
{
  "id": "webhook-1",
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [240, 300],
  "parameters": {
    "httpMethod": "POST",
    "path": "lead-acme",
    "responseMode": "lastNode",
    "options": {}
  }
}
```

## URL вебхука

- Боевой: `<N8N_API_URL>/webhook/<path>`
- Тестовый (до активации воркфлоу, для одного запуска): `<N8N_API_URL>/webhook-test/<path>`

Менеджеру в форму нужно вписать боевой URL и метод POST, тело — JSON с полями формы.

## Замечания

- Данные формы лежат в `$json.body` (объект). Поля — `$json.body.email`, `$json.body.name` и т.п.
- Уточняй точные имена опций через `n8n-mcp:get_node` (mode `full`) — между версиями
  `options` отличается.
