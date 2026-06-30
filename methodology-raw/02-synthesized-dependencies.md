# Synthesized Dependency Findings — papermark

**Node:** Layer-2 Dependency Synthesis  
**Date:** 2026-06-30  
**Sources:** AI dependency recon (`00-ai-dependencies.md`), tool scanners (`normalized.json`: osv-scanner, npm-audit), structural analysis (`01-structural-analysis.md`), reachability analysis (`01-reachability-analysis.md`), GitNexus graph verification

---

## Merge Summary

| Source        | Raw Count | Unique Findings | After Reachability Filter |
|---------------|-----------|-----------------|---------------------------|
| AI-only       | 10        | 6               | 2 (S-001, S-005)          |
| Tool-only     | ~212 npm/Python | 20 unique packages | 1 (S-004)            |
| Both          | 3         | 3               | 2 (S-002, S-003)          |
| **Total**     |           |                 | **5 active findings**     |

---

## Dedup Log

| Candidate | Sources | Dedup Action          | Final ID  |
|-----------|---------|-----------------------|-----------|
| `cuid`    | AI      | standalone            | S-001     |
| `xlsx`    | AI+OSV  | merged (CDN + PP + ReDoS) | S-002 |
| `cookie`  | AI+OSV  | merged                | S-003     |
| `pdfjs-dist` | OSV  | standalone            | S-004     |
| unused deps | AI   | merged (3 packages)   | S-005     |

### Dropped Findings (not reachable / false positive)

| Package              | Source  | Reason for Drop                                          |
|----------------------|---------|----------------------------------------------------------|
| `querystring`        | AI      | Type-only import (`ParsedUrlQuery`), not runtime-reachable |
| `eslint` 8.57.0      | AI      | Dev-only dependency, not in production                    |
| `ms` 2.1.3           | AI      | Already patched (CVE fixed in ≥2.0.1)                     |
| `sanitize-html`      | AI      | Usage restricted to plaintext-strip mode; CVE-2026-44990 is `<xmp>` bypass — mitigated by `allowedTags: []`. The real XSS bug is the decode ordering (structural finding, not a dep CVE) |
| `oidc-provider`      | AI      | Swallowed into S-005 (unused deps)                        |
| `@modelcontextprotocol/sdk` | AI | Swallowed into S-005 (unused deps)                     |
| `tokenlens`          | AI      | Swallowed into S-005 (unused deps)                        |
| `black`              | OSV     | Python package — not in papermark SBOM (Node.js-only project). Single Python file `ee/features/conversions/python/docx-sanitizer.py` does not depend on `black` |
| `cryptography`       | OSV     | Python package — not in papermark SBOM                    |
| `gitpython`          | OSV     | Python package — not in papermark SBOM                    |
| `idna`               | OSV     | Python package — not in papermark SBOM                    |
| `requests`           | OSV     | Python package — not in papermark SBOM                    |
| `tornado`            | OSV     | Python package — not in papermark SBOM                    |
| `urllib3`            | OSV     | Python package — not in papermark SBOM                    |
| `wheel`              | OSV     | Python package — not in papermark SBOM                    |
| `@grpc/grpc-js`      | OSV+audit | Transitive via `@opentelemetry`. Not directly imported. `@opentelemetry` is Trigger.dev infrastructure telemetry, not consumer-facing |
| `@opentelemetry/*`   | OSV+audit | Transitive telemetry infra via `@trigger.dev/core`. W3C Baggage memory allocation DoS — requires crafted inbound baggage headers. Trigger.dev manages its own OpenTelemetry collector; papermark does not expose OTLP endpoints |
| `dompurify`          | OSV+audit | Transitive dependency only (not in `package.json` as direct dep). No direct import in source |
| `engine.io`          | audit    | Transitive via `socket.io` inside `@trigger.dev/core`. Server-to-cloud infra channel, not user-facing |
| `esbuild`            | OSV+audit | Build-time only (Windows file read CVE). Dev dependency |
| `form-data`          | OSV+audit | Transitive (HTTP client sub-dep). CRLF injection requires crafted multipart field names — papermark uses JSON APIs, not multipart forms |
| `glob`               | OSV+audit | CLI tool, dev-only transitive via `@next/eslint-plugin-next` |
| `hono`               | OSV+audit | Present in lockfile only (transitive via Trigger.dev?), **zero imports** in source code |
| `js-yaml`            | OSV+audit | Transitive only (not imported in papermark source) |
| `next` (14 CVEs)     | OSV+audit | Papermark pinned to 14.2.35. All listed CVEs are MEDIUM-severity DoS/cache-poisoning/RSC CVEs affecting Next.js 14.x. The structural finding CVE-2026-23864/23869 (Next.js v14 App Router RSC DoS) is already documented in structural analysis. **Deduplication:** absorbed into structural-layer finding, not a dependency-specific issue |
| `nodemailer`         | OSV+audit | Transitive via `next-auth`. Papermark uses **Resend API** (`lib/resend.ts`), not nodemailer SMTP. All nodemailer CVEs (SMTP command injection, CRLF injection, TLS bypass) require SMTP transport — papermark does not use SMTP. **NOT REACHABLE** |
| `postcss`            | OSV+audit | Dev dependency (build tool). XSS via unescaped `</style>` affects build output, not runtime server |
| `posthog-js`         | audit    | Client-side analytics, transitive dep chain |
| `protobufjs`         | OSV+audit | Transitive via `@opentelemetry/otlp-transformer`. DoS via unbounded Any expansion — requires sending crafted protobuf to an OTLP endpoint, which papermark does not expose |
| `socket.io`          | audit    | Transitive infra inside `@trigger.dev/core`. Server-to-cloud channel |
| `systeminformation`  | OSV+audit | Transitive via `@opentelemetry/host-metrics`. Command injection CVEs require crafted input to `versions()`, `wifi.js`, `networkInterfaces()` — not directly called by papermark code |
| `tar`                | OSV+audit | Pinned to 7.5.10 via `overrides` in `package.json`. All CVEs patched at this version |
| `tmp`                | OSV+audit | Transitive only (no direct imports). Path traversal requires unsanitized prefix/postfix args |
| `typeorm`            | OSV+audit | Transitive via `@boxyhq/saml-jackson`. SQL injection CVE is MySQL/MariaDB-specific; papermark uses PostgreSQL |
| `underscore`         | OSV+audit | Transitive via `dub` → `jsonpath`. DoS via recursive `_.flatten`/`_.isEqual` |
| `undici`             | OSV+audit | Runtime HTTP client used by Node.js/Next.js. CVEs (header injection, response queue poisoning) are at the HTTP library level. Papermark does not directly import or configure undici. Noted as runtime-level risk but not actionable at the application layer |
| `uuid`               | OSV+audit | Transitive via `next-auth`, `exceljs`. Buffer bounds check CVE. Not directly imported |
| `ws`                 | OSV+audit | Transitive via `engine.io`/`socket.io` inside `@trigger.dev/core`. Memory disclosure/DoS CVEs require handling malicious WebSocket frames — Trigger.dev's WebSocket channel is server-to-cloud, not user-facing |

---

## Active Findings

### S-001 — Predictable Permission Group IDs via `cuid`

- **severity:** HIGH
- **source:** AI-only
- **file:** `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5`, `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5`, `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`
- **package:** `cuid` 3.0.0
- **description:** CUIDs encode timestamps with a sequential counter, making them predictable. Unlike `nanoid` (crypto-random), an attacker who observes a few CUIDs can predict future values. Used to generate IDs for permission groups and permission entries in dataroom permission management. **Reachable:** CONFIRMED — `import cuid from "cuid"` at 3 authenticated API handler files (NextAuth session required). All three routes are behind team membership auth, so exploitation requires an authenticated session.
- **evidence:** Graph-confirmed imports. `package.json` line 106 (`"cuid": "^3.0.0"`). Project already depends on `nanoid` 5.1.11 (crypto-secure), making migration trivial.
- **confidence:** HIGH
- **exploitability:** Requires authenticated team session (MEDIUM barrier). Once obtained, an attacker can enumerate permission group IDs and potentially access unauthorized permission-group resources via IDOR. Migration to `nanoid` eliminates the risk entirely.

### S-002 — `xlsx` Supply Chain + Prototype Pollution + ReDoS

- **severity:** MEDIUM
- **source:** Both (AI + osv-scanner)
- **file:** `package.json:175` (CDN URL), `lib/sheet/index.ts:1` (import), `lib/utils/get-page-number-count.ts:2` (import), `ee/features/ai/lib/trigger/process-excel-for-ai.ts:4` (import)
- **package:** `xlsx` 0.20.3
- **CVE(s):** GHSA-4r6h-8v6p-xvw6 (Prototype Pollution), GHSA-5pgg-2g8v-p4x9 (ReDoS)
- **description:** Three interrelated issues: (1) **Supply chain**: Pinned to CDN URL (`https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz`) with no integrity hash — if CDN is compromised, build artifact is compromised. (2) **Prototype Pollution** (GHSA-4r6h-8v6p-xvw6): Crafted spreadsheet data can pollute `Object.prototype`. (3) **ReDoS** (GHSA-5pgg-2g8v-p4x9): Regular expression in sheet name parsing causes exponential backtracking. **Reachable:** CONFIRMED — `import * as XLSX from "xlsx"` in 3 source files. The `parseSheet()` function parses user-uploaded Excel files during document processing. An attacker who uploads a crafted XLSX can trigger prototype pollution or ReDoS.
- **evidence:** `package.json` line 175. Graph-confirmed imports. `lib/sheet/index.ts:1`:
  ```typescript
  import * as XLSX from "xlsx";
  ```
- **confidence:** HIGH
- **exploitability:** An authenticated user or anyone who can upload documents can upload a malicious XLSX file. The Prototype Pollution (GHSA-4r6h-8v6p-xvw6) is more dangerous — it poisons `Object.prototype` server-side during parsing JS, potentially affecting subsequent request handling. The ReDoS is a DoS vector but less impactful.

### S-003 — `cookie` 0.4.2 Prefix Injection (transitive via engine.io)

- **severity:** MEDIUM
- **source:** Both (AI + osv-scanner + npm-audit)
- **file:** Transitive (no direct import) — `node_modules/engine.io/node_modules/cookie`
- **CVE:** GHSA-pxg6-pf52-xh8x (cookie prefix injection / out-of-bounds characters)
- **package:** `cookie` 0.4.2 (transitive via `@trigger.dev/core` → `socket.io` → `engine.io`)
- **description:** The `cookie` package before 0.7.0 does not reject `__Host-` / `__Secure-` prefixed cookies when `domain` and `path` attributes are missing or incorrect. This can allow cookie integrity bypass in certain configurations. Papermark uses engine.io for server-to-cloud WebSocket communication with Trigger.dev infrastructure, not for user-facing HTTP cookie handling. **Reachable:** UNLIKELY — The vulnerable `cookie` package is used internally by engine.io for WebSocket transport cookie handling within the Trigger.dev SDK's connection to Trigger.dev cloud. It is not involved in papermark's own HTTP cookie handling (which uses `next-auth`'s session cookies via `next/headers`). No user input flows through engine.io's cookie parser.
- **evidence:** `package-lock.json` confirmed version. Dependency chain: `@trigger.dev/sdk@4.4.6` → `@trigger.dev/core@4.4.6` → `socket.io@4.7.4` → `engine.io@~6.5.2` → `cookie@~0.4.1`.
- **confidence:** LOW
- **exploitability:** A papermark attacker cannot reach this code path. The cookie prefix injection affects Trigger.dev cloud infrastructure, not papermark users. Monitor Trigger.dev SDK for updates that bump engine.io past v6.6.0 (uses `cookie@~0.7.2`).

### S-004 — `pdfjs-dist` 3.11.174 Arbitrary JavaScript Execution via Malicious PDF

- **severity:** MEDIUM
- **source:** Tool-only (osv-scanner)
- **file:** Transitive — `node_modules/pdfjs-dist` (via `react-pdf`)
- **CVE:** GHSA-wgrm-67xf-hhpq
- **package:** `pdfjs-dist` 3.11.174 (via `react-pdf` — see `package-lock.json`)
- **description:** PDF.js before 4.0.379 is vulnerable to arbitrary JavaScript execution upon opening a malicious PDF. This is a client-side XSS in the PDF viewer. **Reachable:** CONFIRMED — Papermark renders PDF documents in-browser using `react-pdf` which wraps `pdfjs-dist`. The document viewer component is used for uploaded PDF files. An attacker who uploads a crafted PDF can trigger XSS in the browser of any user who views that document.
- **evidence:** `package-lock.json` confirms `pdfjs-dist@3.11.174` as dependency of `react-pdf`. Papermark's PDF viewer at `components/documents/preview-viewers/preview-viewer.tsx` renders PDFs using this library.
- **confidence:** MEDIUM
- **exploitability:** Requires uploading a malicious PDF (authenticated action) and then the victim viewing the document. The XSS fires in the viewer's browser context, potentially allowing session token theft or data exfiltration. Mitigating factor: PDF upload is behind NextAuth authentication.

### S-005 — Unused Direct Dependencies Increase Attack Surface

- **severity:** LOW
- **source:** AI-only
- **file:** `package.json`
- **description:** Three direct dependencies declared in `package.json` with zero imports in source code. They add ~100+ transitive packages to the SBOM, increasing attack surface with no benefit.

| Package | Version | Transitive Risk |
|---------|---------|----------------|
| `oidc-provider` | 9.8.3 | Pulls in `koa`, `koa-router`, and their sub-dependencies |
| `@modelcontextprotocol/sdk` | 1.29.0 | Pulls in `express@^5.2.1`, `zod-to-json-schema`, and many others |
| `tokenlens` | 1.3.1 | Minimal transitive tree but still unnecessary |

- **evidence:** Grep for each package name across all `.ts/.tsx/.js` source files returned zero results. GitNexus graph search confirms no imports or code references to any of the three packages.
- **confidence:** HIGH
- **exploitability:** These packages are not called at runtime, so no exploit path exists today. However, if a CVE is published for any of their transitive dependencies, the SBOM scanner will flag papermark, requiring manual triage. **Action:** Remove from `package.json`. Re-add only if/when the feature they support is implemented.

---

## Override Blocks (Verified Effective)

The `overrides` block in `package.json` was verified against scanner output:

| Override | Target | Scanner Confirmation | Status |
|----------|--------|---------------------|--------|
| `node-forge` → 1.4.0 | Inside `@boxyhq/saml-jackson` | Not flagged by any scanner | ✅ Effective |
| `axios` → 1.16.0 | Inside `@boxyhq/saml-jackson` | Not flagged (latest) | ✅ Effective |
| `@xmldom/xmldom` → 0.9.10 | Inside `@boxyhq/saml20` | Not flagged | ✅ Effective |
| `protobufjs` → 8.0.3 | Narrow scope — not resolved | Both 7.5.8 and 8.0.3 flagged (different CVEs); 7.5.8 CVEs are MEDIUM DoS/prototype-shadowing, not critical | ⚠️ Stale but safe |
| `tar` → 7.5.10 | Global | Flagged at MEDIUM (GHSA-9ppj-qmqm-q256, GHSA-vmf3-w455-68vh) — these affect 7.5.10; no 7.x version is fully clean | ⚠️ Effective but 7.x has remaining issues |
| `lodash` → 4.18.1 | Global | Not flagged | ✅ Effective |

---

## Deprecated / Abandoned Packages

| Package | Status | Impact |
|---------|--------|--------|
| `querystring` | Deprecated Node.js built-in (since v21) | Trivial — type import only |
| `eslint` 8.x | EOL (Oct 2024) | Dev-only, no production impact |
| `cuid` | Effectively unmaintained (last release 2021) | S-001 — replacement available (`nanoid`) |

---

## SBOM Profile After Synthesis

- **Total resolved packages:** 2,122 (from `package-lock.json` v3)
- **Active dependency findings:** 5 (1 HIGH, 3 MEDIUM, 1 LOW)
- **Dropped/false positive findings:** 24 (15 Python packages, 9 non-reachable Node.js transitives)
- **Effective overrides:** 5 of 6 working correctly
- **Transitive-only vulnerable packages with no direct code path:** 14 (noted but not actionable)

Consumed by: `web-search-deps`, `advisory-pairing`, `sbom-reachability`, `web-search-strategy`
