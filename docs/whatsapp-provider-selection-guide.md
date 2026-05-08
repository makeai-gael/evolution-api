# WhatsApp Provider Selection Guide

## Purpose

This document defines the WhatsApp provider capabilities that matter most for the current Support Bot platform.

The goal is not to list every possible WhatsApp API feature. The goal is to identify:

- which provider capabilities are essential for the current platform shape
- which capabilities are useful but not required on day one
- how each capability maps to actual platform features already present in the backend and frontend design

This platform is not a simple message relay. It is a multi-tenant WhatsApp bot platform with:

- tenant-managed WhatsApp numbers
- AI chat bots
- flow bots
- external webhook-driven bots
- voice bot support in the broader architecture
- human handoff / escalation flows
- media and file handling
- usage, billing, and delivery observability
- simulation and live channel modes

Because of that, provider selection should favor operational reliability and runtime coverage over a long tail of optional WhatsApp features.

## Platform Context

The current platform architecture and product behavior imply these non-negotiable needs:

- inbound WhatsApp events must be persisted quickly and processed asynchronously
- one conversation can contain text, media, tool calls, escalation, and outbound follow-ups
- bots can send text, templates, interactive messages, and media
- the system tracks conversations, messages, delivery states, webhook logs, escalation logs, and billing usage
- numbers and provider metadata are visible in tenant and admin workflows
- simulation mode is important, so live-provider behavior should be cleanly abstractable

That means the best provider is not the one with the largest feature catalog. It is the one whose API surface maps cleanly to the platform's actual runtime and operating model.

## Most Useful Capabilities

These are the provider API capabilities that should be treated as first-priority requirements.

### 1. Inbound Webhooks for Messages and Events

**Why it matters**

This is the core ingress path for the whole system. Without reliable inbound webhooks, the platform cannot receive user messages, reactions, delivery updates, read receipts, or other channel events in a timely way.

**What the provider should support**

- inbound user message webhooks
- delivery lifecycle webhooks
- read receipt webhooks
- status change or message-state webhooks
- stable event identifiers for deduplication and idempotency
- webhook retry behavior that is documented and predictable

**Why this is essential for this platform**

The backend architecture depends on fast event persistence, conversation upsert, message logging, job enqueueing, and asynchronous bot execution. Webhook quality directly affects reliability, observability, and duplicate protection.

### 2. Outbound Message Sending

**Why it matters**

Every bot type depends on outbound delivery. This is the most basic provider responsibility after inbound webhook delivery.

**What the provider should support**

- outbound text messages
- replies into existing conversations
- provider-side message IDs returned immediately
- clear error responses for rate limits, invalid payloads, and failed sends

**Why this is essential for this platform**

The runtime emits outbound bot actions and expects provider acknowledgements, delivery tracking, and retry handling. If sending is inconsistent or underspecified, the platform's conversation engine becomes unreliable.

### 3. Media Upload and Media Retrieval

**Why it matters**

The platform supports media-capable bots, runtime message files, and file-driven interactions. A provider that only handles text well is not enough.

**What the provider should support**

- upload media for outbound messages
- retrieve or download inbound media
- stable media identifiers
- content-type and filename metadata
- support for image, audio, and document flows

**Why this is essential for this platform**

The architecture separates bot setup files from runtime conversation files and explicitly supports inbound and outbound media. This capability is required for image-based support, audio handling, document delivery, and future multimodal improvements.

### 4. Template Message API

**Why it matters**

Template messages are required for structured outbound messaging outside the normal customer-care window and for approved notification-style communications.

**What the provider should support**

- send template messages
- support template parameters
- template language selection
- clear template send errors and status visibility

**Why this is essential for this platform**

The bot model and runtime already include `send_template` as a first-class action. Without template support, the platform is constrained in re-engagement, notification, and operational follow-up flows.

### 5. Interactive Message API

**Why it matters**

Interactive messages are one of the highest-value features for flow bots because they reduce ambiguity and increase completion rates.

**What the provider should support**

- reply buttons
- list messages
- interactive response callbacks with selected option IDs
- consistent payload structure for outbound interactive messages and inbound user selections

**Why this is essential for this platform**

The frontend bot builder already includes interactive flows and wait-for-response patterns. Interactive provider support maps directly to the platform's canvas model and is far more valuable than many more advanced but less-used WhatsApp features.

### 6. Delivery Status and Read Receipt Tracking

**Why it matters**

Outbound send success is not enough. The platform needs to know whether messages were sent, delivered, read, or failed.

**What the provider should support**

- message status callbacks
- sent, delivered, read, and failed states
- correlation between provider message IDs and status events
- enough error detail to distinguish transient failures from permanent failures

**Why this is essential for this platform**

The backend already models outbound actions, outbound deliveries, webhook logs, and conversation state. Delivery events improve retries, support analytics, troubleshooting, and more accurate operational reporting.

### 7. Phone Number and Number-Metadata APIs

**Why it matters**

The platform is multi-tenant and number-oriented. Admins and tenant operators need visibility into the numbers bound to their bots.

**What the provider should support**

- list phone numbers
- get phone number details
- number status
- verified name
- quality rating
- messaging limit tier
- throughput or capacity metadata when available

**Why this is essential for this platform**

The architecture and data model explicitly track provider number metadata. This is important for admin workflows, operational support, and understanding channel quality and scale limits.

### 8. Mark-as-Read and Presence-Type Controls

**Why it matters**

This is not as foundational as sending or webhooks, but it materially improves user experience.

**What the provider should support**

- mark message as read
- typing indicator or similar presence control if the provider exposes it

**Why this is still high-value**

The runtime architecture already includes bot actions such as `mark_read` and `typing_indicator`. These features make bot interactions feel more natural and responsive, especially for AI-driven turns with non-trivial latency.

## Good to Have Capabilities

These capabilities are useful, but they should not outrank the first-priority set above when choosing a provider for the current platform.

### 1. Conversation and Contact Lookup APIs

**Why it is useful**

These APIs help with reconciliation, support tooling, audit workflows, and backfilling operational data.

**Why it is not first priority**

The backend is designed as the system of record. If inbound webhooks and message send APIs are strong, the platform can maintain most of the required state itself.

### 2. Template Management APIs

**Why it is useful**

Programmatic template create, update, and sync flows can improve admin automation.

**Why it is not first priority**

The platform mainly needs to send templates reliably. Template lifecycle management can still be handled in the provider dashboard if necessary.

### 3. Provider-Native Flow APIs

**Why it is useful**

Native WhatsApp flow support can be helpful for specific commerce or structured data-capture use cases.

**Why it is not first priority**

The platform already has its own flow runtime, visual builder, state handling, and routing model. Provider-native flow features are additive, not foundational.

### 4. Contact or Profile Enrichment APIs

**Why it is useful**

Extra contact metadata can help customer support and analytics.

**Why it is not first priority**

It is less important than stable delivery, media handling, and interactive messaging. It improves context, but it does not define whether the platform can operate correctly.

### 5. Voice or Calling APIs

**Why it is useful**

These matter if the voice-bot mode needs to run through the same provider stack.

**Why it is not first priority**

Voice is part of the broader architecture, but WhatsApp messaging remains the primary product surface. A provider should not win purely because of voice support if its core messaging APIs are weaker.

### 6. Catalog or Commerce APIs

**Why it is useful**

These can support native shopping and product presentation workflows inside WhatsApp.

**Why it is not first priority**

The current platform can already implement recommendation, lead capture, and order-routing logic in bot flows without needing provider-native commerce features on day one.

## Recommended Evaluation Order

When comparing providers, evaluate in this order:

1. webhook reliability and event completeness
2. outbound send reliability and error model
3. media upload and inbound media retrieval
4. interactive message support
5. template support
6. delivery/read status tracking
7. number metadata and account visibility
8. operational extras such as template management, native flows, and commerce APIs

This order reflects the actual runtime dependency chain of the platform.

## Minimum Acceptance Checklist

A WhatsApp provider is a practical fit for this platform only if it can support all of the following at a reasonable quality level:

- receive inbound message webhooks reliably
- send outbound text messages reliably
- support media upload and inbound media retrieval
- support template sends
- support interactive messages and callbacks
- provide delivery and read status events
- expose phone number metadata useful for admin and tenant operations
- return stable provider IDs for messages and media
- document auth, retries, rate limits, and webhook behavior clearly

If any of those are weak or missing, integration cost and operational risk will rise sharply.

## Final Guidance

For this platform, the best WhatsApp provider is the one that is strongest in:

- event reliability
- outbound messaging consistency
- media handling
- interactive and template support
- operational metadata and delivery tracking

The best provider is not necessarily the one with the broadest optional feature catalog.

For this product, `most useful` should beat `good to have` every time.
