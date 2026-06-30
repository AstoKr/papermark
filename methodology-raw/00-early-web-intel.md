# Early Web CVE Intelligence — papermark

## Detected Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | TypeScript | ^5 |
| Runtime | Node.js | >=24 |
| Frontend Framework | Next.js | ^14.2.35 |
| UI Library | React | ^18.3.1 |
| CSS | Tailwind CSS | ^3.4.19 |
| UI Component Library | Tremor React | ^3.18.7 |
| Rich Text | TipTap | ^3.22.5 |
| ORM | Prisma | 6.5.0 |
| Auth Framework | NextAuth.js | ^4.24.13 |
| SAML SSO | BoxyHQ SAML Jackson | ^26.2.0 |
| OAuth/OIDC | oidc-provider | ^9.8.3 |
| WebAuthn | Hanko Passkeys | ^0.3.1 |
| Validation | Zod | ^3.25.76 |
| HTML Sanitization | sanitize-html | ^2.17.3 |
| Password Hashing | bcryptjs | ^3.0.3 |
| JWT | jsonwebtoken | ^9.0.3 |
| File Upload | @tus/server | ^1.10.2 |
| Background Jobs | Trigger.dev | 4.4.6 |
| Queue | Upstash QStash | ^2.11.0 |
| Redis | Upstash Redis | ^1.38.0 |
| Rate Limiting | Upstash Ratelimit | ^2.0.8 |
| Object Storage | AWS S3 SDK | ^3.1053.0 |
| AI SDK | Vercel AI SDK | ^6.0.191 |
| Internationalization | i18next | ^26.3.0 |
| Payments | Stripe | ^16.12.0 |
| Email | Resend | ^6.12.3 |

---

## CVEs Found

### 1. CVE-2026-44990 — sanitize-html `<xmp>` XSS (CRITICAL)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2026-44990 |
| **Package** | sanitize-html |
| **Affected Versions** | All versions prior to 2.17.4 |
| **Fixed In** | 2.17.4 |
| **CVSS** | **9.3 (Critical)** |
| **Type** | Sanitizer bypass -> Stored XSS |
| **Description** | Under default `disallowedTagsMode: 'discard'`, `<xmp>` raw-text element content is appended unescaped to output. The `nonTextTags` list does not include `<xmp>`, but the `ontext` handler special-cases it, appending content directly without escaping. Markup inside `<xmp>` becomes live HTML/JS in output. |
| **Exploit Available** | **YES** — multiple public PoCs exist |
| **Project Status** | ❌ **VULNERABLE** — on 2.17.3, must upgrade to 2.17.4 |
| **Search Directive** | In structural-analysis: trace all call chains from `lib/utils/sanitize-html.ts` through to DOM insertion points, particularly `document-header.tsx:604` and `form.tsx:145`. Verify whether sanitized output reaches `dangerouslySetInnerHTML`. |

---

### 2. CVE-2026-23864 — Next.js v14 DoS via RSC (HIGH, NO PATCH)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2026-23864 |
| **Package** | Next.js (React Server Components) |
| **Affected Versions** | 13.3.0+ through 14.x (App Router only) |
| **Fixed In** | **NO FIX for v14** — Next.js 15.0.8+ / 16+ |
| **CVSS** | **7.5 (High)** |
| **Type** | Denial of Service — memory exhaustion |
| **Description** | Specially crafted HTTP request to any App Router Server Function endpoint triggers infinite resource consumption during RSC deserialization. **No authentication required.** Next.js v14 reached EOL on Oct 26, 2025 — Vercel will not backport a fix. |
| **Exploit Available** | **YES** — public PoCs observed |
| **Project Status** | ❌ **VULNERABLE WITH NO PATCH** — on v14.2.35 (EOL branch). Pages Router is unaffected; App Router is exposed. |
| **Search Directive** | In reachability-analysis: identify all App Router Server Action endpoints and assess whether unauthenticated access to any `server-only` page or action is possible. In structural-analysis: enumerate all files using `"use server"` directive and App Router page handlers in `app/` directory. |

---

### 3. CVE-2026-23869 — Next.js v14 Cyclic Deserialization DoS (HIGH, NO PATCH)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2026-23869 |
| **Package** | Next.js (React Server Components) |
| **Affected Versions** | 13.x through 16.x (App Router only) |
| **Fixed In** | **NO FIX for v14** — Next.js 15.5.15+ / 16.2.3+ |
| **CVSS** | **7.5 (High)** |
| **Type** | Denial of Service — CPU exhaustion via cyclic deserialization |
| **Description** | A crafted HTTP request triggers cyclic deserialization in `ReactFlightReplyServer.js` (`createMap`, `createSet`, `extractIterator`), causing ~1 minute CPU spike per request. Repeated requests sustain the DoS. No auth required. |
| **Exploit Available** | **YES** — exploit code reportedly available |
| **Project Status** | ❌ **VULNERABLE WITH NO PATCH** — same EOL constraint as CVE-2026-23864 |
| **Search Directive** | In reachability-analysis: check whether any unauthenticated route exposes the RSC Flight protocol endpoint (POST to any App Router page with `next-action` header). |

---

### 4. CVE-2025-55182 — "React2Shell" Next.js RCE (CRITICAL)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2025-55182 |
| **Package** | Next.js / React Server Components |
| **Affected Versions** | Next.js < 14.2.35 (13.x, 14.x); React RSC 19.0.0–19.2.0 |
| **Fixed In** | **14.2.35** (current version) |
| **CVSS** | **10.0 (Critical)** |
| **Type** | Unauthenticated Remote Code Execution |
| **Description** | Unsafe deserialization in React Flight protocol. Attacker sends crafted `multipart/form-data` POST with `next-action` header, triggering prototype pollution gadget chain (`Array.constructor → Function()`) leading to full RCE. |
| **Exploit Available** | **YES** — widespread active exploitation observed (cryptominers, credential harvesting, C2 implants) |
| **Project Status** | ✅ **PATCHED** — on 14.2.35 which includes the fix |
| **Search Directive** | In structural-analysis: verify no legacy RSC endpoints or Server Actions bypass the `next-action` handler protection. |

---

### 5. CVE-2025-55184 — Next.js DoS (HIGH)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2025-55184 |
| **Package** | Next.js |
| **Affected Versions** | < 14.2.35 |
| **Fixed In** | **14.2.35** |
| **CVSS** | **7.5 (High)** |
| **Type** | Denial of Service — infinite loop in deserialization |
| **Description** | Crafted HTTP request causes server process to hang in infinite loop, consuming CPU. |
| **Exploit Available** | No known active exploits |
| **Project Status** | ✅ **PATCHED** (incomplete fix followed by CVE-2025-67779, also patched in 14.2.35) |

---

### 6. CVE-2025-55183 — Next.js Source Code Exposure (MEDIUM)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2025-55183 |
| **Package** | Next.js |
| **Affected Versions** | 13.3+ to < 14.2.35 |
| **Fixed In** | **14.2.35** |
| **CVSS** | **5.3 (Medium)** |
| **Type** | Source code disclosure via App Router |
| **Exploit Available** | No known active exploits |
| **Project Status** | ✅ **PATCHED** |

---

### 7. CVE-2026-33506 — BoxyHQ/Ory Polis DOM-based XSS (HIGH)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2026-33506 |
| **Package** | Ory Polis (formerly BoxyHQ Jackson / @boxyhq/saml-jackson) |
| **Affected Versions** | All versions prior to 26.2.0 |
| **Fixed In** | **26.2.0** |
| **CVSS** | **8.8 (High)** |
| **Type** | DOM-based XSS via callbackUrl parameter |
| **Description** | Improper trust of `callbackUrl` URL parameter, passed to `router.push` without sanitization. Attacker can craft malicious link that executes arbitrary JS in victim's browser context when they authenticate. Can lead to credential theft, session hijacking. |
| **Exploit Available** | PoC exists on GitHub; no confirmed active exploitation |
| **Project Status** | ✅ **PATCHED** — on 26.2.0 which is the fixed version |
| **Search Directive** | In reachability-analysis: verify the `callbackUrl` parameter is properly sanitized server-side and is not reflected unsanitized in any SAML redirect flows. |

---

### 8. CVE-2025-48985 — Vercel AI SDK Filetype Whitelist Bypass (LOW-MEDIUM)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2025-48985 |
| **Package** | Vercel AI SDK |
| **Affected Versions** | <= 5.0.51, 5.1.0-beta.0 through 5.1.0-beta.8 |
| **Fixed In** | 5.0.52 / 5.1.0-beta.9+ |
| **CVSS** | 3.7–5.3 (Low to Medium) |
| **Type** | Filetype whitelist bypass |
| **Description** | Input validation flaw allows bypassing filetype restrictions on uploads, enabling unauthorized file types to be processed by AI features. |
| **Exploit Available** | No known public exploit |
| **Project Status** | ✅ **PATCHED** — on 6.0.191 which is post-fix |

---

### 9. CVE-2026-40186 — sanitize-html Entity Decoding Bypass (MEDIUM)

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2026-40186 |
| **Package** | sanitize-html |
| **Affected Versions** | 2.17.1 to < 2.17.2 |
| **Fixed In** | 2.17.2 |
| **CVSS** | 6.1 (Medium) |
| **Type** | allowedTags bypass via entity decoding |
| **Description** | Regression where htmlparser2 entity decoding inside `textarea`/`option` elements allowed entity-encoded tags to bypass `allowedTags` filter. Affects non-default configs where these tags are allowed. |
| **Exploit Available** | PoC exists |
| **Project Status** | ✅ **NOT RELEVANT** — project configuration uses `allowedTags: []` (no tags allowed), but the CVE-2026-44990 `<xmp>` bypass applies to the default config path which IS used |

---

### 10. AIKIDO-2025-10105 — BoxyHQ SAML Jackson Timing Attack (MEDIUM)

| Field | Value |
|-------|-------|
| **CVE ID** | AIKIDO-2025-10105 |
| **Package** | @boxyhq/saml-jackson |
| **Affected Versions** | 0.4.1 to 1.38.0 |
| **Fixed In** | 1.39.0 |
| **Type** | Timing attack — unsafe comparison |
| **Description** | Authentication and secret comparisons lack constant-time cryptographic comparison methods, potentially allowing credential inference via timing side-channels. |
| **Project Status** | ✅ **ABOVE FIXED VERSION** — project uses 26.2.0 |

---

### 11. AIKIDO-2025-10182 — BoxyHQ SAML Jackson XSS (LOW)

| Field | Value |
|-------|-------|
| **CVE ID** | AIKIDO-2025-10182 |
| **Package** | @boxyhq/saml-jackson |
| **Affected Versions** | 1.3.0 to 1.43.0 |
| **Fixed In** | 1.44.0 |
| **Type** | XSS via SSO URL injection |
| **Description** | Insufficient validation of SSO URLs in SAML metadata allows script injection via manipulated URLs. |
| **Project Status** | ✅ **ABOVE FIXED VERSION** — project uses 26.2.0 |

---

### 12. AIKIDO-2026-10464 — BoxyHQ SAML Jackson XSS (HIGH)

| Field | Value |
|-------|-------|
| **CVE ID** | AIKIDO-2026-10464 |
| **Package** | @boxyhq/saml-jackson |
| **Affected Versions** | 0.0.1 to 1.52.2 |
| **Fixed In** | Not yet specified (advisory pending) |
| **Type** | Cross-Site Scripting |
| **Description** | High-severity XSS affecting broad version range. Details still emerging. |
| **Project Status** | ⚠️ **POTENTIALLY RELEVANT** — version 26.2.0 uses a different versioning scheme (Ory Polis), but may embed affected SAML Jackson code. Needs verification. |
| **Search Directive** | In structural-analysis: check whether @boxyhq/saml-jackson 26.2.0 depends on a vulnerable internal version of the SAML core. Trace all SAML response handling paths for unescaped user-controlled data. |

---

### 13. NextAuth.js — Preact Transitive Dependency (LOW)

| Field | Value |
|-------|-------|
| **Package** | next-auth / preact (transitive) |
| **Description** | GitHub discussion #13361 identifies that the bundled Preact version in next-auth may contain outdated dependencies. No CVE assigned. |
| **Project Status** | ✅ **LOW RISK** — Preact is used for client-side UI in NextAuth pages only, not for data handling |

---

## No CVEs Found (Clean)

The following packages were searched and returned **no known CVEs** for their respective versions:

| Package | Version | Notes |
|---------|---------|-------|
| Prisma ORM | 6.5.0 | No CVEs for ORM (Palo Alto Prisma SD-WAN is unrelated) |
| React | 18.3.1 | CVE-2025-55182 targets RSC layer, fixed by Next.js patch |
| TipTap | 3.22.5 | No CVEs found |
| Tailwind CSS | 3.4.19 | No CVEs found |
| Tremor React | 3.18.7 | No CVEs found |
| Zod | 3.25.76 | ReDoS fixed before 3.22.3; no CVEs on current version |
| jsonwebtoken | 9.0.3 | All CVEs fixed before 9.0.0 |
| bcryptjs | 3.0.3 | No CVEs (72-byte truncation is a bcrypt feature, not a vuln) |
| Stripe SDK | 16.12.0 | No CVEs |
| Resend | 6.12.3 | No CVEs |
| i18next | 26.3.0 | No CVEs |
| Hanko Passkeys | 0.3.1 | No CVEs |
| @tus/server | 1.10.2 | No CVEs (CORS misconfig is a config issue, not a package vuln) |
| oidc-provider | 9.8.3 | No recent CVEs |
| Upstash Redis/Ratelimit | various | No CVEs |
| Trigger.dev | 4.4.6 | No CVEs |
| AWS S3 SDK | 3.1053.0 | No CVEs |

---

## Targeted Search Directives for Structural Analysis

### Critical Priority

1. **In structural-analysis**: trace all call chains that use `sanitize-html` starting from `lib/utils/sanitize-html.ts`. Map every code path where the sanitized output reaches a DOM insertion point (particularly `dangerouslySetInnerHTML`). Focus on:
   - `components/documents/document-header.tsx:604` (stored XSS path)
   - `components/ui/form.tsx:145` (helpText prop)
   - `components/account/upload-avatar.tsx:96` (helpText prop)
   - Any new paths that bypass `sanitize-html` entirely

2. **In reachability-analysis**: identify all App Router routes in `app/` directory that use `"use server"` actions or Server Functions. Assess which of these are reachable without authentication. The `middleware.ts` explicitly excludes `/api/*`, `/oauth/*`, `/.well-known/*`, and `/mcp/*` from middleware checks — verify no App Router Server Actions are similarly unprotected.

3. **In reachability-analysis**: check if RSC Flight protocol endpoint (`next-action` POST handler) is exposed on any unauthenticated route. This is the attack vector for CVE-2026-23864 and CVE-2026-23869 (both unpatchable on v14).

### High Priority

4. **In structural-analysis**: audit all `dangerouslySetInnerHTML` usage sites (5 confirmed) for whether the input data flows through `lib/utils/sanitize-html.ts`. If any bypass the sanitize-html path, they are vulnerable to stored XSS even after the sanitizer is fixed.

5. **In encoding-analysis**: verify the entity decoding order in `lib/utils/sanitize-html.ts:13-14`. The current code calls `sanitizeHtml()` BEFORE `decodeHTML()`, which is the root cause of the entity-stripping bypass. Confirm this ordering, and whether the `<xmp>` bypass (CVE-2026-44990) exists independently of the entity ordering bug.

6. **In reachability-analysis**: confirm that `callbackUrl` / `next` parameter validation in `lib/middleware/app.ts:12-48` covers all authentication redirect paths, including SAML flows from BoxyHQ Jackson. Cross-reference with CVE-2026-33506 pattern.

7. **In structural-analysis**: trace the configuration of `oidc-provider` to verify:
   - PKCE is enforced for all OAuth flows
   - Dynamic Client Registration (DCR) scope is properly restricted
   - Redirect URI allowlist is exhaustive

### Medium Priority

8. **In structural-analysis**: enumerate all API routes in `pages/api/` and identify which lack Upstash Ratelimit integration. Cross-reference against SAST finding #4 (no auth rate limiting).

9. **In structural-analysis**: verify that `@boxyhq/saml-jackson` 26.2.0 does not embed any code affected by AIKIDO-2026-10464 (unpatched XSS in SAML Jackson core).

10. **In sbom-reachability**: confirm that `oidc-provider`, `@modelcontextprotocol/sdk`, and `tokenlens` are truly dead code (zero imports), removing unnecessary attack surface.

---

## CVE Risk Matrix

| CVE ID | Package | CVSS | Exploit Public | Project Status | Action Needed |
|--------|---------|------|----------------|---------------|---------------|
| CVE-2026-44990 | sanitize-html | **9.3** | **YES** | ❌ VULNERABLE | Upgrade to 2.17.4 immediately |
| CVE-2026-23864 | Next.js | **7.5** | **YES** | ❌ NO PATCH (EOL) | Migrate to Next.js 15+ |
| CVE-2026-23869 | Next.js | **7.5** | **YES** | ❌ NO PATCH (EOL) | Migrate to Next.js 15.5.15+ |
| CVE-2025-55182 | Next.js | **10.0** | **YES** | ✅ Patched (14.2.35) | None |
| CVE-2025-55184 | Next.js | **7.5** | No | ✅ Patched (14.2.35) | None |
| CVE-2025-55183 | Next.js | **5.3** | No | ✅ Patched (14.2.35) | None |
| CVE-2026-33506 | BoxyHQ Jackson | **8.8** | PoC | ✅ Patched (26.2.0) | Verify callbacks are sanitized |
| CVE-2025-48985 | Vercel AI SDK | **3.7-5.3** | No | ✅ Patched (6.x) | None |
| AIKIDO-2026-10464 | BoxyHQ Jackson | HIGH | TBD | ⚠️ Investigate | Trace SAML response paths |

---

## Critical Action Items

1. **IMMEDIATE**: Upgrade `sanitize-html` from `2.17.3` to `2.17.4` to fix CVE-2026-44990 (Critical, CVSS 9.3, public PoC)
2. **PLAN**: Migrate Next.js from v14 to v15+ — v14 is EOL and two unpatchable DoS CVEs (CVE-2026-23864, CVE-2026-23869) affect the App Router with no backport available
3. **VERIFY**: Confirm all SAML Jackson callback paths properly sanitize URL parameters (CVE-2026-33506 pattern)
4. **INVESTIGATE**: Determine whether AIKIDO-2026-10464 affects `@boxyhq/saml-jackson` 26.2.0
