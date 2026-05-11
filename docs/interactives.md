# Interactive Messages Reference (Code-Reviewed)

This document summarizes interactive message routes and payload formats by reviewing the current codebase implementation.

Scope reviewed:
- Router endpoints in `src/api/routes/sendMessage.router.ts`
- Validation schemas in `src/validate/message.schema.ts`
- DTOs in `src/api/dto/sendMessage.dto.ts`
- Provider behavior in:
  - `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts`
  - `src/api/integrations/channel/meta/whatsapp.business.service.ts`
  - `src/api/integrations/channel/evolution/evolution.channel.service.ts`

## Base URL and Auth

All routes below are mounted under `/message` and require:
- `apikey` header (global key or instance key)
- `:instanceName` route param

Pattern:
- `POST /message/<route>/:instanceName`

Example:
- `POST /message/sendButtons/my-instance`

## Provider Support Matrix

Instance integrations (from `src/api/types/wa.types.ts`):
- `WHATSAPP-BAILEYS`
- `WHATSAPP-BUSINESS`
- `EVOLUTION`

Interactive support by provider:

| Route | WHATSAPP-BAILEYS | WHATSAPP-BUSINESS | EVOLUTION |
|---|---|---|---|
| `sendButtons` | Yes | Yes (reply-style mapping) | Yes |
| `sendList` | Yes | Yes | No (`Method not available on Evolution Channel`) |
| `sendPoll` | Yes | No (`Method not available on WhatsApp Business API`) | No |
| `sendTemplate` | No (`Method not available in the Baileys service`) | Yes | No |
| `sendStatus` | Yes | No | No |
| `sendReaction` | Yes | Yes | No |

If your instance integration does not support the route, the API will accept the endpoint but fail at service execution with a provider-specific error.

## Common Optional Fields (most send routes)

Many message routes also accept:
- `delay` (ms)
- `quoted`
- `mentioned`
- mention-all flag in schema as `everyOne` (note: services read `mentionsEveryOne`)

Quoted format:
```json
{
  "quoted": {
    "key": {
      "id": "ABCD1234",
      "remoteJid": "5511999999999@s.whatsapp.net",
      "fromMe": false
    },
    "message": {}
  }
}
```

## 1) Buttons

Route:
- `POST /message/sendButtons/:instanceName`

Schema-level fields:
- `number` (required)
- `thumbnailUrl` (optional)
- `title` (optional in schema, but required in DTO/used in service)
- `description` (optional)
- `footer` (optional)
- `buttons` (array)

Button item fields:
- `type`: one of `reply`, `copy`, `url`, `call`, `pix` (required)
- `displayText`
- `id`
- `url`
- `phoneNumber`
- `currency`
- `name`
- `keyType`: `phone` | `email` | `cpf` | `cnpj` | `random`
- `key`
- `copyCode` (used in code for `copy`, defined in DTO)

Baileys runtime constraints:
- At least 1 button required.
- If any `reply` button exists:
  - max 3 buttons
  - cannot mix with other button types.
- If any `pix` button exists:
  - only 1 button allowed
  - cannot mix with other types.

Business API behavior:
- Buttons are mapped as reply buttons (`type: 'reply'`) using `displayText` and `id`.
- Advanced types (`copy`, `url`, `call`, `pix`) are not sent as native specialized button types here.

Example (reply buttons):
```json
{
  "number": "5511999999999",
  "title": "Choose an option",
  "description": "Quick actions",
  "footer": "Footer text",
  "buttons": [
    { "type": "reply", "displayText": "Yes", "id": "opt_yes" },
    { "type": "reply", "displayText": "No", "id": "opt_no" }
  ]
}
```

Example (URL button for Baileys):
```json
{
  "number": "5511999999999",
  "title": "Open website",
  "buttons": [
    {
      "type": "url",
      "displayText": "Visit",
      "url": "https://example.com"
    }
  ]
}
```

Example (PIX button for Baileys):
```json
{
  "number": "5511999999999",
  "title": "Pay with PIX",
  "buttons": [
    {
      "type": "pix",
      "currency": "BRL",
      "name": "Merchant Name",
      "keyType": "cpf",
      "key": "12345678900"
    }
  ]
}
```

## 2) Lists

Route:
- `POST /message/sendList/:instanceName`

Required fields (schema):
- `number`
- `title`
- `footerText`
- `buttonText`
- `sections`

Section fields:
- `title` (required)
- `rows` (required, at least 1)

Row fields:
- `title` (required)
- `rowId` (required)
- `description` (optional in schema definition, but often expected by clients)

Example:
```json
{
  "number": "5511999999999",
  "title": "Main menu",
  "description": "Select one option",
  "footerText": "Powered by Evolution",
  "buttonText": "View options",
  "sections": [
    {
      "title": "Support",
      "rows": [
        {
          "title": "Billing",
          "description": "Questions about invoices",
          "rowId": "support_billing"
        },
        {
          "title": "Technical",
          "description": "API and integration help",
          "rowId": "support_tech"
        }
      ]
    }
  ]
}
```

Business API notes:
- Section titles must be unique.
- Row description is truncated to 72 chars in service.
- `rowId` is mapped to `id` before send.

## 3) Polls

Route:
- `POST /message/sendPoll/:instanceName`

Required fields:
- `number`
- `name`
- `selectableCount` (0..10)
- `values` (2..10 unique items)

Example:
```json
{
  "number": "5511999999999",
  "name": "Which feature first?",
  "selectableCount": 1,
  "values": ["Buttons", "Lists", "Polls"]
}
```

Provider availability:
- Works on `WHATSAPP-BAILEYS`.
- Not available on `WHATSAPP-BUSINESS` and `EVOLUTION`.

## 4) Templates (Interactive by Meta template model)

Route:
- `POST /message/sendTemplate/:instanceName`

Required fields:
- `name`
- `language`

Common fields:
- `number` (practically needed by service call path)
- `components` (array/object according to your Meta template)

Example:
```json
{
  "number": "5511999999999",
  "name": "order_update",
  "language": "en_US",
  "components": [
    {
      "type": "body",
      "parameters": [
        { "type": "text", "text": "John" },
        { "type": "text", "text": "#1234" }
      ]
    }
  ]
}
```

Provider availability:
- Implemented for `WHATSAPP-BUSINESS`.
- Not available on `WHATSAPP-BAILEYS` and `EVOLUTION`.

## 5) Reactions

Route:
- `POST /message/sendReaction/:instanceName`

Required fields:
- `key.id`
- `key.remoteJid`
- `key.fromMe`
- `reaction` (must be a single emoji or empty string)

Example:
```json
{
  "key": {
    "id": "ABCD1234",
    "remoteJid": "5511999999999@s.whatsapp.net",
    "fromMe": false
  },
  "reaction": "👍"
}
```

## 6) Status (not chat interactive UI, but rich send flow)

Route:
- `POST /message/sendStatus/:instanceName`

Required fields:
- `type`: `text` | `image` | `audio` | `video`
- `content`

Conditional requirements:
- For `text`: `backgroundColor` and `font` are required.
- Must provide one of:
  - `statusJidList` (array), or
  - `allContacts: true`.

Example (text status):
```json
{
  "type": "text",
  "content": "Hello status",
  "backgroundColor": "#008069",
  "font": 1,
  "statusJidList": ["5511999999999@s.whatsapp.net"]
}
```

Provider availability:
- Implemented only on `WHATSAPP-BAILEYS`.

## Troubleshooting Checklist for "interactive not rendering"

1. Confirm instance integration supports the route:
   - If using `WHATSAPP-BUSINESS`, `sendPoll`/`sendStatus` are unavailable.
   - If using `EVOLUTION`, only `sendButtons` is implemented among interactive routes.

2. For `sendButtons` on Baileys:
   - Do not mix `reply` with other button types.
   - Keep reply buttons <= 3.
   - Use exactly one `pix` button and no mixed types.

3. For `sendButtons` on Business API:
   - Use reply-style semantics (`displayText` + `id`) for best compatibility.

4. For `sendList`:
   - Ensure `sections` and `rows` are non-empty.
   - Ensure each row has stable `rowId`.

5. Ensure `number` is in expected numeric format string.

6. Ensure you are sending with correct auth header:
   - `apikey: <global-or-instance-key>`

## Quick cURL Samples

Buttons:
```bash
curl -X POST "http://<host>:8080/message/sendButtons/<instanceName>" \
  -H "Content-Type: application/json" \
  -H "apikey: <API_KEY>" \
  -d '{
    "number":"5511999999999",
    "title":"Choose",
    "buttons":[
      {"type":"reply","displayText":"Option A","id":"a"},
      {"type":"reply","displayText":"Option B","id":"b"}
    ]
  }'
```

List:
```bash
curl -X POST "http://<host>:8080/message/sendList/<instanceName>" \
  -H "Content-Type: application/json" \
  -H "apikey: <API_KEY>" \
  -d '{
    "number":"5511999999999",
    "title":"Menu",
    "footerText":"Footer",
    "buttonText":"Open",
    "sections":[
      {
        "title":"Section 1",
        "rows":[{"title":"Row 1","description":"Desc","rowId":"row_1"}]
      }
    ]
  }'
```
