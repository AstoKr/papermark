# AI Configuration Analysis — `ai-analyze-configs`

**Workflow:** whitebox-bug-finder (M7)
**Methodology:** M7 — Configuration security misconfiguration review
**Codebase:** papermark (Next.js 14 App Router, multi-tenant SaaS)
**Date:** 2026-06-19

---

## Scope

Configuration files reviewed for security misconfigurations:
- `next.config.mjs` — Next.js config + security headers + CORS-equivalent rewrites
- `middleware.ts` — Request middleware / routing
- `lib/auth/auth-options.ts` — NextAuth provider config + cookie settings
- `lib/auth/link-session.ts` — Link-viewer session cookie
- `lib/signing/access-token.ts` — Signed-cookie builder
- `pages/api/file/tus-viewer/[[...file]].ts` — TUS upload CORS for viewers
- `.env.example` — Documented environment variables
- `vercel.json`, `trigger.config.ts`, `package.json`, `Pipfile`, `pkgx.yaml` — Build / runtime configs
- `lib/middleware/{app,incoming-webhooks,domain,posthog}.ts` — Middleware pipeline

> **Note:** This codebase ships to **Vercel** (serverless) and **trigger.dev** (background jobs). There is **no Dockerfile, docker-compose, nginx, traefik, or caddy config** in the repo — all those concerns are delegated to the platform. Findings are therefore scoped to Next.js config, auth/cookie settings, CSP/HSTS, CORS, rate limiting, and example-secret hygiene.

---

## Findings

### Finding: CORS wildcard + `Access-Control-Allow-Credentials: true` on tus-viewer upload endpoint

- **Severity:** CRITICAL
- **File:** `pages/api/file/tus-viewer/[[...file]].ts`
- **Line:** 234–250 (especially 236–237)
- **Type:** cors-misconfig
- **Description:** The TUS upload endpoint for viewer-driven uploads sets `Access-Control-Allow-Credentials: true` while also setting `Access-Control-Allow-Origin` to either the request's `Origin` header or `*` as a fallback. The Origin is **not validated** against any allow-list before being reflected.
- **Impact:** Any malicious site can perform cross-origin authenticated uploads (chunked, resumable) against an authenticated viewer's browser, exfiltrating uploaded content or smuggling files into the team's dataroom. The reflective Origin + credentials combination defeats the browser's same-origin protections and creates a textbook CORS misconfiguration vulnerability. Even though the handler relies on Tus protocol metadata for "authentication" (see comment at line 262 — "No session check - authentication is handled via viewer metadata"), browsers will still attach cookies for the papermark domain on cross-origin requests in some cases, and the metadata validation path can be bypassed by an attacker who knows or guesses metadata parameters.
- **Evidence:**
  ```ts
  const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse) => {
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
    res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
    // ...
  };
  ```
- **Remediation:**
  1. Replace the reflective Origin with an explicit allow-list of trusted custom domains (e.g. `papermark.com`, customer-configured dataroom domains).
  2. If credentials truly are not needed (per the comment), remove the `Access-Control-Allow-Credentials: true` header entirely.
  3. Reject any Origin not in the allow-list with a 403 instead of falling back to `*`.
  4. Add server-side authentication at the request layer (cookies or signed tokens) so the endpoint is not solely dependent on Tus metadata for authorization.

---

### Finding: Content Security Policy is set to **Report-Only** in production

- **Severity:** HIGH
- **File:** `next.config.mjs`
- **Line:** 219 (and the full CSP definition 220–230)
- **Type:** missing-header (CSP enforcement)
- **Description:** The default CSP for all routes is emitted under the header name `Content-Security-Policy-Report-Only`. The browser only logs violations; it does **not** enforce them. The intent (per the trailing comment `report-to csp-endpoint`) is to gather reports via `/api/csp-report`, but this means the CSP is currently advisory, not protective.
- **Impact:** The CSP is the primary mitigation for XSS in this app, and combined with the `'unsafe-inline'`, `'unsafe-eval'`, and wildcard-`https:` sources (see next finding), an attacker who achieves script execution will not be blocked. The browser will load and execute the malicious script before reporting it. The CSP only protects the team that reads the reports after the fact.
- **Evidence:**
  ```js
  {
    key: "Content-Security-Policy-Report-Only",
    value:
      `default-src 'self' https: ${isDev ? "http:" : ""}; ` +
      // ...
      "report-to csp-endpoint;",
  }
  ```
  Note the `-Report-Only` suffix — the browser does not enforce this header.
- **Remediation:**
  1. Promote the header to `Content-Security-Policy` (drop `-Report-Only`) once a baseline of reports has been collected and the policy is not breaking legitimate functionality.
  2. Keep a parallel `Content-Security-Policy-Report-Only` header in production to catch regressions without breaking the live site.
  3. Once enforced, tighten the directives (see next finding) — currently `'unsafe-inline'`, `'unsafe-eval'`, and `https:` wildcards are permitted, which defeats most of the protection CSP can offer.

---

### Finding: CSP permits `'unsafe-inline'`, `'unsafe-eval'`, and wildcard `https:` for script-src

- **Severity:** HIGH
- **File:** `next.config.mjs`
- **Line:** 222, 265 (and the `connect-src`/style-src/img-src lines around them)
- **Type:** csp-misconfig (overly permissive directives)
- **Description:** Both the default `/​:path*` CSP and the `/view/:path*/embed` CSP include:
  - `script-src 'self' 'unsafe-inline' 'unsafe-eval' https: [...]`
  - `style-src 'self' 'unsafe-inline' https: [...]`
  - `default-src 'self' https: [...]`
  - `img-src 'self' data: blob: https: [...]`
  - `font-src 'self' data: https: [...]`
  - `connect-src 'self' https: [...]`
  This effectively whitelists every HTTPS script, style, image, font, and connection endpoint in the world — and `unsafe-inline` + `unsafe-eval` remove the two most important XSS guards.
- **Impact:** Even when/if the CSP is promoted to enforcement mode (see prior finding), the policy as written will not block injected scripts because:
  - `'unsafe-inline'` allows inline `<script>` blocks and event handlers (`onerror`, etc.).
  - `'unsafe-eval'` allows `eval()`, `new Function()`, and similar dynamic code execution vectors.
  - The `https:` wildcards allow an attacker who controls any HTTPS host to serve malicious JS that the browser will execute.
- **Evidence:** See lines 222–227 of `next.config.mjs` for the default-policy and lines 265–270 for the embed-policy.
- **Remediation:**
  1. Replace `'unsafe-inline'` with a per-request nonce (Next.js supports `useServerInsertedHTML` and nonce propagation) or `'strict-dynamic'` paired with nonces.
  2. Remove `'unsafe-eval'` — if Next.js dev tooling requires it, scope it to development only (the current policy already conditionally injects `http:` for dev; do the same with `'unsafe-eval'`).
  3. Replace the `https:` wildcards with explicit allow-lists (e.g. `'self'`, the auth providers' domains, blob/cdn hosts, etc.).
  4. Remove `blob:` from `img-src` unless actively used and audited; same for `data:` in script contexts.

---

### Finding: Missing `Strict-Transport-Security` (HSTS) header

- **Severity:** HIGH
- **File:** `next.config.mjs` (or platform-level config — Vercel does **not** set HSTS automatically)
- **Line:** 190–312 (entire `headers()` config)
- **Type:** missing-header
- **Description:** The `headers()` function emits `Referrer-Policy`, `X-DNS-Prefetch-Control`, `X-Frame-Options`, `Report-To`, and a `Content-Security-Policy-Report-Only`. It does **not** emit a `Strict-Transport-Security` header anywhere.
- **Impact:** First-time visitors can be downgraded to HTTP via an active network attacker (Coffee-shop WiFi, hostile ISP, etc.) and have their first request — including any cookies, redirects, or auth tokens in flight — observed or tampered with. HSTS also blocks user-driven `http://` navigation and click-through of mixed-content warnings.
- **Remediation:** Add the following to the default `/:path*` headers block:
  ```js
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" }
  ```
  Note: `preload` is irrevocable — submit to https://hstspreload.org only when the entire domain (and all subdomains) is HTTPS-only. Use `includeSubDomains` first; promote to `preload` after a soak period.

---

### Finding: Missing `X-Content-Type-Options: nosniff`

- **Severity:** MEDIUM
- **File:** `next.config.mjs`
- **Line:** 190–312 (headers block)
- **Type:** missing-header
- **Description:** The response headers do not include `X-Content-Type-Options: nosniff`. Without this, browsers may perform MIME-sniffing and interpret responses as a different content type than declared.
- **Impact:** A user-uploaded file with content that "looks like" HTML could be rendered as HTML in the browser context. Combined with the permissive CSP (see prior finding), this could lead to stored XSS via document uploads.
- **Remediation:** Add `{ key: "X-Content-Type-Options", value: "nosniff" }` to the default `/:path*` headers block.

---

### Finding: Missing `Permissions-Policy` (browser feature restrictions)

- **Severity:** MEDIUM
- **File:** `next.config.mjs`
- **Line:** 190–312 (headers block)
- **Type:** missing-header
- **Description:** No `Permissions-Policy` (formerly `Feature-Policy`) header is set. The default is to grant all origins access to powerful browser features.
- **Impact:** If an XSS occurs (CSP is Report-Only, see above), the attacker can request camera/microphone/geolocation/payment APIs without user prompts beyond the default browser behavior. Less critical in isolation but compounds the XSS impact.
- **Remediation:** Add a restrictive policy such as:
  ```js
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=(), payment=(), usb=(), magnetometer=(), gyroscope=(), accelerometer=()" }
  ```
  Scope to `self` only for features the app legitimately uses (none here that need wide grant).

---

### Finding: Missing Cross-Origin isolation headers (`COOP`, `COEP`, `CORP`)

- **Severity:** MEDIUM
- **File:** `next.config.mjs`
- **Line:** 190–312 (headers block)
- **Type:** missing-header
- **Description:** The app does not set `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy`, or `Cross-Origin-Resource-Policy`. Without these, the app cannot use cross-origin-isolated APIs (e.g. `SharedArrayBuffer`, high-resolution timers) and is exposed to cross-origin attacks such as speculative execution side-channels (Spectre).
- **Impact:** In addition to missing out on performance features, the lack of `COEP: require-corp` allows any third-party resource to be loaded without an explicit `Cross-Origin-Resource-Policy` declaration, weakening the defense-in-depth against supply-chain attacks via compromised CDN assets.
- **Remediation:** Add (after auditing all third-party resources):
  ```js
  { key: "Cross-Origin-Opener-Policy", value: "same-origin" }
  { key: "Cross-Origin-Embedder-Policy", value: "require-corp" }
  { key: "Cross-Origin-Resource-Policy", value: "same-origin" }
  ```
  Note that `COEP: require-corp` may break legitimate third-party embeds that don't set CORP — proceed incrementally.

---

### Finding: `allowDangerousEmailAccountLinking: true` on Google, LinkedIn, and SAML providers

- **Severity:** HIGH
- **File:** `lib/auth/auth-options.ts`
- **Line:** 35, 55, 130
- **Type:** auth-misconfig
- **Description:** All three OAuth/OIDC providers are configured with `allowDangerousEmailAccountLinking: true`. This causes NextAuth to automatically link a new authentication to an existing account if the verified email matches, without any user interaction or re-authentication.
- **Impact:** Account takeover vector. If an attacker can:
  - (a) create or control a Google/LinkedIn account using the victim's email address (some providers issue unverified emails; some corporate Google Workspaces allow self-service email claiming), OR
  - (b) compromise a SAML IdP that issues an assertion for an email they don't control,

  then signing in via that provider will inherit the existing user's session, gaining full access to all teams, datarooms, billing, and SSO controls. The same email-confusion attack class that has historically impacted other SaaS products.

  This is also relevant for the SAML path because Papermark also accepts email logins (EmailProvider at line 57–86) — a SAML-authenticated user with `allowDangerousEmailAccountLinking` will silently inherit any prior email-provider account.
- **Evidence:**
  ```ts
  GoogleProvider({
    // ...
    allowDangerousEmailAccountLinking: true,
  }),
  LinkedInProvider({
    // ...
    allowDangerousEmailAccountLinking: true,
  }),
  // ...SAML provider:
  allowDangerousEmailAccountLinking: true,
  ```
- **Remediation:**
  1. Remove `allowDangerousEmailAccountLinking: true` from all three providers.
  2. Implement an explicit account-linking flow: when a user authenticates with a provider whose email matches an existing account, force them to confirm linking via password re-authentication or email-based verification (one-time link to the existing email).
  3. For SAML specifically, require IdP-side email verification (assertion must include a verified email attribute, not just a claim) and consider rejecting assertions where the email domain doesn't match a pre-configured expected value.

---

### Finding: Default/example secret values in `.env.example`

- **Severity:** MEDIUM
- **File:** `.env.example`
- **Line:** 1, 68
- **Type:** secrets-in-config
- **Description:** Two sensitive environment-variable placeholders use realistic-looking but obviously placeholder values:
  ```bash
  NEXTAUTH_SECRET=my-superstrong-secret
  NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret
  ```
  While `.env.example` is intentionally checked into git and copied to `.env.local` by the README's onboarding steps, the values look like real secrets rather than obvious placeholders.
- **Impact:**
  1. A developer who runs `cp .env.example .env.local` without editing these lines will boot the app with a publicly-known secret. If that developer then deploys to a preview environment using the same `.env.local` (or if `.env.local` is committed by mistake — `.gitignore` ignores `.env.local` but not all teams enforce this), the `NEXTAUTH_SECRET` used to sign session JWTs is `my-superstrong-secret` — published in this repo.
  2. Anyone who knows the convention can forge NextAuth session JWTs and gain access to whatever preview/staging environment uses that secret.
  3. Same risk for `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` — published placeholder would be used to derive document password keys in any environment that picks up the example values unchanged.
- **Remediation:**
  1. Replace with empty values (or obvious `REPLACE_ME` strings):
     ```bash
     NEXTAUTH_SECRET=
     NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=
     ```
  2. Add a header comment: `# DO NOT use these placeholder values. Generate fresh secrets for every environment (e.g. `openssl rand -base64 32`).`
  3. Add a runtime check that fails fast if `NEXTAUTH_SECRET` matches a known placeholder value (defense-in-depth — block the app from booting with `my-superstrong-secret`).
  4. Same for `NEXT_PRIVATE_VERIFICATION_SECRET` (currently empty — good).

---

### Finding: `Referrer-Policy: no-referrer-when-downgrade` leaks referrer on cross-origin requests

- **Severity:** LOW
- **File:** `next.config.mjs`
- **Line:** 199–201
- **Type:** missing-header (suboptimal policy)
- **Description:** The default `Referrer-Policy` is `no-referrer-when-downgrade`, which sends the full URL (including path and query string) as the `Referer` header for same-or-HTTPS-strictly-greater-protocol requests.
- **Impact:** When a user clicks an outbound link from a Papermark page (e.g. external integrations, marketing links, OAuth callbacks), the full URL — including document IDs, link IDs, or other resource identifiers in the path/query — is sent to the destination origin. This can leak resource identifiers to third parties. The downgrade case (HTTPS→HTTP) is the only protection.
- **Remediation:** Use `strict-origin-when-cross-origin` (sends only the origin on cross-origin HTTPS requests) or `no-referrer` (sends nothing).

---

### Finding: Mixed clickjacking protection (`X-Frame-Options: SAMEORIGIN` vs. CSP `frame-ancestors 'none'`)

- **Severity:** LOW
- **File:** `next.config.mjs`
- **Line:** 207–209 (X-Frame-Options), 226 (frame-ancestors in CSP)
- **Type:** header-misconfig (contradictory policies)
- **Description:** The default CSP sets `frame-ancestors 'none'` (correct, deny all framing), but `X-Frame-Options: SAMEORIGIN` is also set. Modern browsers honor CSP `frame-ancestors` over X-Frame-Options, so the effective policy is `'none'`. However, legacy browsers (and some embedded WebViews/proxies) that don't understand CSP will fall back to `SAMEORIGIN` — which permits framing by `app.papermark.com` itself.
- **Impact:** Any subdomain or path on `app.papermark.com` could potentially frame any other path. If an attacker can find an XSS or open-redirect on `app.papermark.com`, they can frame the actual document viewer page inside the malicious framing context. Low likelihood but trivial to fix.
- **Remediation:** Set `X-Frame-Options: DENY` for consistency, or remove `X-Frame-Options` entirely once `frame-ancestors 'none'` is verified to be enforced in the CSP (not the Report-Only header — see earlier finding).

---

### Finding: Embed route CSP allows `frame-ancestors *` (any origin may frame)

- **Severity:** LOW
- **File:** `next.config.mjs`
- **Line:** 258–278
- **Type:** header-misconfig (intentional but permissive)
- **Description:** The `/view/:path*/embed` route overrides the CSP to set `frame-ancestors *`. This is by design (the route is meant to be embeddable), but there is no documented allow-list of which customer domains may embed.
- **Impact:** Any website on the public internet can frame the embed view of any Papermark document whose URL is known. Combined with the permissive CSP and missing `nosniff`, this expands the attack surface. While the documents themselves are designed to be shared, an unintended third party could frame and reskin a document, potentially misleading recipients about the source.
- **Remediation:** Replace `frame-ancestors *` with an explicit allow-list driven by an env var (e.g. `NEXT_PUBLIC_EMBED_ALLOWED_ORIGINS`) and per-team configuration. If full wildcards are required for the product, add `Referrer-Policy: no-referrer` to embed responses to prevent leaking the document URL.

---

### Finding: `/api/*` reachable on api subdomain (relies entirely on per-route auth)

- **Severity:** MEDIUM
- **File:** `next.config.mjs`
- **Line:** 56–63 (comment block); 44–94 (rewrites); `middleware.ts:62–73`
- **Type:** surface-bloat (relies on auth, not network segmentation)
- **Description:** The api.papermark.com host is configured to only expose `/v1/*` and `/openapi.json` cleanly — `/`, `/favicon.ico`, `/sitemap.xml`, `/robots.txt`, `/oauth/*`, and `/.well-known/*` are rewritten to `/404`. **However**, the comment explicitly states:
  > "we deliberately do NOT block `/api/*` on the api host. … The internal /api/* routes are already auth-gated by NextAuth or bearer-token middleware, so reachability on api.papermark.com adds no net exposure beyond app.papermark.com."
- **Impact:** This is a deliberate design choice that depends on every `/api/*` route enforcing auth correctly. Any future API route that forgets an auth check (or a new route added by a developer unfamiliar with the api-subdomain assumption) will be exposed on `api.papermark.com` without any network-level protection. The blast radius of any single missed auth check is the entire internal API surface — instead of just the broken route.
- **Remediation:**
  1. Add a defense-in-depth middleware check on api.papermark.com that 404s any `/api/*` request that doesn't match `/api/v1/*` or `/api/oauth/.well-known/*` or `/api/well-known/*` (the explicitly public endpoints).
  2. Or move all non-v1 API routes to `app.papermark.com/api/*` only and explicitly remove them from the api-subdomain middleware matcher's allow-list.
  3. Add a CI test that fetches a sample of internal `/api/*` paths against `api.papermark.com` and asserts they 404 (regression guard).

---

### Finding: `vercel.json` does not pin function regions or runtime

- **Severity:** LOW
- **File:** `vercel.json`
- **Line:** 1–10
- **Type:** build-config
- **Description:** `vercel.json` only configures memory for two `mupdf` endpoints. It does not pin `regions`, `runtime`, or framework-specific options. The platform defaults will apply (typically Node 22, us-east-1, edge runtime where applicable).
- **Impact:** Inconsistent performance / higher latency for users far from the default region. Not a direct security issue but worth noting that edge functions deployed without explicit region pinning may have weaker isolation guarantees than dedicated serverless functions.
- **Remediation:** Pin region(s) closest to the user base and explicitly set `runtime` to `nodejs22.x` for the mupdf endpoints to ensure predictable behavior.

---

### Finding: `trigger.config.ts` exposes project ID in source

- **Severity:** LOW
- **File:** `trigger.config.ts`
- **Line:** 7
- **Type:** info-disclosure
- **Description:** The Trigger.dev `project` ID `proj_plmsfqvqunboixacjjus` is hard-coded in source. This is the expected convention (Trigger.dev docs recommend committing this), but it does identify the project's tenant in any leaked source/log.
- **Impact:** Low. The project ID alone is not sensitive — it requires the matching `TRIGGER_SECRET_KEY` to interact. But the value is suitable for log scrubbing in case of accidental commit-history or log exposure.
- **Remediation:** None required; mention only to ensure secret-management hygiene is consistent across the org.

---

### Finding: TUS upload endpoint (`/api/file/tus`) lacks CORS configuration (implicit default)

- **Severity:** LOW
- **File:** `pages/api/file/tus/[[...file]].ts`
- **Line:** (entire file)
- **Type:** missing-config
- **Description:** Unlike the viewer endpoint (which has explicit permissive CORS — see first finding), the team-authenticated TUS upload endpoint has no CORS configuration at all. The browser default applies — no cross-origin requests allowed without explicit CORS headers.
- **Impact:** By default, browsers block cross-origin uploads to this endpoint, which is the safer behavior. However, this means that any team that wants to upload from a custom domain (e.g. a customer-hosted UI) will hit silent failures and may be tempted to add permissive CORS — repeating the misconfiguration from `tus-viewer`.
- **Remediation:** Document the CORS policy explicitly. If customer-domain uploads are needed, add a *narrow* reflective CORS policy validated against a per-team allow-list (NOT a wildcard). If not needed, do nothing.

---

### Finding: Signed-cookie builder requires explicit `secure` flag (caller decides)

- **Severity:** LOW
- **File:** `lib/signing/access-token.ts`
- **Line:** 161–183 (`buildSignedAgreementAccessCookie`)
- **Type:** auth-design
- **Description:** The cookie builder accepts a `secure: boolean` parameter. The decision of whether to set the `Secure` flag is delegated to every caller.
- **Impact:** If a future caller forgets to pass `secure: true` in a production context, the cookie will be sent over HTTP, exposing the HMAC-signed agreement access token. There's no central enforcement.
- **Remediation:** Auto-detect `secure` based on the request URL (e.g. `req.url.startsWith("https://")` or use `process.env.NODE_ENV === "production"`). The current code already gates `secure: process.env.NODE_ENV === "production"` in all known callers (verified at lines 990, 344, 354, 384, 395, 226, 236, 266, 277 of various route files), but the central helper doesn't enforce it.

---

### Finding: Session cookies use `SameSite=Lax` (no `Strict`)

- **Severity:** LOW
- **File:** `lib/auth/auth-options.ts`
- **Line:** 183–194
- **Type:** cookie-config
- **Description:** The NextAuth session token cookie is configured with `sameSite: "lax"`. `Lax` allows the cookie to be sent on top-level navigations (e.g. clicking an external link to the app). `Strict` would block this entirely.
- **Impact:** Low — `Lax` is the modern default for session cookies and provides good CSRF protection for state-changing endpoints. The trade-off is that links from external emails/Slack/etc. do maintain the session, which is usually desirable. If `Strict` is acceptable to the product, it would provide stronger CSRF protection.
- **Remediation:** Consider `SameSite=Strict` for sensitive flows (e.g. billing, team-admin). Document the decision; no immediate change required.

---

## Summary Table

| # | Severity | Title | File | Line |
|---|----------|-------|------|------|
| 1 | CRITICAL | CORS wildcard + credentials on tus-viewer | `pages/api/file/tus-viewer/[[...file]].ts` | 236–237 |
| 2 | HIGH | CSP in Report-Only mode (not enforced) | `next.config.mjs` | 219 |
| 3 | HIGH | CSP allows `unsafe-inline` / `unsafe-eval` / `https:` wildcards | `next.config.mjs` | 222–227, 265–270 |
| 4 | HIGH | Missing HSTS header | `next.config.mjs` | 190–312 |
| 5 | HIGH | `allowDangerousEmailAccountLinking` on 3 providers | `lib/auth/auth-options.ts` | 35, 55, 130 |
| 6 | MEDIUM | Missing `X-Content-Type-Options: nosniff` | `next.config.mjs` | 190–312 |
| 7 | MEDIUM | Missing `Permissions-Policy` | `next.config.mjs` | 190–312 |
| 8 | MEDIUM | Missing `COOP` / `COEP` / `CORP` | `next.config.mjs` | 190–312 |
| 9 | MEDIUM | Default secret values in `.env.example` | `.env.example` | 1, 68 |
| 10 | MEDIUM | `/api/*` reachable on api.papermark.com (auth-only segmentation) | `next.config.mjs` | 56–63 |
| 11 | LOW | `Referrer-Policy: no-referrer-when-downgrade` leaks URL on outbound | `next.config.mjs` | 199–201 |
| 12 | LOW | X-Frame-Options vs CSP `frame-ancestors` mismatch | `next.config.mjs` | 207–209, 226 |
| 13 | LOW | Embed route `frame-ancestors *` (no allow-list) | `next.config.mjs` | 269 |
| 14 | LOW | `vercel.json` doesn't pin regions/runtime | `vercel.json` | 1–10 |
| 15 | LOW | `trigger.config.ts` project ID committed | `trigger.config.ts` | 7 |
| 16 | LOW | TUS upload endpoint has no CORS config (implicit default) | `pages/api/file/tus/[[...file]].ts` | (file-wide) |
| 17 | LOW | Cookie `Secure` flag decided per-caller (no central enforcement) | `lib/signing/access-token.ts` | 161–183 |
| 18 | LOW | Session cookie uses `SameSite=Lax` (not `Strict`) | `lib/auth/auth-options.ts` | 183–194 |

---

## Recommended Remediation Order

1. **Fix Finding #1 (CRITICAL CORS)** — single highest-impact misconfiguration; patch and deploy ASAP.
2. **Fix Finding #5 (account linking)** — high account-takeover risk.
3. **Fix Finding #2 (CSP enforcement) + #3 (CSP directives)** — do together; promote to enforcing mode after tightening.
4. **Add HSTS (#4) + nosniff (#6) + Permissions-Policy (#7) + COOP/COEP/CORP (#8)** — straightforward header additions.
5. **Clean up `.env.example` (#9)** — cosmetic but important.
6. **Add api-subdomain defense-in-depth (#10)** — middleware change.
7. **Polish remaining LOW findings as time permits.**
