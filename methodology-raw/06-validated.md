# Validated Findings — Papermark Security Review

**Validator:** Layer-5 Reconciliation Loop (Judge A + Judge B merged)
**Date:** 2026-06-30
**Method:** Independent read of both judge verdicts + source code verification at resolution points. Every disagreement re-examined by reading the code.

---

## Reconciliation Summary

| Category | Count | Notes |
|----------|-------|-------|
| VALID | 25 findings | Across CRITICAL/HIGH/MEDIUM/LOW |
| REJECTED | 4 findings | False positive or not actionable |
| MERGED | 2 pairs | CUID findings merged; CORS+TUS config merged |
| Confirmed chains | 11 chains | 10 VALID, 1 REJECTED |

### Key Judge Disagreements Resolved

| Disagreement | Code Truth | Resolution |
|---|---|---|
| S-011: "cuid@3.0.0 is CUID2" (Judge A) vs "CUIDs encode timestamps" (Judge B) | `package-lock.json` confirms `cuid@3.0.0` (original CUID, deprecated, NOT CUID2). Deprecation message: "insecure. Use @paralleldrive/cuid2 instead." | **Judge B correct** — VALID at MEDIUM. Judge A was wrong about CUID2. |
| S-012: "Partially mitigated" (Judge A) vs "REJECTED" (Judge B) | Python 3 `xml.etree.ElementTree.parse()` does NOT resolve external entities by default. Billion-laughs DoS still works. | **Both partially correct** — LOW defense-in-depth. XXE file read/SSRF not exploitable in modern Python 3, but billion-laughs DoS is. |
| S-017: "VALID marginal" (Judge A) vs "REJECTED false positive" (Judge B) | JS `console.log` accepts format specifiers but cannot be weaponized for memory read, code execution, or meaningful log forgery in Node.js. | **Judge B correct** — FALSE POSITIVE. Rejected. |
| CVE-2026-44575 in Chain 5: version range | Methodology confirms `>= 15.2.0 < 15.5.16, >= 16.0.0 < 16.2.5`. Papermark on 14.2.35 — NOT affected. | **Both judges correct** — Chain reframed. The actual bypass is `middleware.ts:49` excluding `/api/*`, not a CVE. |

---

## Part 1: VALID Findings (F-001 through F-025)

### F-001 — Conversations API: Zero Authentication on All Handlers

- **source ID:** S-001 (structural)
- **severity:** CRITICAL
- **file:** `ee/features/conversations/api/conversations-route.ts` (all 4 handlers) + `pages/api/conversations/[[...conversations]].ts`
- **line:** route.ts:1-301 (entire file, zero next-auth imports)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **methodology found:** Structural analysis (AI-missed, semgrep-missed)
- **description:** Four handlers (GET list, POST create, POST messages, POST notifications) all accept `viewerId` from client with zero server-side verification. Zero `next-auth` imports. Adjacent `team-conversations-route.ts` demonstrates correct `getServerSession()` pattern — making this clearly a missed implementation pass. Middleware explicitly excludes `/api/*`, so no defense-in-depth layer applies.
- **impact:** Unauthenticated conversation enumeration, data injection into PostgreSQL, email notification trigger via Trigger.dev. Trivially exploitable from the internet.
- **CVSS estimate:** 9.1 (CRITICAL) — network-based, low complexity, no auth, confidentiality + integrity impact

---

### F-002 — Stored XSS via `sanitizePlainText` Entity-Decode Ordering + CVE-2026-44990 `<xmp>` Bypass

- **source ID:** S-002 (structural/SAST)
- **severity:** HIGH
- **files:**
  - `lib/utils/sanitize-html.ts:12-19` (decode-after-strip bug)
  - `components/documents/document-header.tsx:604` (dangerouslySetInnerHTML sink)
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts:15` (Zod transform entry point)
  - `lib/zod/url-validation.ts:207` (document upload schema)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** `sanitizeHtml(content)` called first (strips literal HTML tags), then `decodeHTML(sanitized)` after (decodes entities). Encoded entities like `&lt;img src=x onerror=alert(1)&gt;` survive tag stripping because they aren't literal tags yet, then become real HTML tags after decode. The decoded payload is stored in PostgreSQL `Document.name` and rendered via `dangerouslySetInnerHTML`. Additionally, `sanitize-html@2.17.3` is vulnerable to CVE-2026-44990 (CVSS 9.3): the `<xmp>` raw-text element bypass allows injection even without entity encoding. At least 5 authenticated write paths exist.
- **impact:** Any team member can execute arbitrary JS in the browser of every other team member viewing the document. Session token theft, API key exfiltration, worm propagation.
- **CVSS estimate:** 8.7 (HIGH) — network-based, low complexity, requires authenticated rename, HIGH confidentiality + integrity impact
- **confidence:** HIGH

---

### F-003 — `progress-token.ts`: Unauthenticated Trigger.dev Token Generation

- **source ID:** S-003 (structural)
- **severity:** HIGH
- **file:** `pages/api/progress-token.ts:4-27`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Generates a 15-minute Trigger.dev public access token for any `documentVersionId` without any auth or rate limiting. The same `generateTriggerPublicAccessToken` function is called from three authenticated EE routes — this endpoint is the only unguarded path. CUID leakage from client-side JS, referrer headers, or logs is the only barrier.
- **impact:** Unauthenticated Trigger.dev token generation exposing document processing metadata and workflow run data.
- **CVSS estimate:** 7.5 (HIGH) — network-based, low complexity, no auth
- **confidence:** HIGH

---

### F-004 — Next.js v14 Unpatchable DoS (CVE-2026-23864 / CVE-2026-23869)

- **source ID:** S-004 (structural/web-intel)
- **severity:** HIGH
- **type:** Framework CVE (no application patch available)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Next.js 14.2.35 is EOL (since October 2025). Two RSC Flight protocol CVEs (memory exhaustion, CPU exhaustion via cyclic deserialization) are confirmed and unpatchable — fixes only in Next.js 15.5.15+. Any App Router page accepts POST with RSC headers. No `"use server"` directives found (limiting exploitation surface) but RSC endpoint is always active.
- **impact:** Unauthenticated DoS against the application. Single crafted request exhausts server memory or pegs CPU for ~1 minute.
- **CVSS estimate:** 7.5 (HIGH) — network-based, low complexity, no auth, availability impact only
- **confidence:** HIGH

---

### F-005 — CORS Origin Reflection on TUS Viewer Upload Endpoint

- **source ID:** S-005 (structural/config) — MERGED with config CORS finding
- **severity:** MEDIUM
- **file:** `pages/api/file/tus-viewer/[[...file]].ts:236-237`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **CVE:** CVE-2026-57957 (papermark-specific, filed)
- **description:** `Access-Control-Allow-Origin` reflects `req.headers.origin` verbatim with `Access-Control-Allow-Credentials: true`. Textbook reflected-origin-with-credentials anti-pattern. Regular TUS endpoint has no CORS headers (safe default), making this asymmetric. Uploads still require valid dataroom session via `verifyDataroomSessionInPagesRouter`.
- **impact:** Cross-origin credentialed uploads to a victim's dataroom if victim has active session. Requires victim visit attacker's page.
- **CVSS estimate:** 4.7 (MEDIUM) — per CVE-2026-57957
- **confidence:** HIGH

---

### F-006 — `record_reaction.ts`: Unauthenticated DB Write + Viewer Data Leak

- **source ID:** S-006 (structural)
- **severity:** MEDIUM
- **file:** `pages/api/record_reaction.ts:17-42`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Creates `Reaction` record in PostgreSQL with `include: { view }` that leaks `viewerEmail`, `documentId`, `dataroomId`, `linkId`, `viewerId`, `teamId` in response. Unlike `record_view.ts`/`record_click.ts` (Tinybird pipeline), this writes directly to PostgreSQL.
- **impact:** Unauthenticated viewer metadata (email) extraction, data injection into PostgreSQL, analytics pollution.
- **CVSS estimate:** 5.3 (MEDIUM) — network-based, low complexity, no auth, confidentiality impact
- **confidence:** HIGH

---

### F-007 — `feedback/index.ts`: Unauthenticated Feedback Response Recording

- **source ID:** S-007 (structural)
- **severity:** MEDIUM
- **file:** `pages/api/feedback/index.ts:9-56`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Records feedback responses with no auth. Validates `feedbackId` and `viewId`+`linkId` pair exist but doesn't verify caller ownership. `answer` field stored unsanitized in JSON.
- **impact:** Unauthenticated data injection into PostgreSQL feedback table.
- **CVSS estimate:** 5.3 (MEDIUM) — network-based, low complexity, no auth, integrity impact
- **confidence:** HIGH

---

### F-008 — `revalidate.ts`: Shared Secret in URL Query Parameter

- **source ID:** S-008 (structural)
- **severity:** MEDIUM
- **file:** `pages/api/revalidate.ts:13-14`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **CWE:** CWE-598 (Information Exposure Through Query Strings in GET Request)
- **description:** `req.query.secret !== process.env.REVALIDATE_TOKEN` exposes `REVALIDATE_TOKEN` through server access logs, Referer headers, browser history. Leaked token enables mass ISR cache invalidation and link enumeration (endpoint can revalidate ALL links for a team).
- **impact:** Cache invalidation DoS, link enumeration. Secret leakage via standard web mechanisms.
- **CVSS estimate:** 5.3 (MEDIUM) — network-based, requires secret leakage
- **confidence:** HIGH

---

### F-009 — `dangerouslySetInnerHTML` on `helpText` Props — Latent XSS Sinks

- **source ID:** S-009 (structural/SAST)
- **severity:** LOW (downgraded from MEDIUM in synthesis)
- **files:**
  - `components/ui/form.tsx:145`
  - `components/account/upload-avatar.tsx:96`
- **judge agreement:** ⚠️ BOTH agree severity should be DOWNGRADED from original MEDIUM. Judge A → LOW; Judge B → LOW
- **verification:** All callers of `Form` with `helpText` prop pass hardcoded string literals. No confirmed path for user-controlled data to reach this sink.
- **description:** Two shared components render `helpText` via `dangerouslySetInnerHTML` with no local sanitization. If any future code change feeds user data through `helpText`, XSS results. Latent quality finding, not active vulnerability.
- **confidence:** MEDIUM

---

### F-010 — Zod Validation Error Information Disclosure

- **source ID:** S-010 (SAST)
- **severity:** MEDIUM
- **files:** 9 endpoints including:
  - `pages/api/webhooks/services/[...path]/index.ts:236` (`validationResult.error.format()`)
  - `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:67`
  - 7 additional endpoints (custom semgrep rule `papermark-zod-error-format-disclosure`)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Nine endpoints return detailed Zod validation error messages (`error.format()` or `error.errors`) in HTTP 400 responses. Leaks schema structure, field names, types, enum values, regex constraints.
- **impact:** Enables targeted fuzzing and API recon. Facilitates exploitation of other endpoints by revealing expected input shapes.
- **CVSS estimate:** 4.3 (MEDIUM) — network-based, low complexity, no auth, low confidentiality impact
- **confidence:** HIGH

---

### F-011a — NEXTAUTH_SECRET Default Value (self-hosted)

- **source ID:** S-001 (Secrets)
- **severity:** CRITICAL (self-hosted deployments only)
- **file:** `.env.example:1`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** `NEXTAUTH_SECRET=my-superstrong-secret` is a publicly visible default. Used across 5 security domains: JWT signing, SAML clientSecret, Jackson encryption key, HMAC download/access tokens, API token hashing. Published exploitation tooling (Embrace The Red cookie forging) exists. Self-hosted deployments that follow `.env.example` verbatim are critically vulnerable.
- **impact:** Complete cryptographic trust collapse on vulnerable deployments. Session impersonation, SAML response forgery, document access bypass, API credential recovery.
- **CVSS estimate:** 10.0 (CRITICAL) — network-based, no auth, published exploit tooling, complete compromise
- **confidence:** HIGH
- **scope:** Self-hosted only. Vercel deployments using system env vars are NOT affected.

---

### F-011b — NEXTAUTH_SECRET Reused Across 5 Security Domains

- **source ID:** Config C-008
- **severity:** MEDIUM (amplifying factor for F-011a)
- **files:**
  - `lib/auth/auth-options.ts:128,149` (SAML clientSecret)
  - `lib/jackson.ts:16-20` (Jackson encryption key)
  - `lib/signing/download-token.ts:19` (HMAC download tokens)
  - `lib/signing/access-token.ts:21` (HMAC access tokens)
  - `lib/api/auth/token.ts:12` (API token hash)
- **judge agreement:** ⚠️ BOTH agree finding is valid. Judge A: MEDIUM (+HIGH with default); Judge B: LOW
- **resolution:** MEDIUM — Not a standalone vulnerability (primary protection is NEXTAUTH_SECRET confidentiality), but single compromise undermines all 5 mechanisms simultaneously. Amplifies F-011a.
- **description:** The same root secret used for JWT signing, SAML OAuth client secrets, Jackson encryption, HMAC token signing, and API token hashing. A single secret compromise (leak, default value, server breach) breaks all 5 mechanisms.
- **confidence:** HIGH

---

### F-011c — SAML clientSecret = NEXTAUTH_SECRET → SAML Response Forgery

- **source ID:** Config C-008 subset (SAML-specific)
- **severity:** HIGH (self-hosted, with F-011a)
- **file:** `lib/auth/auth-options.ts:128,149`
- **judge agreement:** ⚠️ BOTH. Judge A: MEDIUM → HIGH with default; Judge B: HIGH (self-hosted)
- **resolution:** HIGH — distinct from F-011a (JWT cookie forging). SAML response forgery provides alternative attack path when JWT forging doesn't work.
- **description:** NEXTAUTH_SECRET used as both SAML OAuth clientSecret and SAML IdP client_secret. Combined with `allowDangerousEmailAccountLinking: true` on SAML provider, knowing the secret enables forging SAML assertions that auto-link to victim accounts.
- **confidence:** MEDIUM (SAML assertion acceptance depends on IdP configuration)

---

### F-012a — NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY Default Value

- **source ID:** S-002 (Secrets)
- **severity:** HIGH
- **file:** `.env.example:68` → `lib/utils.ts:614-628`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Default `my-superstrong-document-secret` used for AES-256-CTR key derivation. Key derivation is unsalted SHA-256 (no KDF, no salt, no key stretching). If default unchanged, all encrypted document passwords in the database are decryptable.
- **impact:** Decryption of all document-level passwords on vulnerable self-hosted deployments.
- **CVSS estimate:** 7.5 (HIGH) — network-based, requires DB access or is self-hosted
- **confidence:** HIGH

---

### F-012b — XXE in `docx-sanitizer.py`: Native XML Parsing Without defusedxml

- **source ID:** S-012 (SAST/semgrep)
- **severity:** LOW (downgraded from MEDIUM in synthesis)
- **file:** `ee/features/conversions/python/docx-sanitizer.py:146`
- **judge agreement:** ⚠️ BOTH agree original MEDIUM was overstated. Judge A: LOW (partially mitigated); Judge B: LOW (FP as actionable XXE)
- **verification:** Python 3 `xml.etree.ElementTree.parse()` does NOT resolve external entities by default. `defusedxml` is not used. XXE file read/SSRF is NOT exploitable in modern Python 3 without explicit parser configuration. BILLION LAUGHS DoS is still possible.
- **description:** `ET.parse(path)` at line 146 parses XML from user-uploaded DOCX files. While external entity resolution is disabled by default in Python 3, billion-laughs/quadratic-blowup DoS attacks still work. Defense-in-depth finding — should use `defusedxml`.
- **confidence:** HIGH (that modern Python 3 is not vulnerable to XXE via ET.parse; MEDIUM on DoS)

---

### F-013a — ReDoS in Workflow Engine (Non-Literal RegExp)

- **source ID:** S-013 (SAST/semgrep)
- **severity:** LOW
- **file:** `ee/features/workflows/lib/engine.ts:307`
- **judge agreement:** ✅ BOTH — Judge A: MEDIUM confidence; Judge B: MEDIUM confidence
- **description:** `new RegExp(condition.value as string)` where `condition.value` is user-configurable. Exponential regex patterns can block the event loop. Wrapped in try/catch which catches AFTER catastrophic backtracking completes — performance impact still occurs. Requires authenticated admin access.
- **impact:** Limited — requires admin workflow config. Bounded by per-request timeout.
- **confidence:** MEDIUM

---

### F-013b — `cuid@3.0.0` Predictable IDs in Permission Groups

- **source ID:** S-011 (structural) + S-001 (Deps) — MERGED
- **severity:** MEDIUM
- **files:**
  - `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`
- **judge disagreement:** Judge A: REJECTED (claimed CUID2 — WRONG). Judge B: VALID at MEDIUM (correct).
- **verification:** `package-lock.json` confirms **`cuid@3.0.0`** (original CUID, deprecated, NOT CUID2). Package deprecation message explicitly warns: "Cuid and other k-sortable and non-cryptographic ids ... are all insecure. Use @paralleldrive/cuid2 instead." CUIDs encode timestamps + sequential counter.
- **description:** CUIDs encode timestamps with sequential counters — predictable. All three handlers are behind NextAuth session + team membership auth, limiting remote exploitability. Project already depends on `nanoid` (cryptographic) for other IDs, making migration trivial.
- **impact:** Authenticated team members can enumerate permission group IDs and potentially access unauthorized resources via IDOR.
- **confidence:** HIGH (on version); MEDIUM (on exploitability via auth gate)

---

### F-014a — No Rate Limiting on Authentication Endpoints

- **source ID:** S-014 (SAST/structural)
- **severity:** LOW
- **file:** `pages/api/auth/[...nextauth].ts`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** NextAuth endpoints have no rate limiter imported or used. Confirmed by custom semgrep rule `papermark-nextauth-no-rate-limit`. Defense-in-depth gap.
- **impact:** Enables brute-force attacks against credentials, email enumeration, mass email-sending. Amplifying factor for other findings (Chain 9).
- **confidence:** HIGH

---

### F-014b — `pdfjs-dist` 3.11.174 Arbitrary JavaScript Execution via Malicious PDF

- **source ID:** S-004 (Deps) / D-003
- **severity:** MEDIUM
- **type:** Transitive dependency via `react-pdf`
- **CVE:** CVE-2024-4367 (GHSA-wgrm-67xf-hhpq)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: MEDIUM confidence
- **description:** `pdfjs-dist@3.11.174` via `react-pdf`. `preview-viewer.tsx` renders PDFs with default `isEvalSupported: true`. An attacker who uploads a malicious PDF (requires auth) can achieve arbitrary JS execution in the browser of any user who views it.
- **impact:** XSS in document viewer — session hijacking, data exfiltration. Requires authenticated upload + victim view.
- **confidence:** HIGH

---

### F-014c — Webhook Event Request/Response Body Stored in Tinybird

- **source ID:** S-015 (SAST/structural)
- **severity:** LOW
- **file:** `app/api/webhooks/callback/route.ts:43-44`
- **judge agreement:** ✅ BOTH — Judge A: MEDIUM confidence; Judge B: MEDIUM confidence
- **description:** Webhook bodies decoded from base64 and stored in Tinybird analytics. If third-party webhook responses contain sensitive data, could be exposed to team members viewing webhook logs.
- **impact:** Limited — requires sensitive data in webhook responses + team member access to logs.
- **confidence:** MEDIUM

---

### F-015a — Embed Route `frame-ancestors *` Allows Framing by Any Origin

- **source ID:** Config C-001
- **severity:** MEDIUM (HIGH when combined with F-002 stored XSS)
- **file:** `next.config.mjs:259-271`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** `/view/:path*/embed` sends `frame-ancestors *;` allowing any website to iframe the document embed page. API-layer `isEmbeddableUrl` validates embed domains, but the CSP is the HTML-level protection and it allows all origins.
- **impact:** Enables clickjacking attacks against document viewers. Combined with F-002 stored XSS, enables credential theft via transparent overlay in iframe.
- **confidence:** HIGH

---

### F-015b — Main CSP is Report-Only (Not Enforced)

- **source ID:** Config C-002
- **severity:** MEDIUM (amplifier for XSS findings)
- **file:** `next.config.mjs:219`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** `Content-Security-Policy-Report-Only` means all CSP directives are advisory. Includes `'unsafe-inline'` and `'unsafe-eval'` in `script-src`. XSS payloads face zero CSP blocking on main app pages.
- **impact:** Nullifies CSP as an XSS defense. Combined with F-002, enables undetectable stored XSS worm.
- **confidence:** HIGH

---

### F-015c — CSP Report Endpoint is a No-Op

- **source ID:** Config C-003
- **severity:** MEDIUM (blocks detection)
- **file:** `app/api/csp-report/route.ts:7`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** CSP report handler receives reports but discards them — `console.log` is commented out, no storage, no alerting. Security team has zero visibility into CSP violations including XSS probing.
- **impact:** XSS exploitation generates no security alerts. Mean time to detection extends from hours/days to potentially indefinite.
- **confidence:** HIGH

---

### F-015d — Missing HSTS Header

- **source ID:** Config C-004
- **severity:** MEDIUM
- **file:** `next.config.mjs:190-312`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Zero hits for `Strict-Transport-Security` across the codebase. Vercel platform may cover `*.vercel.app` domains, but custom domains are unprotected. Enables SSL stripping with network position.
- **impact:** Enables query-param secret interception (amplifies F-008). Particularly relevant for custom domains.
- **confidence:** HIGH

---

### F-015e — Missing X-Content-Type-Options Header

- **source ID:** Config C-005
- **severity:** MEDIUM
- **file:** `next.config.mjs:190-312` (absent from global headers)
- **judge agreement:** ✅ Judge B only (Judge A did not include this finding)
- **description:** Only set on a single agreement download endpoint. Absent from global headers. MIME-sniffing risk for uploaded documents.
- **impact:** Uploaded files could be interpreted as HTML/JavaScript by browsers, leading to XSS in document viewer context.
- **confidence:** HIGH

---

### F-015f — Rate Limiter Fails Open When Redis is Unavailable

- **source ID:** Config C-006
- **severity:** MEDIUM
- **file:** `ee/features/security/lib/ratelimit.ts:43-44`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** `checkRateLimit()` returns `{ success: true }` in catch block when Redis throws error. All rate-limited endpoints become unthrottled during Redis outage. Fail-open is intentional (availability vs. security tradeoff) but means rate-limit checks are no-ops during Redis disruption.
- **impact:** Enables brute-force attacks when Redis is down (amplifies F-014a). Used in Chain 9.
- **confidence:** HIGH

---

### F-015g — `allowDangerousEmailAccountLinking: true` on 3 OAuth Providers

- **source ID:** Config C-007
- **severity:** MEDIUM
- **file:** `lib/auth/auth-options.ts:35,55,130` (Google, LinkedIn, SAML)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** `allowDangerousEmailAccountLinking: true` auto-links new OAuth accounts to existing users by matching email without confirmation. An attacker who controls a provider account with the victim's email can gain access. Practical barrier: major providers (Google, LinkedIn) verify email before allowing account creation.
- **impact:** Account takeover when combined with rate-limit bypass (F-015f) or known SAML secret (F-011a).
- **confidence:** HIGH

---

### F-016a — `xlsx` CDN Tarball Without Integrity Hash

- **source ID:** S-002 (Deps) / D-001
- **severity:** LOW
- **file:** `package.json:175`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **verification:** Both CVEs (CVE-2023-30533 prototype pollution, CVE-2024-22363 ReDoS) are PATCHED at version 0.20.3. Fix thresholds: 0.19.3 (PP) and 0.20.2 (ReDoS). Papermark uses 0.20.3.
- **description:** Pinned to CDN URL with no integrity hash. If CDN is compromised or MITM occurs during install, different tarball could be substituted. The two associated CVEs are already patched at this version — the CVE portion is a false positive.
- **impact:** Supply chain risk during installation only. No runtime vulnerability.
- **confidence:** HIGH

---

### F-016b — Unused Direct Dependencies Increasing Attack Surface

- **source ID:** S-005 (Deps) / D-004
- **severity:** LOW
- **files:** `package.json` — `oidc-provider`, `@modelcontextprotocol/sdk`, `tokenlens`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **description:** Three direct dependencies with zero imports in source code. Add ~100+ transitive packages with no benefit. No current exploit path but increases SBOM noise and future CVE surface.
- **impact:** None today. Increases future CVE triage burden.
- **confidence:** HIGH

---

### F-016c — X-Powered-By Header Leaks Application Info on Custom Domains

- **source ID:** Config C-009
- **severity:** LOW
- **file:** `lib/middleware/domain.ts:40-41`
- **judge agreement:** ✅ Judge B only (Judge A did not include)
- **description:** Custom `X-Powered-By: Papermark - Secure Data Room Infrastructure for the modern web` on all rewritten custom-domain responses. Minor information disclosure.
- **impact:** Minimal — enables targeted vulnerability search for identified instances.
- **confidence:** HIGH

---

### F-016d — Trigger.dev Project ID Hardcoded in Config

- **source ID:** S-003 (Secrets)
- **severity:** LOW
- **file:** `trigger.config.ts:7`
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: MEDIUM confidence
- **description:** `project: "proj_plmsfqvqunboixacjjus"` — project identifier, not a credential. Marginal recon value.
- **impact:** No direct exploit. Minor recon value for targeted attacks.
- **confidence:** MEDIUM

---

## Part 2: REJECTED Findings

| ID | Original ID | Reason | Confidence |
|----|------------|--------|------------|
| R-001 | S-016 (domain config dangerouslySetInnerHTML) | Static instruction content only. No dynamic user input. Not a vulnerability — informational observation. | HIGH |
| R-002 | S-017 (unsafe format strings in console.log) | FALSE POSITIVE. JavaScript `console.log` does NOT interpret format specifiers as dangerous injection vectors the way C `printf` does. Cannot be weaponized for memory read, code execution, or meaningful log forgery in Node.js. The semgrep rule `unsafe-formatstring` is a generic pattern that does not account for JavaScript's runtime behavior. | HIGH |
| R-003 | S-003 (Deps) — `cookie@0.4.2` prefix injection | NOT REACHABLE by papermark attackers. Deep transitive via `@trigger.dev/core` → `socket.io` → `engine.io` → `cookie@0.4.2`. This is server-to-cloud infrastructure channel for Trigger.dev, not user-facing. Papermark uses `next-auth` for its own cookie handling. | HIGH |
| R-004 | S-011 / S-001 (Deps) — REJECTED as standalone HIGH | **Recharacterized:** Finding is VALID at MEDIUM (not REJECTED). See F-013b. The HIGH rating from dep synthesis was overstated. The Judge A rejection based on "it's CUID2" was incorrect — the actual dependency IS the original cuid@3.0.0 (predictable). | MEDIUM |

---

## Part 3: MERGED Findings

| Target ID | Original IDs | Merge Rationale |
|-----------|-------------|-----------------|
| F-005 | S-005 + Config CORS finding (00-ai-configs.md) | Both describe the same CORS origin reflection on the TUS viewer endpoint. Config finding is contextual, not additional. |
| F-013b | S-011 (structural) + S-001 (Deps) | Both describe cuid@3.0.0 predictable IDs in permission groups. Dep synthesis added reachability context. Merged as single finding at MEDIUM. |

---

## Part 4: Chain Verdicts

### Chain 1: Stored XSS Worm → Conversations Hijacking → Team-Wide Data Exfiltration

- **status:** VALID
- **findings involved:** F-002 (stored XSS), F-001 (conversations zero-auth), F-006 (record_reaction viewer leak)
- **S-009 (helpText):** REMOVED from chain — non-essential for exploitation path
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **analysis:** All 4 core links confirmed by source reads. Stored XSS in document name → XSS fires in victim browser → payload calls zero-auth conversations API → exfiltrates conversation data and viewer PII. Worm propagation (renaming documents with same payload) is realistic because XSS runs in victim's authenticated session.
- **missing evidence:** None required — all code paths confirmed.
- **severity:** CRITICAL
- **confidence:** HIGH
- **bounty estimate:** $8,000–$15,000

---

### Chain 2: CORS TUS CSRF Upload → Malicious PDF → Admin Session Hijacking

- **status:** VALID (weakened)
- **findings involved:** F-005 (CORS TUS / CVE-2026-57957), F-014b (pdfjs-dist XSS / CVE-2024-4367)
- **judge agreement:** ⚠️ BOTH agree the chain has SameSite=Lax limitations. Judge A: MEDIUM confidence; Judge B: weakened (SameSite barrier). Both agree MEDIUM is more appropriate than the original HIGH.
- **analysis:** CORS misconfiguration is independently validated as CVE-2026-57957. pdfjs-dist XSS is independently validated. However, SameSite=Lax does NOT send cookies on cross-origin POST from `fetch()`/`XHR` — only on top-level navigations. This is a structural barrier to the CSRF upload step, not merely a "mitigation factor." The chain may work in specific browser configurations (top-level redirect scenarios) but is not reliably exploitable from a standard attacker page.
- **severity:** MEDIUM (CORS finding stands independently at MEDIUM; full chain partial exploitability)
- **confidence:** MEDIUM
- **bounty estimate:** $2,000–$5,000 (reduced from $5K–$15K due to practical limitations)

---

### Chain 3: Zod Schema Disclosure → Unauthenticated ID Enumeration → Bulk PII Extraction

- **status:** VALID
- **findings involved:** F-010 (Zod disclosure, 9 endpoints), F-001 (conversations zero-auth), F-006 (record_reaction), F-007 (feedback)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **analysis:** Each link is independently confirmed. No auth bypass required — the endpoints have no auth to bypass. The `viewerEmail` leak through record_reaction's `include: { view }` clause is the most impactful PII vector. ID enumeration step (response-differentiation between valid/invalid IDs) is practical but depends on endpoint behavior.
- **severity:** HIGH
- **confidence:** HIGH
- **bounty estimate:** $3,000–$8,000

---

### Chain 4: XXE via Crafted DOCX → SSRF → Cloud Metadata → Credential Exfiltration

- **status:** REJECTED
- **findings involved:** F-012b (XXE in docx-sanitizer.py)
- **judge agreement:** ✅ BOTH agree this chain is non-exploitable. Judge A: REJECTED (partially mitigated); Judge B: REJECTED (foundational finding unsound)
- **analysis:** Python 3's `xml.etree.ElementTree.parse()` does NOT resolve external entities by default. Without entity resolution, there is no SSRF, no cloud metadata access, no credential exfiltration. The chain collapses entirely. F-012b is a LOW defense-in-depth concern for billion-laughs DoS, not an exploitable SSRF/credential-theft chain.
- **severity:** LOW (defense-in-depth)
- **confidence:** HIGH
- **bounty estimate:** $0 (defense-in-depth documentation)

---

### Chain 5: Framework EOL + Middleware Bypass → Unauthenticated Data Access

- **status:** VALID (reframed)
- **findings involved:** F-004 (Next.js EOL, unpatched CVEs), F-003 (progress-token), F-001 (conversations zero-auth)
- **judge agreement:** ✅ BOTH agree CVE-2026-44575 does NOT affect v14. Chain needs reframing.
- **reframing:** The original chain claimed CVE-2026-44575 (`.rsc` suffix middleware bypass) as the primary bypass vector. However, this CVE affects Next.js >= 15.2.0 — NOT v14.2.35. The actual bypass mechanism is simpler: `middleware.ts:49` explicitly excludes `/api/*` from the middleware matcher. No CVE bypass is needed — middleware simply does not run for any `/api/*` route. The RSC DoS CVEs (F-004) are a separate concern, not a bypass mechanism.
- **severity:** MEDIUM (reframed: structural knowledge, doesn't itself exploit anything)
- **confidence:** MEDIUM
- **bounty estimate:** $1,000–$4,000 (reduced from $3K–$8K)

---

### Chain 6: Default NEXTAUTH_SECRET + Key Reuse → Complete System Compromise (Self-Hosted)

- **status:** VALID
- **findings involved:** F-011a (default secret), F-012a (document password key), F-011b (key reuse), F-011c (SAML clientSecret reuse)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **analysis:** Well-documented chain. The Embrace The Red cookie-minting tooling is published and operational. For self-hosted deployments that did not override the default, this chain gives complete system compromise (session impersonation, SAML response forgery, document access bypass, API token recovery).
- **severity:** CRITICAL (self-hosted)
- **confidence:** HIGH
- **bounty estimate:** $10,000–$25,000

---

### Chain 7: CORS TUS Metadata Read + progress-token → Trigger.dev Info Leak

- **status:** VALID (speculative in metadata details)
- **findings involved:** F-005 (CORS TUS), F-003 (progress-token)
- **judge agreement:** ⚠️ BOTH — Judge A: MEDIUM confidence; Judge B: LOW-MEDIUM confidence
- **analysis:** CORS reflection is confirmed. Cross-origin HEAD requests to TUS viewer can be made from attacker page. However, the chain assumes TUS `Upload-Metadata` header contains `documentVersionId` — this is speculative and depends on the TUS upload implementation. SameSite=Lax cookies may limit cross-origin credentialed requests. The more reliable part is that F-003 (progress-token) generates tokens for any `documentVersionId` — but that doesn't require the CORS step at all.
- **severity:** MEDIUM
- **confidence:** LOW-MEDIUM
- **bounty estimate:** $2,000–$5,000

---

### Chain 8: Embed Clickjacking + Stored XSS → Credential Theft via Transparent Overlay

- **status:** VALID
- **findings involved:** F-015a (frame-ancestors *), F-002 (stored XSS), F-015b (CSP report-only), F-015c (CSP no-op), F-001 (conversations zero-auth), F-006 (record_reaction viewer leak)
- **judge agreement:** ✅ BOTH — Judge A: MEDIUM confidence; Judge B: HIGH confidence
- **analysis:** `frame-ancestors *` allows iframing. Stored XSS fires in embed context. CSP report-only means XSS faces no resistance. CSP no-op means no alerts. The `linkId` requirement (valid shareable link ID) is the only barrier — leakable through various means. The XSS fires in papermark origin, enabling API access.
- **severity:** HIGH
- **confidence:** HIGH (for the chain structure; MEDIUM for the linkId barrier)
- **bounty estimate:** $5,000–$12,000

---

### Chain 9: Rate Limiter Fail-Open + No Auth Rate Limit → OAuth Account Linking → Account Takeover

- **status:** VALID (weakened)
- **findings involved:** F-015f (rate limiter fail-open), F-014a (no auth rate limit), F-015g (allowDangerousEmailAccountLinking)
- **judge agreement:** ⚠️ BOTH — Judge A: LOW confidence; Judge B: MEDIUM confidence. Both agree the Redis outage step is the weak link.
- **analysis:** Three conditions must simultaneously hold: (1) Redis must be unavailable (requires saturating Upstash endpoint or network partition — non-trivial), (2) attacker must control an OAuth provider account with victim's email (Google/LinkedIn verify email ownership), and (3) account linking must be active during the outage. The OAuth account creation barrier is significant. The "Secondary amplification" via default NEXTAUTH_SECRET is a separate chain (Chain 10).
- **severity:** MEDIUM
- **confidence:** LOW (Redis unavailability step is speculative for application-level attackers)
- **bounty estimate:** $1,000–$3,000 (reduced from $4K–$10K)

---

### Chain 10: SAML clientSecret = NEXTAUTH_SECRET + Default → SAML Response Forgery → Account Takeover (Self-Hosted)

- **status:** VALID
- **findings involved:** F-011a (default NEXTAUTH_SECRET), F-011c (SAML clientSecret reuse), F-015g (allowDangerousEmailAccountLinking on SAML)
- **judge agreement:** ✅ BOTH — Judge A: MEDIUM confidence; Judge B: HIGH confidence
- **analysis:** SAML OAuth `clientSecret` at `auth-options.ts:128` is literally `process.env.NEXTAUTH_SECRET`. Combined with default secret and `allowDangerousEmailAccountLinking: true`, an attacker can forge SAML IdP responses. The chain provides an alternative path to Chain 6's JWT forging — if one doesn't work, the other may. Distinct and independently valuable.
- **severity:** HIGH (self-hosted); distinct from Chain 6's CRITICAL because it requires SAML configured
- **confidence:** HIGH (for the code path); MEDIUM (SAML assertion acceptance depends on IdP)
- **bounty estimate:** $8,000–$18,000

---

### Chain 11: CSP Report-Only + Silent Reporting → Undetectable Stored XSS Worm

- **status:** VALID (amplification chain)
- **findings involved:** F-015b (CSP report-only), F-015c (CSP no-op), F-002 (stored XSS)
- **judge agreement:** ✅ BOTH — Judge A: HIGH confidence; Judge B: HIGH confidence
- **analysis:** Correctly identified as amplification chain, not standalone. CSP report-only with `'unsafe-inline'` means zero XSS blocking. CSP report endpoint is a no-op — zero visibility into exploitation. Amplifies Chain 1's XSS worm from "likely detected within days" to "potentially persistent for weeks."
- **severity:** MEDIUM (amplifier — adds to Chain 1 base severity)
- **confidence:** HIGH
- **bounty estimate:** +$3,000–$5,000 adder to Chain 1

---

### Chain 12: Missing HSTS + Secret in Query Param → Revalidation Token Leak → Mass Link Invalidation

- **status:** VALID (weakened)
- **findings involved:** F-015d (missing HSTS), F-008 (revalidate secret in query param)
- **judge agreement:** ⚠️ BOTH. Judge A: MEDIUM confidence; Judge B: LOW-MEDIUM confidence. Both note the multi-condition requirement.
- **analysis:** Requires simultaneously: (1) MITM network position, (2) SSL stripping during non-HSTS window, (3) someone triggering `/api/revalidate?secret=...` during the MITM window. The query-param secret exposure (F-008) is independently valuable — secret leakage via Referer and logs is more practical than the full SSL stripping chain.
- **severity:** MEDIUM
- **confidence:** LOW-MEDIUM (complex multi-condition chain)
- **bounty estimate:** $1,000–$3,000 (reduced from $3K–$7K)

---

## Part 5: Bounty Summary

| Chain | Name | Severity | Bounty Range | Confidence |
|-------|------|----------|-------------|------------|
| 1 | XSS Worm + Conversations Hijack | CRITICAL | $8K–$15K | HIGH |
| 6 | Default Secret → Full Compromise | CRITICAL | $10K–$25K | HIGH |
| 3 | Zod Recon → Bulk PII Extraction | HIGH | $3K–$8K | HIGH |
| 8 | Embed Clickjacking + XSS → Cred Theft | HIGH | $5K–$12K | HIGH |
| 10 | SAML Response Forgery → Account Takeover | HIGH | $8K–$18K | HIGH |
| 2 | CORS TUS + Malicious PDF → Session Hijack | MEDIUM | $2K–$5K | MEDIUM |
| 5 | EOL + Middleware Gap → Data Access | MEDIUM | $1K–$4K | MEDIUM |
| 7 | CORS TUS → Trigger.dev Token Leak | MEDIUM | $2K–$5K | LOW-MEDIUM |
| 9 | Rate Limit Fail-Open → OAuth AT | MEDIUM | $1K–$3K | LOW |
| 11 | CSP Amplifier (adder to Chain 1) | MEDIUM | +$3K–$5K | HIGH |
| 12 | Missing HSTS → Token Intercept | MEDIUM | $1K–$3K | LOW-MEDIUM |
| 4 | DOCX XXE → Cloud Cred Theft | REJECTED | $0 | HIGH |

**Highest validated bounty:** Chain 6 ($10K–$25K) — Complete system compromise via default root secret (self-hosted only).

**Most reliably exploitable:** Chain 1 ($8K–$15K) — Stored XSS worm with zero-auth exfiltration, works on all deployments.

**Best for hosted bounty programs:** Chain 3 ($3K–$8K) — Bulk PII extraction with no authentication, clear GDPR/DPA impact, no deployment-scope limitations.

---

## Part 6: Severity Adjustment Log

| Finding | Original Severity | Validated Severity | Reason |
|---------|------------------|-------------------|--------|
| S-009 / F-009 | MEDIUM | LOW | No confirmed data path from user input to `helpText` sink. Latent quality finding. |
| S-011 / F-013b | MEDIUM / HIGH (disputed) | MEDIUM | CUID@3.0.0 IS predictable (Judge A wrong about CUID2). But all handlers behind NextAuth. MEDIUM is correct. |
| S-012 / F-012b | MEDIUM | LOW | Python 3 ET.parse doesn't resolve external entities by default. XXE not exploitable; billion-laughs DoS only. |
| S-002 (Deps) / F-016a | MEDIUM | LOW | Both CVEs (PP + ReDoS) already patched at 0.20.3. CDN supply chain is the remaining (LOW) concern. |
| Config CORS / F-005 | CRITICAL (config) | MEDIUM | Config layer flagged as CRITICAL; structural analysis and judges correctly rated MEDIUM. Upload still requires dataroom session. |
| Chain 2 | HIGH | MEDIUM | SameSite=Lax blocks cross-origin credentialed POST from fetch/XHR. Structural barrier, not mitigation. |
| Chain 4 | HIGH | REJECTED | ET.parse doesn't resolve entities in modern Python 3. Chain collapses without working XXE. |
| Chain 5 | HIGH | MEDIUM | CVE-2026-44575 doesn't affect v14. Real bypass is simpler (middleware excludes /api/*). |
| Chain 9 | HIGH | MEDIUM | Redis outage step is speculative for application-level attackers. OAuth account creation is non-trivial. |

---

## Part 7: Data Sources

- All methodology-raw/*.md files
- Judge A verdict (`methodology-raw/judge-a.md`)
- Judge B verdict (`methodology-raw/judge-b.md`)
- Source code verified at: conversations-route.ts, tus-viewer.ts, sanitize-html.ts, document-header.tsx, form.tsx, progress-token.ts, record_reaction.ts, feedback/index.ts, revalidate.ts, docx-sanitizer.py, engine.ts, auth-options.ts, ratelimit.ts, package.json, package-lock.json, next.config.mjs, middleware.ts, csp-report/route.ts, domain-configuration.tsx, webhook-events.tsx

---

<promise>VALIDATION_COMPLETE</promise>
