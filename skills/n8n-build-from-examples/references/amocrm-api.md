# amoCRM (через HTTP Request)

В n8n **нет нативной ноды amoCRM**. Интеграция — через `n8n-nodes-base.httpRequest`
(typeVersion ~`4.2`) к amoCRM REST API v4 с OAuth2.

## Авторизация

amoCRM использует OAuth2 (долгоживущий токен через интеграцию в amoCRM → Настройки →
Интеграции). В n8n менеджер создаёт **OAuth2 API credential** один раз, в ноде HTTP Request
ставит `authentication: "genericCredentialType"`, `genericAuthType: "oAuth2Api"` и выбирает
эту credential. Секреты — только в credential, не в параметрах ноды.

## Создание лида (POST /api/v4/leads)

```json
{
  "id": "amocrm-1",
  "name": "amoCRM",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [900, 200],
  "parameters": {
    "method": "POST",
    "url": "https://<SUBDOMAIN>.amocrm.ru/api/v4/leads",
    "authentication": "genericCredentialType",
    "genericAuthType": "oAuth2Api",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify([{ name: $json.data.name, _embedded: { contacts: [{ first_name: $json.data.name, custom_fields_values: [{ field_code: 'EMAIL', values: [{ value: $json.data.email, enum_code: 'WORK' }] }, { field_code: 'PHONE', values: [{ value: $json.data.phone, enum_code: 'WORK' }] }] }] } }]) }}"
  }
}
```

## Что уточнить у менеджера

- **Поддомен** amoCRM (`<SUBDOMAIN>.amocrm.ru`).
- Какие поля лида/контакта заполнять и их `field_code` (EMAIL, PHONE — стандартные;
  кастомные поля идут по `field_id`).
- Нужно ли создавать контакт вместе с лидом (как выше через `_embedded.contacts`) или
  только сделку.

## Замечания

- Тело — массив объектов (amoCRM принимает батч). Выше создаём один лид со встроенным контактом.
- Точную текущую форму параметров HTTP Request сверь через `n8n-mcp:get_node`
  (`n8n-nodes-base.httpRequest`, mode `full`) — имена `sendBody`/`specifyBody`/`jsonBody`
  зависят от typeVersion.
- amoCRM при ошибке вернёт 4xx — её увидит выполнение воркфлоу; для надёжности можно
  включить в ноде `options.response` обработку ошибок, но в v1 это вне рамок.
