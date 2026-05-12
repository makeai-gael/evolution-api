# Evolution API Full Endpoint Reference

## Scope

This document is a code-based HTTP API reference for the current repository.
It is based on the implemented routers, controllers, DTOs, and event definitions under `src/api/**`.

The goal is to describe **all exposed endpoints** with:

- method
- path
- auth expectations
- request payload or query inputs
- response behavior
- important caveats

This document is not limited to provider-core routes.

## Global Rules

### Base URL

- Base URL: `SERVER_URL` from environment configuration

### Main Auth

- Main auth header: `apikey`

### Route Pattern

Most protected tenant-scoped routes use:

- `/<domain>/<action>/:instanceName`

### Guard Behavior

For most protected routes, the following are required:

- valid `apikey`
- existing instance
- logged-in instance state

Exceptions:

- `POST /instance/create` creates the instance, so it does not require an already connected session
- public/system webhook routes do not use the normal `apikey` guard

### Response Normalization

There are two broad response styles in this codebase:

1. controller-defined or repository-backed responses
   - these are relatively stable and can be described concretely

2. live integration pass-through responses
   - these come from `waMonitor.waInstances[instanceName]` and are not always normalized at the HTTP controller layer
   - for these, the safest contract is:
     - persist the raw response
     - extract the best available identifiers and status fields in your adapter

### Common Error Classes

Observed exception patterns include:

- `BadRequestException`
- `UnauthorizedException`
- `InternalServerErrorException`

Typical failure cases:

- invalid or missing request fields
- invalid schema values
- invalid media source
- invalid proxy settings
- instance does not exist
- instance is not connected
- unauthorized or invalid API key

## Public and System Routes

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/` | None | No body | JSON with `status`, `message`, `version`, `clientName`, optional `manager`, `documentation`, `whatsappWebVersion` | Basic liveness/status route. |
| `POST` | `/verify-creds` | `apikey` | No defined body | JSON with `status`, `message`, `facebookAppId`, `facebookConfigId`, `facebookUserToken` | Validates the global API key and exposes configured Meta/Facebook values. |
| `GET` | `/metrics` | Optional IP whitelist and/or Basic Auth depending on env | No body | Prometheus text payload | Only exists when metrics are enabled by config. |
| `GET` | `/manager/*` | None | No body | Manager SPA HTML/assets | Only present when manager UI is enabled. |
| `GET` | `/assets/*` | None | Asset path in URL | Static asset bytes | Protected against simple path traversal. |

## Public Webhook Receiver Routes

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/webhook/evolution` | None | Raw provider webhook JSON body | JSON result from `evolutionController.receiveWebhook(body)` | Inbound webhook receiver for Evolution integration. |
| `GET` | `/webhook/meta` | None | Query params `hub.verify_token`, `hub.challenge` | Verification response string | Meta webhook verification route. |
| `POST` | `/webhook/meta` | None | Raw Meta webhook JSON body | JSON result from `metaController.receiveWebhook(body)` | Meta inbound webhook receiver. |
| `POST` | `/chatwoot/webhook/:instanceName` or equivalent resolved router path | None | Chatwoot webhook payload, with instance context resolved by route params | JSON result from `chatwootController.receiveWebhook` | Public webhook receiver for Chatwoot integration. |

## Instance Lifecycle Routes

### DTO and Input Fields

The instance creation and settings-related inputs are driven primarily by `InstanceDto`.

Important body fields used in code:

- `instanceName: string`
- `integration?: string`
- `instanceId?: string`
- `qrcode?: boolean`
- `businessId?: string`
- `number?: string`
- `token?: string`
- `status?: string`
- `ownerJid?: string`
- `profileName?: string`
- `profilePicUrl?: string`
- `rejectCall?: boolean`
- `msgCall?: string`
- `groupsIgnore?: boolean`
- `alwaysOnline?: boolean`
- `readMessages?: boolean`
- `readStatus?: boolean`
- `syncFullHistory?: boolean`
- `wavoipToken?: string`
- `proxyHost?: string`
- `proxyPort?: string`
- `proxyProtocol?: string`
- `proxyUsername?: string`
- `proxyPassword?: string`
- `webhook?: { enabled?, events?, headers?, url?, byEvents?, base64? }`
- `chatwoot...` fields
- event transport config fields inherited by integration mixins

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/instance/create` | `apikey` | Body based on `InstanceDto`; key fields are `instanceName`, `integration`, `qrcode`, `number`, optional token, settings, webhook config, event transport config, proxy config, Chatwoot config | JSON usually containing `instance`, `hash`, event transport config echoes, `settings`, and sometimes `qrcode` | Creates instance record, initializes local settings, may connect immediately for QR generation, may create proxy and Chatwoot config. |
| `POST` | `/instance/restart/:instanceName` | `apikey` + instance exists + logged instance guard | No body required | JSON with current/restarted instance state, or error structure | If state is `open` or `connecting`, restart is attempted. If `close`, route errors as not connected. |
| `GET` | `/instance/connect/:instanceName` | `apikey` + instance exists + logged instance guard | No body | JSON with current connection state or QR/pairing data | If state is `close`, connects and returns QR state after short delay. If `connecting`, returns current QR. If `open`, returns connection state. |
| `GET` | `/instance/connectionState/:instanceName` | `apikey` + instance exists + logged instance guard | No body | JSON `{ "instance": { "instanceName": "...", "state": "open|close|connecting|..." } }` | Lightweight connection-state lookup. |
| `GET` | `/instance/fetchInstances` | `apikey` | Optional query filters: `instanceName`, `instanceId`, `number` | Instance inventory/info from monitor layer | If `apikey` is the global key, returns broad inventory; if it is an instance token, result is scoped to token-owned instances. |
| `POST` | `/instance/setPresence/:instanceName` | `apikey` + instance exists + logged instance guard | Body `{ "presence": "<WAPresence>" }` | Presence operation result from live instance | Presence values are integration-driven. |
| `DELETE` | `/instance/logout/:instanceName` | `apikey` + instance exists + logged instance guard | No body | Usually `{ "status": "SUCCESS", "error": false, "response": { "message": "Instance logged out" } }` | Logs out current WhatsApp session. |
| `DELETE` | `/instance/delete/:instanceName` | `apikey` + instance exists + logged instance guard | No body | Usually `{ "status": "SUCCESS", "error": false, "response": { "message": "Instance deleted" } }` | Deletes instance and emits removal events. If connected, it may log out first. |

## Message Sending Routes

### Shared Send Metadata Fields

Many outbound message DTOs share:

- `number: string`
- `delay?: number`
- `quoted?: { key, message }`
- `linkPreview?: boolean`
- `encoding?: boolean`
- `mentionsEveryOne?: boolean`
- `mentioned?: string[]`

### `POST /message/sendTemplate/:instanceName`

Request body fields:

- `number: string`
- `name: string`
- `language: string`
- `components: any`
- `delay?`
- `quoted?`
- `linkPreview?`
- `mentionsEveryOne?`
- `mentioned?`
- `encoding?`
- `webhookUrl?`

Typical request example:

```json
{
  "number": "15551234567",
  "name": "welcome_template",
  "language": "en_US",
  "components": [
    {
      "type": "body",
      "parameters": [
        { "type": "text", "text": "Alice" }
      ]
    }
  ]
}
```

### `POST /message/sendText/:instanceName`

Request body fields:

- `number: string`
- `text: string`
- optional shared send metadata fields

Typical request example:

```json
{
  "number": "15551234567",
  "text": "Hello from Evolution",
  "delay": 0,
  "linkPreview": true,
  "encoding": true
}
```

### `POST /message/sendMedia/:instanceName`

Request body fields:

- `number: string`
- `mediatype: "image" | "document" | "video" | "audio" | "ptv"`
- `mimetype?: string`
- `caption?: string`
- `fileName?: string`
- `media: string`
- optional shared send metadata fields

This route also accepts multipart upload via `file`.

Important validation behavior:

- for base64 `document` sends, `fileName` is required
- media must be one of:
  - uploaded file
  - URL
  - base64

### `POST /message/sendPtv/:instanceName`

Request body fields:

- `number: string`
- `video: string`
- optional shared send metadata fields

Also accepts multipart upload via `file`.

### `POST /message/sendWhatsAppAudio/:instanceName`

Request body fields:

- `number: string`
- `audio: string`
- optional shared send metadata fields

Also accepts multipart upload via `file`.

### `POST /message/sendStatus/:instanceName`

Request body fields:

- `number: string`
- `type: string`
- `content: string`
- `statusJidList?: string[]`
- `allContacts?: boolean`
- `caption?: string`
- `backgroundColor?: string`
- `font?: number`
- optional shared send metadata fields

Also accepts multipart upload via `file`.

### `POST /message/sendSticker/:instanceName`

Request body fields:

- `number: string`
- `sticker: string`
- optional shared send metadata fields

Also accepts multipart upload via `file`.

### `POST /message/sendLocation/:instanceName`

Request body fields:

- `number: string`
- `latitude: number`
- `longitude: number`
- `name?: string`
- `address?: string`
- optional shared send metadata fields

### `POST /message/sendContact/:instanceName`

Request body fields:

- `number: string`
- `contact: ContactMessage[]`

Where each `ContactMessage` includes:

- `fullName: string`
- `wuid: string`
- `phoneNumber: string`
- `organization?: string`
- `email?: string`
- `url?: string`

### `POST /message/sendReaction/:instanceName`

Request body fields:

- `key: proto.IMessageKey`
- `reaction: string`

Validation behavior:

- `reaction` must be a single emoji or an empty string

### `POST /message/sendPoll/:instanceName`

Request body fields:

- `number: string`
- `name: string`
- `selectableCount: number`
- `values: string[]`
- `messageSecret?: Uint8Array`
- optional shared send metadata fields

### `POST /message/sendList/:instanceName`

Request body fields:

- `number: string`
- `title: string`
- `description?: string`
- `footerText?: string`
- `buttonText: string`
- `sections: Section[]`

Where each section includes:

- `title: string`
- `rows: Row[]`

Where each row includes:

- `title: string`
- `description?: string`
- `rowId: string`

### `POST /message/sendButtons/:instanceName`

Request body fields:

- `number: string`
- `thumbnailUrl?: string`
- `title: string` - required and cannot be empty
- `description?: string`
- `footer?: string`
- `buttons: Button[]` - required, minimum 1 item

Each button includes:

- `type: "reply" | "copy" | "url" | "call" | "pix"`
- `displayText?: string`
- `id?: string`
- `url?: string`
- `copyCode?: string`
- `phoneNumber?: string`
- `currency?: string`
- `name?: string`
- `keyType?: "phone" | "email" | "cpf" | "cnpj" | "random"`
- `key?: string`

Conditional required fields by button type:

- `reply` requires `displayText` and `id`
- `copy` requires `displayText` and `copyCode`
- `url` requires `displayText` and `url`
- `call` requires `displayText` and `phoneNumber`
- `pix` requires `currency`, `name`, `keyType`, and `key`

Baileys runtime constraints:

- `reply` buttons: maximum 3
- `reply` buttons cannot be mixed with other button types
- `pix` buttons: maximum 1
- `pix` buttons cannot be mixed with other button types

Acceptable reply-button payload example:

```json
{
  "number": "5511999999999",
  "title": "Choose an option",
  "description": "Quick actions",
  "footer": "Footer text",
  "buttons": [
    { "type": "reply", "displayText": "Yes", "id": "opt_yes" },
    { "type": "reply", "displayText": "No", "id": "opt_no" },
    { "type": "reply", "displayText": "Talk to an agent", "id": "opt_agent" }
  ]
}
```

### Message Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/message/sendTemplate/:instanceName` | `apikey` + instance guards | Body from `SendTemplateDto` | Integration-specific send result | Persist raw response and extract best available message ID. |
| `POST` | `/message/sendText/:instanceName` | `apikey` + instance guards | Body from `SendTextDto` | Integration-specific send result | Core outbound text route. |
| `POST` | `/message/sendMedia/:instanceName` | `apikey` + instance guards | Body from `SendMediaDto`, optional multipart `file` | Integration-specific send result | Supports URL, base64, or file upload. |
| `POST` | `/message/sendPtv/:instanceName` | `apikey` + instance guards | Body from `SendPtvDto`, optional multipart `file` | Integration-specific send result | Video note / PTV route. |
| `POST` | `/message/sendWhatsAppAudio/:instanceName` | `apikey` + instance guards | Body from `SendAudioDto`, optional multipart `file` | Integration-specific send result | WhatsApp-native audio route. |
| `POST` | `/message/sendStatus/:instanceName` | `apikey` + instance guards | Body from `SendStatusDto`, optional multipart `file` | Integration-specific send result | Status sending is marked with TODO comment in code. |
| `POST` | `/message/sendSticker/:instanceName` | `apikey` + instance guards | Body from `SendStickerDto`, optional multipart `file` | Integration-specific send result | Sticker media route. |
| `POST` | `/message/sendLocation/:instanceName` | `apikey` + instance guards | Body from `SendLocationDto` | Integration-specific send result | Sends geolocation payload. |
| `POST` | `/message/sendContact/:instanceName` | `apikey` + instance guards | Body from `SendContactDto` | Integration-specific send result | Sends one or more contacts. |
| `POST` | `/message/sendReaction/:instanceName` | `apikey` + instance guards | Body from `SendReactionDto` | Integration-specific send result | Emoji validation enforced. |
| `POST` | `/message/sendPoll/:instanceName` | `apikey` + instance guards | Body from `SendPollDto` | Integration-specific send result | Sends poll. |
| `POST` | `/message/sendList/:instanceName` | `apikey` + instance guards | Body from `SendListDto` | Integration-specific send result | Interactive list route. |
| `POST` | `/message/sendButtons/:instanceName` | `apikey` + instance guards | Body from `SendButtonsDto` | Integration-specific send result | Interactive buttons route. |

## Chat, Contacts, Presence, Profile, and Privacy Routes

### DTO Inputs

#### `POST /chat/whatsappNumbers/:instanceName`

Body:

```json
{
  "numbers": ["15551234567", "15557654321"]
}
```

#### `POST /chat/markMessageAsRead/:instanceName`

Body:

```json
{
  "readMessages": [
    {
      "id": "wamid-or-message-id",
      "fromMe": false,
      "remoteJid": "15551234567@s.whatsapp.net"
    }
  ]
}
```

#### `POST /chat/archiveChat/:instanceName`

Body fields:

- `lastMessage?: { key, messageTimestamp? }`
- `chat?: string`
- `archive: boolean`

#### `POST /chat/markChatUnread/:instanceName`

Body fields:

- `lastMessage?: { key, messageTimestamp? }`
- `chat?: string`

#### `DELETE /chat/deleteMessageForEveryone/:instanceName`

Body fields:

- `id: string`
- `fromMe: boolean`
- `remoteJid: string`
- `participant?: string`

#### `POST /chat/fetchProfilePictureUrl/:instanceName`

Body:

```json
{
  "number": "15551234567"
}
```

#### `POST /chat/getBase64FromMediaMessage/:instanceName`

Body fields:

- `message: proto.WebMessageInfo`
- `convertToMp4?: boolean`

#### `POST /chat/updateMessage/:instanceName`

Body fields:

- `number: string`
- `key: proto.IMessageKey`
- `text: string`

#### `POST /chat/sendPresence/:instanceName`

Body fields:

- `number: string`
- `presence: WAPresence`
- `delay: number`

#### `POST /chat/updateBlockStatus/:instanceName`

Body:

```json
{
  "number": "15551234567",
  "status": "block"
}
```

#### Repository-style query endpoints

These routes accept query/filter objects rather than narrow simple DTOs:

- `POST /chat/findContacts/:instanceName`
- `POST /chat/findMessages/:instanceName`
- `POST /chat/findStatusMessage/:instanceName`
- `POST /chat/findChats/:instanceName`

Exact filter shape depends on repository query usage.

#### `GET /chat/findChatByRemoteJid/:instanceName`

Query string:

- `remoteJid` required

#### Profile and privacy routes

- `POST /chat/fetchBusinessProfile/:instanceName`
  - body `{ "number": "..." }`
- `POST /chat/fetchProfile/:instanceName`
  - body `{ "number": "..." }`
- `POST /chat/updateProfileName/:instanceName`
  - body `{ "name": "..." }`
- `POST /chat/updateProfileStatus/:instanceName`
  - body `{ "status": "..." }`
- `POST /chat/updateProfilePicture/:instanceName`
  - body `{ "picture": "<url-or-base64>" }`
- `DELETE /chat/removeProfilePicture/:instanceName`
  - no body required
- `GET /chat/fetchPrivacySettings/:instanceName`
  - no body
- `POST /chat/updatePrivacySettings/:instanceName`
  - body with:
    - `readreceipts`
    - `profile`
    - `status`
    - `online`
    - `last`
    - `groupadd`

### Chat Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/chat/whatsappNumbers/:instanceName` | `apikey` + instance guards | Body `{ numbers: string[] }` | Existence lookup results from live instance | Useful for number validation. |
| `POST` | `/chat/markMessageAsRead/:instanceName` | `apikey` + instance guards | Body `{ readMessages: Key[] }` | Integration-specific acknowledgement | Control route for marking read. |
| `POST` | `/chat/archiveChat/:instanceName` | `apikey` + instance guards | Body from `ArchiveChatDto` | Integration-specific result | Archives/unarchives chat. |
| `POST` | `/chat/markChatUnread/:instanceName` | `apikey` + instance guards | Body from `MarkChatUnreadDto` | Integration-specific result | Marks chat unread. |
| `DELETE` | `/chat/deleteMessageForEveryone/:instanceName` | `apikey` + instance guards | Body from `DeleteMessage` | Integration-specific result | Deletes message for everyone. |
| `POST` | `/chat/fetchProfilePictureUrl/:instanceName` | `apikey` + instance guards | Body `{ number }` | Profile picture result from live instance | Fetches profile picture URL/details. |
| `POST` | `/chat/getBase64FromMediaMessage/:instanceName` | `apikey` + instance guards | Body with full message payload | Base64 extraction result | Useful when raw message payload is available. |
| `POST` | `/chat/updateMessage/:instanceName` | `apikey` + instance guards | Body from `UpdateMessageDto` | Integration-specific result | Code comment notes media update support is incomplete. |
| `POST` | `/chat/sendPresence/:instanceName` | `apikey` + instance guards | Body from `SendPresenceDto` | Integration-specific acknowledgement | Typing/presence-like control. |
| `POST` | `/chat/updateBlockStatus/:instanceName` | `apikey` + instance guards | Body `{ number, status }` | Integration-specific result | Block/unblock route. |
| `POST` | `/chat/findContacts/:instanceName` | `apikey` + instance guards | Repository-style query body | Stored contact rows | Platform-side data lookup. |
| `POST` | `/chat/findMessages/:instanceName` | `apikey` + instance guards | Repository-style query body | Stored message rows | Platform-side data lookup. |
| `POST` | `/chat/findStatusMessage/:instanceName` | `apikey` + instance guards | Repository-style query body | Stored message status rows | Useful for delivery/read tracking. |
| `POST` | `/chat/findChats/:instanceName` | `apikey` + instance guards | Repository-style query body | Stored chat rows | Platform-side chat lookup. |
| `GET` | `/chat/findChatByRemoteJid/:instanceName?remoteJid=...` | `apikey` + instance guards | Query `remoteJid` required | Chat lookup result | Returns 400 if `remoteJid` missing. |
| `POST` | `/chat/fetchBusinessProfile/:instanceName` | `apikey` + instance guards | Body `{ number }` | Business profile result | Business profile lookup. |
| `POST` | `/chat/fetchProfile/:instanceName` | `apikey` + instance guards | Body `{ number }` | Profile result | Profile lookup. |
| `POST` | `/chat/updateProfileName/:instanceName` | `apikey` + instance guards | Body `{ name }` | Integration-specific result | Updates own profile name. |
| `POST` | `/chat/updateProfileStatus/:instanceName` | `apikey` + instance guards | Body `{ status }` | Integration-specific result | Updates own profile status. |
| `POST` | `/chat/updateProfilePicture/:instanceName` | `apikey` + instance guards | Body `{ picture }` | Integration-specific result | Updates own profile picture. |
| `DELETE` | `/chat/removeProfilePicture/:instanceName` | `apikey` + instance guards | No body | Integration-specific result | Removes own profile picture. |
| `GET` | `/chat/fetchPrivacySettings/:instanceName` | `apikey` + instance guards | No body | Privacy settings payload from live instance | Current privacy settings lookup. |
| `POST` | `/chat/updatePrivacySettings/:instanceName` | `apikey` + instance guards | Body from `PrivacySettingDto` | Integration-specific result | Updates privacy settings. |

## Group Routes

### DTO Inputs

- `CreateGroupDto`
  - `subject: string`
  - `participants: string[]`
  - `description?: string`
  - `promoteParticipants?: boolean`
- `GroupPictureDto`
  - `groupJid: string`
  - `image: string`
- `GroupSubjectDto`
  - `groupJid: string`
  - `subject: string`
- `GroupDescriptionDto`
  - `groupJid: string`
  - `description: string`
- `GroupJid`
  - `groupJid: string`
- `GetParticipant`
  - `getParticipants: string`
- `GroupInvite`
  - `inviteCode: string`
- `AcceptGroupInvite`
  - `inviteCode: string`
- `GroupSendInvite`
  - `groupJid: string`
  - `description: string`
  - `numbers: string[]`
- `GroupUpdateParticipantDto`
  - `groupJid: string`
  - `action: "add" | "remove" | "promote" | "demote"`
  - `participants: string[]`
- `GroupUpdateSettingDto`
  - `groupJid: string`
  - `action: "announcement" | "not_announcement" | "unlocked" | "locked"`
- `GroupToggleEphemeralDto`
  - `groupJid: string`
  - `expiration: 0 | 86400 | 604800 | 7776000`

### Group Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/group/create/:instanceName` | `apikey` + instance guards | Body from `CreateGroupDto` | Integration-specific group creation result | Creates group with participants. |
| `POST` | `/group/updateGroupSubject/:instanceName` | `apikey` + instance guards | Body `{ groupJid, subject }` | Integration-specific result | `groupJid` normalized to `@g.us` if needed. |
| `POST` | `/group/updateGroupPicture/:instanceName` | `apikey` + instance guards | Body `{ groupJid, image }` | Integration-specific result | Group picture update. |
| `POST` | `/group/updateGroupDescription/:instanceName` | `apikey` + instance guards | Body `{ groupJid, description }` | Integration-specific result | Description update. |
| `GET` | `/group/findGroupInfos/:instanceName` | `apikey` + instance guards | Query `groupJid` required | Group info result | Uses query or body normalization. |
| `GET` | `/group/fetchAllGroups/:instanceName` | `apikey` + instance guards | Query `getParticipants` required | Group list result | Query flag influences participant inclusion. |
| `GET` | `/group/participants/:instanceName` | `apikey` + instance guards | Query `groupJid` required | Participant list result | Group participant lookup. |
| `GET` | `/group/inviteCode/:instanceName` | `apikey` + instance guards | Query `groupJid` required | Invite code result | Fetches group invite code. |
| `GET` | `/group/inviteInfo/:instanceName` | `apikey` + instance guards | Query `inviteCode` required | Invite info result | Reads invite metadata. |
| `GET` | `/group/acceptInviteCode/:instanceName` | `apikey` + instance guards | Query `inviteCode` required | Join result | Accept group invite. |
| `POST` | `/group/sendInvite/:instanceName` | `apikey` + instance guards | Body `{ groupJid, description, numbers[] }` | Integration-specific result | Sends invite to numbers. |
| `POST` | `/group/revokeInviteCode/:instanceName` | `apikey` + instance guards | Body or query with `groupJid` | Integration-specific result | Revokes invite code. |
| `POST` | `/group/updateParticipant/:instanceName` | `apikey` + instance guards | Body from `GroupUpdateParticipantDto` | Integration-specific result | Add/remove/promote/demote participants. |
| `POST` | `/group/updateSetting/:instanceName` | `apikey` + instance guards | Body from `GroupUpdateSettingDto` | Integration-specific result | Group setting update. |
| `POST` | `/group/toggleEphemeral/:instanceName` | `apikey` + instance guards | Body from `GroupToggleEphemeralDto` | Integration-specific result | Toggle disappearing messages. |
| `DELETE` | `/group/leaveGroup/:instanceName` | `apikey` + instance guards | Body or query with `groupJid` | Integration-specific result | Leaves the group. |

## Business Routes

### DTO Inputs

- `getCatalogDto`
  - `number?: string`
  - `limit?: number`
  - `cursor?: string`
- `getCollectionsDto`
  - `number?: string`
  - `limit?: number`

### Business Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/business/getCatalog/:instanceName` | `apikey` + instance guards | Body `{ number?, limit?, cursor? }` | Catalog result or standardized Meta-style error payload | Error path uses `createMetaErrorResponse`. |
| `POST` | `/business/getCollections/:instanceName` | `apikey` + instance guards | Body `{ number?, limit? }` | Collections result or standardized Meta-style error payload | Error path uses `createMetaErrorResponse`. |

## Call Route

### DTO Input

- `OfferCallDto`
  - `number: string`
  - `isVideo?: boolean`
  - `callDuration?: number`

### Call Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/call/offer/:instanceName` | `apikey` + instance guards | Body from `OfferCallDto` | Integration-specific call offer result | Thin call/call-offer support, not a full voice provider API. |

## Template Management Routes

### DTO Inputs

- `TemplateDto`
  - `name: string`
  - `category: string`
  - `allowCategoryChange: boolean`
  - `language: string`
  - `components: any`
  - `webhookUrl?: string`
- `TemplateEditDto`
  - `templateId: string`
  - `category?: "AUTHENTICATION" | "MARKETING" | "UTILITY"`
  - `allowCategoryChange?: boolean`
  - `ttl?: number`
  - `components?: any`
- `TemplateDeleteDto`
  - `name: string`
  - `hsmId?: string`

### Template Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/template/create/:instanceName` | `apikey` + instance guards | Body from `TemplateDto` | Template creation result or standardized Meta-style error payload | Used for template administration, distinct from `sendTemplate`. |
| `POST` | `/template/edit/:instanceName` | `apikey` + instance guards | Body from `TemplateEditDto` | Template edit result or standardized Meta-style error payload | Template update route. |
| `DELETE` | `/template/delete/:instanceName` | `apikey` + instance guards | Body from `TemplateDeleteDto` | Template delete result or standardized Meta-style error payload | Deletes template by name and optional `hsmId`. |
| `GET` | `/template/find/:instanceName` | `apikey` + instance guards | No body | Template list/find result or standardized Meta-style error payload | Fetches template inventory from integration. |

## Label Routes

### DTO Inputs

- `LabelDto`
  - `id?: string`
  - `name: string`
  - `color: string`
  - `predefinedId?: string`
- `HandleLabelDto`
  - `number: string`
  - `labelId: string`
  - `action: "add" | "remove"`

### Label Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/label/findLabels/:instanceName` | `apikey` + instance guards | No body | Label list result | Fetches labels from live instance integration. |
| `POST` | `/label/handleLabel/:instanceName` | `apikey` + instance guards | Body `{ number, labelId, action }` | Label association result | Adds or removes label association. |

## Settings Routes

### DTO Input

- `SettingsDto`
  - `rejectCall?: boolean`
  - `msgCall?: string`
  - `groupsIgnore?: boolean`
  - `alwaysOnline?: boolean`
  - `readMessages?: boolean`
  - `readStatus?: boolean`
  - `syncFullHistory?: boolean`
  - `wavoipToken?: string`

### Settings Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/settings/set/:instanceName` | `apikey` + instance guards | Body from `SettingsDto` | Settings persistence result | Stores local runtime settings for the instance. |
| `GET` | `/settings/find/:instanceName` | `apikey` + instance guards | No body | Current stored settings | Reads local instance settings. |

## Proxy Routes

### DTO Input

- `ProxyDto`
  - `enabled?: boolean`
  - `host: string`
  - `port: string`
  - `protocol: string`
  - `username?: string`
  - `password?: string`

### Proxy Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/proxy/set/:instanceName` | `apikey` + instance guards | Body from `ProxyDto` | Proxy persistence result | Stores instance-level proxy settings. |
| `GET` | `/proxy/find/:instanceName` | `apikey` + instance guards | No body | Current proxy config | Reads stored proxy config. |

## Event Transport Configuration Routes

### Shared Event DTO Input

Body can include any of:

- `webhook?: { enabled?, events?, url?, headers?, byEvents?, base64? }`
- `websocket?: { enabled?, events? }`
- `sqs?: { enabled?, events? }`
- `rabbitmq?: { enabled?, events? }`
- `nats?: { enabled?, events? }`
- `pusher?: { enabled?, appId?, key?, secret?, cluster?, useTLS?, events? }`
- `kafka?: { enabled?, events? }`

### Event Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/webhook/set/:instanceName` | `apikey` + instance guards | Body with `webhook` config | Stored webhook config result | Configures outbound webhook delivery. |
| `GET` | `/webhook/find/:instanceName` | `apikey` + instance guards | No body | Current webhook config | Inspection route. |
| `POST` | `/websocket/set/:instanceName` | `apikey` + instance guards | Body with `websocket` config | Stored websocket config result | Configures websocket event delivery. |
| `GET` | `/websocket/find/:instanceName` | `apikey` + instance guards | No body | Current websocket config | Inspection route. |
| `POST` | `/rabbitmq/set/:instanceName` | `apikey` + instance guards | Body with `rabbitmq` config | Stored RabbitMQ config result | Queue transport config. |
| `GET` | `/rabbitmq/find/:instanceName` | `apikey` + instance guards | No body | Current RabbitMQ config | Inspection route. |
| `POST` | `/nats/set/:instanceName` | `apikey` + instance guards | Body with `nats` config | Stored NATS config result | Queue transport config. |
| `GET` | `/nats/find/:instanceName` | `apikey` + instance guards | No body | Current NATS config | Inspection route. |
| `POST` | `/pusher/set/:instanceName` | `apikey` + instance guards | Body with `pusher` config | Stored Pusher config result | Push transport config. |
| `GET` | `/pusher/find/:instanceName` | `apikey` + instance guards | No body | Current Pusher config | Inspection route. |
| `POST` | `/sqs/set/:instanceName` | `apikey` + instance guards | Body with `sqs` config | Stored SQS config result | Queue transport config. |
| `GET` | `/sqs/find/:instanceName` | `apikey` + instance guards | No body | Current SQS config | Inspection route. |
| `POST` | `/kafka/set/:instanceName` | `apikey` + instance guards | Body with `kafka` config | Stored Kafka config result | Queue transport config. |
| `GET` | `/kafka/find/:instanceName` | `apikey` + instance guards | No body | Current Kafka config | Inspection route. |

## Storage Routes

### DTO Input

- `MediaDto`
  - `id?: string`
  - `type?: string`
  - `messageId?: number`
  - `expiry?: number`

### Storage Route Table

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/s3/getMedia/:instanceName` | `apikey` + instance guards | Body `{ id?, type?, messageId? }` | Array of stored media rows with fields like `id`, `fileName`, `type`, `mimetype`, `createdAt`, `Message` | Queries stored media metadata, not raw bytes. |
| `POST` | `/s3/getMediaUrl/:instanceName` | `apikey` + instance guards | Body `{ id, expiry? }` | JSON with `mediaUrl` plus stored media metadata | Returns signed/generated object URL for stored media. |

## Baileys-Specific Technical Routes

These routes are lower-level technical endpoints tied to the Baileys integration.
They are exposed, but they are not the first-choice abstraction surface for most platform integrations.

Request inputs are usually raw integration-specific bodies plus `instanceName`.

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/baileys/onWhatsapp/:instanceName` | `apikey` + instance guards | Raw body passed to `baileysController.onWhatsapp` | Integration-specific result | Low-level on-WhatsApp lookup helper. |
| `POST` | `/baileys/profilePictureUrl/:instanceName` | `apikey` + instance guards | Raw body passed through | Integration-specific result | Low-level profile picture helper. |
| `POST` | `/baileys/assertSessions/:instanceName` | `apikey` + instance guards | Raw body passed through | Integration-specific result | Session assertion helper. |
| `POST` | `/baileys/createParticipantNodes/:instanceName` | `apikey` + instance guards | Raw body passed through | Integration-specific result | Group/participant node helper. |
| `POST` | `/baileys/getUSyncDevices/:instanceName` | `apikey` + instance guards | Raw body passed through | Integration-specific result | Device sync helper. |
| `POST` | `/baileys/generateMessageTag/:instanceName` | `apikey` + instance guards | No meaningful body required | Integration-specific result | Generates message tag. |
| `POST` | `/baileys/sendNode/:instanceName` | `apikey` + instance guards | Raw body passed through | Integration-specific result | Very low-level node send. |
| `POST` | `/baileys/signalRepositoryDecryptMessage/:instanceName` | `apikey` + instance guards | Raw body passed through | Integration-specific result | Low-level decrypt helper. |
| `POST` | `/baileys/getAuthState/:instanceName` | `apikey` + instance guards | No meaningful body required | Integration-specific result | Returns auth-state details. |

## Chatbot Integration Routes

These routes are available in the codebase and part of the HTTP surface, but they are application integrations rather than core WhatsApp provider routes.

### Chatwoot

#### DTO Input

- `ChatwootDto`
  - `enabled?: boolean`
  - `accountId?: string`
  - `token?: string`
  - `url?: string`
  - `nameInbox?: string`
  - `signMsg?: boolean`
  - `signDelimiter?: string`
  - `number?: string`
  - `reopenConversation?: boolean`
  - `conversationPending?: boolean`
  - `mergeBrazilContacts?: boolean`
  - `importContacts?: boolean`
  - `importMessages?: boolean`
  - `daysLimitImportMessages?: number`
  - `autoCreate?: boolean`
  - `organization?: string`
  - `logo?: string`
  - `ignoreJids?: string[]`

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/chatwoot/set/:instanceName` | `apikey` + instance guards | Body from `ChatwootDto` | Chatwoot config persistence result | Configures Chatwoot integration for instance. |
| `GET` | `/chatwoot/find/:instanceName` | `apikey` + instance guards | No body | Current Chatwoot config | Inspection route. |
| `POST` | `/chatwoot/webhook/:instanceName` or equivalent resolved path | None | Chatwoot webhook payload | Result from webhook handler | Public inbound Chatwoot webhook route. |

### OpenAI

These routes follow a CRUD/settings/session pattern. Exact payload DTOs live inside the OpenAI integration module and are not re-declared here field-by-field.

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/openai/creds/:instanceName` | `apikey` + instance guards | OpenAI credential payload | Credential creation result | Stores OpenAI credentials. |
| `GET` | `/openai/creds/:instanceName` | `apikey` + instance guards | No body | Credential list/details | Reads stored credentials. |
| `DELETE` | `/openai/creds/:instanceName/:openaiCredsId` | `apikey` + instance guards | Path param `openaiCredsId` | Delete result | Deletes credential entry. |
| `POST` | `/openai/create/:instanceName` | `apikey` + instance guards | Bot creation payload | Create result | Creates OpenAI bot config. |
| `GET` | `/openai/find/:instanceName` | `apikey` + instance guards | No body | Bot list result | Lists OpenAI bots. |
| `GET` | `/openai/fetch/:openaiBotId/:instanceName` | `apikey` + instance guards | Path param `openaiBotId` | Bot detail result | Reads one bot. |
| `PUT` | `/openai/update/:openaiBotId/:instanceName` | `apikey` + instance guards | Update payload | Update result | Updates bot config. |
| `DELETE` | `/openai/delete/:openaiBotId/:instanceName` | `apikey` + instance guards | Path param `openaiBotId` | Delete result | Deletes bot config. |
| `POST` | `/openai/settings/:instanceName` | `apikey` + instance guards | Settings payload | Settings persistence result | Stores OpenAI integration settings. |
| `GET` | `/openai/fetchSettings/:instanceName` | `apikey` + instance guards | No body | Settings result | Reads OpenAI settings. |
| `POST` | `/openai/changeStatus/:instanceName` | `apikey` + instance guards | Status payload | Status update result | Enables/disables bot/runtime behavior. |
| `GET` | `/openai/fetchSessions/:openaiBotId/:instanceName` | `apikey` + instance guards | Path param `openaiBotId` | Session list result | Reads bot sessions. |
| `POST` | `/openai/ignoreJid/:instanceName` | `apikey` + instance guards | Ignore list payload | Result | Ignores one or more JIDs. |
| `GET` | `/openai/getModels/:instanceName` | `apikey` + instance guards | No body | Model list result | Reads available models. |

### Typebot

| Method | Path | Auth | Request In | Response Out | Notes |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/typebot/create/:instanceName` | `apikey` + instance guards | Typebot creation payload | Create result | Creates Typebot integration config. |
| `GET` | `/typebot/find/:instanceName` | `apikey` + instance guards | No body | Bot list result | Lists Typebot configs. |
| `GET` | `/typebot/fetch/:typebotId/:instanceName` | `apikey` + instance guards | Path param `typebotId` | Bot detail result | Reads one Typebot config. |
| `PUT` | `/typebot/update/:typebotId/:instanceName` | `apikey` + instance guards | Update payload | Update result | Updates config. |
| `DELETE` | `/typebot/delete/:typebotId/:instanceName` | `apikey` + instance guards | Path param `typebotId` | Delete result | Deletes config. |
| `POST` | `/typebot/settings/:instanceName` | `apikey` + instance guards | Settings payload | Settings result | Stores Typebot settings. |
| `GET` | `/typebot/fetchSettings/:instanceName` | `apikey` + instance guards | No body | Settings result | Reads Typebot settings. |
| `POST` | `/typebot/start/:instanceName` | `apikey` + instance guards | Start payload | Start result | Starts Typebot flow/session. |
| `POST` | `/typebot/changeStatus/:instanceName` | `apikey` + instance guards | Status payload | Status result | Changes Typebot status. |
| `GET` | `/typebot/fetchSessions/:typebotId/:instanceName` | `apikey` + instance guards | Path param `typebotId` | Session list result | Reads sessions. |
| `POST` | `/typebot/ignoreJid/:instanceName` | `apikey` + instance guards | Ignore list payload | Result | Ignore JID route. |

### Repeated CRUD Pattern Integrations

The following prefixes use the same route family shape:

- `evolutionBot`
- `dify`
- `flowise`
- `n8n`
- `evoai`

For each prefix:

- `POST /<prefix>/create/:instanceName`
- `GET /<prefix>/find/:instanceName`
- `GET /<prefix>/fetch/:id/:instanceName`
- `PUT /<prefix>/update/:id/:instanceName`
- `DELETE /<prefix>/delete/:id/:instanceName`
- `POST /<prefix>/settings/:instanceName`
- `GET /<prefix>/fetchSettings/:instanceName`
- `POST /<prefix>/changeStatus/:instanceName`
- `GET /<prefix>/fetchSessions/:id/:instanceName`
- `POST /<prefix>/ignoreJid/:instanceName`

Concrete `:id` params:

- `evolutionBotId`
- `difyId`
- `flowiseId`
- `n8nId`
- `evoaiId`

These all follow the same high-level contract:

- auth: `apikey` + instance guards
- request in: integration-specific creation/update/settings/status payloads
- response out: CRUD/settings/session results for that integration

## Event Names Emitted in Code

The event system defines these names in `wa.types.ts`:

- `application.startup`
- `instance.create`
- `instance.delete`
- `qrcode.updated`
- `connection.update`
- `status.instance`
- `messages.set`
- `messages.upsert`
- `messages.edited`
- `messages.update`
- `messages.delete`
- `send.message`
- `send.message.update`
- `contacts.set`
- `contacts.upsert`
- `contacts.update`
- `presence.update`
- `chats.set`
- `chats.update`
- `chats.upsert`
- `chats.delete`
- `groups.upsert`
- `groups.update`
- `group-participants.update`
- `call`
- `typebot.start`
- `typebot.change-status`
- `labels.edit`
- `labels.association`
- `creds.update`
- `messaging-history.set`
- `remove.instance`
- `logout.instance`

These are not HTTP endpoints, but they are part of the effective API surface because event transport routes can emit them to your platform.

## Known Gaps and Caveats

- There is no built-in hosted QR page URL such as `/instance/:name/qr`.
- There is no provider-grade phone number inventory API with quality/tier/throughput metadata.
- Inbound media retrieval exists, but not as a perfectly clean `provider media ID -> file download` abstraction.
- Many live send and action responses are integration-specific rather than normalized in the route/controller layer.
- For provider adapter work, the safest pattern is:
  - persist raw send response
  - persist raw event payload
  - normalize correlation IDs in your adapter

## Runtime Dependency Summary

- PostgreSQL is used as the main persistent database in the normal setup.
- Redis is enabled by default for cache/session support.
- Persistent volume storage is used for instance/session files under `/evolution/instances`.
- S3/MinIO storage is optional and only needed for the explicit storage integration path.
