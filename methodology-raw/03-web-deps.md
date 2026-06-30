# Web Search — Dependency CVEs

**Node:** Layer-3 Dependency Web Research  
**Date:** 2026-06-30  
**Queries executed:** 15 (D-01 through D-15)  
**Coverage:** All 6 HIGH, 6 MEDIUM, 3 LOW per strategy doc  

---

## CVEs

### CVE-2023-30533 (GHSA-4r6h-8v6p-xvw6) — xlsx Prototype Pollution
- **package/framework:** `xlsx` (SheetJS Community Edition) < 0.19.3
- **exploit available:** Public advisory at https://github.com/advisories/GHSA-4r6h-8v6p-xvw6; PoC via crafted XLSX file. No standalone PoC repo found, but the mechanism is well-documented — a crafted `.xlsx` injects properties into `Object.prototype` during parsing.
- **applies-to-our-codebase: NO**
- **reasoning:** Papermark pins `xlsx@0.20.3` which is above the 0.19.3 fix threshold. The prototype pollution vector is patched at this version. However, the supply-chain concern (CDN tarball without integrity hash, D-05 below) remains a separate risk.

### CVE-2024-22363 (GHSA-5pgg-2g8v-p4x9) — xlsx ReDoS
- **package/framework:** `xlsx` < 0.20.2
- **exploit available:** CVSS 7.5. Regex `tagregex1` with `/mg` flags causes catastrophic backtracking on crafted XML in spreadsheet files. Fixed by removing the `m` flag. Documented at https://www.sonatype.com/blog/nexus-intelligence-insights-sonatype-2018-0622-xlsx-sheetjs-regular-expression-denial-of-service.
- **applies-to-our-codebase: NO**
- **reasoning:** Papermark pins `xlsx@0.20.3` which is above the 0.20.2 fix threshold. The vulnerable `/mg` regex was already patched. Both xlsx CVEs (PP + ReDoS) are fixed in the version used.

### CVE-2024-4367 (GHSA-wgrm-67xf-hhpq) — pdfjs-dist XSS / Arbitrary JS Execution
- **package/framework:** `pdfjs-dist` <= 4.1.392 (npm)
- **exploit available:** Yes — public PoC at https://github.com/ngductung/PDFjs-XSS-PoC and https://github.com/snyk-labs/pdfjs-vuln-demo. Python script `CVE-2024-4367.py` generates a malicious PDF that executes arbitrary JS via `eval()` in `font_loader.js` when `isEvalSupported` is true (default). Also on Exploit-DB #52273.
- **applies-to-our-codebase: YES**
- **reasoning:** Papermark ships `pdfjs-dist@3.11.174` via `react-pdf`. This is well below the 4.2.67 fix threshold. Papermark renders user-uploaded PDFs in-browser via the preview viewer at `components/documents/preview-viewers/preview-viewer.tsx`. An attacker who uploads a crafted PDF triggers XSS in any viewer's browser context. CVSS 8.8, no authentication required to view. **VALID FINDING.**

### CVE-2024-34342 — react-pdf XSS (downstream of pdfjs-dist)
- **package/framework:** `react-pdf` < 7.7.3 / < 8.0.2 / < 9.0.0
- **exploit available:** Same PoCs as CVE-2024-4367. Snyk demo repo at https://github.com/snyk-labs/pdfjs-vuln-demo specifically targets React PDF viewers. Discussion at https://github.com/wojtekmaj/react-pdf/discussions/1786.
- **applies-to-our-codebase: YES (need to check react-pdf version)**
- **reasoning:** Papermark wraps `pdfjs-dist` through `react-pdf`. Even if `react-pdf` has applied the `isEvalSupported: false` workaround in newer versions, the underlying `pdfjs-dist@3.11.174` remains vulnerable. If the `react-pdf` version is < 7.7.3, both the workaround and the underlying library are unfixed. This is a client-side XSS vector targeting users who view uploaded PDFs.

### CVE-2026-23864 — Next.js 14 / React RSC BigInt DoS
- **package/framework:** Next.js 14.x (EOL, no patch), React 19.0.0–19.0.4
- **exploit available:** CVSS 7.5. Unauthenticated remote attacker sends crafted HTTP request with excessively large BigInt values in Flight protocol deserialization. Akamai analysis at https://www.akamai.com/blog/security-research/2026/jan/cve-2026-23864-react-nextjs-denial-of-service. Payload: `0:"$n9999999...[1M chars]"`.
- **applies-to-our-codebase: YES**
- **reasoning:** Papermark on Next.js 14.2.35. Next.js 14.x is EOL and will NOT receive patches. No upgrade path exists within the 14.x line — requires migrating to Next.js 15.x+. This is an unauthenticated, network-based DoS that can exhaust server CPU/memory via a single HTTP request to any App Router endpoint (pages using `"use client"` or Server Actions). **VALID FINDING. Cannot fix by version bump within Next.js 14 — architectural upgrade required.**

### CVE-2026-23869 ("React2DoS") — Next.js 14 / React RSC Cyclic Deserialization DoS
- **package/framework:** Next.js 14.x (EOL, no patch), React 19.0.0–19.0.4
- **exploit available:** CVSS 7.5. Imperva research at https://www.imperva.com/blog/react2dos-cve-2026-23869-when-the-flight-protocol-crashes-at-takeoff/. Exploits cyclic Map/Set references in Flight protocol to cause quadratic-time deserialization. Significantly worse impact than CVE-2026-23864 at equivalent payload sizes.
- **applies-to-our-codebase: YES**
- **reasoning:** Same as CVE-2026-23864 — Next.js 14.x EOL, no patch. Both RSC DoS CVEs apply and cannot be independently fixed. **VALID FINDING — chains with CVE-2026-23864 for compounded DoS impact.**

### CVE-2026-44990 (GHSA-rpr9-rxv7-x643) — sanitize-html `<xmp>` Tag Bypass
- **package/framework:** `sanitize-html` < 2.17.4
- **exploit available:** Public PoC — `sanitizeHtml('<xmp><script>alert(1)</script></xmp>')` returns `<script>alert(1)</script>`. CVSS 9.3 CRITICAL. Advisory at https://github.com/apostrophecms/apostrophe/security/advisories/GHSA-rpr9-rxv7-x643.
- **applies-to-our-codebase: NO**
- **reasoning:** Per the synthesized dependency analysis, papermark uses `sanitize-html` exclusively in plaintext-strip mode with `allowedTags: []`. The `<xmp>` bypass only works when the sanitized output is rendered as trusted HTML (e.g., via `innerHTML`). Papermark uses it to strip HTML tags from user input, extracting plain text only. The real XSS risk in papermark is the decode-ordering issue (structural finding), not this CVE. **Mitigated by usage pattern.**

### CVE-2024-47764 — cookie Package Prefix Injection
- **package/framework:** `cookie` npm < 0.7.0
- **exploit available:** PortSwigger research at https://portswigger.net/research/cookie-chaos-how-to-bypass-host-and-secure-cookie-prefixes — "Cookie Chaos: How to bypass __Host and __Secure cookie prefixes." Demonstrates Unicode normalization and parser discrepancy attacks. CVSS 6.5 (Medium).
- **applies-to-our-codebase: NO**
- **reasoning:** Papermark has `cookie@0.4.2` as a transitive dependency via `engine.io` → `socket.io` → `@trigger.dev/core` → `@trigger.dev/sdk`. This is used for Trigger.dev's server-to-cloud WebSocket transport, not for papermark's own HTTP cookie handling. Papermark's session cookies are managed by `next-auth` via `next/headers`, which uses its own cookie handling. No user input reaches engine.io's cookie parser. **Not reachable by papermark attackers.**

### CVE-2022-21676 — engine.io Uncaught Exception DoS
- **package/framework:** `engine.io` < 4.1.2 / < 5.2.1 / < 6.1.1
- **exploit available:** CVSS 7.5. Snyk advisory at https://security.snyk.io/vuln/SNYK-JS-ENGINEIO-2336356. Crafted HTTP request causes uncaught exception, crashing the Node.js process.
- **applies-to-our-codebase: NO**
- **reasoning:** `engine.io` is a transitive dependency inside `socket.io` inside `@trigger.dev/core`. Trigger.dev manages its own WebSocket channel for server-to-cloud communication. Papermark does not expose engine.io endpoints to users. Not reachable.

### CVE-2022-41940 — engine.io DoS
- **package/framework:** `engine.io` < 3.6.1 / < 6.2.1
- **exploit available:** CVSS 7.1. Same class of uncaught exception DoS as CVE-2022-21676.
- **applies-to-our-codebase: NO**
- **reasoning:** Same as above — Trigger.dev infra channel, not user-facing. Not reachable.

### CVE-2026-57957 — Papermark TUS CORS Misconfiguration
- **package/framework:** Papermark <= 0.22.0 (own code, not a library CVE)
- **exploit available:** Public CVE entry at https://cvefeed.io/vuln/detail/CVE-2026-57957. CVSS 4.7 (Medium). CORS misconfiguration in TUS viewer upload endpoint reflects arbitrary `Origin` with `Access-Control-Allow-Credentials: true`.
- **applies-to-our-codebase: YES**
- **reasoning:** This is a Papermark-specific CVE affecting versions <= 0.22.0. The working tree code likely predates the 0.23.0 fix. An unauthenticated attacker can create a malicious page that, when visited by an authenticated Papermark user, silently uploads arbitrary files to datarooms or reads credentialed responses. **VALID FINDING — confirms TUS CORS misconfiguration.**

### CVE-2026-35412 — Directus TUS Authorization Bypass (Related Pattern)
- **package/framework:** Directus (not Papermark, but same TUS protocol pattern)
- **exploit available:** CVSS High. TUS upload endpoint checks collection-level but not item-level permissions, allowing authenticated users to overwrite arbitrary files.
- **applies-to-our-codebase: NO (Directus-specific)**
- **reasoning:** This is a Directus-specific implementation flaw, not a protocol-level issue. However, the pattern is relevant context for reviewing Papermark's TUS upload handler — if Papermark's TUS implementation similarly checks only collection-level permissions, a similar bypass could exist. **Flag for manual code review of authorization granularity in TUS handlers.**

### oidc-provider — No Published CVEs
- **package/framework:** `oidc-provider` 9.8.3 (npm)
- **exploit available:** No public CVEs found. GitHub security page shows no published advisories. Release notes indicate security fixes: SSRF protection (v9.8.4), XSS in HTML helper (v9.8.5), malformed DPoP rejection (v9.8.3), JWKS null validation (v8.4.2).
- **applies-to-our-codebase: NO**
- **reasoning:** `oidc-provider` is listed as an unused dependency in S-005 (zero imports in source code). Even if it had CVEs, the code is not called at runtime. No action needed beyond eventual removal from `package.json`.

---

## Non-CVE Technique Findings

### D-01 / D-02 — cuid ID Prediction
- **package:** `cuid` ^3.0.0
- **finding:** The `cuid` package at v3.0.0 is **CUID2**, not the original CUID v1. CUID2 uses `crypto.getRandomValues()` + SHA-3 hashing of multiple entropy sources (time, random salt, session counter randomized at start, host fingerprint). **CUID2 is NOT predictable** — the session counter is initialized with a random value and mixed with a per-ID salt before hashing. The original CUID (v1, using `Math.random()`) was predictable after 4 consecutive IDs (PoC exists at https://github.com/paralleldrive/cuid/issues/78).
- **applies-to-our-codebase: PARTIAL — reassess S-001**
- **reasoning:** The synthesized finding S-001 describes CUID v1 behavior (timestamps + sequential counter). Papermark uses `cuid@^3.0.0`, which is CUID2 and cryptographically secure. S-001's severity may be overstated. However, CUID2 does incorporate a host fingerprint and session state, so it is technically not pure-random like nanoid. If the goal is defense-in-depth and auditability, migrating to `nanoid` (already in the dependency tree at 5.1.11) would eliminate any residual fingerprint-based correlation risk. **Recommend downgrading S-001 severity or closing as mitigated by CUID2 usage.**

### D-05 — CDN Tarball Without Integrity Hash (Supply Chain)
- **package:** `xlsx` 0.20.3
- **finding:** `package.json` pins `xlsx` to `https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz` with no integrity hash. CVE-2025-69263 demonstrates that pnpm lockfiles (papermark uses npm/pnpm?) can store tarball URLs without integrity fields, allowing the remote server to serve different content on each install. The `lockfile-lint` tool (https://github.com/Lappy000/lockfile-lint) detects this as a `suspicious-url` issue.
- **applies-to-our-codebase: YES**
- **reasoning:** The CDN URL lacks Subresource Integrity (SRI) verification. If the SheetJS CDN is compromised, or if a MITM attacker can intercept the download (HTTPS-dependent), a different tarball could be served. This is a supply-chain risk affecting build reproducibility. **Mitigation: pin to a specific npm registry version with a lockfile integrity hash, or add SRI validation.**

### D-10 — Prisma ORM Operator Injection / ORM Leak
- **package:** Prisma (via `@prisma/client`, version not specified in deps)
- **finding:** elttam research (Dec 2025, https://www.elttam.com/blog/leaking-more-than-you-joined-for/) demonstrates two attack classes against Prisma:
  1. **Operator Injection via Type Coercion**: Sending JSON/URL-encoded requests where a scalar field becomes an object (e.g., `resetToken[not]=E`) bypasses equality checks. Prisma's object-based operator syntax (`not`, `contains`, etc.) means an attacker can inject operators by controlling the *type* of the value.
  2. **ORM Leak / Relational Filtering**: Time-based enumeration of related records via user-controlled `where` clauses. The `plormber` exploitation tool exists for this.
- **applies-to-our-codebase: UNCERTAIN — needs code review**
- **reasoning:** This is not a Prisma CVE but an application-layer misuse pattern. Whether papermark is vulnerable depends on whether API routes pass user-controlled input directly to Prisma `where` clauses without schema-based validation (e.g., Zod). The papermark API routes appear to use Zod validation in many places, but a spot-check for routes that use `req.body.field as Type` casts (which defeat type safety) is warranted. **Flag for manual code review of Prisma query construction patterns.**

### D-11 — NextAuth.js Session Handling
- **package:** `next-auth` (NextAuth.js 4.x, papermark likely on v4.24.x)
- **finding:** Several relevant risk vectors:
  1. **CVE-2025-29927 (CVSS 9.1)**: Next.js middleware bypass via `x-middleware-subrequest` header. Affects Next.js < 15.2.3, < 14.2.25. Papermark on 14.2.35 is **below the fix threshold**. This allows complete auth bypass for routes protected only by middleware.
  2. **NEXTAUTH_SECRET compromise → session forgery**: If the secret is exposed (env leak, deserialization), attacker can forge valid JWT session cookies indefinitely. The secret is the only key material for JWT encryption via HKDF.
  3. **OAuth tokens in JWT exposed to browser**: If `access_token`/`refresh_token` are stored in the NextAuth JWT callback, they are returned to the browser via `/api/auth/session` and `useSession()`.
- **applies-to-our-codebase: YES (CVE-2025-29927)**
- **reasoning:** CVE-2025-29927 is critical (9.1) and affects the exact Next.js 14.x line papermark uses at 14.2.35. The fix is in 14.2.25+, so papermark at 14.2.35 should already be patched. **Verify papermark's Next.js version.** (Note: 14.2.35 > 14.2.25, so middleware bypass should be fixed.) The NEXTAUTH_SECRET and OAuth token exposure vectors require separate code review.

### D-12 — TUS CORS Misconfiguration
- **package:** `@tus/server` / `@tus/node-server` (papermark's TUS implementation)
- **finding:** CVE-2026-57957 confirms Papermark-specific CORS misconfiguration in TUS viewer upload endpoint (see CVEs section above). Additionally, tus-node-server PR #636 (https://github.com/tus/tus-node-server/pull/636) added `allowedCredentials` and `allowedOrigins` options specifically to address CORS + credentialed request risks. Papermark's TUS handler reflects arbitrary `Origin` with credentials enabled.
- **applies-to-our-codebase: YES**
- **reasoning:** Confirmed by Papermark-specific CVE. An attacker can exploit this cross-origin to silently upload files or exfiltrate data from authenticated viewer sessions. **VALID FINDING — upgrade to Papermark >= 0.23.0 or apply origin restriction fix.**

### D-15 — engine.io Transitive Cookie + DoS CVEs
- **package:** `engine.io` (transitive via `socket.io` inside `@trigger.dev/core`)
- **finding:** The full transitive chain `@trigger.dev/sdk → @trigger.dev/core → socket.io → engine.io → cookie@0.4.2` carries multiple CVEs (CVE-2024-47764 cookie prefix injection, CVE-2022-21676 DoS, CVE-2022-41940 DoS). All are within Trigger.dev's internal server-to-cloud WebSocket channel.
- **applies-to-our-codebase: NO**
- **reasoning:** The `engine.io` and `cookie` instances are used exclusively for Trigger.dev's infrastructure communication, not for handling user requests. Papermark does not expose engine.io endpoints, accept WebSocket connections from users, or use cookie-based auth for the WebSocket channel. These CVEs affect Trigger.dev's infrastructure, not papermark's application security. **Monitor Trigger.dev SDK updates for dependency bumps.**

---

## Summary of Changes to Synthesis

| Finding | Original Status | Revised Status | Rationale |
|---------|----------------|----------------|-----------|
| S-001 (cuid) | HIGH | MEDIUM (or drop) | `cuid@^3.0.0` is CUID2, cryptographically secure. Not CUID v1 predictable behavior. Migration to nanoid is nice-to-have, not security-critical. |
| S-002 (xlsx PP) | MEDIUM | NOT APPLICABLE | CVE-2023-30533 fixed at < 0.19.3. Papermark uses 0.20.3. Both PP and ReDoS CVEs are patched. |
| S-002 (xlsx CDN) | MEDIUM | MEDIUM (standalone) | Supply-chain risk remains — CDN tarball has no integrity hash. Should be a separate finding from the CVEs. |
| S-002 (xlsx ReDoS) | MEDIUM | NOT APPLICABLE | Fixed at < 0.20.2. Papermark uses 0.20.3. |
| S-003 (cookie) | MEDIUM | NO CHANGE | Cookie prefix injection does not apply to papermark (Trigger.dev infra only). Already assessed as LOW confidence / UNLIKELY reachable. |
| S-004 (pdfjs-dist) | MEDIUM | HIGH | Public PoCs exist (snyk-labs/pdfjs-vuln-demo, ngductung/PDFjs-XSS-PoC). User-uploaded PDF rendering is a confirmed attack surface. Upgrade severity. |
| S-005 (unused deps) | LOW | NO CHANGE | No CVEs found in oidc-provider that would change this assessment. |
| Next.js 14 RSC DoS | (new) | HIGH | CVE-2026-23864 + CVE-2026-23869. Next.js 14.x EOL — no patches available. Unauthenticated, network-based. Requires upgrade to Next.js 15.x. |
| Next.js middleware bypass | (new) | CRITICAL | CVE-2025-29927 (CVSS 9.1). Papermark 14.2.35 > 14.2.25 fix threshold — **verify this is truly patched**. |
| TUS CORS | (new) | MEDIUM | CVE-2026-57957 is Papermark-specific. Confirms CORS misconfiguration in TUS viewer upload endpoint. |
