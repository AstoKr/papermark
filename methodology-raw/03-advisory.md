# Advisory & Patch Pairing — papermark

**Node:** Layer-3 Advisory Pairing  
**Date:** 2026-06-30  
**Consumed by:** `chain-detection` loop

---

## Per-CVE Advisory Details

### CVE-2024-4367 / GHSA-wgrm-67xf-hhpq — pdfjs-dist Arbitrary JS Execution

- **package:** `pdfjs-dist` 3.11.174 (transitive via `react-pdf` 8.0.2)
- **vulnerable range:** <= 4.1.392
- **advisory URL:** https://github.com/advisories/GHSA-wgrm-67xf-hhpq
- **patch commit:** https://github.com/mozilla/pdf.js/commit/85e64b5c16c9aaef738f421733c12911a441cec6
- **patch description:** Removes all `eval()`/`new Function()` usage for font glyph rendering. Replaces string-based command objects with integer opcodes (`FontRenderOps`) dispatched through safe closure functions. Also removes `isEvalSupported` option entirely.
- **fix applied:** NOT_APPLIED
- **patch completeness:** COMPLETE (in 4.2.67)
- **fix mechanism required:** VERSION_BUMP — need pdfjs-dist >= 4.2.67 or override `isEvalSupported: false` as a workaround.
- **similar patterns in app code:** NO — `eval()` and `new Function()` are not used anywhere in app source code.
- **description:** Papermark ships pdfjs-dist 3.11.174 via react-pdf. The PDF viewer at `components/view/viewer/pdf-default-viewer.tsx` renders user-uploaded PDFs using `<Document>` from react-pdf. The component does NOT set `isEvalSupported: false` (defaults to `true`). react-pdf 8.0.2 pins `pdfjs-dist: "3.11.174"` in its own dependencies — it does not inherit papermark's pdfjs-dist version. An attacker who uploads a crafted PDF can achieve arbitrary JS execution in the browser of any user who views the document. CVSS 8.8, EPSS 72.6% (99th percentile).

### CVE-2023-30533 / GHSA-4r6h-8v6p-xvw6 — xlsx Prototype Pollution

- **package:** `xlsx` 0.20.3 (direct dependency)
- **vulnerable range:** < 0.19.3
- **advisory URL:** https://github.com/advisories/GHSA-4r6h-8v6p-xvw6
- **patch commit:** No single commit hash published. Fixed at version 0.19.3 tag.
- **fix applied:** VERSION_BUMP (papermark uses 0.20.3 > 0.19.3)
- **patch completeness:** COMPLETE — version 0.20.3 is above the 0.19.3 fix threshold
- **similar patterns in app code:** NO — no prototype pollution patterns found. `eval`/`new Function` not used. Spread operators on parsed user data were checked — no occurrences.
- **description:** Crafted spreadsheet data could pollute `Object.prototype` during parsing. Fixed in 0.19.3. Papermark's version 0.20.3 is patched. The remaining risk is the supply-chain concern (CDN tarball without integrity hash), not this specific CVE.

### CVE-2024-22363 / GHSA-5pgg-2g8v-p4x9 — xlsx ReDoS

- **package:** `xlsx` 0.20.3 (direct dependency)
- **vulnerable range:** < 0.20.2
- **advisory URL:** https://github.com/advisories/GHSA-5pgg-2g8v-p4x9
- **patch commit:** No single commit hash published. Fixed at version 0.20.2 tag.
- **patch change:** Removed `m` flag from `tagregex1` regex to prevent catastrophic backtracking on crafted XML.
- **fix applied:** VERSION_BUMP (papermark uses 0.20.3 > 0.20.2)
- **patch completeness:** COMPLETE — version 0.20.3 is above the 0.20.2 fix threshold
- **similar patterns in app code:** YES — `ee/features/workflows/lib/engine.ts:307` uses `new RegExp(condition.value as string).test(normalizedEmail)` where `condition.value` is a user-configurable workflow condition. Wrapped in try/catch (mitigated). No `/mg` flag usage found. The xlsx ReDoS used exponential backtracking via multiline flag — the workflow engine regex is operator-controlled but single-line. **Risk LOW** due to try/catch and admin-only workflow configuration.
- **description:** Regular expression in sheet name parsing (`tagregex1` with `/mg` flags) caused exponential backtracking on crafted XML. Fixed in 0.20.2. Papermark uses 0.20.3 which includes the fix.

### CVE-2024-47764 / GHSA-pxg6-pf52-xh8x — cookie Prefix Injection

- **package:** `cookie` 0.4.2 (transitive via `engine.io` → `socket.io` → `@trigger.dev/core`)
- **vulnerable range:** < 0.7.0
- **advisory URL:** https://github.com/advisories/GHSA-pxg6-pf52-xh8x
- **patch commit:** https://github.com/jshttp/cookie/commit/e100428 (PR #167)
- **patch description:** Updates validation for `name`, `path`, and `domain` fields to reject `__Host-`/`__Secure-` prefixed cookies when attributes are missing or incorrect.
- **fix applied:** NOT_APPLIED — transitive dependency not bumped
- **patch completeness:** COMPLETE (in 0.7.0)
- **fix mechanism required:** TRANSITIVE_BUMP — needs `engine.io` >= 6.6.0 (uses `cookie@~0.7.2`) or `@trigger.dev/sdk` update that bumps socket.io/engine.io.
- **similar patterns in app code:** NO — papermark does not use the `cookie` package directly. Session cookies are managed by `next-auth` via `next/headers`. No cookie serialization using the `cookie` npm package in app code.
- **description:** The `cookie` package before 0.7.0 allows prefix injection via crafted cookie names. Papermark has `cookie@0.4.2` as a deep transitive via Trigger.dev's WebSocket channel. This channel is server-to-cloud infrastructure, not user-facing. Papermark uses `next-auth` for its own HTTP cookie handling. **Not reachable by papermark attackers.** Monitor Trigger.dev SDK for dependency bumps.

### CVE-2026-23864 — Next.js / React RSC BigInt Deserialization DoS

- **package:** `react-server-dom-webpack` (via Next.js 14.2.35)
- **vulnerable range:** React 19.0.0–19.0.3 (in Next.js 14.x, unfixed)
- **advisory URL:** https://github.com/facebook/react/security/advisories/GHSA-479c-33wc-g2pg (related)
- **patch commit:** React 19.0.4, 19.1.5, 19.2.4 releases. Next.js 14.x EOL — no backports available.
- **fix applied:** NOT_APPLIED — Next.js 14.x is EOL, no patch exists within the 14.x line
- **patch completeness:** COMPLETE (in React 19.0.4+) but NOT AVAILABLE for Next.js 14.x
- **fix mechanism required:** ARCHITECTURAL — requires upgrading to Next.js 15.x+
- **similar patterns in app code:** NO — BigInt deserialization occurs in the React Flight protocol (framework-level), not in application code. App code does not use `BigInt` with user input.
- **description:** Unauthenticated remote attacker sends crafted HTTP request with excessively large BigInt values in Flight protocol deserialization. CVSS 7.5. Affects all Next.js 14.x App Router endpoints. Papermark on 14.2.35 (14.x) is vulnerable. Next.js 14.x is EOL and will not receive patches. **Requires architectural upgrade to Next.js 15+.**

### CVE-2026-23869 ("React2DoS") — React RSC Cyclic Deserialization DoS

- **package:** `react-server-dom-webpack` (via Next.js 14.2.35)
- **vulnerable range:** React 19.0.0–19.0.4, 19.1.0–19.1.5, 19.2.0–19.2.4
- **advisory URL:** https://github.com/facebook/react/security/advisories/GHSA-479c-33wc-g2pg
- **patch version:** Fixed in React 19.0.5, 19.1.6, 19.2.5 (backported). Not available for Next.js 14.x (EOL).
- **fix applied:** NOT_APPLIED — same as CVE-2026-23864
- **patch completeness:** COMPLETE (in React 19.0.5+) but NOT AVAILABLE for Next.js 14.x
- **fix mechanism required:** ARCHITECTURAL — requires upgrading to Next.js 15.x+
- **similar patterns in app code:** NO — cyclic deserialization is a framework-level issue in the React Flight protocol.
- **description:** Crafted cyclic Map/Set references in Flight protocol cause quadratic-time deserialization. CVSS 7.5. Chains with CVE-2026-23864 for compounded DoS impact. **Requires architectural upgrade to Next.js 15+.**

### CVE-2025-29927 — Next.js Middleware Authorization Bypass

- **package:** `next` 14.2.35 (direct dependency)
- **vulnerable range:** >= 14.0.0, < 14.2.25
- **advisory URL:** https://github.com/vercel/next.js/security/advisories/GHSA-f82v-jwr5-mffw
- **patch commit:** https://github.com/vercel/next.js/commit/52a078da3884efe6501613c7834a3d02a91676d2
- **patch description:** Introduces a per-server-session random 8-byte identifier (`x-middleware-subrequest-id`). Middleware sandbox attaches this internal ID to outgoing requests. Incoming requests with a forged `x-middleware-subrequest` header are stripped if the session ID doesn't match the server's internal value. This prevents external spoofing of the middleware subrequest header.
- **fix applied:** VERSION_BUMP (papermark uses 14.2.35 >= 14.2.25)
- **patch completeness:** COMPLETE — 14.2.35 includes the fix. Verified papermark's Next.js version is 14.2.35.
- **similar patterns in app code:** NO — authorization bypass via header injection is a framework-level issue. Papermark's own middleware/auth patterns use NextAuth session validation, not header-based bypass guards.
- **description:** Attackers can bypass authorization checks in Next.js middleware by sending a crafted `x-middleware-subrequest` header. CVSS 9.1. **Papermark is NOT vulnerable** — version 14.2.35 already includes the fix from 14.2.25.

### CVE-2026-57957 — Papermark TUS CORS Misconfiguration

- **package:** Papermark application code (not a library CVE)
- **vulnerable range:** Papermark <= 0.22.0
- **advisory URL:** https://cvefeed.io/vuln/detail/CVE-2026-57957
- **patch commit:** Not publicly identified (likely fixed in Papermark >= 0.23.0)
- **fix applied:** NOT_APPLIED — current working tree contains the vulnerable configuration
- **patch completeness:** UNKNOWN (fix not assessed)
- **fix mechanism required:** CODE_PATCH — restrict `Access-Control-Allow-Origin` to specific allowed origins, or disable `Access-Control-Allow-Credentials` when reflecting arbitrary origins.
- **similar patterns in app code:** YES — `pages/api/file/tus-viewer/[[...file]].ts:234-250` contains the vulnerable CORS configuration. The regular TUS upload endpoint (`pages/api/file/tus/[[...file]].ts`) does NOT set any CORS headers (safe default). No other API handler reflects the Origin header.
- **description:** The TUS viewer upload endpoint reflects arbitrary `Origin` headers with `Access-Control-Allow-Credentials: true`. An unauthenticated attacker can create a malicious page that, when visited by an authenticated Papermark viewer, silently uploads files to datarooms via cross-origin requests. The two TUS endpoints have asymmetric CORS treatment — the viewer endpoint is permissive while the authenticated team upload endpoint is restrictive. This inconsistency itself is a finding.

### D-05 / S-002 (Supply Chain) — xlsx CDN Tarball Without Integrity Hash

- **package:** `xlsx` 0.20.3 (CDN URL in package.json)
- **vulnerable range:** N/A (supply chain, not a version issue)
- **advisory URL:** N/A — finding from structural analysis
- **patch commit:** N/A
- **fix applied:** NOT_APPLIED
- **patch completeness:** N/A
- **fix mechanism required:** CODE_PATCH — pin to npm registry version with lockfile integrity hash, or add SRI verification for CDN tarball.
- **similar patterns in app code:** N/A — not a code pattern vulnerability
- **description:** `package.json` line 175 pins `xlsx` to `https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz` with no integrity hash. If the CDN is compromised or a MITM attack succeeds during install, a different tarball could be substituted. The Prototype Pollution and ReDoS CVEs are patched in 0.20.3, but the distribution mechanism has no tamper protection.

---

## App-Code Pattern Hunt Results

### ReDoS patterns found in application code

| Location | Pattern | User-Controlled? | Mitigated? |
|----------|---------|-----------------|------------|
| `ee/features/workflows/lib/engine.ts:307` | `new RegExp(condition.value as string).test(normalizedEmail)` | Yes — workflow condition value is user-configurable | Partial — wrapped in try/catch, but ReDoS could still cause CPU exhaustion before caught |
| `lib/utils.ts:904` | `new RegExp(\`{{\\s*${key}\\s*}}\`, "gi")` | No — `key` is from whitelist (`allowedVariables`) | Yes — keys are hardcoded and escaped |
| `lib/edge-config/blacklist.ts:20` | `new RegExp(blacklistedEmails.join("|"), "i")` | No — blacklist from Edge Config (server-controlled) | Yes — server-controlled input |
| `lib/utils/watermark-pdf.ts:159` | `new RegExp(original, "g")` | No — `original` is from hardcoded replacements map | Yes — static replacement pairs |

### Prototype pollution patterns

No instances found. Application code does not use `eval()`, `new Function()`, or unsafe object merge patterns with user input.

### CORS misconfiguration patterns

Only one instance found: `pages/api/file/tus-viewer/[[...file]].ts:237`. The regular TUS endpoint has no CORS headers (safe default). All other API handlers use same-origin defaults.

### Cookie serialization patterns

No instances found. Papermark uses `next-auth` session cookies via `next/headers`, not the `cookie` npm package.

### Prisma ORM injection patterns (D-10)

No instances found where user-controlled body values flow directly into Prisma `where` clauses. All queries use URL path parameters (`teamId` from `req.query`) with session-based authorization. Structured compound unique keys prevent operator injection.
