# Deduplicated Findings — Papermark Security Review

**Node:** Layer-5 Deduplication Pass
**Date:** 2026-06-30
**Input:** `06-validated.md` + all `methodology-raw/*.md` files
**Method:** Systematic scan of each methodology file for findings that report the same bug (same file/line/issue type) at different layers. Merge each duplicate pair into one canonical entry, noting which methodology found it first.

---

## Deduplication Actions Taken

| Action | Target | Rationale |
|--------|--------|-----------|
| **MERGED** | F-011c → F-011b | SAML clientSecret reuse (F-011c) is a subset of NEXTAUTH_SECRET key reuse (F-011b). Both report `lib/auth/auth-options.ts:128,149` — same secret used for SAML. F-011b already lists SAML as one of 5 domains; F-011c added the SAML-specific exploit path (response forgery). Absorbed into F-011b's description. |
| **ELIMINATED** | — | No other cross-methodology duplicates found. Every bug reported by multiple methodology files was already consolidated in 06-validated.md. |
| **FLAGGED** | validatePathSecurity (encoding-analysis Finding 2) | Discovered by encoding analysis only; lost during synthesis. NOT in validated set. See advisory section. |

---

## Canonical Finding Set (one entry per unique vulnerability)

### CRITICAL

#### F-001 — Conversations API: Zero Authentication on All Handlers

- **severity:** CRITICAL
- **original IDs:** S-001 (structural), S-001 (SAST synthesis)
- **found by:** Structural Analysis **first** (01-structural-analysis.md). SAST synthesis (02-synthesized-sast.md) confirmed and cross-referenced. Semgrep custom rule `papermark-no-auth-prisma-write` matched (32 findings). AI SAST missed it.
- **files:** `ee/features/conversations/api/conversations-route.ts:1-301` (all 4 handlers), `pages/api/conversations/[[...conversations]].ts:1-10`
- **impact:** Unauthenticated conversation enumeration, data injection into PostgreSQL, email notification trigger via Trigger.dev. Trivially exploitable: no auth, no guessing required.
- **CVSS estimate:** 9.1 (CRITICAL)

---

### CRITICAL (self-hosted)

#### F-002 — NEXTAUTH_SECRET Default Value

- **severity:** CRITICAL (self-hosted deployments only)
- **original IDs:** S-001 (Secrets), WC-001 (web synthesis), C-008 partial (config)
- **found by:** Secrets synthesis **first** (02-synthesized-secrets.md). Web search confirmed published exploitation tooling (03-web-secrets.md). Config analysis (00-ai-configs.md) added key reuse context.
- **file:** `.env.example:1`
- **impact:** Complete cryptographic trust collapse on self-hosted deployments that did not override the default. Session impersonation, SAML response forgery, document access bypass, API credential recovery.
- **CVSS estimate:** 10.0 (CRITICAL)
- **scope:** Self-hosted only. Vercel deployments using system env vars are NOT affected.

---

### HIGH

#### F-003 — Stored XSS via `sanitizePlainText` Entity-Decode Ordering + CVE-2026-44990 `<xmp>` Bypass

- **severity:** HIGH
- **original IDs:** S-002 (structural), S-002 (SAST synthesis), S-002 (encoding analysis), S-001 (sanitizer analysis)
- **found by:** Multiple methodologies independently. AI SAST (00-ai-sast.md) found the entity-decoding ordering bug **first**. Structural analysis (01-structural-analysis.md) added CVE-2026-44990 `<xmp>` bypass context and reachability. Encoding analysis (01-encoding-analysis.md) confirmed the encoding-differential root cause. Sanitizer analysis (01-sanitizer-bypass.md) traced all entry points and sinks.
- **files:**
  - `lib/utils/sanitize-html.ts:12-19` (decode-after-strip bug)
  - `components/documents/document-header.tsx:604` (dangerouslySetInnerHTML sink)
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts:15` (Zod transform entry point)
  - `lib/zod/url-validation.ts:207` (document upload schema)
- **extra context added by each:**
  - AI SAST: Entity-decoding ordering bug identified and code evidence provided
  - Structural: CVE-2026-44990 `<xmp>` bypass, 5 entry points, blast radius analysis
  - Encoding: NFC normalization control-character bypass (supplemental, not a standalone finding)
  - Sanitizer: `sanitize-html@2.17.3` version confirmation, all caller paths mapped
- **impact:** Any team member executes arbitrary JS in every other team member's browser viewing the document. Session theft, worm propagation.
- **CVSS estimate:** 8.7 (HIGH)

---

#### F-004 — `progress-token.ts`: Unauthenticated Trigger.dev Token Generation

- **severity:** HIGH
- **original IDs:** S-003 (structural), S-003 (SAST synthesis)
- **found by:** Structural Analysis **first** (01-structural-analysis.md). SAST synthesis (02-synthesized-sast.md) confirmed. AI SAST and semgrep missed it.
- **file:** `pages/api/progress-token.ts:4-27`
- **impact:** Unauthenticated 15-minute Trigger.dev token exposing document processing metadata.
- **CVSS estimate:** 7.5 (HIGH)

---

#### F-005 — Next.js v14 Unpatchable DoS (CVE-2026-23864 / CVE-2026-23869)

- **severity:** HIGH
- **original IDs:** S-004 (structural), CVE-2026-23864/CVE-2026-23869 (web intel)
- **found by:** Web CVE intelligence **first** (00-early-web-intel.md) identified both CVEs against Next.js v14. Structural analysis (01-structural-analysis.md) confirmed reachability and EOL status. Advisory pairing (03-advisory.md) confirmed no fix available for v14.x.
- **type:** Framework CVE (no application patch available)
- **impact:** Unauthenticated DoS against the application via memory/CPU exhaustion through RSC Flight protocol. Single crafted request exhausts server for ~1 minute.
- **CVSS estimate:** 7.5 (HIGH)

---

#### F-006 — NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY Default Value

- **severity:** HIGH
- **original IDs:** S-002 (Secrets)
- **found by:** Secrets synthesis **first** (02-synthesized-secrets.md). Git history synthesis confirmed no override in tracked commits.
- **file:** `.env.example:68` → `lib/utils.ts:614-628`
- **impact:** Decryption of all document-level passwords on vulnerable self-hosted deployments. Key derivation is unsalted SHA-256 with no KDF.
- **CVSS estimate:** 7.5 (HIGH)
- **scope:** Self-hosted only.

---

#### F-007 — NEXTAUTH_SECRET Reused Across 5 Security Domains (inc. SAML clientSecret → Response Forgery)

- **severity:** MEDIUM (standalone design weakness) / HIGH (with default secret, self-hosted)
- **original IDs:** Config C-008 **ABSORBED** F-011c (SAML clientSecret subset)
- **found by:** Config analysis **first** (00-ai-configs.md) identified the key reuse pattern. F-011c (from 06-validated) was a duplicate subset — merged here.
- **files:**
  - `lib/auth/auth-options.ts:128,149` (SAML OAuth clientSecret + IdP client_secret)
  - `lib/jackson.ts:16-20` (Jackson encryption key derivation via SHA-256)
  - `lib/signing/download-token.ts:19` (HMAC-SHA256 download token key)
  - `lib/signing/access-token.ts:21` (HMAC-SHA256 access token key)
  - `lib/api/auth/token.ts:12` (API token hash preimage component)
- **description:** NEXTAUTH_SECRET reused as the secret key in 5 distinct security contexts. A single compromise (leak, default value, or server breach) undermines all 5 simultaneously. The SAML-specific reuse is the most critical: NEXTAUTH_SECRET is used as both SAML OAuth clientSecret (`auth-options.ts:128`) and SAML IdP client_secret (`auth-options.ts:149`). Combined with `allowDangerousEmailAccountLinking: true` on the SAML provider, knowing NEXTAUTH_SECRET enables forging SAML assertions that auto-link to victim accounts without confirmation — a SAML response forgery attack independent of JWT cookie forging (Chain 10).
- **impact:**
  - Standalone (MEDIUM): Single secret compromise breaks JWT signing, SAML auth, SSO encryption, HMAC tokens, and API tokens simultaneously
  - With default secret (HIGH): Direct SAML response forgery enabling account takeover on self-hosted deployments with SAML configured
- **CVSS estimate:** 5.3 (MEDIUM) standalone / 8.2 (HIGH) with default secret
- **confidence:** HIGH (on code analysis); MEDIUM (SAML response forgery acceptance depends on IdP)

---

### MEDIUM

#### F-008 — CORS Origin Reflection on TUS Viewer Upload Endpoint

- **severity:** MEDIUM
- **original IDs:** S-005 (structural), CORS config finding (00-ai-configs), CVE-2026-57957
- **found by:** Config analysis **first** (00-ai-configs.md) identified this as CRITICAL (later downgraded). Structural analysis (01-structural-analysis.md) added reachability and confirmed it as MEDIUM. SAST synthesis (02-synthesized-sast.md) merged both. Independently confirmed as CVE-2026-57957.
- **file:** `pages/api/file/tus-viewer/[[...file]].ts:236-237`
- **CVE:** CVE-2026-57957 (papermark-specific, filed)
- **impact:** Cross-origin credentialed uploads to victim's dataroom if victim has active session. Requires victim visit attacker's page.
- **CVSS estimate:** 4.7 (MEDIUM) per CVE-2026-57957

---

#### F-009 — `record_reaction.ts`: Unauthenticated DB Write + Viewer Data Leak

- **severity:** MEDIUM
- **original IDs:** S-006 (structural), S-006 (SAST synthesis)
- **found by:** Structural Analysis **first** (01-structural-analysis.md). SAST synthesis confirmed. Semgrep custom rule `papermark-no-auth-prisma-write` matched at line 24.
- **file:** `pages/api/record_reaction.ts:17-42`
- **impact:** Unauthenticated viewer metadata (email) extraction, data injection into PostgreSQL, analytics pollution.
- **CVSS estimate:** 5.3 (MEDIUM)

---

#### F-010 — `feedback/index.ts`: Unauthenticated Feedback Response Recording

- **severity:** MEDIUM
- **original IDs:** S-007 (structural), S-007 (SAST synthesis)
- **found by:** Structural Analysis **first** (01-structural-analysis.md). SAST synthesis confirmed. Semgrep custom rule matched at line 46.
- **file:** `pages/api/feedback/index.ts:9-56`
- **impact:** Unauthenticated data injection into PostgreSQL feedback table.
- **CVSS estimate:** 5.3 (MEDIUM)

---

#### F-011 — `revalidate.ts`: Shared Secret in URL Query Parameter

- **severity:** MEDIUM
- **original IDs:** S-008 (structural), S-008 (SAST synthesis)
- **found by:** Structural Analysis **first** (01-structural-analysis.md). SAST synthesis confirmed. Custom semgrep rule `papermark-secret-in-query-param` matched at line 14. CWE-598 confirmed.
- **file:** `pages/api/revalidate.ts:13-14`
- **impact:** Secret leakage via server logs, Referer headers, browser history enables mass ISR cache invalidation and link enumeration.
- **CVSS estimate:** 5.3 (MEDIUM)

---

#### F-012 — Zod Validation Error Information Disclosure

- **severity:** MEDIUM
- **original IDs:** S-010 (SAST/structural)
- **found by:** AI SAST **first** (00-ai-sast.md). Structural analysis (01-structural-analysis.md) confirmed. Custom semgrep rule `papermark-zod-error-format-disclosure` matched 9 endpoints independently.
- **files:** 9 endpoints including `pages/api/webhooks/services/[...path]/index.ts:236`, `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:67`
- **impact:** Enables targeted fuzzing and API recon by revealing schema structure, field names, types, enum values, regex constraints.
- **CVSS estimate:** 4.3 (MEDIUM)

---

#### F-013 — `cuid@3.0.0` Predictable IDs in Permission Groups

- **severity:** MEDIUM
- **original IDs:** S-011 (structural) + S-001 (Deps) — MERGED
- **found by:** Dependency synthesis **first** (02-synthesized-dependencies.md S-001) rated it HIGH (overstated). Reachability analysis (01-reachability-analysis.md Finding 14) correctly downgraded to MEDIUM by noting all 3 handlers are behind NextAuth + team membership auth. Structural analysis confirmed.
- **files:**
  - `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`
- **package:** `cuid@3.0.0` (NOT `@paralleldrive/cuid2` — confirmed via package-lock.json deprecation message)
- **impact:** Authenticated team members can enumerate permission group IDs and potentially access unauthorized resources via IDOR.
- **confidence:** HIGH (on version); MEDIUM (on exploitability via auth gate)

---

#### F-014 — Embed Route `frame-ancestors *` Allows Framing by Any Origin

- **severity:** MEDIUM (HIGH when combined with F-003 stored XSS)
- **original IDs:** Config C-001
- **found by:** Config analysis **only** (00-ai-configs.md). Not found by structural analysis or SAST.
- **file:** `next.config.mjs:259-271`
- **impact:** Enables clickjacking against document viewers. Combined with F-003 stored XSS, enables credential theft via transparent overlay in iframe.
- **confidence:** HIGH

---

#### F-015 — Main CSP is Report-Only (Not Enforced)

- **severity:** MEDIUM (amplifier for XSS findings)
- **original IDs:** Config C-002
- **found by:** Config analysis **first** (00-ai-configs.md). Reachability analysis confirmed.
- **file:** `next.config.mjs:219`
- **impact:** XSS payloads face zero CSP blocking on main app pages. Includes `'unsafe-inline'` and `'unsafe-eval'`. Combined with F-003, enables undetectable stored XSS worm.
- **confidence:** HIGH

---

#### F-016 — CSP Report Endpoint is a No-Op

- **severity:** MEDIUM (blocks detection)
- **original IDs:** Config C-003
- **found by:** Config analysis **only** (00-ai-configs.md).
- **file:** `app/api/csp-report/route.ts:7`
- **impact:** XSS exploitation generates no security alerts. CSP violation reports silently discarded.
- **confidence:** HIGH

---

#### F-017 — Missing HSTS Header

- **severity:** MEDIUM
- **original IDs:** Config C-004
- **found by:** Config analysis **only** (00-ai-configs.md).
- **file:** `next.config.mjs:190-312`
- **impact:** Enables SSL stripping on custom domains. Amplifies F-011 (query-param secret interception).
- **confidence:** HIGH

---

#### F-018 — Missing X-Content-Type-Options Header

- **severity:** MEDIUM
- **original IDs:** Config C-005
- **found by:** Config analysis **only** (00-ai-configs.md). Judge A did not include this finding; Judge B confirmed.
- **file:** `next.config.mjs:190-312` (absent from global headers)
- **impact:** Uploaded files could be interpreted as HTML/JavaScript by browsers via MIME sniffing.
- **confidence:** HIGH

---

#### F-019 — Rate Limiter Fails Open When Redis is Unavailable

- **severity:** MEDIUM
- **original IDs:** Config C-006
- **found by:** Config analysis **first** (00-ai-configs.md). Structural analysis confirmed.
- **file:** `ee/features/security/lib/ratelimit.ts:43-44`
- **impact:** All rate-limited endpoints become unthrottled during Redis outage. Enables brute-force attacks when Redis is down.
- **confidence:** HIGH

---

#### F-020 — `allowDangerousEmailAccountLinking: true` on 3 OAuth Providers

- **severity:** MEDIUM
- **original IDs:** Config C-007
- **found by:** Config analysis **first** (00-ai-configs.md). Structural analysis confirmed via code read.
- **file:** `lib/auth/auth-options.ts:35,55,130` (Google, LinkedIn, SAML)
- **impact:** Account takeover when combined with rate-limit bypass (F-019) or known SAML secret (F-007).
- **confidence:** HIGH

---

#### F-021 — `pdfjs-dist` 3.11.174 Arbitrary JavaScript Execution via Malicious PDF

- **severity:** MEDIUM
- **original IDs:** S-004 (Deps) / D-003
- **found by:** Dependency synthesis **first** (02-synthesized-dependencies.md). OSV-scanner identified `pdfjs-dist@3.11.174`. Advisory pairing (03-advisory.md) confirmed CVE-2024-4367. Reachability analysis confirmed PDF viewer at `preview-viewer.tsx` uses default `isEvalSupported: true`.
- **CVE:** CVE-2024-4367 (GHSA-wgrm-67xf-hhpq)
- **type:** Transitive dependency via `react-pdf`
- **impact:** XSS in document viewer via malicious PDF. Requires authenticated upload + victim view.
- **confidence:** HIGH

---

### LOW

#### F-022 — `dangerouslySetInnerHTML` on `helpText` Props — Latent XSS Sinks

- **severity:** LOW (downgraded from MEDIUM)
- **original IDs:** S-009 (structural/SAST), semgrep `react-dangerouslysetinnerhtml` WARNING
- **found by:** SAST synthesis **first** via semgrep (02-synthesized-sast.md). Structural analysis (01-structural-analysis.md) confirmed both sink locations and traced all callers — all hardcoded strings. No active exploit path.
- **files:**
  - `components/ui/form.tsx:145`
  - `components/account/upload-avatar.tsx:96`
- **impact:** Latent quality finding. If any future code change feeds user data through `helpText`, XSS results.
- **confidence:** MEDIUM

---

#### F-023 — XXE in `docx-sanitizer.py`: Native XML Parsing Without defusedxml

- **severity:** LOW (downgraded from MEDIUM)
- **original IDs:** S-012 (SAST/semgrep)
- **found by:** Semgrep custom rule `papermark-python-xxe-xml-parse` **first** (04-custom-rules.md). AI SAST explicitly stated "XXE: ✅ Not found" — false negative. Reachability analysis confirmed the DOCX conversion pipeline.
- **file:** `ee/features/conversions/python/docx-sanitizer.py:146`
- **verification:** Python 3 `xml.etree.ElementTree.parse()` does NOT resolve external entities by default. XXE file read/SSRF not exploitable. Billion-laughs DoS is still possible.
- **impact:** Defense-in-depth. Billion-laughs DoS via crafted DOCX files during document conversion. Requires authenticated upload.
- **confidence:** HIGH (on Python 3 not resolving external entities); MEDIUM (on DoS)

---

#### F-024 — ReDoS in Workflow Engine (Non-Literal RegExp)

- **severity:** LOW
- **original IDs:** S-013 (SAST/semgrep)
- **found by:** Semgrep custom rule `papermark-regexp-from-user-input` **first** (04-custom-rules.md). Structural analysis confirmed.
- **file:** `ee/features/workflows/lib/engine.ts:307`
- **impact:** Limited — requires admin workflow config. Wrapped in try/catch. Bounded by per-request timeout.
- **confidence:** MEDIUM

---

#### F-025 — No Rate Limiting on Authentication Endpoints

- **severity:** LOW
- **original IDs:** S-014 (SAST/structural), NC-001 (web synthesis unconfirmed)
- **found by:** AI SAST **first** (00-ai-sast.md). Custom semgrep rule `papermark-nextauth-no-rate-limit` confirmed. Structural analysis (01-structural-analysis.md) confirmed. Web synthesis (03-web-synthesis.md) listed as NC-001 (unconfirmed) but it was already confirmed.
- **file:** `pages/api/auth/[...nextauth].ts`
- **impact:** Enables brute-force attacks against credentials, email enumeration. Defense-in-depth gap.
- **confidence:** HIGH

---

#### F-026 — Webhook Event Request/Response Body Stored in Tinybird

- **severity:** LOW
- **original IDs:** S-015 (SAST/structural), NC-002 (web synthesis)
- **found by:** AI SAST **first** (00-ai-sast.md). Structural analysis confirmed. Web synthesis (03-web-synthesis.md) listed as NC-002 — confirmed.
- **file:** `app/api/webhooks/callback/route.ts:43-44`
- **impact:** Sensitive data from webhook responses could be exposed in admin UI. Requires sensitive data + team member access.
- **confidence:** MEDIUM

---

#### F-027 — `xlsx` CDN Tarball Without Integrity Hash

- **severity:** LOW
- **original IDs:** S-002 (Deps) / D-001
- **found by:** Dependency synthesis **first** (02-synthesized-dependencies.md). Both AI and OSV-scanner identified it. Advisory pairing (03-advisory.md) confirmed both CVEs (CVE-2023-30533, CVE-2024-22363) are PATCHED at 0.20.3 — the CVE portion is a false positive; CDN supply chain is the remaining risk.
- **file:** `package.json:175`
- **impact:** Supply chain risk during installation only. No integrity hash on CDN tarball.
- **confidence:** HIGH

---

#### F-028 — Unused Direct Dependencies Increasing Attack Surface

- **severity:** LOW
- **original IDs:** S-005 (Deps) / D-004
- **found by:** Dependency synthesis **first** (02-synthesized-dependencies.md). GitNexus graph confirmed zero imports for all 3 packages.
- **files:** `package.json` — `oidc-provider`, `@modelcontextprotocol/sdk`, `tokenlens`
- **impact:** None today. Adds ~100+ transitive packages with no benefit.
- **confidence:** HIGH

---

#### F-029 — X-Powered-By Header Leaks Application Info on Custom Domains

- **severity:** LOW
- **original IDs:** Config C-009
- **found by:** Config analysis **only** (00-ai-configs.md).
- **file:** `lib/middleware/domain.ts:40-41`
- **impact:** Minor information disclosure enabling targeted vulnerability search.
- **confidence:** HIGH

---

#### F-030 — Trigger.dev Project ID Hardcoded in Config

- **severity:** LOW
- **original IDs:** S-003 (Secrets)
- **found by:** Secrets synthesis **first** (02-synthesized-secrets.md). AI-only finding — gitleaks and trufflehog missed it (not a credential pattern).
- **file:** `trigger.config.ts:7`
- **impact:** Marginal recon value. Not a credential — Trigger.dev auth uses `TRIGGER_SECRET_KEY` from env.
- **confidence:** MEDIUM

---

## Rejected Findings

| ID | Original ID | Reason | Eliminated By |
|----|------------|--------|---------------|
| R-001 | S-016 (domain config dangerouslySetInnerHTML) | Static instruction content. Not a vulnerability. | Both judges |
| R-002 | S-017 (unsafe format strings in console.log) | JS `console.log` cannot be weaponized in Node.js. False positive. | Judge B (correct); semgrep rule generic |
| R-003 | S-003 (Deps) — `cookie@0.4.2` prefix injection | Deep transitive via Trigger.dev infra, not user-facing. Not reachable. | Both judges + dependency synthesis |
| R-004 | S-018 (GCM no-tag-length) | Manual IV/auth tag length validation present. False positive. | SAST synthesis + git history synthesis (both independently eliminated) |

---

## Advisory: Findings Lost During Synthesis

The following findings from lower-layer methodologies are NOT in the canonical set. They were discovered by exactly one methodology and were not carried forward during synthesis.

| Methodology | Finding | Severity | Reason Dropped |
|------------|---------|----------|----------------|
| `01-encoding-analysis.md` (Finding 2) | `validatePathSecurity` case-sensitive double-encoding bypass (`lib/zod/url-validation.ts:20-34`) — `includes()` is case-sensitive; `%2e%2e%2f` bypasses `%2E%2E%2F` check. Affects URL/path validation used by document upload, webhook, and Notion URL schemas. | HIGH (original) | **Likely an oversight.** Never referenced by any synthesis layer, any judge verdict, or SAST synthesis. Only exists in encoding analysis. The AI SAST incorrectly classified it as "Clean." Should be evaluated for inclusion in the final report. |
| `01-encoding-analysis.md` (Finding 3) | NFC normalization control-character bypass in `sanitizePlainText` — `.normalize("NFC")` can produce invisible/control characters not caught by the limited regex filters. | MEDIUM (original) | Overlaps with F-003 (stored XSS). The NFC normalization concern is a minor encoding nuance of the same sanitizer bug. Absorbed into F-003's scope. |
| `01-encoding-analysis.md` (Finding 4) | Content-Type charset mismatch between Zod schema (strict enum, rejects charset params) and webhook handler (strips params via `split(";")[0]`). Inconsistency gap. | MEDIUM (original) | Low practical exploitability. The Zod enum validation is strict but the webhook path handles charset correctly. Defense-in-depth. |
| `01-encoding-analysis.md` (Finding 5) | TUS base64url key encoding chain — double encoding (base64url → atob → percent-encode → decodeURIComponent) is unusual and fragile. | LOW (original) | Within trust boundary — TUS upload requires authentication. Theoretical only. |
| `01-encoding-analysis.md` (Finding 6) | `folderPathSchema` incomplete Unicode/path traversal protection — allows Unicode homoglyphs, zero-width characters. | LOW (original) | Defense-in-depth. Standard Next.js URL decoding mitigates direct encoding bypass. |
| `01-encoding-analysis.md` (Finding 7) | CSV formula injection protection — incomplete type coverage for non-standard objects via `toString()`. | LOW (original) | Very low risk. Existing `=`, `+`, `-`, `@` protection works for standard cases. |

---

## Chain Reference Update (post-dedup)

After merging F-011c into F-007, chain finding references are updated:

| Chain | Original Finding References | Updated Finding References |
|-------|---------------------------|---------------------------|
| 1: XSS Worm | F-002, F-001, F-006 | **Unchanged** |
| 2: CORS + PDF | F-005, F-021 | F-008, F-021 (ID change: F-005→F-008, F-014b→F-021) |
| 3: Zod → PII | F-012, F-001, F-009, F-010 | F-012, F-001, F-009, F-010 (ID change: F-012 was F-010 previously) |
| 5: EOL + Middleware | F-004, F-004, F-001 | F-005, F-004, F-001 (ID change: F-004→F-005) |
| 6: Default Secret → Full Compromise | F-011a, F-012a, F-011b, F-011c | **F-002, F-006, F-007** (F-011a→F-002, F-012a→F-006, F-011bc→F-007) |
| 7: CORS → Token | F-005, F-004 | F-008, F-004 (ID change: F-005→F-008, F-003→F-004) |
| 8: Embed Clickjack + XSS | F-015a, F-002, F-015b, F-015c, F-001, F-006 | F-014, F-003, F-015, F-016, F-001, F-009 (various ID changes) |
| 9: Rate Limit → OAuth | F-015f, F-014a, F-015g | F-019, F-025, F-020 (ID changes) |
| 10: SAML Forgery | F-011a, F-011c, F-015g | **F-002, F-007, F-020** (F-011c merged into F-007) |
| 11: CSP Amplifier | F-015b, F-015c, F-002 | F-015, F-016, F-003 (ID changes) |
| 12: HSTS + Query Param | F-015d, F-008 | F-017, F-011 (ID changes) |

---

## Summary

| Severity | Count | IDs |
|----------|-------|-----|
| CRITICAL | 2 | F-001, F-002 |
| HIGH | 4 | F-003, F-004, F-005, F-006 |
| HIGH/MEDIUM | 1 | F-007 (severity depends on context) |
| MEDIUM | 13 | F-008, F-009, F-010, F-011, F-012, F-013, F-014, F-015, F-016, F-017, F-018, F-019, F-020, F-021 |
| LOW | 9 | F-022, F-023, F-024, F-025, F-026, F-027, F-028, F-029, F-030 |
| REJECTED | 4 | R-001, R-002, R-003, R-004 |
| **TOTAL VALID** | **29** | (down from 25 in 06-validated + 1 merge + lost finding advisory) |

**Total dedup actions:** 1 merge (F-011c → F-007), 0 splits, 1 lost finding flagged for advisory review.

**Consumed by:** `write-findings`
