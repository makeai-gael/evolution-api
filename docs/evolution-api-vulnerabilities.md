# Evolution API Vulnerability Review

## Review Scope

This review focused on:

- HTTP exposure in `src/main.ts` and `src/api/routes/**`
- auth and websocket guards
- webhook/event configuration
- connection/session handling for WhatsApp onboarding
- container exposure patterns

This is a source review, not a pen test. No formal automated security suite exists in this repository.

## Executive Summary

The codebase is feature-rich, but there are several high-impact issues around exposed internal data, weak defaults, and trust boundaries around webhooks and metrics. The most serious problems are public static exposure of the `store` directory, disabled webhook URL validation, and missing signature validation on inbound Meta webhooks.

## Findings

### 1. Critical: `/store` is exposed as a public static directory

- Evidence: `src/main.ts:71`
- Code: `app.use('/store', express.static(join(ROOT_DIR, 'store')));`
- Risk:
  - makes internal runtime files web-accessible
  - may expose cached media, auth artifacts, chatbot state, or integration state stored under `store/`
  - increases blast radius of any accidental secret or session file written there
- Recommended fix:
  - remove the public static mount entirely unless there is a strict business requirement
  - if any subpath must stay public, expose only that specific subdirectory
  - never expose auth/session storage paths over HTTP

### 2. High: webhook destination validation is commented out, enabling SSRF-style abuse

- Evidence: `src/api/integrations/event/webhook/webhook.controller.ts:20-23`
- Code:
  - URL validation exists but is commented out
  - instance-specific webhook URLs are accepted and later used for outbound HTTP requests
- Risk:
  - a user with API access can point callbacks to internal hosts or sensitive network targets
  - can be abused for SSRF, internal service probing, or credential relay through crafted destinations
- Recommended fix:
  - restore strict URL validation
  - deny localhost, RFC1918, link-local, metadata, and internal DNS targets by default
  - add explicit allowlists when possible

### 3. High: Meta webhook POST has no signature verification

- Evidence: `src/api/integrations/channel/meta/meta.router.ts:15-17`
- Current behavior:
  - GET verification token is checked during subscription handshake
  - POST webhook payload is accepted and forwarded directly to the controller
- Risk:
  - anyone who can reach the endpoint can forge inbound Meta webhook payloads
  - fake messages, status changes, or automation triggers may be injected
- Recommended fix:
  - validate Meta webhook signatures on every POST
  - reject unsigned or invalid payloads before business processing

### 4. High: metrics IP whitelist logic is broken

- Evidence: `src/api/routes/index.router.ts:58`
- Code: `if (allowedIPs.filter((ip) => clientIPs.includes(ip)) === 0) { ... }`
- Risk:
  - `filter()` returns an array, so the comparison to `0` is always false
  - when metrics are enabled with IP filtering, the whitelist does not actually protect the route
- Recommended fix:
  - replace with a length check such as `allowedIPs.filter(...).length === 0`
  - also normalize `x-forwarded-for` handling instead of treating it as a single raw value

### 5. Medium: weak hardcoded fallback global API key

- Evidence: `src/config/env.config.ts:867-870`
- Code: `KEY: process.env.AUTHENTICATION_API_KEY || 'BQYHJGJHJ'`
- Risk:
  - if the environment variable is omitted in any deployment, the API falls back to a predictable key
  - this can silently create an insecure production deployment
- Recommended fix:
  - fail fast when `AUTHENTICATION_API_KEY` is not set
  - never ship a default global credential in code

### 6. Medium: websocket handshake bypass for “allowed hosts”

- Evidence: `src/api/integrations/event/websocket/websocket.controller.ts:33-45`
- Current behavior:
  - if the request looks like a Socket.IO handshake and `remoteAddress` is in the allowed host list, the connection is accepted before API key validation
- Risk:
  - behind reverse proxies or misconfigured network layers, `remoteAddress` may collapse to a trusted local hop
  - this weakens the auth boundary for websocket access
- Recommended fix:
  - require API key validation for all websocket clients
  - use network allowlists only as an additional control, not as an auth bypass

## Operational Risks That Are Not Vulnerabilities But Matter

### No dedicated readiness endpoint

- There is no `/health/ready` equivalent
- The simplest health probe today is `GET /`
- For Docker/Kubernetes parity with `support-bot-be`, add a real readiness route that also checks DB and provider state

### No built-in policy scheduler for inactivity or compliance

- There is no native 10-day activity policy
- There is no native 60-day inactivity SMS workflow
- These are feature gaps, not security flaws, but they matter for production policy enforcement

## Priority Order

Recommended fix order:

1. Remove public `/store`
2. Restore webhook destination validation and SSRF defenses
3. Add Meta webhook signature validation
4. Fix metrics whitelist logic
5. Remove fallback API key
6. Tighten websocket auth
