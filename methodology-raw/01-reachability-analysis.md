# Reachability Analysis — papermark

**Target:** `papermark`
**Node:** Layer-1 Reachability Analysis (M14, heavy tier)
**Date:** 2026-06-30
**Method:** GitNexus impact analysis (direction=upstream, maxDepth=3) + direct source audit of entry points, middleware matcher, and auth patterns. Every finding traced from sink backward to HTTP handler.

---

## Reachability Matrix

| # | Finding | Severity | Reachable | Entry Point | Auth | Exposure |
|---|---------|----------|-----------|-------------|------|----------|
| 1 | Conversations API — zero auth | CRITICAL | CONFIRMED | `GET/POST /api/conversations/*` | NONE | INTERNET |
| 2 | Stored XSS via `sanitizePlainText` decode order | HIGH | CONFIRMED | `POST /api/teams/:teamId/documents/:id/update-name` | JWT (NextAuth) | INTERNET |
| 3 | CORS origin reflection on TUS viewer | MEDIUM | CONFIRMED | `OPTIONS/* /api/file/tus-viewer/**` | Dataroom session | INTERNET |
| 4 | `progress-token.ts` — unauthenticated token gen | HIGH | CONFIRMED | `GET /api/progress-token?documentVersionId=` | NONE | INTERNET |
| 5 | `record_reaction.ts` — unauthenticated DB write + info leak | MEDIUM | CONFIRMED | `POST /api/record_reaction` | NONE | INTERNET |
| 6 | `feedback/index.ts` — unauthenticated feedback recording | MEDIUM | CONFIRMED | `POST /api/feedback` | NONE | INTERNET |
| 7 | `revalidate.ts` — shared secret in query param | MEDIUM | CONFIRMED | `GET /api/revalidate?secret=&linkId=` | Shared secret | INTERNET |
| 8 | CVE-2026-23864/23869 — Next.js v14 App Router DoS | HIGH | CONFIRMED | `POST` any `app/` route with RSC headers | NONE | INTERNET |
| 9 | `dangerouslySetInnerHTML` on `helpText` props | MEDIUM | LIKELY | Component render (auth user context) | N/A (XSS) | INTERNET |
| 10 | `dangerouslySetInnerHTML` on domain config | LOW | UNLIKELY | `components/domains/domain-configuration.tsx:143` | N/A (static) | INTERNET |
| 11 | Zod validation info disclosure | MEDIUM | CONFIRMED | Multiple API endpoints (webhooks, agreements) | Varies | INTERNET |
| 12 | No rate limiting on auth endpoints | LOW | CONFIRMED | `pages/api/auth/[...nextauth].ts` | N/A | INTERNET |
| 13 | Webhook event body storage in Tinybird | LOW | CONFIRMED | `app/api/webhooks/callback/route.ts` | QStash sigs | INTERNET |
| 14 | `cuid` 3.0.0 predictable ID generation (permission groups) | HIGH | CONFIRMED | `POST /api/teams/:teamId/datarooms/:id/permission-groups` | JWT (NextAuth) | INTERNET |

---

## Detailed Findings

### Finding 1: Conversations API — Zero Authentication on All Handlers

- **reachable:** CONFIRMED
- **entry point:** `pages/api/conversations/[[...conversations]].ts:5-9` (Pages Router catch-all)
  - `GET /api/conversations?dataroomId=X&viewerId=Y` — list conversations
  - `POST /api/conversations` — create conversation
  - `POST /api/conversations/messages` — add message
  - `POST /api/conversations/notifications` — toggle notifications
- **input type:** query (GET) / body (POST)
- **auth required:** NONE
- **auth bypassable:** N/A — there is no auth to bypass
- **trace:** HTTP → `middleware.ts:49` (excludes `/api/` from middleware) → `pages/api/conversations/[[...conversations]].ts:5-9:handler` (thin passthrough, zero auth) → `ee/features/conversations/api/conversations-route.ts:276-299:handleRoute` (dispatches via `routeHandlers` map, zero auth) → four handlers (lines 19, 77, 202, 258) → prisma CRUD (`conversation.findMany`, `conversationService.createConversation`, `messageService.addMessage`, `notificationService.toggleNotificationsForConversation`) → optionally `sendConversationTeamMemberNotificationTask.trigger()` (Trigger.dev background job sends email)
- **evidence:**
  - `pages/api/conversations/[[...conversations]].ts:1-10` — imports only `handleRoute`, no auth import, no session check
  - `ee/features/conversations/api/conversations-route.ts:1-14` — zero `next-auth` imports
  - All four handlers accept `viewerId` from body/query on trust with zero server-side verification
  - Adjacent `team-conversations-route.ts` demonstrates correct pattern with `getServerSession()` on every handler
- **deployment exposure:** INTERNET — mounted at `/api/conversations`, middleware explicitly excludes `/api/` (line 49), so no middleware auth layer applies. API is self-authenticating and this route simply forgot auth.
- **priority:** P0 — CRITICAL — unauthenticated internet-facing DB writes and enumeration

---

### Finding 2: Stored XSS via `sanitizePlainText` Entity-Decode Ordering + `<xmp>` Bypass

- **reachable:** CONFIRMED
- **entry point 1 (primary):** `POST /api/teams/:teamId/documents/:id/update-name` — `pages/api/teams/[teamId]/documents/[id]/update-name.ts:24`
- **entry point 2 (upload):** `POST` via `app/(ee)/api/links/[id]/upload/route.ts` and `lib/zod/url-validation.ts:207` (document upload schema)
- **entry point 3 (agreements):** `POST /api/teams/:teamId/documents/agreement`
- **entry point 4 (team name):** `POST /api/teams/:teamId/update-name`
- **entry point 5 (FAQ):** EE FAQ routes PUT/POST through `validateContent`
- **input type:** body (name/content field)
- **auth required:** JWT (NextAuth `getServerSession`) — all direct write paths are authenticated
- **auth bypassable:** YES, via XSS worm — once one authenticated user triggers the XSS, it executes in the browser of any other team member viewing the document, enabling token theft and propagation without direct auth
- **trace:**
  - Primary path: HTTP → middleware.ts:49 (excludes `/api/`) → `update-name.ts:30` (`getServerSession(req, res, authOptions)`) → line 41 (`updateNameSchema.safeParse(req.body)`) → Zod `.transform((value) => sanitizePlainText(value))` at line 15 → `lib/utils/sanitize-html.ts:12-19:sanitizePlainText`
  - Bug trigger: `sanitizeHtml(content, {allowedTags: []})` at line 13 (strips nothing — encoded entities like `&lt;` are not literal tags) → `decodeHTML(sanitized)` at line 14 (decodes `&lt;` → `<`, producing real HTML tags) → sanitized-but-now-dangerous string stored in PostgreSQL `Document.name`
  - Sink: `components/documents/document-header.tsx:604` — `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` renders unescaped HTML whenever any team member views the document page
- **depth-2 callers (additional XSS entry points):** 15+ files including `validateContent` (sanitize-html.ts:24), agreement handlers, team name changes, FAQ management, domain validation
- **deployment exposure:** INTERNET — authenticated team member endpoints
- **priority:** P1 — HIGH — stored XSS that fires on every document view page for all team members. Widespread sanitizer usage means many input paths are affected.

---

### Finding 3: CORS Origin Reflection on TUS Viewer Upload Endpoint

- **reachable:** CONFIRMED
- **entry point:** `OPTIONS/GET/POST/DELETE/PATCH/HEAD /api/file/tus-viewer/**` — `pages/api/file/tus-viewer/[[...file]].ts:252:handler`
- **input type:** `Origin` header (reflected verbatim), HTTP body (TUS upload payload)
- **auth required:** Dataroom session (via `onIncomingRequest` callback → `verifyDataroomSessionInPagesRouter`)
- **auth bypassable:** PARTIALLY — CORS configuration allows arbitrary origins with `Access-Control-Allow-Credentials: true`, enabling CSRF-style attacks. An attacker's website can issue authenticated cross-origin TUS uploads to a victim's dataroom if the victim has an active session cookie. However, the upload itself still requires a valid dataroom session.
- **trace:** HTTP → `middleware.ts:49` (excludes `/api/`) → `tus-viewer.ts:252-263:handler` → `setCorsHeaders(req, res)` at line 260 → `res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*")` at line 237 → `res.setHeader("Access-Control-Allow-Credentials", "true")` at line 236 → `tusServer.handle(req, res)` at line 263
- **deployment exposure:** INTERNET
- **priority:** P2 — MEDIUM — requires victim to have active session, but the CORS config removes the Same-Origin Policy protection entirely

---

### Finding 4: `progress-token.ts` — Unauthenticated Trigger.dev Access Token Generation

- **reachable:** CONFIRMED
- **entry point:** `GET /api/progress-token?documentVersionId=<cuid>` — `pages/api/progress-token.ts:5:handle`
- **input type:** query (`documentVersionId`)
- **auth required:** NONE
- **auth bypassable:** N/A — no auth exists
- **trace:** HTTP → `middleware.ts:49` (excludes `/api/`) → `progress-token.ts:5-27:handle` (zero auth, zero session check) → validates `documentVersionId` is present (line 15-17) → `generateTriggerPublicAccessToken(\`version:${documentVersionId}\`)` at line 20 → `lib/utils/generate-trigger-auth-token.ts:3-10` → `auth.createPublicToken({ scopes: { read: { tags: [version:X] } }, expirationTime: "15m" })` — returns 15-minute Trigger.dev read token
- **contrast:** The same `generateTriggerPublicAccessToken` function is also called from **three authenticated EE routes** (`monitor-token/route.ts:GET`, `retry-archive/route.ts:POST`, `freeze/route.ts:POST`) — all inside `app/(ee)/api/teams/[teamId]/datarooms/[id]/freeze/`, all behind NextAuth session checks. This endpoint is the **only unguarded path** to the same function.
- **deployment exposure:** INTERNET
- **priority:** P1 — HIGH — any unauthenticated client can obtain a Trigger.dev access token by guessing or discovering a `documentVersionId` (CUID). While CUIDs are not trivially guessable, they can leak through client-side JS, referrer headers, logs, or browser history.

---

### Finding 5: `record_reaction.ts` — Unauthenticated Database Write + Viewer Data Leak

- **reachable:** CONFIRMED
- **entry point:** `POST /api/record_reaction` — `pages/api/record_reaction.ts:5:handle`
- **input type:** body (`viewId`, `pageNumber`, `type`)
- **auth required:** NONE
- **auth bypassable:** N/A — no auth exists
- **trace:** HTTP → `middleware.ts:49` (excludes `/api/`) → `record_reaction.ts:6-57:handle` (zero auth, zero session check) → no input validation beyond checking `viewId` presence → `prisma.reaction.create({ data: { viewId, pageNumber, type }, include: { view: { select: { documentId, dataroomId, linkId, viewerEmail, viewerId, teamId } } } })` at lines 24-42 → response includes the created reaction (viewer metadata leaked through query result)
- **deployment exposure:** INTERNET
- **priority:** P2 — MEDIUM — unauthenticated DB write (data injection) plus info disclosure of viewer metadata (email, documentId, dataroomId, linkId, teamId). Unlike `record_view.ts`/`record_click.ts` (which publish to Tinybird analytics pipeline), this writes directly to PostgreSQL.

---

### Finding 6: `feedback/index.ts` — Unauthenticated Feedback Response Recording

- **reachable:** CONFIRMED
- **entry point:** `POST /api/feedback` — `pages/api/feedback/index.ts:5:handle`
- **input type:** body (`answer`, `feedbackId`, `viewId`)
- **auth required:** NONE
- **auth bypassable:** N/A — no auth exists
- **trace:** HTTP → `middleware.ts:49` (excludes `/api/`) → `feedback/index.ts:6-69:handle` (zero auth, zero session check) → validates `feedbackId` exists (line 18-31) and `viewId`+`linkId` pair exists (line 33-43) → `prisma.feedbackResponse.create({ feedbackId, viewId, data: { question, type, answer } })` at lines 46-55 — `answer` field stored unsanitized in JSON data
- **deployment exposure:** INTERNET
- **priority:** P2 — MEDIUM — unauthenticated DB write with unsanitized `answer` content. Validation only confirms IDs exist, but does not verify caller owns those IDs.

---

### Finding 7: `revalidate.ts` — Shared Secret in URL Query Parameter

- **reachable:** CONFIRMED (gated by secret)
- **entry point:** `GET /api/revalidate?secret=<REVALIDATE_TOKEN>&linkId=<id>` — `pages/api/revalidate.ts:5:handler`
- **input type:** query (`secret`, `linkId`, `documentId`, `teamId`)
- **auth required:** Shared secret (`REVALIDATE_TOKEN` env var) passed as query parameter
- **auth bypassable:** YES — secret in query param is leaked via server access logs (Vercel, CDN, reverse proxy), `Referer` header, browser history. If the secret is exposed, an attacker can:
  - Revalidate any single link view page
  - Revalidate ALL links for a document (up to all links)
  - Revalidate ALL links for a team (mass revalidation)
  - Enumerate which links exist for a team/document via timing/success responses
- **trace:** HTTP → `middleware.ts:49` (excludes `/api/`) → `revalidate.ts:6-110:handler` → line 14 compares `req.query.secret` with `process.env.REVALIDATE_TOKEN` (string comparison, no timing-safe comparison) → if match: for `linkId` → `res.revalidate(\`/view/${linkId}\`)` (line 44); for `documentId` → fetch all links for doc → revalidate each (lines 49-70); for `teamId` → fetch all non-archived links for team → revalidate each (lines 72-101)
- **deployment exposure:** INTERNET
- **priority:** P2 — MEDIUM — requires secret knowledge, but the query-param pattern makes secret leakage highly likely in production environments

---

### Finding 8: CVE-2026-23864 / CVE-2026-23869 — Next.js v14 App Router Denial of Service

- **reachable:** CONFIRMED
- **entry point:** Any Next.js App Router page (`app/` directory) that accepts HTTP POST with `content-type: text/x-component` or `next-action` or RSC Flight protocol headers
- **input type:** body (crafted RSC Flight deserialization payload)
- **auth required:** NONE — the RSC Flight protocol handler is a Next.js infrastructure-level handler that processes POST requests before any application auth
- **auth bypassable:** N/A — RSC endpoint is framework-level, not application-level
- **trace:** HTTP POST → any App Router page → Next.js RSC Flight deserialization handler (framework internals, not application code) → CVE-2026-23864: memory exhaustion during deserialization; CVE-2026-23869: CPU exhaustion via cyclic deserialization
- **mitigation factors:**
  - Papermark uses Next.js 14.2.35 (EOL since October 2025, no backport available)
  - No explicit `"use server"` directives found in App Router files (no explicit Server Actions), but the RSC Flight POST endpoint is always active on App Router pages regardless
  - `middleware.ts:49` excludes `/api/*` but does NOT exclude App Router pages from RSC handling
  - Publicly accessible `app/` pages include viewer, auth, and marketing routes that accept POST
- **patch status:** NO FIX AVAILABLE for Next.js v14 — fixes only in v15.5.15+ / v16.2.3+
- **deployment exposure:** INTERNET — any App Router page is reachable by unauthenticated clients
- **priority:** P1 — HIGH — unauthenticated DoS on framework-level handler with no available patch. A single crafted request can exhaust server memory or peg CPU for ~1 minute.

---

### Finding 9: `dangerouslySetInnerHTML` on `helpText` Props — Secondary XSS Sinks

- **reachable:** LIKELY
- **entry point:** Component render path — any parent component that passes user-controlled data as `helpText` prop
- **sink:** `components/ui/form.tsx:145` — `<p dangerouslySetInnerHTML={{ __html: helpText || "" }} />`
- **secondary sink:** `components/account/upload-avatar.tsx:96` — same pattern
- **auth required:** N/A (client-side XSS if triggered)
- **trace:** Parent component passes `helpText` prop → `Form` component renders `dangerouslySetInnerHTML` with prop value unsanitized → XSS fires if `helpText` contains attacker-controlled HTML
- **current risk:** These components are used extensively throughout the codebase with both static and potentially dynamic content. If any feature routes user input through `helpText`, stored XSS becomes possible. This broadens the XSS blast radius beyond `document-header.tsx`.
- **deployment exposure:** INTERNET (authenticated team member context)
- **priority:** P3 — MEDIUM — secondary sinks; currently no confirmed path for user-controlled input to reach `helpText`, but the pattern is dangerous

---

### Finding 10: `dangerouslySetInnerHTML` on Domain Configuration — Static Content

- **reachable:** UNLIKELY
- **entry point:** `components/domains/domain-configuration.tsx:143` — `MarkdownText` component
- **sink:** `dangerouslySetInnerHTML` on rendered markdown
- **current risk:** Renders static instruction content only. No dynamic user input feeds this sink.
- **deployment exposure:** INTERNET
- **priority:** P4 — LOW — informational, no dynamic data path

---

### Finding 11: Zod Validation Error Information Disclosure

- **reachable:** CONFIRMED
- **entry points:**
  - `pages/api/webhooks/services/[...path]/index.ts:232-238` — returns `validationResult.error.format()` with full schema structure
  - `pages/api/teams/[teamId]/documents/agreement.ts:44-48` — returns `validationResult.error.errors` with field-level details
- **input type:** body (malformed/invalid)
- **auth required:** Varies by endpoint (webhooks endpoint may have auth, agreements requires team auth)
- **trace:** HTTP request → API handler → Zod `.safeParseAsync(req.body)` → on validation failure → `validationResult.error.format()` or `.errors` returned to client in HTTP 400 response → leaks schema field names, types, constraints
- **deployment exposure:** INTERNET
- **priority:** P3 — MEDIUM — schema structure disclosure facilitates targeted fuzzing and API recon

---

### Finding 12: No Rate Limiting on Authentication Endpoints

- **reachable:** CONFIRMED
- **entry point:** `pages/api/auth/[...nextauth].ts` — NextAuth.js endpoints for sign-in, email verification, password reset
- **auth required:** N/A
- **impact:** Allows brute-force attacks against user credentials, email enumeration, and mass email-sending via the Resend SMTP relay. The incoming webhooks API does implement token-based rate limiting (via Upstash Ratelimit), but auth endpoints were not included.
- **deployment exposure:** INTERNET
- **priority:** P4 — LOW — missing defense-in-depth

---

### Finding 13: Webhook Event Request/Response Body Storage

- **reachable:** CONFIRMED
- **entry point:** `app/api/webhooks/callback/route.ts:43-44` — QStash webhook callback handler
- **input type:** body (base64-encoded QStash payload)
- **auth required:** QStash signature verification
- **trace:** HTTP POST → `app/api/webhooks/callback/route.ts` → `Buffer.from(sourceBody, "base64").toString("utf-8")` decodes request body → `Buffer.from(body, "base64").toString("utf-8")` decodes response body → stored in Tinybird via `recordWebhookEvent()` → rendered in webhook events admin UI
- **risk:** Webhook bodies may contain sensitive data from third-party responses (error pages, tokens). The Shiki syntax-highlighted rendering in the UI properly escapes HTML, but the stored data in Tinybird is not sanitized before storage.
- **deployment exposure:** INTERNET
- **priority:** P4 — LOW — information disclosure through webhook admin UI

---

### Finding 14: `cuid` 3.0.0 — Predictable ID Generation in Permission Groups

- **reachable:** CONFIRMED (authenticated)
- **entry points:**
  - `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5` — generates permission IDs
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5` — generates permission group IDs
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4` — creates permission groups
- **input type:** body (permission data, authenticated)
- **auth required:** JWT (NextAuth session + team membership check)
- **trace:** HTTP → authenticated team member request → handler imports `cuid` → `cuid()` generates sequential-predictable ID → stored in database as resource identifier
- **risk:** CUIDs encode timestamps and use a sequential counter — an attacker who observes a few IDs can predict future values. This enables ID enumeration and potential IDOR against permission group resources. The project already depends on `nanoid` 5.1.11 (cryptographically secure), so migration is trivial.
- **deployment exposure:** INTERNET (authenticated team members)
- **priority:** P2 — MEDIUM (elevated from dependency recon's HIGH due to auth gate) — requires team membership to exploit, but enables ID enumeration within an authenticated session

---

## Auth Coverage Summary

| Route Area | Auth Pattern | Reachability Impact |
|-----------|-------------|-------------------|
| `app/(ee)/api/**` | NextAuth + dataroom sessions + API keys | ✅ Protected |
| `pages/api/teams/[teamId]/**` | NextAuth `getServerSession` | ✅ Protected |
| `pages/api/account/**` | NextAuth `getServerSession` | ✅ Protected |
| `pages/api/file/tus-viewer/**` | Dataroom sessions (CORS bypassable) | ⚠️ CORS weakens |
| `pages/api/conversations/**` | **NONE — CRITICAL GAP** | ❌ Fully exposed |
| `pages/api/progress-token.ts` | **NONE — HIGH GAP** | ❌ Fully exposed |
| `pages/api/record_reaction.ts` | **NONE — MEDIUM GAP** | ❌ Fully exposed |
| `pages/api/feedback/index.ts` | **NONE — MEDIUM GAP** | ❌ Fully exposed |
| `pages/api/revalidate.ts` | Shared secret in query param | ⚠️ Leakable secret |

## Key Findings Summary

### Critical findings (internet-facing, no auth, data-write capable):
1. **Conversations API** — 4 unauthenticated handlers performing CRUD on PostgreSQL + triggering email notifications

### High findings (internet-facing, limited/no auth):
2. **Stored XSS via sanitizePlainText** — affects all team members viewing documents; multiple input paths
4. **progress-token.ts** — unauthenticated access token generation
8. **Next.js v14 App Router DoS** — unpatchable, unauthenticated memory/CPU exhaustion

### Medium findings (internet-facing, exploitable):
3. **CORS reflection on TUS** — CSRF-style upload attacks
5. **record_reaction.ts** — unauthenticated DB writes + viewer metadata leak
6. **feedback/index.ts** — unauthenticated feedback injection
7. **revalidate.ts** — secret leakage enables mass cache invalidation
9. **helpText XSS sinks** — secondary XSS blast radius expansion
11. **Zod info disclosure** — schema structure leaks
14. **cuid predictable IDs** — permission group ID enumeration

### Low findings (minor risk):
10. **Domain config dangerouslySetInnerHTML** — static content only
12. **No auth rate limiting** — missing brute-force protection
13. **Webhook body storage** — information disclosure in admin UI

---

## Data Sources

- GitNexus impact analysis (direction=upstream) on: `handleRoute` (conversations), `sanitizePlainText`, `setCorsHeaders`, `handle` (progress-token), `handle` (record_reaction), `handle` (feedback), `handler` (revalidate), `generateTriggerPublicAccessToken`
- Direct source reads of all handler files, `middleware.ts`, `sanitize-html.ts`, `generate-trigger-auth-token.ts`, `document-header.tsx`, `form.tsx`, `upload-avatar.tsx`
- Middleware matcher pattern analysis from `middleware.ts:49` confirming `/api/` exclusion
- Auth pattern cross-reference from structural analysis, SAST recon, framework recon, API spec recon, and dependency recon
