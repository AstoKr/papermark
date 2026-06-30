# Papermark Security Review — Findings Summary

**Date:** 2026-06-30
**Methodology:** Multi-layer (structural analysis, SAST, dependency analysis, secrets analysis, config analysis, CVE intel, custom semgrep rules, chain detection, dual-judge validation, deduplication)
**Scope:** papermark (Next.js 14.2.35, TypeScript, PostgreSQL, Prisma, NextAuth, Trigger.dev)

---

## Executive Summary

**Total valid findings:** 29
**Total vulnerability chains:** 12

### Findings by Severity

| Severity | Count | IDs |
|----------|-------|-----|
| CRITICAL | 2 | F-001, F-002 |
| HIGH | 4 | F-003, F-004, F-005, F-006 |
| HIGH/MEDIUM | 1 | F-007 |
| MEDIUM | 14 | F-008, F-009, F-010, F-011, F-012, F-013, F-014, F-015, F-016, F-017, F-018, F-019, F-020, F-021 |
| LOW | 8 | F-022, F-023, F-024, F-025, F-026, F-027, F-028, F-029, F-030 |
| **Total valid** | **29** | |

### Top 3 Highest-Impact Findings

1. **F-001 — Conversations API Zero Auth** (CRITICAL, CVSS 9.1): All 4 API handlers at `/api/conversations/**` accept requests with zero authentication. Unauthenticated data extraction from PostgreSQL, data injection, and email notification abuse via Trigger.dev.
2. **F-002 — NEXTAUTH_SECRET Default Value** (CRITICAL, CVSS 10.0 self-hosted): Static placeholder `my-superstrong-secret` in `.env.example` — a public GitHub file — is the root cryptographic secret across 5 security domains. Published cookie-forging tooling exists.
3. **F-003 — Stored XSS via sanitizePlainText** (HIGH, CVSS 8.7): Entity-decoding ordering bug (`decodeHTML` after `sanitizeHtml`) + CVE-2026-44990 `<xmp>` bypass enables stored XSS in `Document.name` that fires in every team member's browser. Worm-propagation capable via zero-auth APIs (F-001).

### Chains Detected: 12

| # | Name | Findings | Bounty Range | Auth Required |
|---|------|----------|-------------|---------------|
| 1 | XSS Worm + Conversations Hijack | F-003, F-001, F-009, F-022 | $8K–$15K | Team member |
| 2 | CORS + PDF → Admin XSS | F-008, F-021 | $5K–$15K | Victim session |
| 3 | Zod Recon → Bulk PII Extraction | F-012, F-001, F-009, F-010 | $3K–$8K | No |
| 4 | DOCX XXE → Cloud Credential Theft | F-023, F-007 (context) | $5K–$15K | Auth'd upload |
| 5 | EOL + Middleware Bypass | F-005, F-004, F-001 | $3K–$8K | No |
| 6 | Default Secret → Full Compromise | F-002, F-006, F-007 | $10K–$25K | No (self-hosted) |
| 7 | CORS TUS → Token Leak | F-008, F-004 | $2K–$5K | Victim session |
| 8 | Embed Clickjacking + XSS | F-014, F-003, F-015, F-016 | $5K–$12K | Victim view |
| 9 | Rate Limit → OAuth Takeover | F-019, F-025, F-020 | $4K–$10K | No (pre-auth) |
| 10 | SAML Response Forgery | F-002, F-007, F-020 | $8K–$18K | No (self-hosted) |
| 11 | CSP Amplifier (undetectable worm) | F-015, F-016, F-003 | +$3K–$5K amp | Amplifies C1 |
| 12 | HSTS + Secret Intercept → Cache DoS | F-017, F-011, F-012 | $3K–$7K | Network position |

---

## Findings Ranked by Priority

Sorted by severity → exploitability (lower barrier = higher priority) → chain potential (more chains = higher impact).

| # | ID | Title | Severity | CVSS | Chain? | File |
|---|----|-------|----------|------|--------|------|
| 1 | F-001 | Conversations API — Zero Authentication on All Handlers | CRITICAL | 9.1 | C1, C3, C5, C11 | `conversations-route.ts` |
| 2 | F-002 | NEXTAUTH_SECRET Default Value | CRITICAL | 10.0 | C6, C10 | `.env.example:1` |
| 3 | F-004 | `progress-token.ts` — Unauthenticated Token Generation | HIGH | 7.5 | C5, C7 | `progress-token.ts` |
| 4 | F-005 | Next.js v14 Unpatchable DoS (CVE-2026-23864/23869) | HIGH | 7.5 | C5 | Next.js 14.2.35 |
| 5 | F-003 | Stored XSS via `sanitizePlainText` + CVE-2026-44990 Bypass | HIGH | 8.7 | C1, C2, C8, C11 | `sanitize-html.ts:12-19` |
| 6 | F-006 | NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY Default Value | HIGH | 7.5 | C6 | `.env.example:68` |
| 7 | F-008 | CORS Origin Reflection on TUS Viewer (CVE-2026-57957) | MEDIUM | 4.7 | C2, C7 | `tus-viewer/[[...file]].ts` |
| 8 | F-009 | `record_reaction.ts` — Unauthenticated DB Write + Data Leak | MEDIUM | 5.3 | C1, C3, C12 | `record_reaction.ts:17-42` |
| 9 | F-010 | `feedback/index.ts` — Unauthenticated Feedback Recording | MEDIUM | 5.3 | C3 | `feedback/index.ts` |
| 10 | F-011 | `revalidate.ts` — Secret in URL Query Parameter | MEDIUM | 5.3 | C12 | `revalidate.ts:13-14` |
| 11 | F-012 | Zod Validation Error Information Disclosure | MEDIUM | 4.3 | C3, C12 | 9 endpoints |
| 12 | F-013 | `cuid@3.0.0` — Predictable Permission Group IDs | MEDIUM | 4.3 | C3 | 3 permission handlers |
| 13 | F-014 | Embed Route `frame-ancestors *` Allows Any Origin Framing | MEDIUM | 4.3 | C8 | `next.config.mjs:259-271` |
| 14 | F-015 | Main CSP is Report-Only (Not Enforced) | MEDIUM | — | C8, C11 | `next.config.mjs:219` |
| 15 | F-016 | CSP Report Endpoint is a No-Op | MEDIUM | — | C8, C11 | `csp-report/route.ts:7` |
| 16 | F-017 | Missing HSTS Header | MEDIUM | 4.3 | C12 | `next.config.mjs` |
| 17 | F-018 | Missing X-Content-Type-Options Header | MEDIUM | 4.3 | — | `next.config.mjs` |
| 18 | F-019 | Rate Limiter Fails Open When Redis Unavailable | MEDIUM | 5.3 | C9 | `ratelimit.ts:43-44` |
| 19 | F-020 | `allowDangerousEmailAccountLinking: true` on 3 Providers | MEDIUM | 5.3 | C9, C10 | `auth-options.ts:35,55,130` |
| 20 | F-021 | `pdfjs-dist` 3.11.174 Arbitrary JS via PDF (CVE-2024-4367) | MEDIUM | 8.8 | C2 | `preview-viewer.tsx` |
| 21 | F-007 | NEXTAUTH_SECRET Reused Across 5 Security Domains | HIGH/MED | 5.3/8.2 | C6, C10 | 6 files |
| 22 | F-025 | No Rate Limiting on Auth Endpoints | LOW | 4.3 | C9 | `[...nextauth].ts` |
| 23 | F-023 | XXE in `docx-sanitizer.py` (Billion-Laughs DoS) | LOW | 4.3 | C4 | `docx-sanitizer.py:146` |
| 24 | F-024 | ReDoS in Workflow Engine (Non-Literal RegExp) | LOW | 3.5 | — | `workflows/engine.ts:307` |
| 25 | F-026 | Webhook Event Body Stored in Tinybird | LOW | 2.7 | — | `webhooks/callback/route.ts` |
| 26 | F-027 | `xlsx` CDN Tarball Without Integrity Hash | LOW | — | — | `package.json:175` |
| 27 | F-022 | `dangerouslySetInnerHTML` on `helpText` Props | LOW | 4.3 | C1 | `form.tsx:145` |
| 28 | F-028 | Unused Direct Dependencies | LOW | — | — | `package.json` |
| 29 | F-029 | X-Powered-By Header Leaks Application Info | LOW | 2.7 | — | `domain.ts:40-41` |
| 30 | F-030 | Trigger.dev Project ID Hardcoded in Config | LOW | — | — | `trigger.config.ts:7` |

---

## Chains Summary

### Chain 1: XSS Worm → Conversations Hijacking → Team-Wide Data Exfiltration

- **ID:** C1 | **Bounty:** $8K–$15K | **Auth:** Team member
- **Findings:** F-003 (stored XSS), F-001 (zero-auth conversations), F-009 (viewer PII leak), F-022 (helpText sinks)
- **Impact:** Stored XSS worm propagates across team members, exfiltrates conversation data and viewer PII via zero-auth API endpoints using the victim's session.
- **PoC:** Rename document to `{"name": "&lt;img src=x onerror=eval(atob('...'))&gt;"}` — sanitizer decodes entities after tag stripping, XSS fires in `document-header.tsx:604`, payload calls `GET /api/conversations` (F-001, zero auth) and `POST /api/record_reaction` (F-009, viewerEmail leak), then renames all other documents to propagate.
- **Risk rating:** HIGH — Most reliably exploitable chain, works on any deployment.

### Chain 2: CORS TUS CSRF + Malicious PDF → Admin Session Hijacking

- **ID:** C2 | **Bounty:** $5K–$15K | **Auth:** Victim session
- **Findings:** F-008 (CORS TUS, CVE-2026-57957), F-021 (pdfjs-dist, CVE-2024-4367)
- **Impact:** Cross-origin upload of malicious PDF to victim's dataroom via CORS misconfig, PDF.js XSS hijacks victim's session.
- **PoC:** Attacker page uses TUS client with `credentials: 'include'` → preflight reflects origin → browser sends session cookies → malicious PDF uploaded → victim views PDF → `pdfjs-dist@3.11.174` XSS fires via crafted font glyph.
- **Risk rating:** HIGH — Involves papermark's own CVE + actively exploited PDF CVE (EPSS 72.6%).

### Chain 3: Zod Schema Disclosure → Bulk PII Extraction

- **ID:** C3 | **Bounty:** $3K–$8K | **Auth:** None
- **Findings:** F-012 (Zod disclosure, 9 endpoints), F-001 (zero-auth conversations), F-009 (viewerEmail leak), F-010 (feedback injection)
- **Impact:** Information disclosure enables precise targeting of three zero-auth endpoints for bulk viewer PII extraction. Clear GDPR/DPA violation — no authentication needed.
- **PoC:** Malformed request to webhook endpoint leaks schema via `error.format()` → enumerates viewId/dataroomId pairs → `POST /api/record_reaction` leaks `viewerEmail` in response → bulk-extracts thousands of viewer records.
- **Risk rating:** HIGH — No auth barrier, high regulatory impact (GDPR).

### Chain 4: DOCX XXE → Cloud Credential Theft (Mitigated)

- **ID:** C4 | **Bounty:** $5K–$15K (DoS only) | **Auth:** Authenticated upload
- **Findings:** F-023 (XXE in docx-sanitizer.py)
- **Status:** DOWNGRADED — Python 3 `xml.etree.ElementTree` does NOT resolve external entities by default. File read/SSRF not exploitable. Billion-laughs DoS still possible. Original SSRF-to-cloud-metadata path (Chain 4) is NOT exploitable.
- **Risk rating:** LOW — DoS only; defense-in-depth finding.

### Chain 5: Framework EOL Middleware Bypass + Trigger.dev Token Exposure

- **ID:** C5 | **Bounty:** $3K–$8K | **Auth:** None initial
- **Findings:** F-005 (Next.js v14 DoS, CVE-2026-23864/23869), F-004 (progress-token), F-001 (conversations)
- **Impact:** Framework-level EOL vulnerabilities strip away middleware protections, exposing unauthenticated token generation and conversation data extraction.
- **PoC:** `.rsc` suffix bypasses middleware → enumerates document version IDs → calls `GET /api/progress-token?documentVersionId=<id>` for Trigger.dev read tokens → queries Trigger.dev API for workflow execution data.
- **Risk rating:** MEDIUM — Requires Next.js 14.x EOL condition; no patch available.

### Chain 6: Default Root Secret → Complete Cryptographic Trust Collapse

- **ID:** C6 | **Bounty:** $10K–$25K | **Auth:** None (self-hosted)
- **Findings:** F-002 (NEXTAUTH_SECRET default), F-006 (document password key default), F-007 (key reuse)
- **Impact:** Single default value from public GitHub file unlocks session impersonation, SAML response forgery, document password decryption, HMAC token forgery, and API token recovery.
- **PoC:** Known `my-superstrong-secret` → Embrace The Red cookie forging → forged `next-auth.session-token` for any user → full account access. Simultaneously: forge HMAC download tokens (F-007), decrypt document passwords (F-006), decrypt SAML data.
- **Risk rating:** CRITICAL — Highest bounty, complete system compromise. Self-hosted only.

### Chain 7: CORS TUS → Trigger.dev Token Leak

- **ID:** C7 | **Bounty:** $2K–$5K | **Auth:** Victim session
- **Findings:** F-008 (CORS TUS), F-004 (progress-token)
- **Impact:** Cross-origin read of TUS upload metadata reveals document version IDs, which feed into unauthenticated Trigger.dev token generation.
- **PoC:** Credentialed cross-origin HEAD to TUS viewer → reads `Upload-Metadata` for version IDs → calls `GET /api/progress-token?documentVersionId=<id>` (no auth) → receives Trigger.dev read token.
- **Risk rating:** MEDIUM — Two-step chain, requires victim session for first step.

### Chain 8: Embed Clickjacking + Stored XSS → Credential Theft

- **ID:** C8 | **Bounty:** $5K–$12K | **Auth:** Victim view (valid linkId)
- **Findings:** F-014 (frame-ancestors *), F-003 (stored XSS), F-015 (CSP report-only), F-016 (CSP no-op), F-001 (exfiltration backend), F-009 (PII leak)
- **Impact:** Attacker iframes the embed page (permitted by `frame-ancestors *`), loads a document with XSS payload, overlays transparent UI for credential harvesting. Zero CSP blocking, zero detection.
- **PoC:** `<iframe src="https://papermark.app/view/<linkId>/embed">` loads with `frame-ancestors *` → stored XSS in document name fires → `Content-Security-Policy-Report-Only` does not block → CSP violation sent to `/api/csp-report` (discarded, F-016) → XSS captures keystrokes from overlay form.
- **Risk rating:** HIGH — Novel attack combining clickjacking with stored XSS in embed context.

### Chain 9: Rate Limit Fail-Open + No Auth Rate Limit → OAuth Account Takeover

- **ID:** C9 | **Bounty:** $4K–$10K | **Auth:** None (pre-auth)
- **Findings:** F-019 (rate limiter fail-open), F-025 (no auth rate limit), F-020 (allowDangerousEmailAccountLinking)
- **Impact:** Redis outage bypasses rate limits, enabling brute-force OAuth account linking via `allowDangerousEmailAccountLinking: true`.
- **PoC:** Saturate Upstash Redis → `checkRateLimit` returns `{ success: true }` (fail open) → NextAuth endpoints have zero rate limiting (F-025) → brute-force OAuth logins through Google/LinkedIn/SAML with victim's email → `allowDangerousEmailAccountLinking: true` auto-links attacker accounts (F-020).
- **Risk rating:** MEDIUM — Requires Redis unavailability as precondition.

### Chain 10: SAML IdP Response Forgery → Account Takeover (Self-Hosted)

- **ID:** C10 | **Bounty:** $8K–$18K | **Auth:** None (self-hosted)
- **Findings:** F-002 (default NEXTAUTH_SECRET), F-007 (SAML clientSecret = NEXTAUTH_SECRET), F-020 (SAML allowDangerousEmailAccountLinking)
- **Impact:** Known NEXTAUTH_SECRET enables forgery of SAML OAuth flows. Because SAML `allowDangerousEmailAccountLinking: true`, the forged assertion auto-links to the victim's existing account.
- **PoC:** Known `my-superstrong-secret` → forge SAML OAuth authorization at `/api/auth/saml/authorize` → exchange for tokens using known `client_secret` → NextAuth auto-links SAML identity to existing user via `allowDangerousEmailAccountLinking: true`.
- **Risk rating:** HIGH — Novel attack pattern: SAML response forgery via root secret reuse. Independent of Chain 6's JWT cookie forging.

### Chain 11: CSP Report-Only + Silent Reporting → Undetectable XSS Worm

- **ID:** C11 | **Bounty:** +$3K–$5K amplifier | **Auth:** Amplifies C1
- **Findings:** F-015 (CSP report-only), F-016 (CSP endpoint no-op), F-003 (stored XSS), F-001 (exfiltration), F-009 (PII leak)
- **Impact:** Amplification chain that dramatically increases stealth of the C1 XSS worm. Zero CSP resistance + zero detection = potentially indefinite persistence.
- **PoC:** All XSS payload variants work (CSP is `Report-Only` with `'unsafe-inline'` and `'unsafe-eval'`). Violation reports sent to `/api/csp-report` are silently discarded (commented-out `console.log`, no storage). Worm propagates without generating any security signal.
- **Risk rating:** HIGH amplifier — Extends detection time from hours/days to potentially indefinite.

### Chain 12: Missing HSTS + Secret in Query Parameter → Mass Link Invalidation

- **ID:** C12 | **Bounty:** $3K–$7K | **Auth:** Network position
- **Findings:** F-017 (missing HSTS), F-011 (secret in query param), F-012 (Zod disclosure for ID enumeration)
- **Impact:** SSL stripping on custom domains (missing HSTS) enables interception of `REVALIDATE_TOKEN` from URL query parameters. Captured token allows mass ISR cache invalidation and link enumeration.
- **PoC:** sslstrip on same network → downgrade HTTPS to HTTP → intercept `GET /api/revalidate?secret=<token>&teamId=xxx` → use captured token to call revalidation for all team links → cache stampede DoS.
- **Risk rating:** MEDIUM — Requires network position, but missing HSTS lowers the barrier significantly on custom domains.

---

## Submission Priority

### Tier 1: Chains (highest impact × novelty × confidence)

| Priority | Chain | Bounty | Rationale |
|----------|-------|--------|-----------|
| 1 | C6 — Default Secret → Full Compromise | $10K–$25K | Highest bounty. Complete system compromise on self-hosted. Novelty: 5-domain key reuse pattern amplifies impact. |
| 2 | C10 — SAML Response Forgery | $8K–$18K | Most original attack pattern. SAML forgery via root secret reuse + auto-linking is novel and independently validated. |
| 3 | C1 — XSS Worm | $8K–$15K | Most reliably exploitable on any deployment. Production-quality PoC with worm propagation. |
| 4 | C2 — CORS + PDF (CVE-2026-57957 + CVE-2024-4367) | $5K–$15K | Involves papermark's own CVE + actively exploited PDF CVE. Best for hosted/Vercel bounty programs. |
| 5 | C8 — Embed Clickjacking + XSS | $5K–$12K | Novel clickjacking-in-embed-context attack. CSP bypassed and detection silent. |
| 6 | C9 — Rate Limit → OAuth Takeover | $4K–$10K | Multi-step authentication bypass with clear exploit path. |
| 7 | C3 — Zod Recon → Bulk PII Extraction | $3K–$8K | Clear GDPR/DPA violation. No authentication needed for bulk viewer email extraction. |
| 8 | C12 — HSTS + Query Param Intercept | $3K–$7K | Network-based attack enabled by missing HSTS on custom domains. |
| 9 | C5 — EOL Middleware Bypass | $3K–$8K | Framework EOL exploitation; no patch available. |
| 10 | C7 — CORS → Token Leak | $2K–$5K | Cross-origin info leak enabling Trigger.dev token generation. |
| 11 | C11 — CSP Amplifier | +$3K–$5K | Amplifier for C1; reported as combined chain with C1. |

### Tier 2: Standalone CRITICAL Findings

| Priority | ID | Title | CVSS | Rationale |
|----------|----|-------|------|-----------|
| 12 | F-001 | Conversations API Zero Auth | 9.1 | Trivially exploitable, no auth barrier, direct DB access. Backend for 4 chains. |
| 13 | F-002 | NEXTAUTH_SECRET Default | 10.0 | Root secret compromise. Usually folded into C6 report. |

### Tier 3: Standalone HIGH Findings

| Priority | ID | Title | CVSS | Rationale |
|----------|----|-------|------|-----------|
| 14 | F-004 | progress-token Unauthenticated Token | 7.5 | No auth, just a version CUID. |
| 15 | F-005 | Next.js v14 Unpatchable DoS | 7.5 | Framework EOL, no fix; WAF rule as mitigation. |
| 16 | F-003 | Stored XSS | 8.7 | Usually folded into C1. |
| 17 | F-006 | Document Password Key Default | 7.5 | Self-hosted only; folded into C6. |

### Tier 4: MEDIUM/LOW Standalone Findings

| Priority | ID | Title | Rationale |
|----------|----|-------|-----------|
| 18 | F-008 | CORS TUS (CVE-2026-57957) | Papermark-specific CVE, filed. |
| 19 | F-009 | record_reaction Unauthenticated | Data leak + DB write, no auth. |
| 20 | F-010 | feedback Unauthenticated | Data injection, no auth. |
| 21 | F-012 | Zod Error Disclosure | Aids all other attacks via schema recon. |
| 22 | F-011 | revalidate Query Param Secret | Log leakage + cache DoS. |
| 23 | F-014 | frame-ancestors * | Clickjacking enabler. |
| 24 | F-019 | Rate Limiter Fail-Open | Auth bypass enabler (C9). |
| 25 | F-020 | allowDangerousEmailAccountLinking | OAuth takeover enabler (C9, C10). |
| 26 | F-021 | pdfjs-dist CVE-2024-4367 | Usually folded into C2. |
| 27 | F-013 | cuid Predictable IDs | Authenticated IDOR enabler. |
| 28–30 | F-015, F-016, F-017 | CSP/Missing Headers | Defense-in-depth; amplifiers for chains. |
| 31+ | F-022–F-030 | LOW findings | Defense-in-depth, informational, or marginal impact. |

---

*Generated by `write-summary` — consumed by `validate-report` and `create-pr`.*
