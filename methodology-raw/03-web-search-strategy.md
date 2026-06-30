# Web Search Strategy — papermark bug bounty

**Node:** Layer-3 Web Search Strategy (M4, light tier)
**Date:** 2026-06-30
**Inputs:** `00-ai-frameworks.md`, `00-ai-dependencies.md`, `02-synthesized-secrets.md`, `02-synthesized-sast.md`, `02-synthesized-dependencies.md`, `02-synthesized-git-history.md`

---

## Per-Domain Search Agenda

### DEPS — Dependency Vulnerability Queries (15 queries)

| # | Query | Serves Finding | Query Type | Priority |
|---|-------|---------------|------------|----------|
| D-01 | `cuid2 predictable id generation exploitation proof of concept bug bounty` | S-001 (cuid HIGH) | exploit-writeup / PoC | HIGH |
| D-02 | `cuid vs nanoid security comparison id prediction attack sequential counter` | S-001 (cuid HIGH) | technique / comparison | HIGH |
| D-03 | `xlsx prototype pollution GHSA-4r6h-8v6p-xvw6 exploit spreadsheet upload` | S-002 (xlsx PP MEDIUM) | CVE exploit-writeup | HIGH |
| D-04 | `xlsx sheetjs ReDoS GHSA-5pgg-2g8v-p4x9 denial of service regex backtracking` | S-002 (xlsx ReDoS MEDIUM) | CVE exploit-writeup | MEDIUM |
| D-05 | `npm dependency pinned CDN URL supply chain attack no integrity hash mitigation` | S-002 (xlsx CDN MEDIUM) | technique | MEDIUM |
| D-06 | `pdfjs-dist arbitrary javascript execution GHSA-wgrm-67xf-hhpq malicious PDF` | S-004 (pdfjs-dist MEDIUM) | CVE exploit-writeup | HIGH |
| D-07 | `react-pdf pdfjs-dist XSS bypass malicious PDF viewer exploitation` | S-004 (pdfjs-dist MEDIUM) | technique / bug bounty writeup | HIGH |
| D-08 | `Next.js 14 RSC Flight protocol denial of service CVE-2026-23864 CVE-2026-23869 memory exhaustion` | S-003 (Next.js DoS HIGH) | CVE exploit-writeup | HIGH |
| D-09 | `sanitize-html xmp tag bypass CVE-2026-44990 raw text element injection` | S-002 (SAST XSS) / deps cross-ref | CVE PoC | HIGH |
| D-10 | `Prisma 6 SQL injection bypass technique bug bounty 2026` | Stack-level (Prisma 6.5.0) | technique / bug bounty writeup | MEDIUM |
| D-11 | `NextAuth.js 4 vulnerability exploitation session handling misconfiguration` | Stack-level (NextAuth 4.24.13) | technique / bug bounty writeup | MEDIUM |
| D-12 | `TUS protocol server CORS misconfiguration upload exploitation bypass` | Stack-level (TUS 1.10.2) | technique / bug bounty writeup | HIGH |
| D-13 | `oidc-provider node js vulnerability exploitation DCR dynamic registration` | S-005 (unused deps LOW) | CVE / technique | LOW |
| D-14 | `cookie package prefix injection __Host __Secure bypass CVE-2024-47764` | S-003 (cookie 0.4.2 MEDIUM) | CVE technique | LOW |
| D-15 | `engine.io socket.io cookie vulnerability transitive dependency exploitation` | S-003 (transitive chain) | technique | LOW |

---

### SAST — Static Analysis Pattern Queries (14 queries)

| # | Query | Serves Finding | Query Type | Priority |
|---|-------|---------------|------------|----------|
| S-01 | `Next.js API route missing authentication zero auth bypass exploitation bug bounty` | S-001 (conversations CRITICAL) | bug bounty writeup | CRITICAL |
| S-02 | `Next.js catch-all route unauthenticated handler privilege escalation` | S-001 (conversations CRITICAL) | technique | CRITICAL |
| S-03 | `sanitize-html decode after strip entity encoding XSS bypass stored cross site` | S-002 (stored XSS HIGH) | technique / writeup | HIGH |
| S-04 | `dangerouslySetInnerHTML stored XSS document name header React Next.js bug bounty` | S-002 (stored XSS HIGH) | bug bounty writeup | HIGH |
| S-05 | `Trigger.dev public access token unauthenticated API createPublicToken exploitation` | S-003 (progress-token HIGH) | technique | HIGH |
| S-06 | `Next.js 14 App Router RSC POST denial of service unpatched vulnerability` | S-004 (Next.js DoS HIGH) | CVE technique | HIGH |
| S-07 | `CORS origin reflection with credentials true CSRF file upload exploit` | S-005 (TUS CORS MEDIUM) | bug bounty writeup | HIGH |
| S-08 | `unauthenticated POST endpoint database write no session check bug bounty` | S-006 + S-007 (record_reaction/feedback MEDIUM) | bug bounty writeup | HIGH |
| S-09 | `Next.js revalidation API secret token query parameter leakage exploit` | S-008 (revalidate MEDIUM) | technique | MEDIUM |
| S-10 | `dangerouslySetInnerHTML helpText prop secondary XSS sink React component` | S-009 (helpText XSS MEDIUM) | technique / bug bounty | MEDIUM |
| S-11 | `Zod validation error format information disclosure API schema enumeration` | S-010 (Zod info disclosure MEDIUM) | technique / bug bounty | MEDIUM |
| S-12 | `Python xml etree parse XXE file read server side request forgery docx` | S-012 (XXE docx-sanitizer MEDIUM) | technique / CVE | MEDIUM |
| S-13 | `non-literal RegExp ReDoS Node js workflow engine condition injection` | S-013 (workflow ReDoS LOW) | technique | LOW |
| S-14 | `XSS via dangerouslySetInnerHTML React shiki syntax highlighted webhook body` | S-010 (webhook-events LOW) | technique | LOW |

---

### SECRETS — Credential / Secret Exposure Queries (12 queries)

| # | Query | Serves Finding | Query Type | Priority |
|---|-------|---------------|------------|----------|
| K-01 | `NEXTAUTH_SECRET default value next auth session forgery exploitation` | S-001 (default secret CRITICAL) | exploit-writeup | CRITICAL |
| K-02 | `NextAuth.js default secret session token hijacking CVE known compromise` | S-001 (default secret CRITICAL) | CVE / technique | CRITICAL |
| K-03 | `boxyhq saml jackson NEXTAUTH_SECRET derived encryption key compromise SSO` | S-001 (cross-domain key CRITICAL) | technique | CRITICAL |
| K-04 | `NextAuth JWT session token forgery with known secret signing key` | S-001 (JWT forgery CRITICAL) | PoC / technique | CRITICAL |
| K-05 | `AES-256-CTR document password encryption default key exploit decryption` | S-002 (doc password key HIGH) | exploit-writeup | HIGH |
| K-06 | `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY default value encrypted password decrypt` | S-002 (doc password key HIGH) | technique / PoC | HIGH |
| K-07 | `SHA-256 key derivation without salt deterministic encryption weakness` | S-002 (key derivation HIGH) | technique | HIGH |
| K-08 | `dot env example default credentials production deployment vulnerability bug bounty` | S-001 + S-002 (pattern) | bug bounty writeup | HIGH |
| K-09 | `HMAC-SHA256 known secret key signature forgery download token bypass` | S-001 (signed tokens) | technique | HIGH |
| K-10 | `Next.js API token hash preimage attack known secret credential recovery` | S-001 (API tokens) | technique | MEDIUM |
| K-11 | `self-hosted Next.js deployment common misconfiguration secrets checklist bug bounty 2026` | S-001 + S-002 (deployment risk) | bug bounty writeup | MEDIUM |
| K-12 | `Trigger.dev project ID hardcoded recon attack surface information disclosure` | S-003 (Trigger.dev ID LOW) | technique / recon | LOW |

---

### GIT-HISTORY — Historical Pattern / Framework-Level Queries (12 queries)

| # | Query | Serves Finding / Pattern | Query Type | Priority |
|---|-------|--------------------------|------------|----------|
| G-01 | `Next.js API routes authentication bypass pattern middleware exclusion bug bounty 2026` | Pattern: auth gap by omission | bug bounty writeup | HIGH |
| G-02 | `sanitize-html entity decode bypass stored cross site scripting strip tags` | Pattern: sanitizer decode ordering | technique / writeup | HIGH |
| G-03 | `unauthenticated feedback webhook data injection endpoint bug bounty Next.js` | Pattern: tracking endpoints → DB vectors | bug bounty writeup | HIGH |
| G-04 | `Next.js 14 end of life unpatched vulnerabilities 2026 upgrade required` | Pattern: EOL framework risk | CVE roundup | HIGH |
| G-05 | `CORS misconfiguration origin reflect with credentials file upload CSRF writeup` | Pattern: CORS + TUS upload | bug bounty writeup | HIGH |
| G-06 | `SSRF bypass Next.js server side request forgery technique 2026` | Historical VFC (fixed, pattern for context) | technique | MEDIUM |
| G-07 | `open redirect callbackUrl bypass NextAuth normalization encoding` | Historical VFC (fixed, pattern) | technique | MEDIUM |
| G-08 | `Next.js middleware bypass matcher exclusion leading to unprotected routes` | Pattern: middleware exclusions | technique | HIGH |
| G-09 | `Python xml etree parse XXE document conversion pipeline exploit` | Pattern: XXE in conversion pipeline | technique / PoC | MEDIUM |
| G-10 | `TUS protocol resumable upload CSRF CORS security analysis` | Pattern: TUS + CORS cross-domain | technique / security analysis | HIGH |
| G-11 | `Prisma ORM mass assignment field exposure data leak bug bounty` | Stack-level (Prisma pattern) | bug bounty writeup | MEDIUM |
| G-12 | `Next.js 14 security misconfiguration chain exploitation multiple findings writeup` | Stack-level (multi-finding chaining) | bug bounty writeup | HIGH |

---

## Summary

| Domain | Queries | Priority Distribution |
|--------|---------|----------------------|
| DEPS | 15 | 6 HIGH, 6 MEDIUM, 3 LOW |
| SAST | 14 | 4 CRITICAL, 6 HIGH, 3 MEDIUM, 1 LOW |
| SECRETS | 12 | 4 CRITICAL, 5 HIGH, 2 MEDIUM, 1 LOW |
| GIT-HISTORY | 12 | 7 HIGH, 4 MEDIUM, 1 LOW |
| **Total** | **53** | **8 CRITICAL, 24 HIGH, 15 MEDIUM, 6 LOW** |

## Consumed By

- `web-search-deps` — runs D-01 through D-15
- `web-search-sast` — runs S-01 through S-14
- `web-search-secrets` — runs K-01 through K-12
- `web-search-git-history` — runs G-01 through G-12
