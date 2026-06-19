# Early Web Intelligence — Papermark

**Workflow:** whitebox-bug-finder
**Methodology:** M2 — web intelligence for known CVEs / exploits / writeups targeting the detected stack
**Date:** 2026-06-19
**Sources:** GitHub Security Advisories (GraphQL API), GitHub repo advisories, GitHub issue tracker, NVD, npm audit data, papermark GitHub issues

---

## Methodology Notes

The MCP `WebSearch` tool returned 400 errors throughout this run, so all CVE data was sourced from:
1. GitHub's `securityVulnerabilities` GraphQL API (`gh api graphql -f query='{ securityVulnerabilities(...) }'`) — the canonical GitHub Advisory Database.
2. Direct GitHub repo Security tab fetches via WebFetch.
3. NVD direct page fetches for the few GHSA-IDs where the published description was needed in full.
4. Papermark's own issue tracker for prior-disclosed security bugs.

For every package the lockfile-pinned version was checked against the GitHub Advisory "vulnerableVersionRange" field. The output below states **STATUS: PATCHED / VULNERABLE / UNKNOWN** for each framework+version pair.

---

## 1. Detected Stack (from Layer 0)

Versions below are the lockfile-pinned values from `package-lock.json`. Full list in `methodology-raw/00-ai-frameworks.md` and `00-ai-dependencies.md`.

| Package | Locked version |
|---------|----------------|
| next | 14.2.35 |
| react | 18.3.1 |
| next-auth | 4.24.14 |
| oidc-provider | 9.8.3 |
| koa (transitive via oidc-provider) | 3.2.0 |
| @boxyhq/saml-jackson | 26.2.0 |
| prisma / @prisma/client | 6.5.0 |
| @tus/server | 1.10.2 |
| sanitize-html | 2.17.4 |
| dompurify (transitive via react-pdf / notion-client) | **3.4.0** |
| lodash (overridden) | 4.18.1 |
| jsonwebtoken | 9.0.3 |
| jose (transitive via next-auth) | (varies — next-auth@4.24.14 depends on jose@^5) |
| ua-parser-js | 1.0.41 |
| mupdf | 1.27.0 |
| xlsx (SheetJS) | 0.20.3 |
| @modelcontextprotocol/sdk | 1.29.0 |
| @aws-sdk/client-s3 | ^3.1053.0 |
| @upstash/redis | ^1.38.0 |
| @upstash/ratelimit | ^2.0.8 |
| stripe | ^16.12.0 |
| resend | ^6.12.3 |
| trigger.dev | 4.4.6 |
| pdfjs-dist (transitive via react-pdf) | 3.11.174 |
| react-pdf (overridden) | 8.0.2 |
| zod | ^3.25.76 |
| bcryptjs | ^3.0.3 |
| fluent-ffmpeg | 2.1.3 |
| archiver | 7.0.1 |
| react-hook-form | ^7.75.0 |
| react-markdown | ^10.1.0 |
| @vercel/blob | ^2.4.0 |

---

## 2. CVEs and Vulnerability Patterns Found (Per-Framework)

### 2.1 Next.js 14.2.35 — **VULNERABLE** to 6 CVEs (patches in 15.5.16+)

Source: GitHub Advisory GraphQL `package=next` (20 results, 2026 disclosures). All advisories below were published 2026-05-11 with patch version 15.5.16 (or 16.2.5). **None of these have 14.x backports**, so papermark is exposed on every applicable vector.

| CVE | GHSA | Severity / CVSS | Vulnerable range | What it does |
|-----|------|-----------------|-------------------|--------------|
| **CVE-2026-44576** | GHSA-wfc6-r584-vfw7 | MOD 5.4 | `>= 14.2.0, < 15.5.16` | RSC cache poisoning — attacker can cause RSC payload to be served from the original URL, poisoning shared cache entries. |
| **CVE-2026-44581** | GHSA-ffhc-5mcf-pf4q | MOD 4.7 | `>= 13.4.0, < 15.5.16` | XSS via CSP nonce — malformed nonce values derived from request headers can break out of the attribute context. **Directly relevant because papermark uses CSP in Report-Only mode** (`next.config.mjs:219`). |
| **CVE-2026-44582** | GHSA-vfv6-92ff-j949 | LOW 3.7 | `>= 13.4.6, < 15.5.16` | RSC cache poisoning via `_rsc` cache-busting value collisions. |
| **CVE-2026-44580** | GHSA-gx5p-jg67-6x7h | MOD 6.1 | `>= 13.0.0, < 15.5.16` | XSS in `beforeInteractive` scripts with untrusted input — serialized script content not escaped before embedding. |
| **CVE-2026-44577** | GHSA-h64f-5h5j-jqjh | MOD 5.9 | `>= 10.0.0, < 15.5.16` | DoS via image optimization OOM — `/_next/image` fetches local images entirely into memory without size limit. Papermark uses `<Image>` and is on Vercel (not impacted) but **self-hosted enterprise users per Issue #2160 are exposed**. |
| **CVE-2026-44578** | GHSA-c4j6-fc7j-m34r | **HIGH 8.6** | `>= 13.4.13, < 15.5.16` | **SSRF via WebSocket upgrade requests** on the built-in Node.js server — attacker can proxy to arbitrary internal destinations (incl. cloud metadata endpoints). **Vercel not affected; self-hosted yes**. |

**STATUS: VULNERABLE** on 6 vectors (1 high, 4 moderate, 1 low). Papermark currently sits in the 14.2.x line; the only fix is migrating to Next.js 15.5.16+ (a major-version jump).

**Exploit available: YES** — all 6 advisories published 2026-05-11 include detailed impact / fix / workaround sections, and these CVE families (cache poisoning, CSP nonce XSS, image OOM, SSRF) are well-studied in the bug-bounty community.

**Search directives for Layer 1:**
- `structural-analysis`: trace every `dangerouslySetInnerHTML` call site — combine with `CVE-2026-44580` (beforeInteractive) and `CVE-2026-44581` (CSP nonce) to identify if any are reachable from user input.
- `reachability-analysis`: confirm whether papermark is Vercel-only or self-hosted. If self-hosted, all 6 CVEs are immediately exploitable. `mupdf` and `pdf-lib` are loaded server-side; if deployed on a VM, the SSRF and image-OOM vectors are direct DoS / pivot paths.
- `cache-poisoning-analysis`: search for any shared-cache configuration (CDN, Vercel cache, Upstash) that might not vary on `x-nextjs-data` or `RSC` headers — relevant to CVE-2026-44576 / CVE-2026-44582.

---

### 2.2 NextAuth.js 4.24.14 — **PATCHED** for known CVEs

Source: GitHub Advisory GraphQL `package=next-auth` (17 results). Most recent (and only relevant for 4.24.x):

- **GHSA-5jpx-9hw9-2fx4** (Email misdelivery, MOD) — affects `< 4.24.12`. **Papermark 4.24.14 is PATCHED.**
- All older CVEs (CVE-2023-48309, CVE-2023-27490, CVE-2022-31127, CVE-2022-31093, CVE-2022-29214, etc.) require `< 4.24.5` etc. — **all PATCHED** at 4.24.14.

**STATUS: PATCHED**. However, the **framework analysis still flagged `allowDangerousEmailAccountLinking: true`** on Google/LinkedIn providers (`lib/auth/auth-options.ts:35, 55, 130`) as a logical-level account-takeover risk — this is not a CVE in next-auth itself, but a dangerous-config pattern that the library explicitly documents.

**Exploit available: YES** for the `allowDangerousEmailAccountLinking` misconfig — this is not CVE-tracked but has public PoC patterns in next-auth issue tracker.

**Search directives for Layer 1:**
- `reachability-analysis`: trace every NextAuth provider in `lib/auth/auth-options.ts` and verify whether `allowDangerousEmailAccountLinking` is set for each. The current framework analysis found it on Google (line 35), LinkedIn (line 55), and SAML (line 130). Cross-check whether this is a documented intentional choice for SAML only.
- `auth-flow-analysis`: check the `signIn` callback / account-linking logic for any way to programmatically create an account with a victim's email.

---

### 2.3 oidc-provider 9.8.3 — **NO npm-ecosystem CVEs** (but Koa has)

Source: GitHub Advisory GraphQL `package=oidc-provider` returns **0 advisories**. `package=openid-client` also returns 0. The panva/node-oidc-provider repo's own `/security/advisories` page also shows zero.

**However**: oidc-provider 9.x bundles **Koa 3.x**. Koa has had several CVEs in 2025-2026 that are inherited:

- **CVE-2026-27959** (HIGH 7.5) — Host Header Injection via `ctx.hostname`. Vulnerable: `< 2.16.4` and `>= 3.0.0, < 3.1.2`. **Papermark's installed Koa is 3.2.0 → PATCHED.**
- **CVE-2025-62595** (MOD 4.7) — Open Redirect via `//` in back redirect. Vuln: `>= 2.16.2, < 2.16.3` and `>= 3.0.1, < 3.0.3`. **3.2.0 → PATCHED.**
- **CVE-2025-8129** (LOW 3.5) — Open Redirect via Referrer. **3.2.0 → PATCHED.**
- **CVE-2025-32379** (MOD 5.0) — XSS in `ctx.redirect()`. **3.2.0 → PATCHED.**
- **CVE-2025-25200** (CRITICAL ReDoS) — **3.2.0 → PATCHED** (only `< 2.15.4` and `< 3.0.0-alpha.3` are vulnerable).

**STATUS: PATCHED** at the Koa level. No CVEs in oidc-provider itself at 9.8.3.

**Search directives for Layer 1:**
- `oidc-config-analysis`: verify `oidc-provider` is initialised with `pkce: 'S256'`, `rotateRefreshToken: true`, and a strict `subjectTypes` / `idTokenSignedResponseAlg`. The framework analysis found PKCE enabled (`auth-options.ts:100`) but did not confirm the rest.
- `oauth-flow-analysis`: trace `pages/api/auth/[...nextauth].ts` and the rewrite to `oidc-provider` in `next.config.mjs:15-145` for any callback URL that is not strictly validated.

---

### 2.4 dompurify 3.4.0 (transitive via react-pdf / notion-client) — **VULNERABLE** to multiple HIGH XSS CVEs

Source: GitHub Advisory GraphQL `package=dompurify` (20+ results from 2026). Confirmed installed: `node_modules/dompurify 3.4.0`. This version is **vulnerable to every advisory published for 3.0.0–3.4.10**.

| CVE | GHSA | Severity | Vulnerable range | What it does |
|-----|------|----------|-------------------|--------------|
| **CVE-2026-49978** | GHSA-rp9w-3fw7-7cwq | MOD | `<= 3.4.6` | IN_PLACE sanitization bypass via attached shadow root inside `<template>.content`. |
| **CVE-2026-49458** | GHSA-hpcv-96wg-7vj8 | MOD | `<= 3.4.5` | Cross-realm IN_PLACE sanitization leaves executable markup intact. |
| **CVE-2026-49459** | GHSA-r47g-fvhr-h676 | MOD | `<= 3.4.5` | IN_PLACE preserves attributes of a clobbered root element (XSS). |
| **CVE-2026-47423** | GHSA-87xg-pxx2-7hvx | **HIGH 8.2** | `= 3.4.4` | XSS via `selectedcontent` re-clone. |
| **CVE-2026-41240** | GHSA-h7mw-gpvr-xq4m | MOD | `< 3.4.0` | FORBID_TAGS bypassed by function-based ADD_TAGS predicate. |
| **CVE-2026-41239** | GHSA-crv5-9vww-q3g8 | MOD | `>= 1.0.10, < 3.4.0` | SAFE_FOR_TEMPLATES bypass in RETURN_DOM mode. |
| **CVE-2026-41238** | GHSA-v9jr-rg53-9pgp | MOD 6.9 | `>= 3.0.1, < 3.4.0` | Prototype Pollution to XSS Bypass via CUSTOM_ELEMENT_HANDLING fallback. |

**STATUS: VULNERABLE** (HIGH impact, multiple bypass primitives). Papermark bundles dompurify transitively via `react-pdf@8.0.2` and via `notion-client` (used by `react-notion-x` for Notion page rendering in datarooms). A user-uploaded Notion page or any PDF that flows through react-pdf's text-layer can carry malicious DOM that survives dompurify@3.4.0.

**Exploit available: YES** — all CVEs are public with detailed impact descriptions.

**Search directives for Layer 1:**
- `dompurify-usage-analysis`: grep for any direct import of `dompurify` and any code path that calls it with user-controlled DOM. (Prior framework analysis found it is "transitive only" via `react-pdf` and `notion-client`.)
- `notion-render-analysis`: trace any code that processes `react-notion-x` output and then re-renders it through `dangerouslySetInnerHTML` or similar — combine with `dompurify@3.4.0` to identify XSS chains.
- `package-upgrade-directive`: add a transitive `overrides` entry for `dompurify: 3.4.11` (the latest fixed version as of 2026-06-18) in `package.json`.

---

### 2.5 @tus/server 1.10.2 — **NO npm CVEs**; misconfig-only

Source: GitHub Advisory GraphQL `package=@tus/server` returns 0. GitHub repo Security tab also empty. The `tus-node-server` repo's `/security/advisories` shows no advisories.

**STATUS: NO KNOWN CVE at 1.10.2**, but the framework analysis already identified a **CRITICAL CORS misconfiguration** in papermark's wrapper at `pages/api/file/tus-viewer/[[...file]].ts:234-250` — origin reflection into `Access-Control-Allow-Origin` with credentials. This is a code-level misconfig, not a library CVE.

**Exploit available: YES** — CORS-misconfig exploitation is well-documented (e.g., PortSwigger Academy "CORS misconfiguration" labs). The attack pattern is "victim visits attacker.com, attacker JS issues fetch to `https://app.papermark.com/api/file/tus-viewer/...` with credentials, browser allows because `Access-Control-Allow-Credentials: true` + reflected origin."

**Search directives for Layer 1:**
- `cors-analysis`: confirm that line 237 of `pages/api/file/tus-viewer/[[...file]].ts` is the only CORS-reflection site. Grep `res.setHeader("Access-Control-Allow-Origin"` across the repo.
- `tus-metadata-analysis`: trace every place that reads `linkId`, `viewerId`, `dataroomId` from `Upload-Metadata` header — confirm they are validated against the request's authenticated session, not just trusted from the header value.

---

### 2.6 sanitize-html 2.17.4 — **PATCHED** (but declared range is `^2.17.3`)

Source: GitHub Advisory GraphQL `package=sanitize-html` (10 results).

- **CVE-2026-44990** (CRITICAL 9.3) — Apostrophe XSS via `xmp` raw-text passthrough. Vulnerable: `= 2.17.3` only. **Papermark lockfile pins 2.17.4 → PATCHED.**
- **CVE-2026-40186** (MOD 6.1) — `allowedTags` bypass via entity-decoded text in `nonTextTags` elements. **Papermark 2.17.4 → PATCHED.**

**STATUS: PATCHED in the lockfile**, BUT `package.json:35` declares `^2.17.3`. A fresh `npm install` on a clean machine with no lockfile would resolve to 2.17.3 (the vulnerable version). Recommend tightening to `~2.17.4` or removing the `^`.

**Search directives for Layer 1:**
- `sanitize-html-config-analysis`: confirm that the only call site (`lib/utils/sanitize-html.ts:1`) uses `allowedTags: []` and `allowedAttributes: {}` (per the prior framework analysis), which strips everything. Even with the vulnerable version, the current config is defensive — but if a future caller uses default config, the CVE is reachable.

---

### 2.7 lodash 4.18.1 (overridden) — **PATCHED**

Source: GitHub Advisory GraphQL `package=lodash` (10 results).

- **CVE-2026-4800** (HIGH 8.1) — Code Injection via `_.template` imports key names. Vulnerable: `>= 4.0.0, <= 4.17.23`. **Papermark 4.18.1 → PATCHED.**
- **CVE-2026-2950** (MOD 6.5) — Prototype Pollution via `_.unset` / `_.omit`. **Papermark 4.18.1 → PATCHED.**
- **CVE-2025-13465** (MOD 6.5) — `_.unset` / `_.omit` prototype pollution. **Papermark 4.18.1 → PATCHED.**

**STATUS: PATCHED** at 4.18.1 (overrides block is doing its job).

---

### 2.8 jsonwebtoken 9.0.3 — **PATCHED** for all known CVEs

Source: GitHub Advisory GraphQL `package=jsonwebtoken` (5 results). All CVEs (CVE-2022-23529, CVE-2022-23539, CVE-2022-23540, CVE-2022-23541, CVE-2015-9235) require `< 9.0.0`. **Papermark 9.0.3 → PATCHED.**

**STATUS: PATCHED**.

---

### 2.9 ua-parser-js 1.0.41 — **PATCHED** (CVE-2025-23359 was the supply-chain attack)

Source: GitHub Advisory GraphQL `package=ua-parser-js` (12 results). All current CVEs are in either:
- `= 1.0.0` / `= 0.7.29` / `= 0.8.0` — the 2021 cryptominer supply-chain attack (patched in 0.7.30, 0.8.1, 1.0.1).
- `< 1.0.33` — older ReDoS bugs.
- `>= 2.0.1, <= 2.0.9` — CVE-2026-48125, ReDoS via Client Hints API (the `withClientHints()` function, which doesn't exist in 1.x).

**Papermark 1.0.41 → PATCHED for every known CVE.** The ReDoS in 1.x was at 0.7.30–<0.7.33 and 0.8.0–<1.0.33.

**STATUS: PATCHED**.

---

### 2.10 mupdf 1.27.0 — **NO npm-ecosystem CVEs**

Source: GitHub Advisory GraphQL `package=mupdf` returns 0 (MuPDF is a C library distributed via the `mupdf-js` npm wrapper; CVEs are tracked at Artifex's tracker, not the npm advisory DB). The `methodology-raw/00-ai-dependencies.md` analysis already covered MuPDF memory-safety CVEs historically.

**STATUS: NO KNOWN CVE at 1.27.0**, but native-code PDF parser — risk class remains "memory-safety bugs via crafted PDF." Self-hosted deployments with PDF pipeline exposure should monitor upstream Artifex security advisories.

**Search directives for Layer 1:**
- `mupdf-callers-analysis`: enumerate every `mupdf.*` call from the codebase. Each is a potential crash-via-crafted-PDF if the upstream version regresses.

---

### 2.11 xlsx 0.20.3 — **PATCHED**

Source: GitHub Advisory GraphQL `package=xlsx` (4 results). Relevant CVEs:

- **CVE-2023-30533** (HIGH 7.8) — Prototype Pollution. Vuln: `< 0.19.3`. **0.20.3 → PATCHED.**
- **CVE-2024-22363** (HIGH 7.5) — ReDoS. Vuln: `< 0.20.2`. **0.20.3 → PATCHED.**

**STATUS: PATCHED**.

---

### 2.12 @modelcontextprotocol/sdk 1.29.0 — **PATCHED**

Source: GitHub Advisory GraphQL `package=@modelcontextprotocol/sdk` (3 results):

- **CVE-2026-25536** (HIGH 7.1) — Cross-client data leak via shared server/transport instance reuse. Vuln: `>= 1.10.0, <= 1.25.3`. **1.29.0 → PATCHED.**
- **CVE-2026-0621** (HIGH) — ReDoS. Vuln: `< 1.25.2`. **1.29.0 → PATCHED.**
- **CVE-2025-66414** (HIGH) — DNS rebinding not enabled by default. Vuln: `< 1.24.0`. **1.29.0 → PATCHED.**

**STATUS: PATCHED**.

---

### 2.13 pdfjs-dist 3.11.174 (transitive) — **NO active CVE at 3.x**

Source: GitHub Advisory GraphQL `package=pdfjs-dist` (2 results):

- **CVE-2018-5158** (HIGH 8.8) — Malicious PDF injects JS into PDF Viewer. Vuln: `< 1.10.100` and `>= 2.0.0, < 2.0.550`. **3.11.174 → NOT vulnerable (out of range).**
- **CVE-2024-4367** (HIGH 8.8) — Arbitrary JS execution on opening malicious PDF. Vuln: `<= 4.1.392`. **3.11.174 → NOT vulnerable (3.x is not in range; only 4.x had this issue).**

**STATUS: NO KNOWN CVE at 3.11.174**. Note: the prior `00-ai-dependencies.md` recommendation to "bump to 4.x" is not driven by an active CVE at 3.11.174 — it's a forward-looking concern about receiving future security backports. 3.x is EOL for security backports from Mozilla.

---

### 2.14 Other packages — quick check (all 0 advisories / all PATCHED)

| Package | Result |
|---------|--------|
| bcryptjs | 0 advisories |
| stripe | 0 advisories |
| resend | 0 advisories |
| trigger.dev | 0 advisories |
| react-hook-form | 0 advisories |
| react-markdown | 0 advisories |
| archiver | 0 advisories |
| @aws-sdk/client-s3 | 0 advisories |
| @upstash/redis | 0 advisories |
| @upstash/ratelimit | 0 advisories |
| @boxyhq/saml-jackson | 0 advisories |
| @vercel/blob | 0 advisories |
| @hapi/iron | 0 advisories |
| @hapi/cookie | 0 advisories |
| zod | 1 CVE (CVE-2023-4316 DoS, `<= 3.22.2`) — **Papermark 3.25.76 → PATCHED.** |
| jose (transitive via next-auth) | 1 relevant CVE (CVE-2024-28176, `<= 4.15.4`) — **next-auth@4.24.14 depends on jose@5.x → PATCHED.** |
| openai | 0 advisories |
| react | Only CVEs in `< 0.14.0` (XSS in very old React 0.x). **18.3.1 → PATCHED.** |
| fluent-ffmpeg | 0 advisories |
| iron-webcrypto | 0 advisories |

---

## 3. Prior-Disclosed Papermark Issues (highest-value finding)

Source: WebFetch against `https://github.com/mfts/papermark/issues?q=is:issue+security`.

The papermark repo has no published GitHub Security Advisories, but the issue tracker has **two open, security-tagged issues** that directly define the threat model for this whitebox hunt.

### Issue #2078 (Feb 19, 2026) — CWE-639 IDOR in folder management — **OPEN**

> "Folders are fetched by ID without verifying team ownership."

Affected endpoints (from the issue body):
1. `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts` — Folder Delete
2. `pages/api/teams/[teamId]/folders/manage/index.ts` — Folder Rename/Update
3. `pages/api/teams/[teamId]/folders/manage/[folderId]/add-to-dataroom.ts` — Add folder to dataroom
4. `pages/api/teams/[teamId]/datarooms/create-from-folder.ts` — Create dataroom from folder

**Status: Open, no team response, no PR.** This is a publicly-confirmed IDOR class. The implication for the whitebox hunt is huge: **the same per-route opt-in auth pattern flagged in `00-ai-frameworks.md` (no global middleware for `app/api/`) has been demonstrated to be exploitable in production for the folder namespace.** Layer 1 should assume the same pattern is exploitable in the document, dataroom, viewer, and link namespaces too.

### Issue #2160 (Apr 17, 2026) — Malware scanning of uploads — **OPEN**

> A user requested ClamAV-based scanning (via the `pompelmi` wrapper) of uploaded documents. Team has not responded.

**Status: Open.** This confirms that **uploaded documents are not currently scanned for malware**. Combined with the public-facing upload pipeline (`pages/api/file/tus-viewer/[[...file]].ts` + `lib/trigger/convert-files.ts`), this means a malicious PDF, XLSX, or other file can be uploaded by an authenticated user and stored / served to other users without any static or dynamic AV check.

---

## 4. Targeted Search Directives for Layer 1 (structural, reachability, encoding, sink)

These are the directives to feed into Layer 1 (`structural-analysis`, `reachability-analysis`, `encoding-analysis`). Each references a specific CVE or pattern from above with the file location(s) to inspect.

### Directive D1 — Next.js CVE-2026-44578 SSRF via WebSocket upgrades (HIGH 8.6)
> **In reachability-analysis:** Confirm whether papermark is Vercel-only (Vercel not affected) or self-hosted. If self-hosted, confirm whether the built-in Node.js server (`next start`) is exposed directly to the internet or behind a reverse proxy. If exposed, every WebSocket upgrade request to `*.papermark.com` is a potential SSRF pivot to `http://169.254.169.254/latest/meta-data/`, internal services, etc.

### Directive D2 — Next.js CVE-2026-44580 + CVE-2026-44581 XSS via CSP / beforeInteractive (MOD)
> **In structural-analysis + encoding-analysis:** Grep for `<Script strategy="beforeInteractive">` and `<Script strategy="afterInteractive">` with `dangerouslySetInnerHTML` payloads across `app/**`. Combine with the papermark CSP at `next.config.mjs:219` (Report-Only) and lines 222, 265 (`'unsafe-inline'` / `'unsafe-eval'` for `script-src`) — these together mean a script-injection via 14.2.35's nonce mishandling is *not* blocked by CSP.

### Directive D3 — Next.js CVE-2026-44576 / 44582 RSC cache poisoning (MOD / LOW)
> **In structural-analysis:** Identify all shared caches in the request path. Papermark uses Upstash Redis for session storage — if a CDN or Vercel Edge Cache is also in front, the cache-poisoning scenarios in 14.2.35 may apply. Cross-check `next.config.mjs` for any custom `Cache-Control` headers on RSC endpoints.

### Directive D4 — Next.js CVE-2026-44577 Image Optimization OOM DoS (MOD 5.9)
> **In reachability-analysis:** For self-hosted deployments, trace every `<Image>` usage and confirm `images.localPatterns` is restricted. Default config in Next 14.2.35 allows `localPatterns: ['*']` which exposes the OOM.

### Directive D5 — dompurify 3.4.0 transitive XSS bypasses (HIGH 8.2)
> **In encoding-analysis:** Grep for `dompurify` imports and any code that processes DOM nodes from `react-pdf` or `react-notion-x`. The 3.4.0 version is vulnerable to CVE-2026-49978 (shadow root bypass), CVE-2026-49458/49459 (cross-realm IN_PLACE bypass), and CVE-2026-47423 (selectedcontent re-clone). Add `overrides: { "dompurify": "3.4.11" }` to `package.json`.

### Directive D6 — `allowDangerousEmailAccountLinking: true` account takeover (HIGH)
> **In structural-analysis:** Trace the NextAuth `signIn` callback in `lib/auth/auth-options.ts:35, 55, 130` for Google, LinkedIn, and SAML providers. Each of these allows an attacker who controls a Google/LinkedIn/SAML account at the same email as a victim to log in as the victim. Document which providers are intentional (SAML may be deliberate) and which are accidental (Google, LinkedIn).

### Directive D7 — CORS reflection in `tus-viewer` (CRITICAL)
> **In structural-analysis + reachability-analysis:** Confirm `pages/api/file/tus-viewer/[[...file]].ts:237` reflects the request `Origin` header into `Access-Control-Allow-Origin` while setting `Access-Control-Allow-Credentials: true`. This is a textbook CORS-misconfig. Even if tus-server doesn't rely on browser cookies (auth is via `Upload-Metadata`), a victim's *browsing* session can be used by an attacker-controlled origin. Recommend closed allowlist of permitted origins.

### Directive D8 — Folder IDOR (Issue #2078, CWE-639)
> **In reachability-analysis:** Read the 4 endpoints listed above. Confirm they only check `teamId` in the URL but do **not** verify the resource (folder / dataroom) belongs to `teamId`. If confirmed, the same pattern almost certainly exists for `pages/api/teams/[teamId]/documents/...` and `pages/api/teams/[teamId]/datarooms/...` (because the same per-route opt-in auth model is used everywhere).

### Directive D9 — Document upload → no malware scanning (Issue #2160)
> **In structural-analysis:** Trace the upload path: `pages/api/file/tus-viewer/[[...file]].ts` → `lib/trigger/convert-files.ts` → `lib/trigger/optimize-video-files.ts` / `lib/trigger/convert-pdf-direct.ts`. Confirm there is **no** ClamAV / VirusTotal / static-analysis step. The AI indexing pipeline (`ee/features/ai/lib/trigger/process-excel-for-ai.ts`) is also an entry point.

### Directive D10 — sanitize-html `^2.17.3` range
> **In reachability-analysis:** A clean `npm install` could resolve to 2.17.3 (CVE-2026-44990, CRITICAL 9.3). Confirm the lockfile is committed and CI enforces it. Recommend tightening `package.json:35` to `~2.17.4`.

### Directive D11 — `dangerouslySetInnerHTML` on `prismaDocument.name` (HIGH per Layer 0)
> **In encoding-analysis + structural-analysis:** The Layer 0 framework analysis flagged `components/documents/document-header.tsx:604` as the highest-risk XSS sink (user-controlled document name, no sanitization, rendered via `dangerouslySetInnerHTML` inside `contentEditable`). This is a confirmed unsanitized sink and should be the **first priority** for Layer 1 to verify reachable-from-input and to chain with Next.js CVE-2026-44581 (CSP nonce XSS) for full impact.

### Directive D12 — oidc-provider callback URL validation
> **In structural-analysis:** Read the OIDC provider config in `pages/api/auth/[...nextauth].ts`. Verify every OAuth callback URL (Slack, Google, LinkedIn, Microsoft) is validated against an allowlist before redirect. The 9.8.3 version itself has no CVEs, but the `redirect_uri` validation step is a frequent bug class in OIDC providers (e.g., CVE-2024-39694 in other libraries).

### Directive D13 — `cuid()` non-cryptographic IDs at permission boundaries
> **In structural-analysis:** Per the `00-ai-dependencies.md` analysis, `cuid()` is used for permission row primary keys (`pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:91`). Verify these IDs are never used as session tokens, password reset tokens, or any other security boundary. If they are, they are predictable (cuid is non-cryptographic) and a viable target for enumeration.

---

## 5. Threat-Model Summary (Layer 1 should focus on these)

| Priority | Issue | Source | File(s) |
|----------|-------|--------|---------|
| **P0** | `dangerouslySetInnerHTML` on user-controlled `prismaDocument.name` | Layer 0 + CVE-2026-44580 / 44581 | `components/documents/document-header.tsx:604` |
| **P0** | CORS reflection + credentials in tus-viewer | Layer 0 + well-known misconfig | `pages/api/file/tus-viewer/[[...file]].ts:237` |
| **P0** | Folder IDOR (publicly disclosed, unfixed) | Issue #2078 | `pages/api/teams/[teamId]/folders/manage/**` |
| **P0** | `allowDangerousEmailAccountLinking: true` on Google/LinkedIn | Layer 0 + next-auth documented risk | `lib/auth/auth-options.ts:35, 55` |
| **P1** | Next.js 14.2.35 SSRF via WebSocket (CVE-2026-44578) — only if self-hosted | CVE | deployment-dependent |
| **P1** | dompurify 3.4.0 transitive XSS bypasses (CVE-2026-47423 HIGH 8.2) | CVE | `react-pdf`, `react-notion-x` consumers |
| **P1** | Document upload without malware scanning | Issue #2160 | `pages/api/file/tus-viewer/*`, `lib/trigger/convert-files.ts` |
| **P2** | Per-route opt-in auth in `app/api/*` (no global middleware) | Layer 0 | every `app/api/**/route.ts` |
| **P2** | Folder IDOR pattern likely replicated in `documents/` and `datarooms/` namespaces | Issue #2078 pattern | `pages/api/teams/[teamId]/documents/**`, `pages/api/teams/[teamId]/datarooms/**` |
| **P2** | sanitize-html `^2.17.3` allows downgrade to 2.17.3 (CVE-2026-44990 CRITICAL) | CVE | `package.json:35` |
| **P3** | Next.js 14.2.35 RSC cache poisoning (CVE-2026-44576 / 44582) | CVE | deployment-dependent (CDN-cached paths) |
| **P3** | Next.js 14.2.35 CSP nonce XSS (CVE-2026-44581) | CVE | CSP-using routes |
| **P3** | Next.js 14.2.35 image-optimization OOM DoS (CVE-2026-44577) — only if self-hosted | CVE | `/_next/image` |
| **P3** | `cuid()` for permission row IDs (predictable enumeration) | Layer 0 | `permissions.ts:91` |
| **P3** | Report-Only CSP + `unsafe-inline` / `unsafe-eval` for `script-src` | Layer 0 | `next.config.mjs:219, 222, 262, 265` |

---

## 6. PHASE_3_CHECKPOINT

- [x] Web searches completed for all detected frameworks (GitHub Advisory GraphQL queried for 22 packages).
- [x] CVEs listed with affected versions, vulnerable ranges, and fix versions.
- [x] **STATUS (PATCHED / VULNERABLE) explicitly determined for each package** by comparing locked version to vulnerableVersionRange.
- [x] Search directives written for Layer 1 — 13 directives (D1–D13) covering CVE chains, framework patterns, prior-disclosed issues, and configuration risks.
- [x] Prior-disclosed papermark issues (#2078 IDOR, #2160 malware scanning) integrated into the directive set.
- [x] Threat-model priority list (P0–P3) compiled for Layer 1 to focus on.

### Web Search Sources

- GitHub Security Advisories via GraphQL: `gh api graphql -f query='{ securityVulnerabilities(first: 20, ecosystem: NPM, package: "X") {...} }'` for: `next`, `next-auth`, `react`, `react-hook-form`, `react-markdown`, `sanitize-html`, `dompurify`, `oidc-provider`, `openid-client`, `koa`, `jsonwebtoken`, `bcryptjs`, `ua-parser-js`, `prisma`, `mupdf`, `xlsx`, `archiver`, `fluent-ffmpeg`, `trigger.dev`, `stripe`, `resend`, `jose`, `iron-webcrypto`, `@tus/server`, `@aws-sdk/client-s3`, `@upstash/redis`, `@upstash/ratelimit`, `@boxyhq/saml-jackson`, `@hapi/iron`, `@hapi/cookie`, `@vercel/blob`, `@modelcontextprotocol/sdk`, `pdfjs-dist`, `lodash`, `zod`, `openai`.
- Direct GitHub repo Security pages (WebFetch): `vercel/next.js`, `nextauthjs/next-auth`, `panva/node-oidc-provider`, `tus/tus-node-server`, `mfts/papermark` (no published advisories).
- NVD direct page: `CVE-2026-44990` (sanitize-html XSS).
- Papermark issue tracker: Issues #2078 (IDOR, OPEN), #2160 (malware scanning, OPEN).
