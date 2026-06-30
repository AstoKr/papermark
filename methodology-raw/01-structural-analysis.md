# Structural Analysis — Deep Call-Chain & Data-Flow Report

**Target:** `papermark`  
**Node:** Layer-1 Structural Analysis  
**Date:** 2026-06-30  
**Method:** GitNexus 6-step pattern (query → context → cypher → process → impact → read) + live source audit

---

## Summary

| Severity | Count | Key Issues |
|----------|-------|------------|
| CRITICAL | 1 | Conversations API — zero auth on any handler; unauthenticated DB writes |
| HIGH | 4 | Stored XSS (sanitize-html entity-decode + `<xmp>` CVE-2026-44990); progress-token unauthenticated token generation; Next.js v14 unpatchable DoS (CVE-2026-23864/23869); `dangerouslySetInnerHTML` on untrusted document names |
| MEDIUM | 6 | CORS origin reflection on TUS viewer; Zod info disclosure; record_reaction unauthenticated DB write + data leak; feedback unauthenticated DB write; revalidate secret-in-query-param; helpText `dangerouslySetInnerHTML` in form/avatar |
| LOW | 2 | Webhook event body storage; no auth rate limiting on NextAuth |

---

## Finding 1: Conversations API — Zero Authentication on All Handlers

- **severity:** CRITICAL
- **source handler:** `pages/api/conversations/[[...conversations]].ts:4-9`
- **router:** `ee/features/conversations/api/conversations-route.ts:276-299` (`handleRoute`)
- **sink:** Direct Prisma DB operations — conversation creation, message creation, notification toggling, dataroom data reads
- **call chain:**
  1. `POST/GET /api/conversations` → `pages/api/conversations/[[...conversations]].ts:handler` (thin passthrough, no auth)
  2. → `ee/features/conversations/api/conversations-route.ts:handleRoute` (dispatches to routeHandlers map, no auth)
  3. → Four handlers: `GET /` (list conversations, line 19), `POST /` (create conversation, line 77), `POST /messages` (add message, line 202), `POST /notifications` (toggle notifications, line 258)
  4. → `prisma.conversation.findMany()` / `conversationService.createConversation()` / `messageService.addMessage()` / `notificationService.toggleNotificationsForConversation()`
- **auth:** **NONE** — zero authentication on all four handlers
- **description:** The conversations route has **four handlers (GET list, POST create, POST messages, POST notifications) — none perform any authentication**. There is no `getServerSession()` call, no token check, no API key validation. The `viewerId` parameter is accepted on trust from query parameters or request body with zero server-side verification that the caller owns that viewer identity. An attacker can enumerate conversations by providing any `dataroomId`+`viewerId` pair, create fake conversations and messages, inject message content, and toggle notification preferences for any conversation.
- **evidence:**
  - File `ee/features/conversations/api/conversations-route.ts` — **zero imports from `next-auth`**, **zero `getServerSession` calls**:
    ```typescript
    // conversations-route.ts:1-14 — imports are: next, @trigger.dev/sdk, @vercel/functions,
    // prisma, conversationService, messageService, notificationService, conversation-message-notification
    // *** NO next-auth imports ***
    ```
  - All four handlers accept `viewerId` from request body/query without validation:
    ```typescript
    // Line 21-23: GET / — viewerId taken from query params
    const { dataroomId, viewerId } = req.query as { dataroomId?: string; viewerId?: string };
    
    // Line 78-85: POST / — viewerId taken from body
    const { dataroomId, viewId, viewerId, documentId, pageNumber, ...data } = req.body;
    
    // Line 203-208: POST /messages — viewerId taken from body
    const { content, viewId, viewerId, conversationId } = req.body;
    
    // Line 259-263: POST /notifications — viewerId taken from body
    const { enabled, viewerId, conversationId } = req.body;
    ```
  - **Contrast** — the adjacent file `ee/features/conversations/api/team-conversations-route.ts` demonstrates the correct pattern:
    ```typescript
    // team-conversations-route.ts:1-4
    import { authOptions } from "@/lib/auth/auth-options";
    import { getServerSession } from "next-auth/next";
    
    // team-conversations-route.ts:40-43 — EVERY handler starts with:
    const session = await getServerSession(req, res, authOptions);
    if (!session) {
      return res.status(401).json({ error: "Unauthorized" });
    }
    ```
- **impact:** An unauthenticated attacker can:
  - List all conversations for any `dataroomId`+`viewerId` pair (enumeration of conversation content, messages, and participant data)
  - Create conversations under any viewer identity (data injection into the PostgreSQL database)
  - Add messages to any conversation (content injection)
  - Toggle notification preferences for any conversation participant
  - Trigger `sendConversationTeamMemberNotificationTask` (Trigger.dev background job) to send email notifications to team members, potentially enabling harassment/phishing via the trusted notification channel

---

## Finding 2: Stored XSS via `sanitizePlainText` Entity-Decode Ordering + `<xmp>` Bypass (CVE-2026-44990)

- **severity:** HIGH (CRITICAL with CVE-2026-44990 `<xmp>` bypass combining)
- **source handler:** `pages/api/teams/[teamId]/documents/[id]/update-name.ts:15` (Zod transform) and `lib/zod/url-validation.ts:207` (document upload)
- **sink:** `components/documents/document-header.tsx:604` — `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}`
- **call chain:**
  1. POST `/api/teams/:teamId/documents/:id/update-name` → `update-name.ts:24` (handler, has NextAuth auth)
  2. → Zod schema `updateNameSchema` line 12-22 → `.transform((value) => sanitizePlainText(value))` line 15
  3. → `lib/utils/sanitize-html.ts:sanitizePlainText` line 12-19
  4. → `sanitizeHtml(content, {allowedTags: []})` strips literal HTML tags (line 13)
  5. → `decodeHTML(sanitized)` decodes HTML entities AFTER tag stripping (line 14) — **THIS IS THE BUG**
  6. → Decoded string stored in PostgreSQL `Document.name` column
  7. → `components/documents/document-header.tsx:604` renders via `dangerouslySetInnerHTML`
- **auth:** YES — `getServerSession` at update-name.ts:30
- **description:** The `sanitizePlainText` function calls `sanitizeHtml` first (stripping literal HTML tags), then `decodeHTML` (decoding entities). This ordering is **wrong**: encoded entities like `&lt;img src=x onerror=alert(1)&gt;` survive tag stripping because they aren't literal tags yet, then become real HTML tags after `decodeHTML` runs. The result is stored XSS that fires whenever a team member views the document page. Additionally, `sanitize-html` version 2.17.3 is vulnerable to **CVE-2026-44990** (CVSS 9.3): the `<xmp>` raw-text element bypass allows arbitrary HTML/JS injection even without entity encoding.
- **evidence:**
  ```typescript
  // lib/utils/sanitize-html.ts:12-19 — BUG: decode AFTER strip
  export function sanitizePlainText(content: string) {
    const sanitized = sanitizeHtml(content, plainTextSanitizeConfig);  // Step 1: strip tags
    const decoded = decodeHTML(sanitized).normalize("NFC");            // Step 2: decode entities — TOO LATE
    return decoded.replace(controlCharsRegex, " ").replace(invisibleControlRegex, "").trim();
  }
  ```
  ```typescript
  // components/documents/document-header.tsx:604 — XSS sink
  dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
  ```
  ```typescript
  // pages/api/teams/[teamId]/documents/[id]/update-name.ts:12-22 — input via Zod
  const updateNameSchema = z.object({
    name: z.string().transform((value) => sanitizePlainText(value)).pipe(
      z.string().min(1).max(255),
    ),
  });
  ```
  ```typescript
  // lib/zod/url-validation.ts:207 — same bug in document upload schema
  .transform((value) => sanitizePlainText(value))
  ```
- **impact:** Any team member (or anyone who can create/rename documents) can execute arbitrary JavaScript in the browser of every other team member who views the document. This includes: session token theft, API key exfiltration, team data exfiltration, UI redressing for phishing. The XSS fires on every document view page load, making it a persistent compromise vector.

---

## Finding 3: CORS Origin Reflection on TUS Viewer Upload Endpoint

- **severity:** MEDIUM
- **source handler:** `pages/api/file/tus-viewer/[[...file]].ts:252-263` (handler)
- **sink:** `Access-Control-Allow-Origin` response header at `pages/api/file/tus-viewer/[[...file]].ts:237`
- **call chain:**
  1. Browser makes cross-origin request to TUS viewer endpoint with crafted `Origin` header
  2. → `handler()` line 252 → `setCorsHeaders(req, res)` line 260
  3. → `res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*")` line 237 — **reflects origin verbatim**
  4. → Also sets `Access-Control-Allow-Credentials: "true"` line 236
  5. → Request forwarded to `tusServer.handle(req, res)` line 263
- **auth:** Session check present via `onIncomingRequest` → `verifyDataroomSessionInPagesRouter()` but CORS allows arbitrary origins with credentials
- **description:** The TUS viewer upload endpoint reflects the request `Origin` header verbatim into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`. This allows any website to make authenticated cross-origin requests (including uploads) on behalf of a logged-in user. Although the endpoint does validate the dataroom session for new uploads via `verifyDataroomSessionInPagesRouter`, the CORS configuration enables CSRF-style attacks. An attacker's page can issue TUS uploads to a victim's dataroom if the victim has an active session.
- **evidence:**
  ```typescript
  // pages/api/file/tus-viewer/[[...file]].ts:233-250
  const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse) => {
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*"); // <-- arbitrary origin
    res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
    // ...
  };
  ```
- **impact:** An attacker-controlled website can make authenticated cross-origin uploads to a victim's dataroom, if the victim has an active session cookie. The `Access-Control-Allow-Credentials: true` means browser cookies are sent cross-origin. Uploaded files could contain malicious content or be used to fill storage quotas.

---

## Finding 4: `progress-token.ts` — Unauthenticated Trigger.dev Access Token Generation

- **severity:** HIGH
- **source handler:** `pages/api/progress-token.ts:4-27` (`handle`)
- **sink:** `lib/utils/generate-trigger-auth-token.ts:3-10` → `auth.createPublicToken()`
- **call chain:**
  1. GET `/api/progress-token?documentVersionId=xxx` → `handle()` line 5
  2. → No session check, no auth of any kind
  3. → `generateTriggerPublicAccessToken(\`version:${documentVersionId}\`)` line 22
  4. → `auth.createPublicToken({ scopes: { read: { tags: [`version:${documentVersionId}`] } }, expirationTime: "15m" })`
  5. → Returns Trigger.dev public access token scoped to that document version ID
- **auth:** **NONE** — no authentication, no authorization, no rate limiting
- **description:** This endpoint generates a Trigger.dev public access token for any `documentVersionId` passed as a query parameter, without any authentication or authorization check. The token grants read access to Trigger.dev runs tagged with `version:{documentVersionId}`. Anyone who can discover or guess a valid `documentVersionId` can obtain a public access token. While CUIDs are not trivially guessable, they can be leaked through client-side code, referrer headers, logs, or other means. The same `generateTriggerPublicAccessToken` function is also called from authenticated ee routes (freeze monitor-token, retry-archive, freeze), confirming that the **only** unguarded path is this endpoint.
- **evidence:**
  ```typescript
  // pages/api/progress-token.ts:4-27 — full file, no auth
  export default async function handle(req, res) {
    if (req.method !== "GET") { return res.status(405)... }
    const { documentVersionId } = req.query;
    if (!documentVersionId || typeof documentVersionId !== "string") { ... }
    const publicAccessToken = await generateTriggerPublicAccessToken(`version:${documentVersionId}`);
    return res.status(200).json({ publicAccessToken });
  }
  ```
  ```typescript
  // lib/utils/generate-trigger-auth-token.ts:2-11 — the token creation
  export async function generateTriggerPublicAccessToken(tag: string) {
    return auth.createPublicToken({
      scopes: { read: { tags: [tag] } },
      expirationTime: "15m",
    });
  }
  ```
- **impact:** An attacker who discovers a `documentVersionId` can obtain a 15-minute Trigger.dev public access token that grants read access to Trigger.dev runs for that document version. This could expose internal processing data, document processing status, and potentially workflow metadata. The contrast with the authenticated ee routes (`monitor-token`, `retry-archive`, `freeze`) that use the same function with proper auth makes this an obvious auth gap.

---

## Finding 5: `record_reaction.ts` — Unauthenticated Database Write + Viewer Data Leak

- **severity:** MEDIUM
- **source handler:** `pages/api/record_reaction.ts:5-57`
- **sink:** `prisma.reaction.create()` at line 24
- **call chain:**
  1. POST `/api/record_reaction` → `handle()` line 6
  2. → `viewId`, `pageNumber`, `type` extracted from body line 17-21 (no validation of caller authorization)
  3. → `prisma.reaction.create({ data: { viewId, pageNumber, type }, include: { view: { select: { documentId, dataroomId, linkId, viewerEmail, viewerId, teamId } } } })` line 24-42
  4. → Returns viewer metadata in response
- **auth:** **NONE**
- **description:** This endpoint creates a `Reaction` record in the database with no authentication or session validation. It accepts `viewId`, `pageNumber`, and `type` from the request body and does not verify the caller is authorized. The endpoint also includes the full `view` relation in the creation response (line 30-41), leaking `documentId`, `dataroomId`, `linkId`, `viewerEmail`, `viewerId`, and `teamId` back to the caller. Unlike `record_view.ts` and `record_click.ts` (which publish to Tinybird analytics), this endpoint writes directly to the PostgreSQL `Reaction` table, making it a data-injection vector.
- **evidence:**
  ```typescript
  // pages/api/record_reaction.ts:17-42 — no auth, creates DB record with viewer data leak
  const { viewId, pageNumber, type } = req.body;
  const reaction = await prisma.reaction.create({
    data: { viewId, pageNumber, type },
    include: {
      view: {
        select: {
          documentId: true,
          dataroomId: true,
          linkId: true,
          viewerEmail: true,
          viewerId: true,
          teamId: true,
        },
      },
    },
  });
  ```
- **impact:** Unauthenticated attackers can:
  - Inject arbitrary reaction data tied to any known `viewId`
  - Enumerate valid `viewId` values by observing error responses
  - Leak protected viewer metadata (`viewerEmail`, `viewerId`, `documentId`, `dataroomId`, `linkId`, `teamId`) through the response's `include` clause
  - Pollute analytics/reaction data that team members see in dashboards

---

## Finding 6: `feedback/index.ts` — Unauthenticated Feedback Response Recording

- **severity:** MEDIUM
- **source handler:** `pages/api/feedback/index.ts:5-69`
- **sink:** `prisma.feedbackResponse.create()` at line 46
- **call chain:**
  1. POST `/api/feedback` → `handle()` line 6
  2. → `answer`, `feedbackId`, `viewId` extracted from body line 11-15 (no auth)
  3. → Validates `feedbackId` exists (line 18-31) and `viewId`+`linkId` match (line 33-43)
  4. → `prisma.feedbackResponse.create({ data: { feedbackId, viewId, data: { ... } } })` line 46-55
- **auth:** **NONE**
- **description:** This endpoint records feedback responses with no authentication. It validates that the `feedbackId` and `viewId`+`linkId` pair exist, but any unauthenticated client with knowledge of valid IDs can submit arbitrary feedback responses. The `answer` field is stored as part of JSON data in the database without sanitization.
- **evidence:**
  ```typescript
  // pages/api/feedback/index.ts:9-56 — no auth
  const { answer, feedbackId, viewId } = req.body;
  // ... validation that feedbackId and viewId exist ...
  await prisma.feedbackResponse.create({
    data: {
      feedbackId,
      viewId,
      data: { ...(feedback.data as { question: string; type: string }), answer },
    },
  });
  ```
- **impact:** Unauthenticated attackers can inject arbitrary feedback response data, polluting analytics and potentially injecting unsanitized content into the JSON `data` field. The `answer` field is stored without sanitization.

---

## Finding 7: `revalidate.ts` — Shared Secret in URL Query Parameter

- **severity:** MEDIUM
- **source handler:** `pages/api/revalidate.ts:5-110`
- **auth:** Shared secret passed via `?secret=xxx` query parameter
- **call chain:**
  1. GET `/api/revalidate?secret=xxx&linkId=yyy` → `handler()` line 6
  2. → `req.query.secret !== process.env.REVALIDATE_TOKEN` comparison line 14
  3. → If valid, revalidates Next.js ISR pages for linkId, documentId, or teamId
  4. → Can revalidate individual view pages or ALL links for a document/team
- **description:** The revalidation endpoint authenticates via `req.query.secret !== process.env.REVALIDATE_TOKEN`. Passing the shared secret as a URL query parameter exposes it to leakage through: (1) server access logs (Vercel, CDN, reverse proxy), (2) `Referer` header when the page loads external resources, (3) browser history if triggered from client-side navigation. If the `REVALIDATE_TOKEN` is ever exposed, an attacker can force mass revalidation of all team links (potentially causing DoS via cache stampede) or enumerate which links exist for a given team/document.
- **evidence:**
  ```typescript
  // pages/api/revalidate.ts:13-14 — secret in query param
  if (req.query.secret !== process.env.REVALIDATE_TOKEN) {
    return res.status(401).json({ message: "Invalid token" });
  }
  ```
  ```typescript
  // pages/api/revalidate.ts:48-100 — can revalidate ALL links for a team
  if (teamId) {
    const allLinks = await prisma.link.findMany({ where: { teamId, isArchived: false, deletedAt: null } });
    for (const link of allLinks) { await res.revalidate(...); }
  }
  ```
- **impact:** Secret leakage enables mass ISR cache invalidation and link enumeration. The endpoint accepts `teamId`, meaning a leaked secret allows an attacker to enumerate all links for any team and force revalidation of all their view pages.

---

## Finding 8: CVE-2026-23864 / CVE-2026-23869 — Next.js v14 App Router Denial of Service (No Patch)

- **severity:** HIGH
- **type:** Memory/CPU exhaustion via crafted RSC requests to App Router
- **CVE-2026-23864:** Memory exhaustion during RSC deserialization (CVSS 7.5)
- **CVE-2026-23869:** CPU exhaustion via cyclic deserialization (CVSS 7.5)
- **auth:** NONE required for either — unauthenticated HTTP requests
- **patch status:** **NO FIX AVAILABLE** — Next.js v14 reached EOL October 2025; fixes only in v15.5.15+ / v16.2.3+
- **project version:** Next.js 14.2.35 (EOL, no backport)
- **exposure analysis:**
  - The `middleware.ts:49` matcher **excludes `/api/*` from middleware** but does NOT exclude App Router pages from RSC Flight protocol handling
  - Any App Router page (in `app/` directory) exposes the RSC Flight POST endpoint
  - No `"use server"` directives found in App Router files — this means no explicit Server Actions exist, but the **RSC Flight protocol endpoint** (`POST` to any App Router page with `next-action` or RSC content-type headers) is a Next.js infrastructure-level handler that processes all POST requests to App Router pages
  - Unauthenticated pages in `app/` that accept POST could trigger RSC deserialization:
    - `app/(auth)/` pages are behind `SessionProvider` but rendered server-side
    - `/view/` pages in `app/` are publicly accessible
    - Any `app/` route that uses dynamic rendering could be targeted
  - The `/api/` exclusion in middleware means API routes are not relevant here — the attack targets App Router page routes
  - **Mitigation factor:** Papermark uses Next.js 14.2.35 which includes the CVE-2025-55182 fix, but the RSC infrastructure-level handler is still vulnerable to the DoS CVEs
- **impact:** An unauthenticated attacker can send a crafted HTTP request to any App Router page, causing the server process to exhaust memory or spin CPU for ~1 minute per request. Repeated requests sustain a DoS against the application. No authentication required, no complex payload needed.

---

## Finding 9: `dangerouslySetInnerHTML` on `helpText` Props — Secondary XSS Sinks

- **severity:** MEDIUM
- **components:** `components/ui/form.tsx:145` and `components/account/upload-avatar.tsx:96`
- **sink:** `dangerouslySetInnerHTML={{ __html: helpText || "" }}`
- **call chain:**
  1. Parent component passes `helpText` prop (may contain user-controlled content)
  2. → `components/ui/form.tsx` renders `<p dangerouslySetInnerHTML={{ __html: helpText }} />` at line 145
  3. OR `components/account/upload-avatar.tsx` renders same at line 96
- **description:** The shared `Form` component and `UploadAvatar` component use `dangerouslySetInnerHTML` to render the `helpText` prop. While these are currently used with mostly static content, if any caller passes user-controlled data through `helpText`, it will render unsanitized HTML. This is a secondary XSS sink that increases the attack surface beyond the primary `document-header.tsx` instance.
- **evidence:**
  ```typescript
  // components/ui/form.tsx:142-146
  {typeof helpText === "string" ? (
    <p className="text-sm text-muted-foreground transition-colors"
       dangerouslySetInnerHTML={{ __html: helpText || "" }} />
  ) : helpText}
  ```
  ```typescript
  // components/account/upload-avatar.tsx:93-97
  {typeof helpText === "string" ? (
    <p className="text-sm text-muted-foreground transition-colors"
       dangerouslySetInnerHTML={{ __html: helpText || "" }} />
  ) : helpText}
  ```
- **impact:** If any route or component feeds user-controlled data into the `helpText` prop of `Form` or `UploadAvatar`, stored XSS is possible. This broadens the XSS blast radius beyond `document-header.tsx`.

---

## Finding 10: `dangerouslySetInnerHTML` on Domain Configuration — Static Content (Low Risk)

- **severity:** LOW (informational)
- **component:** `components/domains/domain-configuration.tsx:143` (`MarkdownText`)
- **sink:** `dangerouslySetInnerHTML` on rendered markdown
- **risk:** Currently renders static instruction content. Low risk unless dynamic markdown is introduced.

---

## Auth Coverage Heatmap (Cross-Reference from API Spec Recon)

| Area | Auth Pattern | Coverage |
|------|-------------|----------|
| `app/api/` (views, auth, cron, webhooks) | Link/dataroom sessions + QStash sigs + NextAuth | ✅ Complete |
| `app/(ee)/api/` | NextAuth + dataroom sessions + API keys | ✅ Complete |
| `pages/api/teams/[teamId]/**` | NextAuth `getServerSession` | ✅ Full |
| `pages/api/account/**` | NextAuth `getServerSession` | ✅ Full |
| `pages/api/file/tus-viewer/**` | Dataroom sessions | ✅ Present (CORS bypassable) |
| `pages/api/conversations/**` | **NONE** | ❌ **MISSING — CRITICAL** |
| `pages/api/progress-token.ts` | **NONE** | ❌ **MISSING — HIGH** |
| `pages/api/record_reaction.ts` | **NONE** | ❌ **MISSING — MEDIUM** |
| `pages/api/feedback/index.ts` | **NONE** | ❌ **MISSING — MEDIUM** |
| `pages/api/revalidate.ts` | Shared secret in query param | ⚠️ **BYPASSABLE — MEDIUM** |
| `pages/api/record_view.ts` | None (by design, public tracking) | ⚠️ Analytics pollution |
| `pages/api/record_click.ts` | None (by design, public tracking) | ⚠️ Analytics pollution |

---

## Blast Radius Analysis

### Conversations API (handleRoute)
- **direct callers:** 1 (pages router handler `pages/api/conversations/[[...conversations]].ts`)
- **modules affected:** Api (direct)
- **depth-2 callees:** prisma CRUD operations, conversationService, messageService, notificationService, Trigger.dev task triggering
- **risk assessment:** LOW impact count (only 1 direct caller) but **CRITICAL** severity because there is zero auth — the handler is a public API endpoint

### sanitizePlainText
- **direct callers:** 4 (update-name.ts Zod transform, url-validation.ts Zod transform, validateContent, POST upload route)
- **depth-2 callers:** 10+ files including agreement creation, team name changes, FAQ management
- **depth-3 callers:** 4 files (branding, domains)
- **risk assessment:** Widespread use across the codebase means the sanitizer bug affects many input paths. Every file that imports `sanitize-html.ts` is a potential XSS entry point.

### progress-token (handle)
- **direct callers:** 0 external (public API endpoint with no caller restrictions)
- **risk assessment:** Any unauthenticated client can reach this endpoint. No rate limiting.

---

## Key Patterns Discovered

1. **Auth gap pattern:** The `conversations-route.ts` file shares the same structure as `team-conversations-route.ts` but was missed during the auth implementation pass. The team route has `getServerSession` on every handler; the public route has none.

2. **Sanitizer ordering bug:** The `sanitizeHtml → decodeHTML` ordering in `sanitizePlainText` is fundamentally broken. Entities decoded after tag stripping become real HTML. This is compounded by CVE-2026-44990 (sanitize-html `<xmp>` bypass on 2.17.3).

3. **Public tracking endpoints becoming data-injection vectors:** `record_reaction.ts` and `feedback/index.ts` were likely intended as public tracking endpoints like `record_view.ts` and `record_click.ts`, but unlike those (which publish to Tinybird analytics pipeline), these write directly to PostgreSQL tables — making them data-injection vectors.

4. **TUS CORS misconfiguration:** The endpoint reflects arbitrary origins with credentials allowed, which is a standard CSRF-style attack pattern for TUS upload endpoints.

5. **No Server Actions found:** No `"use server"` directives were found in App Router files, which partially mitigates the practical exploitability of CVE-2026-23864/23869, but the RSC Flight protocol handler remains exposed at the framework level.

---

## Data Sources

- GitNexus query + context on `handleRoute` (conversations route), `sanitizePlainText`, `progress-token`, `setCorsHeaders`, `generateTriggerPublicAccessToken`
- Direct source reads: `conversations-route.ts`, `team-conversations-route.ts`, `tus-viewer/[[...file]].ts`, `update-name.ts`, `record_reaction.ts`, `feedback/index.ts`, `revalidate.ts`, `sanitize-html.ts`, `middleware.ts`, `generate-trigger-auth-token.ts`, `record_view.ts`, `record_click.ts`, `url-validation.ts`, `form.tsx`, `upload-avatar.tsx`, `document-header.tsx`
- Impact analysis: `sanitizePlainText` (23 impacted symbols, depth 1-3), `handleRoute` (1 direct caller)
- Cypher query: `dangerouslySetInnerHTML` usage sites (4 components confirmed)
- Recon cross-reference: SAST findings, framework analysis, API spec recon, CVE intel
