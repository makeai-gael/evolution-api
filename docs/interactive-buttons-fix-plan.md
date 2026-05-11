# Interactive Buttons and Lists Fix Plan

## Purpose

This plan merges:
- validated upstream fixes from the official Evolution repository
- additional findings from local code-path review
- a safe rollout strategy that avoids impacting the currently running instance

Scope: `WHATSAPP-BAILEYS` interactive rendering problems (`sendButtons`, `sendList`, related message relay paths).

## Verified Context

Current project baseline:
- Evolution API: 2.3.7
- Baileys: 7.0.0-rc.9

Local code paths reviewed:
- `POST /message/sendButtons/:instanceName`
- `buttonsMessageSchema`
- DTO mapping for button fields
- Baileys interactive payload build and relay path

Related official upstream evidence:
- Reports of 201/success but no render/delivery for buttons and lists
- Merged fixes targeting interactive rendering and device/web compatibility

## Latest Investigation Evidence

### Scope of this check

This verification was performed without changing source behavior.
Evidence was collected from:
- API container logs
- PostgreSQL persisted `Message` and `MessageUpdate` rows
- current validation code path
- a live validator reproduction inside the running API container

### Active instance and provider

Database query result confirmed the failing tenant is:
- instance: `demo-tenant-27705441611`
- integration: `WHATSAPP-BAILEYS`
- connection status: `open`

### Latest outbound interactive message evidence

The latest relevant outbound record in `Message` shows:
- `messageTimestamp`: `1778514549`
- `messageType`: `viewOnceMessage`
- `remoteJid`: `27730328615@s.whatsapp.net`
- outbound (`fromMe = true`)

Persisted payload excerpt:
- `viewOnceMessage.message.interactiveMessage.nativeFlowMessage.buttons`
- two buttons with `name = quick_reply`
- button params contain valid `display_text` and `id`
- body text persisted as `**\n\nDo you want to continue?\n`

The matching `MessageUpdate` evidence shows:
- `DELIVERY_ACK` for the interactive row

Interpretation:
- transport to WhatsApp succeeded
- the failure is not a send exception or routing failure
- the message is being accepted upstream but is still failing to render on the companion client

### What the stored body proves about the latest input

The service builds button body text as:
- `'*' + data.title + '*'`
- then appends `description` if present

The persisted body was:
- `**\n\nDo you want to continue?\n`

That means the latest accepted payload had:
- `title = ''` or effectively blank
- `description = 'Do you want to continue?'`

So the latest request did not fully conform to the intended semantic input for `sendButtons`.
The button items themselves were structurally valid, but the message heading/title was blank.

### Why the blank title was not rejected by validation

This is a schema enforcement bug, not an API logging issue.

Current validation path:
- route validation runs through `dataValidate(...)`
- the request is rejected only when `jsonschema.validate(...)` returns `valid = false`

Current button schema intends to require non-empty title with:
- `required: ['number', 'title', 'buttons']`
- `...isNotEmpty('title')`

However, `required` only guarantees that the property exists.
It does not reject an empty string.

The intended non-empty enforcement comes from `isNotEmpty(...)`, but that helper is implemented in a way that does not match normal multi-property request objects.

Current helper shape:
- `if: { propertyNames: { enum: [...] } }`
- `then: { properties: { <field>: { minLength: 1 } } }`

Why this fails for the real request object:
- the request object contains keys such as `number`, `title`, and `buttons`
- `propertyNames` applies to every property name in the object
- with `enum: ['title']`, validation only passes if every property name in the object is literally `title`
- because the object also contains `number` and `buttons`, the `if` condition fails
- when the `if` condition fails, the `then` branch with `minLength: 1` is never applied

Result:
- `title: ''` still satisfies `type: 'string'`
- `title` is present, so `required` is satisfied
- the validator returns success even though the title is blank

### Live reproduction of the validation bug

A live check was executed inside the running `evolution_api` container using the current compiled schema.

Test payload:
- `number`: valid string
- `title`: empty string
- `description`: `Do you want to continue?`
- `buttons`: two valid reply buttons

Validator result:
- `valid: true`
- `errors: []`

This confirms the expected rejection did not happen.

### Why buttons still do not render after transport succeeds

The latest evidence indicates two separate problems with different severities:

1. Secondary problem: payload quality gap
- the latest input had a blank title
- validation should have rejected it but did not
- this produced a malformed visible body (`**`) and weakens client compatibility further

2. Primary problem: interactive render format remains incompatible for companion clients
- the persisted outbound message is still a `viewOnceMessage` wrapping `interactiveMessage` and `nativeFlowMessage`
- the message reaches WhatsApp and gets `DELIVERY_ACK`
- the companion client still shows the classic failure state: the message exists but cannot be decoded/rendered there

Conclusion:
- the title issue is real and should have been rejected, but it is not the main cause of the render failure
- the main cause remains the encoded outbound interactive message shape used for Baileys buttons
- delivery success must not be treated as proof of render compatibility

### Practical meaning for future verification

When validating future fixes, success criteria must include all of the following:
- request schema rejects blank `title`
- outbound stored message body reflects a real title, not `**`
- the interactive row still receives normal ack updates
- WhatsApp Web/Desktop renders the buttons instead of showing `This message couldn't load`

## Root Cause Candidates (Ranked)

### 1) High confidence: interactive payload shape/relay strategy mismatch on Baileys

Current implementation builds `viewOnceMessage -> interactiveMessage -> nativeFlowMessage` for buttons.

This path is known to be fragile across WA clients and versions, especially for Web/Desktop and some mobile combinations.

### 2) High confidence: low-level relay behavior for viewOnce interactive path

The send path generates a normalized WA message object but relays the raw object branch for viewOnce handling.

Risk: transport acceptance and message ID creation can still happen while client rendering fails.

### 3) High confidence: missing stanza-level compatibility nodes for some interactive relays

Upstream fix notes indicate interactive messages may require additional biz nodes in relay options to render consistently on Web/Desktop/iOS.

### 4) Medium confidence: schema is permissive for button payload quality

The schema allows minimal payloads that can pass validation but produce poor/invalid runtime button objects (for example missing button text/id combinations).

### 5) Medium confidence: list/button cross-client strategy regression

Upstream notes indicate format split may be required:
- direct native flow for certain interactive types
- legacy list strategy for broad cross-client compatibility

## Upstream Fixes to Port (Official Repo)

### A) Port interactive rendering changes from official PRs

Reference targets from official repository discussion:
- interactive device-sent/button handling refactor
- rendering fixes on Web/Desktop with relay additional nodes

Expected port outcomes:
- no blanket `viewOnceMessage` wrapping for all interactive sends
- correct relay route for top-level interactive/list messages
- optional relay additional nodes for client compatibility

### B) Keep strict button mixing rules

Preserve and enforce runtime guards:
- reply buttons max 3
- reply cannot mix with CTA/other types
- pix single-only and exclusive

### C) Apply list strategy compatible with current clients

Use the list shape and relay metadata that matches tested cross-client rendering behavior from upstream changes.

## Additional Local Fixes (Beyond Upstream)

### 1) Harden schema-level validation for buttons

At minimum enforce for reply buttons:
- `displayText` required
- `id` required

At request level enforce:
- `buttons` non-empty
- `title` required when rendering body from title

### 2) Validate button param JSON before relay

Before send, ensure mapped `buttonParamsJson` is valid and semantically complete by type.

### 3) Add structured debug logs for interactive send flow

For each interactive request, log:
- incoming sanitized payload
- normalized DTO
- final relay message object
- relay options (`additionalNodes`, message ID)
- provider ack/status details

### 4) Add feature-flag fallback path

Add a config flag for interactive strategy selection, for example:
- `native_flow` (new path)
- `legacy` (fallback)

This allows quick rollback without code re-edit.

### 5) Update parser support for modern interactive responses

Current parsing often focuses on older response types.
Add/confirm handling for modern interactive response shapes to avoid partial feature behavior.

## Implementation Plan (Phased)

### Phase 1: Safe patch in isolated branch

1. Create branch for interactive fixes only.
2. Port upstream interactive send/relay strategy.
3. Add schema hardening and guardrails.
4. Add trace logs for interactive sends.

### Phase 2: Compatibility verification matrix

Test each flow on:
- Android WhatsApp Business (free app)
- Android WhatsApp consumer
- WhatsApp Desktop
- WhatsApp Web
- iOS (if available)

Test message types:
- buttons (reply)
- buttons (url/call/copy)
- pix
- lists

### Phase 3: Controlled release

1. Build a separate image tag (do not overwrite current tag).
2. Run separate container stack on different API port.
3. Validate with same tenant-like data pattern.
4. Promote only after successful matrix.

## Rollback Plan

- Keep current image and compose stack untouched.
- If new stack fails, stop only new stack and keep old stack active.
- No migration changes in this fix scope.

## Non-Impact Strategy for Current Running Instance

## Short answer

Yes, it is possible to make these changes without affecting the currently running instance, if you isolate build and runtime.

## Rules to guarantee no impact

1. Do not modify the running container.
2. Do not reuse the same image tag used by production/local active stack.
3. Do not run fixed container with same compose project name and same published ports.
4. Do not point fixed container to the same live database unless explicitly intended for shared-state test.

## Recommended isolation setup

- Keep current stack exactly as is (for example API on 8080).
- Run fix stack with:
  - different compose project name
  - different container names
  - different host ports (for example API on 8081)
  - separate volumes
  - optional separate database

## What would affect current instance

- Restarting/rebuilding the same container from same compose service name and tag will replace runtime behavior.
- Reusing the same database with incompatible schema changes could impact both stacks (not expected in this fix if schema is unchanged).

## Practical Decision

If you need zero risk while continuing normal operations:
- implement fixes on separate branch
- build separate image tag
- run side-by-side stack on alternate port
- switch traffic only after validation

## Acceptance Criteria

The fix is accepted when all are true:
- `sendButtons` renders reliably on phone and Web/Desktop for reply buttons
- no "message couldn't load" in validated clients
- `sendList` renders in validated clients
- API still returns normal send response and persisted message tracking
- fallback/rollback path is available without downtime
