# amoCRM — через community-ноду `n8n-nodes-amocrm` (yatolstoy)

Используем **community-ноду** [`n8n-nodes-amocrm`](https://github.com/yatolstoy/n8n-node-amocrm),
а не HTTP Request. Она удобнее, нативная для amoCRM, поддерживает ресурсы leads/contacts/companies
и т.д.

## Установка

Только **self-hosted n8n** (или Enterprise Cloud с разрешёнными community-нодами):

1. n8n → **Settings → Community Nodes → Install**.
2. NPM-имя пакета: `n8n-nodes-amocrm`.
3. Поставь галку «I understand the risks» → Install.

На обычном n8n Cloud unverified community nodes недоступны. Если у менеджера именно Cloud —
объясни ему ограничение и либо предложи self-hosted, либо вернись к HTTP Request как запасному варианту.

## Технические факты ноды

- `type`: `n8n-nodes-amocrm.amocrm`
- `typeVersion`: `1` (единственная версия)
- Ресурсы: `leads`, `contacts`, `companies`, `catalogs`, `notes`, `tasks`, `account`.
- Операции для лидов: `getLeads`, `createLeads`, `updateLeads`.
- Операции для контактов: `getContacts`, `createContacts`, `updateContacts`.

## Credentials

Два варианта в `credentials/`:

### Long-Lived токен — **рекомендую** (проще)
- Тип: `amocrmLongLivedApi`
- Поля: `subdomain` (без `.amocrm.ru`), `apiKey` (Long-Lived токен из amoCRM → Интеграции → создать → Long-Lived).
- В параметрах ноды: `"authentication": "longLivedToken"`.

### OAuth2 (если нужен)
- Тип: `amocrmOAuth2Api`
- Поля: `subdomain`, `clientId`, `clientSecret` (из amoCRM-интеграции). Refresh/access токены n8n крутит сам по своему OAuth2-callback (`https://<n8n>/rest/oauth2-credential/callback`).
- В параметрах ноды: `"authentication": "oAuth2"`.

В обоих случаях менеджер создаёт credential в n8n один раз; в ноде только ссылка на неё:

```json
"credentials": {
  "amocrmLongLivedApi": { "id": "<cred-id>", "name": "AmoCRM Long-Lived" }
}
```

## ⚠️ Главный подводный камень: лида с НОВЫМ контактом в одной ноде НЕЛЬЗЯ

Нода вызывает `POST /api/v4/leads`, а этот endpoint в `_embedded.contacts` принимает только
**id уже существующего контакта**. Нет встроенного «создай лид и контакт одной операцией».

Поэтому шаблон **двухнодный**:

```
... → amoCRM Create Contact → amoCRM Create Lead (ссылается на id из контакта)
```

Это **главное отличие** от HTTP-варианта (где в одном теле можно было создать лид+контакт).

В шаблоне обработчика форм fan-out IF(true) указывает на ТРИ цели: первая — `amoCRM Create Contact`,
дальше Contact → Lead это уже цепочка. Telegram и Postgres по-прежнему параллельно.

## Создание контакта (JSON-mode — самый чистый способ для phone/email)

Поля email/phone в amoCRM — это `custom_fields_values` с `field_code: "EMAIL" | "PHONE"`.
В form-mode `data` требует dropdown-кодированный JSON — статически собирать неудобно. Поэтому
используем `json: true` и пишем тело массивом:

```json
{
  "type": "n8n-nodes-amocrm.amocrm",
  "typeVersion": 1,
  "parameters": {
    "authentication": "longLivedToken",
    "resource": "contacts",
    "operation": "createContacts",
    "json": true,
    "jsonString": "={{ JSON.stringify([{ name: $json.data.name, custom_fields_values: [ { field_code: 'PHONE', values: [ { value: $json.data.phone, enum_code: 'WORK' } ] }, { field_code: 'EMAIL', values: [ { value: $json.data.email, enum_code: 'WORK' } ] } ] }]) }}"
  },
  "credentials": {
    "amocrmLongLivedApi": { "id": "<cred-id>", "name": "AmoCRM Long-Lived" }
  }
}
```

`enum_code` — категория: `WORK | MOB | PRIV | FAX | OTHER`. Поле возвращает массив контактов;
id нового — `$json._embedded.contacts[0].id`.

## Создание лида (form-mode, ссылка на id контакта)

`_embedded.contacts` в form-mode имеет НЕОЧЕВИДНУЮ форму с двойной вложенностью
(`{id: {contact: [{id, is_main}]}}`) — не упрощай, иначе нода не распарсит:

```json
{
  "type": "n8n-nodes-amocrm.amocrm",
  "typeVersion": 1,
  "parameters": {
    "authentication": "longLivedToken",
    "resource": "leads",
    "operation": "createLeads",
    "json": false,
    "collection": {
      "lead": [
        {
          "name": "=Заявка с сайта: {{ $json.data.name }}",
          "pipeline_id": 0,
          "status_id": 0,
          "_embedded": {
            "contacts": [
              { "id": { "contact": [ { "id": "={{ $json._embedded.contacts[0].id }}", "is_main": true } ] } }
            ]
          }
        }
      ]
    }
  },
  "credentials": {
    "amocrmLongLivedApi": { "id": "<cred-id>", "name": "AmoCRM Long-Lived" }
  }
}
```

`pipeline_id` и `status_id` — обязательны для конкретной воронки. **Спроси у менеджера** их id
(amoCRM → Сделки → нужная воронка → URL содержит pipeline id; этапы видны там же).

## Что уточнить у менеджера

- **Поддомен** amoCRM (без `.amocrm.ru`).
- Какой **Long-Lived токен** (или OAuth2 credentials) — он создаёт credential в n8n; токен в чат не присылает.
- **pipeline_id** и **status_id** воронки/этапа, куда падают лиды.
- Нужно ли заполнять кастомные поля помимо email/phone (тогда понадобятся их `field_id`).

## Прочие подводные камни (из источников)

- Нода работает только на **self-hosted n8n** (или Enterprise Cloud).
- Phone/email в form-mode реально не собрать без dropdown — всегда JSON-mode для контакта.
- API amoCRM возвращает ошибки **по-русски** — это нормально, не пугайся.
- Open issue #6 — совместимость с n8n ≥ 1.94 не отрейтена; если у менеджера новый n8n, прогоняй
  тестовый запуск воркфлоу сразу после `n8n_create_workflow`.
- В ноде нет триггера для входящих событий amoCRM — для них используется отдельная Webhook-нода + amoCRM Webhooks API (вне рамок шаблона форм).

## Источники

- Репозиторий: https://github.com/yatolstoy/n8n-node-amocrm
- `nodes/Amocrm/V1/AmocrmV1.node.ts`, `nodes/Amocrm/V1/resources/{leads,contacts}/...`
- `credentials/amocrmLongLivedApi.credentials.ts`, `credentials/amocrmOAuth2Api.credentials.ts`
