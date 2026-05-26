# Telegram-нода (назначение + алерт ошибок)

Тип: `n8n-nodes-base.telegram`. Используется в шаблоне ДВАЖДЫ:
1. ветка `valid=true` — отправка лида в рабочий чат/группу;
2. ветка `valid=false` — алерт о невалидной заявке (чтобы ничего не потерять).

## Credential

Менеджер один раз создаёт в n8n **Telegram API credential** с токеном бота (от @BotFather).
Ноды ссылаются на эту credential; токен в параметры не вписываем.

## Отправка сообщения (успех)

```json
{
  "id": "telegram-ok",
  "name": "Telegram",
  "type": "n8n-nodes-base.telegram",
  "typeVersion": 1.2,
  "position": [900, 300],
  "parameters": {
    "resource": "message",
    "operation": "sendMessage",
    "chatId": "<CHAT_ID>",
    "text": "=🆕 Новая заявка\nИмя: {{ $json.data.name }}\nEmail: {{ $json.data.email }}\nТел: {{ $json.data.phone }}",
    "additionalFields": { "parse_mode": "HTML" }
  }
}
```

## Алерт об ошибке валидации (ветка false)

```json
{
  "id": "telegram-alert",
  "name": "Telegram Алерт",
  "type": "n8n-nodes-base.telegram",
  "typeVersion": 1.2,
  "position": [900, 460],
  "parameters": {
    "resource": "message",
    "operation": "sendMessage",
    "chatId": "<CHAT_ID>",
    "text": "=⚠️ Невалидная заявка с формы\nОшибки: {{ $json.errors.join('; ') }}\nСырые данные: {{ JSON.stringify($json.data) }}"
  }
}
```

## Что уточнить у менеджера

- **chat_id** рабочего чата/группы (можно один и тот же для лидов и алертов, или разные).
  Подскажи: чтобы узнать chat_id, добавить бота в чат и посмотреть update, либо использовать
  @userinfobot для личного чата.
- Формат сообщения (какие поля показывать).

## Замечания

- Бот должен быть добавлен в группу и иметь право писать.
- `typeVersion` Telegram-ноды сверь через `n8n-mcp:get_node` — параметры
  (`additionalFields`, `parse_mode`) зависят от версии.
