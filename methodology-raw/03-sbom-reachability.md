# SBOM Reachability Filter — papermark

**Node:** Layer-3 SBOM Reachability Filter  
**Date:** 2026-06-30  
**Target repo:** `papermark`  
**Method:** GitNexus query + context + impact analysis, direct source audit (Grep), OSV-scanner + npm-audit cross-reference, structural analysis (01-structural-analysis.md), reachability analysis (01-reachability-analysis.md), synthesized dependencies (02-synthesized-dependencies.md)

---

## Classification Results Summary

| ID | Package | CVEs | Classification | Actionable |
|----|---------|------|----------------|------------|
| S-001 | `cuid` 3.0.0 | Design weakness (no CVE) | **REACHABLE** | YES (HIGH) |
| S-002 | `xlsx` 0.20.3 | GHSA-4r6h-8v6p-xvw6, GHSA-5pgg-2g8v-p4x9 | **REACHABLE** | YES (MEDIUM) |
| S-003 | `cookie` 0.4.2 (transitive) | GHSA-pxg6-pf52-xh8x | **NOT-REACHABLE** | Informational |
| S-004 | `pdfjs-dist` 3.11.174 (via react-pdf) | GHSA-wgrm-67xf-hhpq | **REACHABLE** | YES (MEDIUM) |
| S-005 | `oidc-provider`, `@modelcontextprotocol/sdk`, `tokenlens` | None | **NOT-IMPORTED** | Informational |

---

## Reachable CVEs (Actionable)

### S-001 — `cuid` 3.0.0 Predictable ID Generation

| Field | Value |
|-------|-------|
| **CVE** | None (design weakness) |
| **Package** | `cuid` ^3.0.0 |
| **Severity** | HIGH |
| **Vulnerable Function** | `cuid()` — generates IDs using timestamp + sequential counter |
| **Called From** | 3 API handler files: |
| | `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4` (creating permission groups) |
| | `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5` (updating permission groups) |
| | `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5` (creating permission entries) |
| **Call Chain from User Input** | |
| | `POST/GET /api/teams/:teamId/datarooms/:id/permission-groups` |
| | → `middleware.ts:49` (excludes `/api/` from middleware) |
| | → Pages Router handler → `getServerSession()` NextAuth auth check |
| | → Handler calls `cuid()` to generate resource ID |
| | → ID stored in PostgreSQL → returned in API response |
| **Evidence** | GitNexus graph-confirmed imports in all 3 files. `package.json` line 106. Each handler imports `cuid` at module level and calls `cuid()` to generate IDs for DB records. The handlers are behind NextAuth session checks with team membership verification. |
| **Reachability** | **CONFIRMED** — Authenticated team members trigger `cuid()` on every permission group/entry creation. An attacker with team membership can observe CUID patterns and predict future IDs, enabling horizontal privilege escalation (IDOR) within the team scope. |
| **Mitigation** | Project already depends on `nanoid` 5.1.11 (crypto-secure). Migration is trivial — replace `cuid()` with `nanoid()`. |

---

### S-002 — `xlsx` 0.20.3 Prototype Pollution + ReDoS + Supply Chain

| Field | Value |
|-------|-------|
| **CVE(s)** | GHSA-4r6h-8v6p-xvw6 (Prototype Pollution, CVSS 6.3), GHSA-5pgg-2g8v-p4x9 (ReDoS, CVSS 5.3) |
| **Package** | `xlsx` 0.20.3 (CDN URL: `https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz`, no integrity hash) |
| **Severity** | MEDIUM |
| **Vulnerable Functions** | `XLSX.read()` (Prototype Pollution via crafted cell data), sheet name regex (ReDoS) |
| **Called From** | 3 source files (2 actively used, 1 dead code): |

| **File** | **Function** | **Status** |
|----------|-------------|------------|
| `lib/sheet/index.ts:18` | `parseSheet({fileUrl})` → `XLSX.read(data, {type: "array"})` | **ACTIVE** — called from views routes |
| `lib/utils/get-page-number-count.ts:19` | `getSheetsCount(arrayBuffer)` → `XLSX.read(data, {type: "array"})` | **DEAD CODE** — defined but never imported/called anywhere |
| `ee/features/ai/lib/trigger/process-excel-for-ai.ts:136` | `excelToMarkdown(workbook)` → `XLSX.utils.sheet_to_json()` | **ACTIVE** — Trigger.dev AI processing job |

| **Call Chain from User Input** | |
| | **Path A (views routes — parseSheet):** |
| | User uploads XLSX → stored in blob storage |
| | → `app/api/views/route.ts:868` or `app/api/views-dataroom/route.ts:1362` |
| | → `parseSheet({ fileUrl })` → `fetch(fileUrl)` → `XLSX.read(data, {type: "array"})` |
| | → Prototype Pollution: crafted cell names (e.g. `__proto__`, `constructor`) pollute `Object.prototype` |
| | → ReDoS: crafted sheet name triggers exponential regex backtracking |
| | |
| | **Path B (AI processing — processExcelForAITask):** |
| | User uploads XLSX → AI document processing enabled |
| | → `ee/features/ai/lib/trigger/process-document-for-ai.ts` |
| | → Triggers `processExcelForAITask` (Trigger.dev job) |
| | → `run()` → `getFile()` → `XLSX.read()` → `excelToMarkdown()` → Prototype Pollution / ReDoS |

| **Evidence** | `package.json` line 175 (CDN URL with no integrity hash). Graph-confirmed imports in 3 files. The `parseSheet` function is called from `app/api/views/route.ts:868` and `app/api/views-dataroom/route.ts:1362` — both are Next.js App Router API handlers that process uploaded document data. The `processExcelForAITask` is triggered from the AI document processing pipeline (confirmed by `lib/trigger/queues.ts:33` queue definition). |
| **Reachability** | **CONFIRMED** — Both active paths parse user-uploaded XLSX files server-side. The Prototype Pollution (GHSA-4r6h-8v6p-xvw6) affects `Object.prototype` during `XLSX.read()`, which then persists for the lifetime of the server process, potentially affecting subsequent requests. The ReDoS (GHSA-5pgg-2g8v-p4x9) causes CPU exhaustion. The supply chain risk from the CDN URL with no integrity hash is a build-time compromise vector. |
| **Note on dead code** | `lib/utils/get-page-number-count.ts:getSheetsCount()` defines the same vulnerable `XLSX.read()` call but is **never invoked** — zero imports across all source files. Its Prototype Pollution risk is NOT actionable until the function is actually called. |
| **Mitigation** | Upgrade to `xlsx` 0.20.4+ (if available) or migrate to `exceljs` / `xlsx-populate`. Add integrity hash to CDN URL. Remove dead `getSheetsCount()` function. |

---

### S-004 — `pdfjs-dist` 3.11.174 Arbitrary JavaScript Execution via Malicious PDF

| Field | Value |
|-------|-------|
| **CVE** | GHSA-wgrm-67xf-hhpq (CVSS 7.3 — arbitrary JavaScript execution in PDF viewer) |
| **Package** | `pdfjs-dist` 3.11.174 (transitive via `react-pdf` 8.0.2 — declared in `package.json` line 150) |
| **Severity** | MEDIUM |
| **Vulnerable Function** | PDF.js core document rendering — executes JavaScript embedded in PDF documents during viewer load |
| **Called From** | `components/view/viewer/pdf-default-viewer.tsx:5` — imports `{ Document, Page, pdfjs } from "react-pdf"` |
| **Call Chain from User Input** | |
| | Attacker uploads malicious PDF (requires NextAuth authenticated session) |
| | → PDF stored in blob storage → document link created |
| | → Victim views document (via shared link or authenticated view) |
| | → `PDFViewer` component renders PDF using react-pdf |
| | → react-pdf calls pdfjs-dist to parse and render the PDF |
| | → Malicious JavaScript embedded in PDF executes in victim's browser |
| | **XSS impact:** session token theft, API key exfiltration, team data access |
| **Evidence** | `package.json` line 150 declares `react-pdf: ^8.0.2`. `package-lock.json` resolves `pdfjs-dist@3.11.174` as a dependency of `react-pdf`. Grep confirms `react-pdf` import in `components/view/viewer/pdf-default-viewer.tsx:5`. The `PDFViewer` component is used in document view pages (`pages/view/[linkId]/d/[documentId].tsx`) which render uploaded PDFs. |
| **Reachability** | **CONFIRMED** — This is a client-side XSS vulnerability triggered when a victim views a malicious PDF. Upload requires authentication (NextAuth session), but the document viewing can be done via shared links (which may not require auth depending on link configuration). The PDF viewer is the core document viewing experience for papermark — every uploaded PDF is rendered through react-pdf. |
| **Mitigation** | Upgrade `react-pdf` to a version that bundles `pdfjs-dist` 4.0.379+. Pin `pdfjs-dist` via overrides: `"pdfjs-dist": "4.0.379"`. Alternatively, use a server-side PDF sanitizer to strip embedded JavaScript before serving PDFs to viewers. |

---

## Non-Reachable CVEs (Informational)

### S-003 — `cookie` 0.4.2 Prefix Injection (Transitive via engine.io)

| Field | Value |
|-------|-------|
| **CVE** | GHSA-pxg6-pf52-xh8x — cookie accepts `__Host-`/`__Secure-` prefixed names without requiring correct `domain`/`path` attributes |
| **Package** | `cookie` ~0.4.1 (transitive: `@trigger.dev/core` → `socket.io` → `engine.io@~6.5.2` → `cookie@~0.4.1`) |
| **Severity** | MEDIUM |
| **Vulnerable Function** | `cookie.parse()` — used internally by engine.io for WebSocket transport cookie handling |
| **Why NOT reachable** | |
| | 1. The vulnerable `cookie@0.4.1` is only used by `engine.io@~6.5.2` for WebSocket transport cookie parsing within Trigger.dev's server-to-cloud connection. |
| | 2. Papermark does not expose any user-facing WebSocket endpoints. The socket.io/engine.io connection is internal infrastructure between the Trigger.dev SDK and Trigger.dev cloud. |
| | 3. Papermark directly imports the `cookie` package for HTTP cookie parsing in `lib/auth/dataroom-auth.ts`, `lib/auth/link-session.ts`, and `pages/api/links/download/verify.ts`, but the resolved version in `package-lock.json` for these direct imports is `cookie@^0.7.x` (CVE-fixed). |
| | 4. No user input flows through engine.io's cookie parser — the Trigger.dev SDK manages its own WebSocket connection with fixed infrastructure endpoints. |
| | 5. The newer engine.io version in the lockfile (`~6.6.0` with `cookie@~0.7.2`) is already patched. Only the older nested version is vulnerable, and it's never exposed to user requests. |
| **Classification** | **NOT-REACHABLE** — The vulnerable code path exists in the dependency tree but is unreachable from user input. The CVE affects Trigger.dev cloud infrastructure, not papermark users. |
| **Recommendation** | Monitor Trigger.dev SDK updates that bump `socket.io`/`engine.io` past v6.6.0 (which uses `cookie@~0.7.2`). No urgent action needed. |

### S-005 — Unused Direct Dependencies

| Field | Value |
|-------|-------|
| **Packages** | `oidc-provider` 9.8.3, `@modelcontextprotocol/sdk` 1.29.0, `tokenlens` 1.3.1 |
| **Severity** | LOW |
| **Why NOT reachable** | |
| | 1. Zero imports of any of these three packages across all `.ts/.tsx/.js` source files — confirmed by both Grep and GitNexus graph search. |
| | 2. No code references, no imports, no require() calls exist for these packages. |
| | 3. They add ~100+ transitive packages to the SBOM but are never loaded at runtime. |
| **Classification** | **NOT-IMPORTED** — These packages are declared in `package.json` but never imported or called. No exploit path exists today. They increase attack surface only in the sense that if a CVE is published for their transitive deps, the SBOM scanner will flag papermark. |
| **Recommendation** | Remove from `package.json`. Re-add only if/when the feature they support is implemented. |

---

## Additional Notes

### Dead Code Detected

`lib/utils/get-page-number-count.ts:getSheetsCount()` imports `xlsx` and calls `XLSX.read()` but is **never invoked** anywhere in the codebase. Zero imports across all source files. While `getPagesCount()` from the same file is actively used (in `lib/files/put-file.ts` and upload components), `getSheetsCount()` is dead code. Should be removed to reduce false positive SBOM noise.

### Supply Chain Risk (xlsx CDN)

The `xlsx` package is pinned to a CDN URL in `package.json`:
```
"xlsx": "https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz"
```
No integrity hash (SRI) is specified. If the CDN is compromised, the build artifact would include a backdoored version of xlsx. This is a build-time supply chain risk that compounds the Prototype Pollution CVE.

### engine.io Version Duplication

The `package-lock.json` shows two versions of `engine.io`:
- `engine.io@~6.5.2` (via `socket.io@4.7.4`) — uses `cookie@~0.4.1` (vulnerable)
- `engine.io@~6.6.0` (via `socket.io` newer) — uses `cookie@~0.7.2` (fixed)

Both are transitive via different sub-dependency resolutions of `@trigger.dev/core`. The older version is only reachable if Trigger.dev SDK still resolves to `socket.io@4.7.4`. Updating Trigger.dev SDK should deduplicate to the newer engine.io.

---

## Data Sources

- **GitNexus query + context**: `xlsx`, `cuid`, `react-pdf`, `cookie` — verified imports, usage patterns, call chains
- **GitNexus impact analysis**: `parseSheet`, `getSheetsCount`, `run` (process-excel-for-ai), `excelToMarkdown`, `PDFViewer`, permission group handlers
- **Direct source reads**: All referenced handler files, `lib/sheet/index.ts`, `lib/utils/get-page-number-count.ts`, `ee/features/ai/lib/trigger/process-excel-for-ai.ts`, `components/view/viewer/pdf-default-viewer.tsx`, `package.json`, `package-lock.json`
- **Tool scanners**: `tools-raw/osv-scanner.json`, `tools-raw/npm-audit.json`
- **Methodology input**: `01-structural-analysis.md` (call chains), `01-reachability-analysis.md` (reachability traces), `02-synthesized-dependencies.md` (deduped findings)
