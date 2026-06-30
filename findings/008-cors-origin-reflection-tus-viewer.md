# Finding: CORS Origin Reflection on TUS Viewer Upload Endpoint

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:L/A:N = 4.7 (per CVE-2026-57957)

## Affected Code
- **File:** `pages/api/file/tus-viewer/[[...file]].ts:236-237`
- **Endpoint:** `/api/file/tus-viewer/**`
- **CVE:** CVE-2026-57957 (papermark-specific, filed)

## Description
The TUS viewer upload endpoint reflects the request `Origin` header verbatim into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`. This is the textbook "reflected origin with credentials" anti-pattern — it allows any website to make credentialed cross-origin requests to this endpoint on behalf of a logged-in user.

The TUS viewer endpoint serves file upload operations (chunked upload, metadata, copy). An attacker who tricks a logged-in viewer into visiting their website can perform upload operations to the victim's dataroom, or read upload metadata. The regular TUS upload endpoint (`pages/api/file/tus/[[...file]].ts`) does NOT set any CORS headers (safe default), making the asymmetric treatment itself a finding.

This finding is independently confirmed as CVE-2026-57957.

## Proof of Concept
1. Host a page at `https://attacker.com/exploit.html`:
   ```javascript
   fetch('https://papermark.app/api/file/tus-viewer/upload-id', {
     credentials: 'include',
     method: 'HEAD'
   }).then(r => {
     console.log(r.headers.get('Upload-Metadata'));
     // Cross-origin read of TUS metadata
   });
   ```
2. When a logged-in papermark viewer visits the page, the browser sends the request with cookies due to `Access-Control-Allow-Credentials: true`.
3. The response includes `Access-Control-Allow-Origin: https://attacker.com`, permitting the cross-origin read.

## Impact
An attacker who lures a logged-in papermark viewer to their website can:
- Read TUS upload metadata containing document version identifiers
- Perform TUS upload operations to the victim's dataroom (CSRF)
- Enumerate upload state and file information
- The `Access-Control-Allow-Credentials: true` flag means browser cookies (including session tokens) are sent cross-origin

## Evidence
```typescript
// pages/api/file/tus-viewer/[[...file]].ts:233-250
const setCorsHeaders = (req, res) => {
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*"); // <-- reflects arbitrary origin
  res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
  // ...
};
```

## Methodology
- **First found by:** Config analysis (00-ai-configs.md) identified as CRITICAL (later downgraded)
- **Confirmed by:** Structural analysis (01-structural-analysis.md) added reachability and downgraded to MEDIUM
- **Confirmed by:** SAST synthesis (02-synthesized-sast.md S-005) merged both
- **Independently confirmed as:** CVE-2026-57957 (papermark-specific)

## Chain Potential
**Chainable.**
- **Chain 2 (CORS + PDF):** CSRF upload of malicious PDF via CORS misconfig + PDF.js XSS for admin session takeover.
- **Chain 7 (CORS → Token Leak):** Cross-origin read of TUS upload metadata containing document version IDs, feeding into unauthenticated progress-token endpoint.

## Remediation
Replace the dynamic origin reflection with an explicit allowlist. Check `req.headers.origin` against a set of known origins before reflecting it. Do not combine `Access-Control-Allow-Credentials: true` with dynamic origin reflection. The two TUS endpoints should have symmetric CORS treatment.
