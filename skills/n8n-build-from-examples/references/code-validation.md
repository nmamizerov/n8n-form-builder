# Code-нода (валидация полей формы)

Тип: `n8n-nodes-base.code`, typeVersion `2`. Язык: JavaScript (`language: "javaScript"`,
`mode: "runOnceForAllItems"`).

Задача ноды — проверить поля по правилам менеджера и вернуть единый объект
`{ valid, data, errors }`, по которому дальше ветвится IF.

## Контракт выхода

```js
return [{ json: {
  valid: true | false,
  data:  { /* нормализованные поля для записи в источники */ },
  errors: [ /* строки с описанием, что не прошло */ ]
}}];
```

## Шаблон кода (генерируй под конкретные поля/правила)

```js
const body = $input.first().json.body || {};
const errors = [];

// --- поля и правила менеджера ---
const name  = (body.name  || '').trim();
const email = (body.email || '').trim();
const phone = (body.phone || '').trim();

if (!name)  errors.push('Не заполнено имя');
if (!email) errors.push('Не заполнен email');
else if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) errors.push('Некорректный email');
// телефон необязателен в этом примере

const data = { name, email, phone: phone || null, received_at: new Date().toISOString() };

return [{ json: { valid: errors.length === 0, data, errors } }];
```

## Как превращать правила менеджера в код

| Правило менеджера | Проверка в коде |
|-------------------|-----------------|
| «поле обязательно» | `if (!value) errors.push(...)` |
| «это email» | regex `^[^@\s]+@[^@\s]+\.[^@\s]+$` |
| «это телефон» | normalize: `value.replace(/\D/g,'')`; проверка длины 10–15 |
| «число от A до B» | `Number(value)` + диапазон |
| «один из вариантов» | `['a','b'].includes(value)` |
| «не короче N символов» | `value.length >= N` |

## Замечания

- Всегда нормализуй `data` к тому виду, который удобно писать в amoCRM/Telegram/Postgres.
- Не бросай исключение на невалидных данных — вместо этого верни `valid:false` + `errors`,
  чтобы ветка IF(false) отправила Telegram-алерт (иначе воркфлоу упадёт без уведомления).
- Точные имена параметров Code-ноды сверяй через `n8n-mcp:get_node`.
