# Configuration Misconfiguration Analysis

## Summary

10 findings identified — 1 CRITICAL, 1 HIGH, 6 MEDIUM, 2 LOW. The most impactful issue is a reflected-origin CORS misconfiguration on the tus file-upload endpoint that allows any attacker origin to make credentialed cross-origin requests.

---

### CORS misconfiguration — reflected origin with credentials in tus-viewer

- file: `pages/api/file/tus-viewer/[[...file]].ts`
- line: 236–237
- type: cors-misconfig
- severity: CRITICAL
- description: The tus-viewer handler sets `Access-Control-Allow-Origin` to the value of the incoming `Origin` header (falling back to `*`) while simultaneously setting `Access-Control-Allow-Credentials: true`. This is the textbook "reflected origin with credentials" anti-pattern — it tells the browser to expose authenticated responses to any cross-origin page that asks for them.
- impact: Any attacker-controlled website can make credentialed `fetch()` requests to this endpoint and read the response. The tus viewer serves file-upload operations (chunked upload, metadata, copy). An attacker who tricks a logged-in viewer into visiting their site can perform arbitrary upload operations on the victim's behalf, or read upload metadata. The `viewerId` and `teamId` are passed in upload metadata, so an attacker can enumerate or tamper with uploads by controlling these values.
- evidence: Config at `pages/api/file/tus-viewer/[[...file]].ts:234-250`:
  ```typescript
  const setCorsHeaders = (req, res) => {
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
    // ...
  };
  ```
  Credentials (`Access-Control-Allow-Credentials: true`) + dynamic origin reflection = any origin can make authenticated requests. The tus server authenticates via session cookies (`next-auth.session-token`) that are sent on cross-origin requests when `credentials: "include"` is used. There is no origin allowlist check before reflecting the Origin header.
- remediation: Replace the dynamic origin with an explicit allowlist. Check `req.headers.origin` against a set of known origins before reflecting it. Do not combine credentials with dynamic origins.

---

### Embed route allows framing by any origin

- file: `next.config.mjs`
- line: 259–271
- type: missing-header
- severity: HIGH
- description: The `/view/:path*/embed` route sends `Content-Security-Policy: frame-ancestors *;` which allows any website to load the document viewer in an iframe. Unlike the main route CSP (which is report-only), this one is a real enforced header, so the `frame-ancestors *` directive is actively allowing cross-origin framing.
- impact: An attacker can iframe the embed page on their own site and overlay UI elements to perform clickjacking attacks against document viewers. While the API layer validates the embeddable-domains allowlist (via `isEmbeddableUrl` in `lib/edge-config/embeddable-domains.ts`), the HTML page itself loads in the iframe — allowing an attacker to present a transparent overlay over the document view and trick users into interacting with UI controls (e.g., granting permissions, clicking "download"). The UI-layer protection (CSP) should match the API-layer protection.
- evidence: Config at `next.config.mjs:259-271`:
  ```javascript
  {
    source: "/view/:path*/embed",
    headers: [{
      key: "Content-Security-Policy",
      value: "... frame-ancestors *; ..."
    }]
  }
  ```
  The embed-page HTML is served with `frame-ancestors *` allowing any domain to iframe it. The API backends do check `isEmbeddableUrl()` (`lib/edge-config/embeddable-domains.ts:49-67`), which validates the Referer against an Edge Config allowlist, but the CSP should also be tightened:
  ```typescript
  export const isEmbeddableUrl = async (url) => {
    if (!url) return false;
    let parsed = new URL(url);
    if (parsed.protocol !== "https:") return false;
    const allowlist = await getEmbeddableDomains();
    return allowlist.some((host) => matchesHost(parsed.hostname, host));
  };
  ```
- remediation: Replace `frame-ancestors *` with `frame-ancestors 'self'` or a specific allowlist of embed domains. Consider reading the embeddable-domains from Edge Config at build time to generate the CSP value dynamically.

---

### Main CSP is report-only and not enforced

- file: `next.config.mjs`
- line: 219
- type: missing-header
- severity: MEDIUM
- description: The main Content Security Policy is set via `Content-Security-Policy-Report-Only` rather than `Content-Security-Policy`. This means all CSP directives on the main application pages are advisory only — violations are reported but never blocked. Combined with the permissive policy that includes `'unsafe-inline'` and `'unsafe-eval'` in `script-src`, XSS attacks against main application pages face minimal CSP-based resistance.
- impact: An attacker who finds an XSS vulnerability faces no CSP blocking on main app pages. The report-only policy provides visibility into violations but no actual protection. The permissive directives (`'unsafe-inline'`, `'unsafe-eval'`, `https:` protocol wildcards) mean most XSS payloads will execute without CSP interference.
- evidence: Config at `next.config.mjs:219`:
  ```javascript
  key: "Content-Security-Policy-Report-Only",  // ← NOT enforced
  value: `default-src 'self' https: ... script-src 'self' 'unsafe-inline' 'unsafe-eval' https: ...`
  ```
- remediation: Migrate to an enforced `Content-Security-Policy` header. Remove `'unsafe-inline'` and `'unsafe-eval'` in favor of nonces or hashes. Tighten protocol wildcards (`https:`) to specific domains.

---

### CSP report endpoint is non-functional

- file: `app/api/csp-report/route.ts`
- line: 1–16
- type: debug-mode
- severity: MEDIUM
- description: The CSP report ingestion endpoint at `/api/csp-report` receives POST requests from browsers but discards all reports. The `console.log` line that would record violations is commented out, and there is no storage, forwarding, or alerting logic. The infrastructure for CSP violation monitoring is wired up in `next.config.mjs` (via the `Report-To` header and `report-to csp-endpoint;` directive) but the actual ingestion is a no-op.
- impact: CSP violations are silently discarded. Security teams receive no visibility into attempted CSP violations, which could indicate XSS probing, content injection attacks, or misconfigured integrations. The reporting infrastructure exists declaratively but produces no actionable data.
- evidence: Route handler at `app/api/csp-report/route.ts:3-15`:
  ```typescript
  export async function POST(request: Request) {
    const report = await request.json();
    // console.log("CSP Violation:", report);  // ← commented out
    // No storage, no forwarding, no alerting
    return NextResponse.json({ success: true });
  }
  ```
- remediation: Implement CSP report logging — write to a database, forward to an external SIEM/logging service, or at minimum enable the commented-out `console.log`. Consider alerting on patterns that indicate XSS probing.

---

### Missing HSTS header

- file: `next.config.mjs`
- line: 190–312
- type: missing-header
- severity: MEDIUM
- description: The application does not set a `Strict-Transport-Security` header in any response. While Vercel's edge network may add HSTS at the platform level for vercel.app domains, custom domains and non-Vercel deployments receive no HSTS policy. Without HSTS, an attacker with a network position (e.g., on public Wi-Fi) can perform SSL stripping attacks against users accessing custom domains or direct deployments.
- impact: Users on HTTP connections or behind an active MITM receive no browser-enforced HTTPS upgrade. This allows SSL stripping for first-time visitors or users who previously visited over HTTP. The impact is partially mitigated by Vercel's global edge network for `*.vercel.app` domains, but custom domains configured by users rely entirely on this configuration.
- evidence: No `Strict-Transport-Security` header is present in the `headers()` function at `next.config.mjs:190-312`. Grep confirms zero hits for `Strict-Transport-Security` across the entire codebase.
- remediation: Add to the global `/:path*` header block:
  ```javascript
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" }
  ```

---

### Missing X-Content-Type-Options nosniff header

- file: `next.config.mjs`
- line: 190–312
- type: missing-header
- severity: MEDIUM
- description: The `X-Content-Type-Options: nosniff` header is only set on a single agreement-signing download endpoint (`pages/api/agreements/signing/[agreementResponseId]/download.ts:211`) and is absent from the global headers configuration. Without this header, browsers may perform MIME-type sniffing, potentially interpreting a user-uploaded file as a different MIME type (e.g., treating a PDF as HTML).
- impact: An attacker who uploads a document with crafted content and a manipulated Content-Type header could potentially cause browsers to render it as HTML/JavaScript, leading to XSS in the document viewer context. The risk is partially mitigated because uploaded documents are served from blob storage/CDN rather than the app origin, but any view that renders user-uploaded content without explicit MIME handling is affected.
- evidence: Global headers at `next.config.mjs:190-312` do not include `X-Content-Type-Options`. Only one endpoint sets it:
  ```
  pages/api/agreements/signing/[agreementResponseId]/download.ts:211:
    res.setHeader("X-Content-Type-Options", "nosniff");
  ```
- remediation: Add to the global `/:path*` header block:
  ```javascript
  { key: "X-Content-Type-Options", value: "nosniff" }
  ```

---

### Rate limiter fails open when Redis is unavailable

- file: `ee/features/security/lib/ratelimit.ts`
- line: 38
- type: other
- severity: MEDIUM
- description: The `checkRateLimit` function catches all errors from the Upstash rate limiter and returns `{ success: true, error: "Rate limiting unavailable" }`, which callers treat as a successful rate-limit pass. If Redis is down (network partition, outage, or DoS against the Redis endpoint), all rate-limited endpoints become unthrottled.
- impact: An attacker who can cause or observe a Redis outage (e.g., by saturating the Upstash rate limit endpoint with requests) can bypass all rate limits. This enables unlimited brute-force attempts against auth endpoints, unlimited billing operations, and unlimited view-recording requests. The `enableProtection: true` flag on the Upstash Ratelimit instances only kicks in during extreme abuse, not when Redis is simply unreachable.
- evidence: Code at `ee/features/security/lib/ratelimit.ts:31-46`:
  ```typescript
  export async function checkRateLimit(limiter, identifier) {
    try {
      const result = await limiter.limit(identifier);
      return { success: result.success, remaining: result.remaining };
    } catch (error) {
      console.error("Rate limiting error:", error);
      return { success: true, error: "Rate limiting unavailable" }; // ← fail open
    }
  }
  ```
  The rate limiters are defined with fairly generous limits (10 requests per 20 minutes for auth and billing), and the fail-open path bypasses them entirely.
- remediation: Change the error path to fail closed by default:
  ```typescript
  return { success: false, error: "Rate limiting unavailable" };
  ```
  Or implement a circuit-breaker that preserves the last-known rate-limit state.

---

### allowDangerousEmailAccountLinking enabled on three OAuth providers

- file: `lib/auth/auth-options.ts`
- lines: 35, 55, 130
- type: other
- severity: MEDIUM
- description: The `allowDangerousEmailAccountLinking: true` flag is set on Google, LinkedIn, and SAML providers. This allows NextAuth to automatically link a new OAuth account to an existing user if the email addresses match, without any confirmation step. An attacker who can sign up with the victim's email on a provider the victim hasn't used yet could gain access to the victim's account.
- impact: If an attacker controls any of the configured OAuth providers with the victim's email address (e.g., the victim has not yet linked their Google account, but the attacker creates a Google account with the victim's email via Google's account recovery flows), the attacker can sign in and automatically be linked to the existing Papermark account. This is compounded by the fact that `NEXTAUTH_SECRET` is used as the SAML client secret, reducing the attacker's uncertainty about SAML flow parameters.
- evidence: Three instances in `lib/auth/auth-options.ts`:
  ```typescript
  GoogleProvider({ ..., allowDangerousEmailAccountLinking: true }),     // line 35
  LinkedInProvider({ ..., allowDangerousEmailAccountLinking: true }),   // line 55
  { id: "saml", ..., allowDangerousEmailAccountLinking: true },        // line 130
  ```
- remediation: Remove `allowDangerousEmailAccountLinking` where not needed, or implement an email verification step before linking accounts. Set it to `false` on providers where account linking confirmation is desired.

---

### NEXTAUTH_SECRET reused across multiple security contexts

- file: Multiple files
- lines: `lib/auth/auth-options.ts:128,149`, `lib/jackson.ts:18,72`, `lib/signing/download-token.ts:19`, `lib/signing/access-token.ts:21`, `lib/api/auth/token.ts:12`
- type: other
- severity: LOW
- description: The `NEXTAUTH_SECRET` environment variable is used as the encryption/secret key in at least 6 distinct security contexts: JWT signing (expected), SAML OAuth client secret, SAML IdP client secret, Jackson (SSO) encryption key derivation, Jackson clientSecretVerifier, signing download token HMAC, signing access token HMAC, and API token hashing. This means a single secret compromise undermines all of these mechanisms simultaneously.
- impact: If the NEXTAUTH_SECRET is compromised (e.g., leaked via build logs, exposed in a `.env` file committed to the repo, or extracted from a server compromise), the attacker can: forge JWT session tokens, impersonate SAML IdP responses, decrypt Jackson SSO data, forge signing download/access tokens, and compute valid API token hashes.
- evidence: Direct references across the codebase:
  ```typescript
  // lib/auth/auth-options.ts — SAML OAuth clientSecret
  clientSecret: process.env.NEXTAUTH_SECRET as string;

  // lib/jackson.ts — encryption key derivation
  .update(process.env.NEXTAUTH_SECRET || "").digest("base64url").substring(0, 32);

  // lib/signing/download-token.ts — download token signing
  const secret = process.env.NEXTAUTH_SECRET;

  // lib/signing/access-token.ts — access token signing
  const secret = process.env.NEXTAUTH_SECRET;
  ```
- remediation: Use separate, dedicated secrets for each security context. Create `SAML_CLIENT_SECRET`, `JACKSON_ENCRYPTION_KEY`, `SIGNING_TOKEN_SECRET` environment variables and use them in their respective contexts instead of reusing `NEXTAUTH_SECRET`.

---

### X-Powered-By header leaks application information on custom domains

- file: `lib/middleware/domain.ts`
- line: 40–41
- type: other
- severity: LOW
- description: The DomainMiddleware adds a custom `X-Powered-By` header with the value `"Papermark - Secure Data Room Infrastructure for the modern web"` on all rewritten custom-domain responses. This leaks the specific software stack to anyone probing custom domains, which could be useful for targeted attacks.
- impact: An attacker scanning custom domains can fingerprint the backend software as Papermark, enabling targeted vulnerability searches. This is a minor information disclosure issue with limited practical impact since the main application already identifies itself via standard responses.
- evidence: Code at `lib/middleware/domain.ts:37-43`:
  ```typescript
  return NextResponse.rewrite(url, {
    headers: {
      "X-Robots-Tag": "noindex",
      "X-Powered-By": "Papermark - Secure Data Room Infrastructure for the modern web",
    },
  });
  ```
- remediation: Remove the custom `X-Powered-By` header or set it to a generic value.
