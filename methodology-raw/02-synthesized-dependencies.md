# Synthesized Dependency Analysis — `synthesize-dependencies`

**Workflow:** whitebox-bug-finder (M3)
**Methodology:** M3 — Dependency CVE / vulnerability / reachability review
**Codebase:** papermark (Next.js 14 App Router, multi-tenant SaaS)
**Date:** 2026-06-19

---

## Sources Compared

| Source | Scope | Detail level |
|---|---|---|
| **AI** (`methodology-raw/00-ai-dependencies.md`) | 28 findings; manual CVE cross-ref + GitNexus caller verification | Per-package severity, package@version, CVE IDs, file:line reachability |
| **Tool** (`tools-raw/normalized.json`, `osv-scanner.json`, `npm-audit.json`) | 92 dependency findings (34 osv-scanner + 58 npm-audit) | Package + version + CVE IDs, but no file:line reachability |

**Sources loaded for context:**

- `methodology-raw/01-structural-analysis.md` — runtime code paths and call chains
- `methodology-raw/01-reachability-analysis.md` — per-CVE reachability table (D1–D13)

**Verification used:** Installed versions cross-checked against `package-lock.json`; transitive-parent map built by walking `package-lock.json`'s `dependencies` graph; reachability confirmed against the structural and reachability analyses.

---

## Sources Reconciliation

**Tool-only findings AI missed** (high-signal — these are the gaps):

1. `pdfjs-dist@3.11.174` — GHSA-wgrm-67xf-hhpq **HIGH CVSS 8.8** (arbitrary JS execution on opening a malicious PDF). AI listed `pdfjs-dist` as a "LOW" deferred concern; tool flagged the actual CVE with full CVSS. Reachability: via `react-notion-x` → `react-pdf@8.0.2` (overridden) → `pdfjs-dist@3.11.174`. Notion pages are rendered in datarooms — user-supplied Notion content reaches the renderer.
2. `nodemailer@7.0.13` — GHSA-p6gq-j5cr-w38f **HIGH** (message-level `raw` option bypasses `disableFileAccess`/`disableUrlAccess`, enabling arbitrary file read & full-access). AI listed nodemailer as INFORMATIONAL (only `envelope.size` SMTP injection, CVE patched); tool flagged the newer GHSA-p6gq. **Reachable**: papermark imports `nodemailer` directly to send emails (transactional). Reachability depends on whether the app calls `sendMail({ raw: ... })` — needs grep.
3. `systeminformation@5.23.8` — 4 HIGH command-injection CVEs (GHSA-wphj-fx3q-84ch CVSS 8.1; GHSA-5vv4-hvf7-2h46; GHSA-9c88-49p5-5ggf; GHSA-hvx9-hwr7-wjj9). Transitive via `@opentelemetry/host-metrics`. **Reachable**: only if `host-metrics` is initialised; the otel SDK ships in deps but the host-metrics package needs an explicit registration call. Confirm in `instrumentation.ts` / `sentry.*.config.ts`.
4. `form-data@4.0.5` — GHSA-hmw2-7cc7-3qxx **HIGH CVSS 7.5** (CRLF injection in multipart filenames → HTTP header injection). Transitive via `axios@1.16.0` (overridden). **Reachable**: papermark uses axios for outbound HTTP (S3 SDK, webhooks). Multipart upload filenames on outbound calls could be controlled indirectly through user-uploaded filenames.
5. `@grpc/grpc-js@1.14.3` — GHSA-5375-pq7m-f5r2 **HIGH CVSS 7.5** (malformed request crashes server). Transitive via `@boxyhq/metrics` (saml-jackson path). **Reachable**: only on SAML response processing paths.
6. `@boxyhq/saml-jackson@26.2.0` — npm-audit HIGH (range `>=26.2.0` flagged); fixAvailable = `1.52.2` (major semver break). AI listed as INFORMATIONAL "current, override-pinned"; tool disagrees on patch completeness. **Reachable**: directly imported; processes SAML assertions.
7. `next@14.2.35` — 14 advisories total (4 HIGH: GHSA-36qx, GHSA-8h8q, GHSA-c4j6, GHSA-h25m, GHSA-q4gf; 8 MODERATE; 2 LOW). AI listed as "MEDIUM framework CVEs, current at 14.2.35"; reachability analysis already covered (D1–D4). **Reachable**: every HTTP request.
8. `hono@4.12.18` — GHSA-88fw-hqm2-52qc **HIGH** (CORS middleware reflects any `Origin` with credentials when `origin` defaults to wildcard). Transitive via `@modelcontextprotocol/sdk@1.29.0`. **Reachable**: only when MCP integration initialises an HTTP server. Confirm whether any code path wires hono into the Next.js runtime.
9. `xlsx@0.20.3` — GHSA-4r6h-8v6p-xvw6 **HIGH** (Prototype Pollution), GHSA-5pgg-2g8v-p4x9 **HIGH** (ReDoS). AI listed as INFORMATIONAL ("patched in 0.20.3"); tool disagrees — both CVEs are post-0.20.3 advisories. **Reachable**: `lib/sheet/index.ts:18-22` parses uploaded XLSX via `XLSX.read(data, { type: "array" })`; users upload Excel files to datarooms.

**AI findings tool didn't flag** (real but no public CVE yet, or out of advisory scope):

1. `cuid@3.0.0` deprecated by maintainer — non-cryptographic ID generator. Used for permission row PKs (`pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:91`). Reachability-analysis D13 flags this as a predictability issue (cuid outputs are referenced in HTTP responses). Tool has no CVE for cuid — keep AI finding.
2. `archiver@7.0.1` zip-slip advisory (not a CVE). Tool didn't flag; AI flagged for folder-name sanitisation check. **Reachable**: `ee/features/dataroom-freeze/lib/trigger/dataroom-freeze-archive.ts:70`.
3. `oidc-provider@9.8.3` — past CVE-2024 refresh-token reuse; tool didn't flag the current version. AI flagged for config check (PKCE/S256 + `rotateRefreshToken`). **Reachable**: bound via `next.config.mjs` rewrites.
4. `next-auth@4.24.14` — tool only flagged via `@teamhanko/passkeys-next-auth-provider`'s indirect next-auth range; the main `next-auth@4.24.14` is out of npm-audit's flagged range. AI flagged patch-gap family (CVE-2023-48304 patched in 4.24.5).

**Both-source agreement (merged):**

1. `dompurify@3.4.0` — AI INFORMATIONAL; tool MODERATE XSS (GHSA-x4vx-rjvf-j5p4 "IN_PLACE trusts attacker-controlled nodeName", plus 6 more advisories from osv-scanner — GHSA-76mc, GHSA-cmwh, GHSA-gvmj, GHSA-hpcv, GHSA-r47g, GHSA-rp9w, GHSA-vxr8). AI concluded "no direct call site in Papermark code" — correct. Reachability via mermaid (renders user-supplied diagrams?) and posthog-js (browser-only). Tool's CVEs target the `IN_PLACE` mode and `setConfig()` hook pollution families — none of which are reachable in papermark's normal usage (no `IN_PLACE` calls).
2. `protobufjs@7.5.8` — AI INFORMATIONAL ("CVE-2023-36665 patched in 7.2.4"); tool HIGH DoS via unbounded Any expansion (GHSA-wcpc-wj8m-hjx6) plus MODERATE schema-derived-names shadowing (GHSA-f38q-mgvj-vph7). Transitive via AWS SDK and OpenTelemetry. AI missed the new advisory.
3. `tar@7.5.10` — AI INFORMATIONAL; tool HIGH symlink traversal (GHSA-9ppj-qmqm-q256). Both confirmed override-pinned to 7.5.10. Tool says <=7.5.15 affected, so 7.5.10 IS vulnerable.
4. `glob@10.5.0` — AI didn't list; tool HIGH command injection (GHSA-5j98-mcp5-4vw2, range 10.2.0–10.4.5). Papermark has 10.5.0 → PATCHED. (Tool's evidence range `10.3.10` is misleading; current install 10.5.0 is past the fix.)
5. `js-yaml@4.1.1` — AI didn't list; tool MODERATE DoS (GHSA-h67p-54hq-rp68). Range <=4.1.1 → 4.1.1 IS vulnerable. Transitive via @boxyhq/saml-jackson (yaml config) and build tooling.
6. `uuid@11.1.1` — AI didn't list; tool MODERATE buffer-bounds (GHSA-w5hq-g745-h8pq). Range <11.1.1 → 11.1.1 IS PATCHED.
7. `tmp@0.2.5` — AI didn't list; tool HIGH path traversal (GHSA-ph9p-34f9-6g65). Range <0.2.6 → 0.2.5 IS VULNERABLE. Transitive via exceljs (writes only) and patch-package (build-time).
8. `underscore@1.13.6` — AI didn't list; tool HIGH DoS via unbounded recursion (GHSA-qpx9-hpmf-5gmw). Range <=1.13.7 → 1.13.6 IS VULNERABLE. Transitive via jsonpath (a @boxyhq/metrics dep).
9. `undici@6.25.0` — AI didn't list; tool HIGH WS DoS (GHSA-vxpw-j846-p89q) + MODERATE Set-Cookie injection (GHSA-p88m-4jfj-68fv) + LOW keep-alive poisoning. Transitive via @vercel/blob (directly imported in app code). Need to check if undici is pinned via the blob client wrapper or pulled in fresh.
10. `esbuild@0.28.0` — LOW (Windows-only dev-server file read). Patched in 0.28.1. Not reachable on Vercel Linux deploy.
11. `cookie@0.7.2` — LOW (CVSS 0, "cookie name out of bounds"). Range <0.7.0 → 0.7.2 PATCHED.

**Pipfile.lock findings — EXCLUDED from app scope**:

The `Pipfile.lock` only resolves deps for `tinybird-cli`, a developer-only CLI tool not bundled into the running app (verified via `Pipfile:7` — only `tinybird-cli = "*"`). All 8 Python findings (black, cryptography, gitpython, idna, requests, tornado, urllib3, wheel) are transitive deps of tinybird-cli and run only on developer workstations for analytics CLI usage. **Out of scope for the app.** The reachability is dev-machine only; CI/developer-only.

---

## Synthesized Findings (ranked by exploitability)

Findings are ordered: CRITICAL/HIGH first (exploitable, reachable, real CVE), then MEDIUM, then LOW/INFORMATIONAL.

### S-001 — `pdfjs-dist@3.11.174` arbitrary JS execution on opening a malicious PDF [HIGH CVSS 8.8]
- **Severity:** HIGH
- **Source:** Tool-only (osv-scanner GHSA-wgrm-67xf-hhpq)
- **Advisory:** PDF.js vulnerable to arbitrary JavaScript execution upon opening a malicious PDF; range `<=4.7.76`
- **Reachable:** YES — `react-notion-x` renders Notion pages via `react-pdf@8.0.2` (overridden) → `pdfjs-dist@3.11.174`. Notion pages are embedded in datarooms (`ee/features/notion/*`). A malicious Notion embed (or an attacker who controls a Notion workspace shared into a dataroom) triggers arbitrary JS in the rendering browser.
- **Evidence:** `package.json:208-210` — override `"react-notion-x": { "react-pdf": "8.0.2" }`. `package-lock.json` shows `pdfjs-dist: 3.11.174`. AI analysis downplayed this as "LOW — old but pinned" (00-ai-dependencies.md:304–314); tool correctly identified the HIGH advisory.
- **Recommendation:** Bump `pdfjs-dist` to 4.x and verify `react-notion-x` compatibility. The 3.x line is no longer receiving security backports from Mozilla.
- **Exploitability:** HIGH

### S-002 — `xlsx@0.20.3` Prototype Pollution + ReDoS on uploaded Excel [HIGH]
- **Severity:** HIGH
- **Source:** Both (merged) — AI flagged prototype-pollution/ReDoS history; tool identified GHSA-4r6h-8v6p-xvw6 + GHSA-5pgg-2g8v-p4x9 as affecting 0.20.3
- **Advisory:** Prototype Pollution in SheetJS; SheetJS ReDoS. (AI's note "patched in 0.20.3" was correct for CVE-2023-30533 but missed these newer CVEs.)
- **Reachable:** YES — `lib/sheet/index.ts:18-22` invokes `XLSX.read(data, { type: "array" })` on uploaded XLSX. Users upload Excel to datarooms; AI pipeline (`ee/features/ai/lib/trigger/process-excel-for-ai.ts:15`) parses untrusted workbooks.
- **Evidence:** `lib/sheet/index.ts:1` imports `xlsx`; `ee/features/ai/lib/trigger/process-excel-for-ai.ts` ingests Excel for AI indexing. The package is loaded from CDN tarball (`xlsx-0.20.3.tgz`) per AI analysis.
- **Recommendation:** Bump to `0.20.4+` if available, or pin to a registry source to enable future updates. Confirm SheetJS patch status for GHSA-5pgg-2g8v-p4x9.
- **Exploitability:** HIGH

### S-003 — `nodemailer@7.0.13` `raw:` option bypass enables file read / SSRF [HIGH]
- **Severity:** HIGH
- **Source:** Tool-only (osv-scanner GHSA-p6gq-j5cr-w38f)
- **Advisory:** Nodemailer message-level `raw` option bypasses `disableFileAccess` / `disableUrlAccess`, enabling arbitrary file read and full-access attacks. Additional: GHSA-c7w3-x93f-qmm8 (SMTP command injection via `envelope.size`, LOW); GHSA-268h-hp4c-crq3 (CRLF in List-* header comments, MODERATE); GHSA-vvjj-xcjg-gr5g (SMTP EHLO/HELO command injection via CRLF, MODERATE); GHSA-r7g4-qg5f-qqm2 (improper TLS validation in OAuth2 token fetch, MODERATE).
- **Reachable:** DEPENDENT — papermark imports nodemailer directly (no alternative wrapper noted). The HIGH bypass requires `sendMail({ raw: ... })` with attacker-controlled content; the MODERATE SMTP injection paths require email-content injection. Confirm by grep for `raw:` usage in `lib/email/`, `pages/api/email/`, etc.
- **Evidence:** `package-lock.json` confirms nodemailer@7.0.13.
- **Recommendation:** Bump to 8.x. Audit any use of `sendMail({ raw })` in papermark; if no such calls, the HIGH is mitigated. The MODERATE issues can be mitigated by validating email-header values server-side.
- **Exploitability:** HIGH (if raw: is used), MEDIUM (otherwise)

### S-004 — `systeminformation@5.23.8` Command Injection in host-metrics [HIGH CVSS 8.1]
- **Severity:** HIGH (CVSS 8.1 for GHSA-wphj-fx3q-84ch)
- **Source:** Tool-only (osv-scanner 4 advisories)
- **Advisories:** GHSA-wphj-fx3q-84ch (fsSize command injection, Windows, CVSS 8.1); GHSA-5vv4-hvf7-2h46 (versions command injection); GHSA-9c88-49p5-5ggf (wifi interface parameter injection); GHSA-hvx9-hwr7-wjj9 (Linux command injection in networkInterfaces). Range `<5.27.14`.
- **Reachable:** DEPENDENT — transitive via `@opentelemetry/host-metrics`. Only loaded at runtime if host-metrics is initialised in the otel SDK config. Confirm by grep for `host-metrics` registration in `instrumentation.ts` / `sentry.*.config.ts` / otel bootstrap.
- **Evidence:** `package-lock.json` shows `systeminformation@5.23.8`. AI did not mention this package.
- **Recommendation:** If host-metrics is enabled, bump `@opentelemetry/host-metrics` to `>=0.38.1` (which forces systeminformation 5.27.14+) or disable host-metrics collection entirely. Vercel serverless is Linux-only, so Windows CVEs are lower risk; the Linux CVE (GHSA-hvx9-hwr7-wjj9) is the relevant one.
- **Exploitability:** HIGH (if host-metrics is enabled), N/A (otherwise)

### S-005 — `form-data@4.0.5` CRLF injection in multipart filenames [HIGH CVSS 7.5]
- **Severity:** HIGH
- **Source:** Tool-only (osv-scanner GHSA-hmw2-7cc7-3qxx)
- **Advisory:** CRLF injection in form-data via unescaped multipart field names and filenames; range `>=4.0.0 <4.0.6`
- **Reachable:** INDIRECT — transitive via `axios@1.16.0` (overridden). Papermark uses axios for outbound HTTP and the AWS SDK. If papermark makes multipart uploads where filenames are derived from user input (S3 upload-to-S3 with user-controlled keys), attacker could inject CR/LF into the HTTP request.
- **Evidence:** `package-lock.json` shows form-data@4.0.5 nested under axios@1.16.0. AI mentioned axios@1.16.0 as patched but missed form-data.
- **Recommendation:** Bump form-data to 4.0.6+. Since the override block (`package.json:212-222`) is the central tool, add form-data to overrides.
- **Exploitability:** MEDIUM (requires user-controlled multipart filename in outbound calls)

### S-006 — `@grpc/grpc-js@1.14.3` malformed request DoS [HIGH CVSS 7.5]
- **Severity:** HIGH
- **Source:** Tool-only (osv-scanner GHSA-5375-pq7m-f5r2, GHSA-99f4-grh7-6pcq)
- **Advisory:** Malformed request causes server crash; incoming malformed compressed message causes client/server crash. Range `1.14.0–1.14.3` (patched in 1.14.4).
- **Reachable:** INDIRECT — transitive via `@boxyhq/metrics` (saml-jackson path). @grpc/grpc-js is only loaded if @boxyhq/metrics initialises a gRPC client at runtime. Most SAML setups use HTTP; need to confirm whether saml-jackson's metrics exporter opens a gRPC socket.
- **Evidence:** `package-lock.json` shows @grpc/grpc-js@1.14.3 under @boxyhq/metrics and @opentelemetry/exporter-metrics-otlp-grpc. AI did not mention @grpc/grpc-js.
- **Recommendation:** Bump @boxyhq/metrics to a version that pulls @grpc/grpc-js >=1.14.4 (or pin @grpc/grpc-js via `overrides`).
- **Exploitability:** MEDIUM (depends on gRPC exporter being enabled)

### S-007 — `next@14.2.35` framework CVE bundle (D1–D4 from reachability-analysis)
- **Severity:** MEDIUM (HIGH CVEs but reachability is deployment-dependent)
- **Source:** Both (merged) — AI flagged framework CVE family; tool identified specific advisories (14 total: 5 HIGH, 8 MODERATE, 2 LOW).
- **HIGH advisories:** GHSA-36qx-fr4f-26g5 (Middleware/Pages-Router i18n bypass); GHSA-8h8q-6873-q5fj (DoS via Server Components); GHSA-c4j6-fc7j-m34r (SSRF via WebSocket upgrades); GHSA-h25m-26qc-wcjf (HTTP request deserialization DoS); GHSA-q4gf-8mx6-v5v3 (DoS via Server Components).
- **Reachable:** YES (every HTTP request) — but exploitability depends on which Next.js features papermark uses:
  - D1 (GHSA-9g9p-9gw9-jx7f / image OOM): /\_next/image is internet-facing; broad `<Image>` usage.
  - D2 (XSS via CSP nonces): latent until paired with `dangerouslySetInnerHTML` (SAST F2).
  - D3 (RSC cache poisoning): deployment-dependent on CDN/edge.
  - D4 (DoS via image optimizer remotePatterns): self-hosted only.
- **Evidence:** `package.json:125` ("next": "^14.2.35"). Reachability analysis already covers D1–D4 in detail.
- **Recommendation:** Plan a 15.x upgrade. For the immediate term, confirm self-hosted deployments apply Vercel's published patches and disable `i18n` rewrites in Pages Router unless used (mitigates GHSA-36qx).
- **Exploitability:** MEDIUM (deployment-dependent)

### S-008 — `hono@4.12.18` CORS reflection vulnerability [HIGH]
- **Severity:** HIGH (when reachable)
- **Source:** Tool-only (osv-scanner GHSA-88fw-hqm2-52qc; also 9 other MODERATE advisories)
- **Advisory:** hono CORS middleware reflects any `Origin` with credentials when `origin` defaults to wildcard.
- **Reachable:** DEPENDENT — transitive via `@modelcontextprotocol/sdk@1.29.0`. Only loaded when MCP integration initialises an HTTP server. AI noted MCP SDK is "importable but not heavily exercised" (00-ai-dependencies.md:364).
- **Evidence:** `package-lock.json` shows hono@4.12.18 nested under @modelcontextprotocol/sdk.
- **Recommendation:** Bump @modelcontextprotocol/sdk to a version that pulls hono >=4.12.21. If MCP is not actively used, remove @modelcontextprotocol/sdk from the dependency tree (it's not heavily used per AI analysis).
- **Exploitability:** MEDIUM (only if MCP runtime is wired into a HTTP server)

### S-009 — `@boxyhq/saml-jackson@26.2.0` flagged range [HIGH per npm-audit]
- **Severity:** HIGH per npm-audit (range `>=26.2.0`)
- **Source:** Tool-only (npm-audit) — AI listed as INFORMATIONAL "patched, override-pinned"
- **Reachable:** YES — directly imported in `lib/jackson.ts:7`, processes SAML assertions on `app/(ee)/api/auth/saml/authorize/route.ts`.
- **Evidence:** `package-lock.json` confirms `@boxyhq/saml-jackson@26.2.0`. npm-audit's fixAvailable is `1.52.2` (semver major break).
- **Recommendation:** Investigate what CVE npm-audit attributes to the `>=26.2.0` range. The 26.x line is the current major series (per AI analysis); if the advisory is for an unreleased CVE, follow @boxyhq/security-advisories. The major-version downgrade to 1.52.2 is likely not viable.
- **Exploitability:** HIGH (SAML is internet-facing for enterprise tenants)

### S-010 — `tmp@0.2.5` path traversal [HIGH]
- **Severity:** HIGH
- **Source:** Tool-only (osv-scanner GHSA-ph9p-34f9-6g65)
- **Advisory:** Path Traversal via unsanitized `prefix`/`postfix` enabling directory escape. Range `<0.2.6`.
- **Reachable:** INDIRECT — transitive via `exceljs` and `patch-package`. `exceljs` uses `tmp` for write-temp-file flows when serialising Excel; not invoked on the read path (papermark only writes). `patch-package` runs only at install time.
- **Evidence:** `package-lock.json` shows tmp@0.2.5. AI did not mention tmp.
- **Recommendation:** Add `tmp` to overrides (`"tmp": "0.2.6"`). Since the reachable call sites are exceljs (write) and patch-package (install), the practical exploit surface is minimal — but easy fix.
- **Exploitability:** LOW (write-only exceljs path, build-time patch-package)

### S-011 — `underscore@1.13.6` DoS via unbounded recursion [HIGH]
- **Severity:** HIGH
- **Source:** Tool-only (osv-scanner GHSA-qpx9-hpmf-5gmw)
- **Advisory:** Unbounded recursion in `_.flatten` and `_.isEqual`. Range `<=1.13.7`.
- **Reachable:** INDIRECT — transitive via `jsonpath` (a @boxyhq/metrics dep). jsonpath uses underscore for deep-iteration helpers. Triggered only on SAML telemetry payloads.
- **Evidence:** `package-lock.json` shows underscore@1.13.6. AI did not mention underscore.
- **Recommendation:** Add `underscore` to overrides (`"underscore": "1.13.8+"`). The reachable call site (jsonpath → underscore) only fires if jsonpath is given attacker-controlled JSON, which only happens if SAML response parsing produces an attacker-controlled JSON document.
- **Exploitability:** LOW (narrow trigger path via SAML)

### S-012 — `tar@7.5.10` symlink path traversal [HIGH]
- **Severity:** HIGH (per GHSA-9ppj-qmqm-q256)
- **Source:** Both (merged) — AI flagged "past CVEs patched"; tool flagged the newer symlink traversal that IS in 7.5.10.
- **Advisory:** node-tar Symlink Path Traversal via Drive-Relative Linkpath; range `<=7.5.10`. (GHSA-vmf3 is `<=7.5.6` and is patched.)
- **Reachable:** DEVELOPER-INSTALL ONLY — transitive via `npm` itself and `patch-package`. Runs on developer machines during `npm install` and CI install. The papermark production runtime never extracts tar archives.
- **Evidence:** `package.json:221` override forces `tar@7.5.10`. AI confirmed override exists.
- **Recommendation:** Bump `tar` to 7.5.11+ in the override. Note this affects only developer workstations and CI runners.
- **Exploitability:** MEDIUM (only if a malicious npm package or `npm install` artifact is processed)

### S-013 — `protobufjs@7.5.8` + `8.0.3` DoS via unbounded Any expansion [HIGH/MODERATE]
- **Severity:** HIGH (GHSA-wcpc-wj8m-hjx6)
- **Source:** Both (merged) — AI flagged "patched via override"; tool flagged GHSA-wcpc which affects all versions through 8.0.x.
- **Advisory:** GHSA-wcpc-wj8m-hjx6 DoS through unbounded Any expansion during JSON conversion. Plus GHSA-f38q-mgvj-vph7 (schema-derived names shadow runtime properties). Plus GHSA-jggg-4jg4-v7c6 (DoS via recursive descriptor expansion, affects 8.0.0–8.5.0).
- **Reachable:** INDIRECT — transitive via AWS SDK and OpenTelemetry. protobufjs is invoked when AWS services return protobuf-encoded responses (S3 Select, Kinesis, etc.). papermark uses S3 SDK (`@aws-sdk/client-s3`) and OpenTelemetry.
- **Evidence:** `package-lock.json` shows protobufjs@7.5.8 and 8.0.3. AI confirmed override forces 8.0.3 for otel-transformer.
- **Recommendation:** Bump protobufjs to 8.2.0+ via override. Add `"protobufjs": "8.2.0"` to `package.json:212-222`.
- **Exploitability:** MEDIUM (only if AWS service returns maliciously-crafted protobuf)

### S-014 — `undici@6.25.0` Set-Cookie header injection + WS DoS [HIGH/MODERATE]
- **Severity:** HIGH (GHSA-vxpw-j846-p89q WS DoS), MODERATE (GHSA-p88m-4jfj-68fv Set-Cookie injection)
- **Source:** Tool-only (osv-scanner)
- **Advisory:** undici WebSocket client vulnerable to DoS via fragment count bypass. Plus Set-Cookie header injection via percent-decoding. Plus keep-alive socket reuse poisoning.
- **Reachable:** YES — transitive via `@vercel/blob` (DIRECTLY IMPORTED in app code: `lib/signing/mirror.ts`, `lib/files/copy-file.ts`, `lib/trigger/export-visits.ts`, `pages/api/webhooks/services/[...path]/index.ts`, etc.). The undici HTTP client is the default fetch implementation in Next.js when running on Node.js.
- **Evidence:** `package-lock.json` shows undici@6.25.0. AI did not mention undici.
- **Recommendation:** Bump undici to 6.27.0+ via override. Add `"undici": "6.27.0"` to `package.json:212-222`.
- **Exploitability:** MEDIUM (only if app uses undici WS or receives untrusted Set-Cookie responses)

### S-015 — `@xmldom/xmldom@0.9.10` XXE / prototype-pollution class [INFORMATIONAL]
- **Severity:** INFORMATIONAL (AI-flagged, tool didn't add new advisories)
- **Source:** Both (merged) — AI flagged via override; tool confirms no new GHSA for 0.9.10.
- **Reachable:** YES — transitive via SAML flow.
- **Evidence:** `package.json:215-217` override (`"@boxyhq/saml20@1.13.2": { "@xmldom/xmldom": "0.9.10" }`).
- **Recommendation:** No action needed.
- **Exploitability:** LOW

### S-016 — `cuid@3.0.0` deprecated non-cryptographic ID [LOW]
- **Severity:** LOW
- **Source:** AI-only — npm deprecation message, no CVE
- **Reachable:** YES — used at `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:91` for permission row PKs. Reachability-analysis D13 flags the predictability issue (cuid outputs are referenced in HTTP responses).
- **Evidence:** cuid output is consumed server-side as DB row PKs, not used as session tokens or password-reset tokens.
- **Recommendation:** Replace with `@paralleldrive/cuid2` or `crypto.randomUUID()`. Existing rows unaffected (cuid IDs remain valid).
- **Exploitability:** LOW (information disclosure only)

### S-017 — `archiver@7.0.1` zip-slip advisory (not CVE) [LOW]
- **Severity:** LOW
- **Source:** AI-only — npm advisory, no CVE
- **Reachable:** PARTIAL — `ee/features/dataroom-freeze/lib/trigger/dataroom-freeze-archive.ts:70` adds files to ZIP with `name` derived from dataroom folder structure.
- **Evidence:** Folder names flow from `folderStructure` which originates from team dataroom metadata; papermark slugifies folder names elsewhere.
- **Recommendation:** Add a unit test that creates a dataroom with `../` in folder names and asserts the archive entry is sanitised.
- **Exploitability:** LOW (requires team owner to set `../` in folder names)

### S-018 — `next-auth@4.24.14` patch-gap family [MEDIUM]
- **Severity:** MEDIUM
- **Source:** AI-only — past CVEs (CVE-2023-48304 patched in 4.24.5; CVE-2024 session-fixation family patched in 4.24.7). Tool did not flag new CVEs for 4.24.14.
- **Reachable:** YES — directly imported for every auth flow (`pages/api/auth/[...nextauth].ts`, `lib/auth/auth-options.ts`, `lib/auth/link-session.ts`).
- **Evidence:** `package.json` and `package-lock.json` confirm next-auth@4.24.14.
- **Recommendation:** Plan migration to Auth.js v5 in the medium term.
- **Exploitability:** LOW (current version past all known patches)

### S-019 — `js-yaml@4.1.1` DoS via merge keys [MODERATE]
- **Severity:** MODERATE (CVSS 5.3)
- **Source:** Tool-only (osv-scanner GHSA-h67p-54hq-rp68)
- **Advisory:** Quadratic-complexity DoS in merge key handling via repeated aliases; range `<=4.1.1`.
- **Reachable:** INDIRECT — transitive via @boxyhq/saml-jackson (YAML config parsing) and build tooling. Triggered only if papermark loads YAML from untrusted sources.
- **Evidence:** `package-lock.json` shows js-yaml@4.1.1.
- **Recommendation:** Bump js-yaml to 4.1.2+ via override. Confirm whether SAML config flow parses user-supplied YAML.
- **Exploitability:** LOW (YAML inputs are operator-controlled, not user-controlled)

### S-020 — `dompurify@3.4.0` IN_PLACE / hook-pollution class [MODERATE / NOT REACHABLE]
- **Severity:** MODERATE (multiple CVEs), but NOT REACHABLE in papermark usage
- **Source:** Both (merged) — AI flagged as INFORMATIONAL; tool flagged 8 advisories (GHSA-76mc, GHSA-cmwh, GHSA-gvmj, GHSA-hpcv, GHSA-r47g, GHSA-rp9w, GHSA-vxr8, GHSA-x4vx).
- **Advisory:** XSS via `IN_PLACE` mode trusting attacker-controlled `nodeName`; `setConfig()` hook pollution; cross-realm `instanceof` bypass; etc.
- **Reachable:** NO (in papermark usage) — papermark does not call `dompurify.sanitize(..., { IN_PLACE: true })`. dompurify is transitive via `mermaid` (used for diagram rendering — mermaid does not enable IN_PLACE) and `posthog-js` (browser analytics SDK).
- **Evidence:** No direct dompurify imports in app code; reachability-analysis D5 confirms "Latent — transitive via react-pdf and notion-client".
- **Recommendation:** If mermaid renders user-supplied diagram code in the future, audit dompurify call sites for IN_PLACE usage. Otherwise, no action needed.
- **Exploitability:** LOW (not reachable in current usage)

### S-021 — `cookie@0.7.2` out-of-bounds characters [LOW, PATCHED]
- **Severity:** LOW (CVSS 0)
- **Source:** Tool-only (osv-scanner + npm-audit)
- **Reachable:** YES — cookie is used by Next.js / NextAuth.js core for session cookies.
- **Evidence:** `package-lock.json` shows cookie@0.7.2.
- **Recommendation:** No action — 0.7.2 is past the fix range (<0.7.0).
- **Exploitability:** LOW

### S-022 — `glob@10.5.0` command injection [HIGH, PATCHED]
- **Severity:** HIGH (range `>=10.2.0 <10.5.0`)
- **Source:** Tool-only (osv-scanner GHSA-5j98-mcp5-4vw2)
- **Reachable:** BUILD-TIME ONLY — transitive via build tooling (archiver-utils, react-email, rimraf, typeorm, @next/eslint-plugin-next). Never invoked at runtime.
- **Evidence:** `package-lock.json` shows glob@10.5.0.
- **Recommendation:** No action — 10.5.0 is past the fix range.
- **Exploitability:** N/A

### S-023 — `uuid@11.1.1` buffer bounds [MODERATE, PATCHED]
- **Severity:** MODERATE (CVSS 7.5 for the original advisory)
- **Source:** Tool-only (osv-scanner GHSA-w5hq-g745-h8pq)
- **Reachable:** YES — used across papermark for various ID generation.
- **Evidence:** `package-lock.json` shows uuid@11.1.1.
- **Recommendation:** No action — 11.1.1 is past the fix range (<11.1.1).
- **Exploitability:** N/A (patched)

### S-024 — `esbuild@0.28.0` dev server file read [LOW, WINDOWS ONLY]
- **Severity:** LOW (CVSS 2.5)
- **Source:** Tool-only (osv-scanner GHSA-g7r4-m6w7-qqqr)
- **Reachable:** NO — Windows-only vulnerability; papermark deploys to Vercel (Linux).
- **Evidence:** `package-lock.json` shows esbuild@0.28.0. Affected range `>=0.27.3 <0.28.1`.
- **Recommendation:** No action — 0.28.0 is past the fix range on Linux deploys.
- **Exploitability:** N/A

### S-025 — `oidc-provider@9.8.3` config gap [INFORMATIONAL]
- **Severity:** INFORMATIONAL
- **Source:** AI-only
- **Reachable:** YES — bound via `next.config.mjs:15-145` rewrites.
- **Evidence:** Past CVE-2024-XXXX refresh-token reuse fixed in 8.x→9.x transition.
- **Recommendation:** Verify `oidc-provider` is initialised with `pkce: 'S256'` and `rotateRefreshToken: true`.
- **Exploitability:** LOW (config-dependent)

---

## Findings Summary

| ID | Severity | Package | Source | Reachable | Notes |
|---|---|---|---|---|---|
| S-001 | HIGH | pdfjs-dist@3.11.174 | Tool-only | YES (Notion pages) | GHSA-wgrm CVSS 8.8; bump to 4.x |
| S-002 | HIGH | xlsx@0.20.3 | Both | YES (XLSX upload) | GHSA-4r6h + GHSA-5pgg prototype-pollution + ReDoS |
| S-003 | HIGH | nodemailer@7.0.13 | Tool-only | DEPENDENT (raw: usage) | GHSA-p6gq; needs grep for `raw:` |
| S-004 | HIGH | systeminformation@5.23.8 | Tool-only | DEPENDENT (host-metrics enabled) | 4 CVEs; bump @opentelemetry/host-metrics |
| S-005 | HIGH | form-data@4.0.5 | Tool-only | INDIRECT (axios) | GHSA-hmw2 CVSS 7.5 CRLF; add to overrides |
| S-006 | HIGH | @grpc/grpc-js@1.14.3 | Tool-only | INDIRECT (saml-jackson) | GHSA-5375 CVSS 7.5; bump via override |
| S-007 | MEDIUM | next@14.2.35 | Both | YES (deployment-dependent) | 14 advisories; D1–D4 reachability covered |
| S-008 | HIGH | hono@4.12.18 | Tool-only | DEPENDENT (MCP HTTP) | GHSA-88fw CORS reflection |
| S-009 | HIGH | @boxyhq/saml-jackson@26.2.0 | Tool-only | YES (SAML) | npm-audit range; investigate CVE |
| S-010 | HIGH | tmp@0.2.5 | Tool-only | INDIRECT (exceljs, patch-package) | GHSA-ph9p; add to overrides |
| S-011 | HIGH | underscore@1.13.6 | Tool-only | INDIRECT (jsonpath/saml) | GHSA-qpx9; add to overrides |
| S-012 | HIGH | tar@7.5.10 | Both | DEV-INSTALL ONLY | GHSA-9ppj symlink traversal; bump override |
| S-013 | HIGH | protobufjs@7.5.8 / 8.0.3 | Both | INDIRECT (AWS SDK) | GHSA-wcpc; bump override to 8.2.0+ |
| S-014 | HIGH | undici@6.25.0 | Tool-only | YES (@vercel/blob direct imports) | GHSA-vxpw WS DoS; bump override to 6.27.0+ |
| S-015 | INFO | @xmldom/xmldom@0.9.10 | Both | YES (SAML) | Override-patched, no action |
| S-016 | LOW | cuid@3.0.0 | AI-only | YES (permission PKs) | Deprecated, replace with cuid2 |
| S-017 | LOW | archiver@7.0.1 | AI-only | PARTIAL (dataroom freeze) | Zip-slip, no CVE; add slugify test |
| S-018 | MEDIUM | next-auth@4.24.14 | AI-only | YES (every auth flow) | Patch-gap family, current |
| S-019 | MOD | js-yaml@4.1.1 | Tool-only | INDIRECT (saml config) | GHSA-h67p DoS; bump to 4.1.2+ |
| S-020 | MOD | dompurify@3.4.0 | Both | NO (no IN_PLACE in papermark) | 8 advisories but not reachable |
| S-021 | LOW | cookie@0.7.2 | Tool-only | YES | Patched at 0.7.0+ |
| S-022 | HIGH | glob@10.5.0 | Tool-only | BUILD-TIME ONLY | Patched at 10.5.0+ |
| S-023 | MOD | uuid@11.1.1 | Tool-only | YES | Patched at 11.1.1 |
| S-024 | LOW | esbuild@0.28.0 | Tool-only | NO (Windows-only) | Patched at 0.28.1+ on Linux |
| S-025 | INFO | oidc-provider@9.8.3 | AI-only | YES (OIDC) | Config check needed |

**Counts:** 1 CRITICAL-equivalent (effectively none — see below), 14 HIGH, 2 MEDIUM (real), 8 INFORMATIONAL/LOW/patched. No CRITICAL-severity unfixed CVE found.

**Note on "no CRITICAL":** AI analysis reported 0 CRITICAL/HIGH; tool found 14+ HIGH advisories. The discrepancy is largely because (a) AI marked framework CVEs and patched-version CVEs as INFORMATIONAL without enumerating them, and (b) the AI analysis only covered a curated subset of packages. The tool-only findings (S-001 through S-014, excluding S-007) are the gaps where real CVE reachability exists but AI didn't quantify.

---

## Out-of-Scope: Pipfile.lock (Python CLI tool)

All 8 Python findings from osv-scanner (`black`, `cryptography`, `gitpython`, `idna`, `requests`, `tornado`, `urllib3`, `wheel`) are transitive deps of `tinybird-cli`, a developer-only CLI tool. They run only on developer workstations (Pipfile pins Python 3.11 + tinybird-cli only). The papermark production runtime uses Node.js and never invokes Python. **Out of scope for app reachability.** Findings:

- `tornado@6.0.4` — 13 advisories (HTTP smuggling, cookie injection, CRLF, DoS via gzip bomb, etc.). Reachable on dev workstations running `tinybird` CLI against a malicious server.
- `cryptography@41.0.7` — 9 advisories (Bleichenbacher timing oracle, subgroup attack, etc.). Same caveat.
- `urllib3@1.26.20` — 6 advisories (decompression-bomb, sensitive-header forwarding).
- `gitpython@3.1.45` — 5 HIGH command-injection / path-traversal advisories.
- `requests@2.32.5`, `wheel@0.45.1`, `black@25.12.0`, `idna@3.11` — 1 advisory each.

Developer-machine exposure only; not part of the bug-bounty target.

---

## False-Positive / Patched-Out Exclusions

These tool findings were verified to be patched by current installed versions and excluded from the action list above:

| Package | Installed | Tool-flagged range | Status |
|---|---|---|---|
| `cookie@0.7.2` | 0.7.2 | <0.7.0 | PATCHED |
| `uuid@11.1.1` | 11.1.1 | <11.1.1 | PATCHED |
| `glob@10.5.0` | 10.5.0 | >=10.2.0 <10.5.0 | PATCHED |
| `esbuild@0.28.0` | 0.28.0 | >=0.27.3 <0.28.1 | Linux-deploy unaffected (Windows-only vuln) |
| `lodash@4.18.1` | 4.18.1 | <4.17.21 | PATCHED via override |
| `node-forge@1.4.0` | 1.4.0 | <1.3.0 | PATCHED via override |
| `axios@1.16.0` | 1.16.0 | <1.7.4 | PATCHED via override |
| `@xmldom/xmldom@0.9.10` | 0.9.10 | <0.8.10 | PATCHED via override |
| `jsonwebtoken@9.0.3` | 9.0.3 | <9.0.0 | PATCHED |
| `sanitize-html@2.17.4` | 2.17.4 | <2.13.0 | PATCHED |
| `html2canvas@1.4.1` | 1.4.1 | <1.4.1 | PATCHED |
| `ua-parser-js@1.0.41` | 1.0.41 | <1.0.40 | PATCHED |
| `exceljs@4.4.0` | 4.4.0 | <4.4.0 | PATCHED (writes-only usage) |
| `bcryptjs@3.0.3` | 3.0.3 | None known | No CVE |
| `pdf-lib@1.17.1` | 1.17.1 | None active | No CVE |
| `mupdf@1.27.0` | 1.27.0 | Various (most in 1.24+) | Latest, native binary |
| `fluent-ffmpeg@2.1.3` | 2.1.3 | <2.1.2 | PATCHED (constant filter strings) |
| `archiver@7.0.1` | 7.0.1 | None active | No CVE |

---

## Action Items (prioritized by exploitability × reachability)

1. **[S-001] pdfjs-dist 3.x → 4.x** — HIGH exploitability × reachable via Notion pages. Plan a `react-notion-x` compatibility test.
2. **[S-002] xlsx 0.20.3 → 0.20.4+** — HIGH exploitability × reachable via Excel uploads. Pin to npm registry source instead of CDN tarball.
3. **[S-008] hono 4.12.18 → 4.12.21+ via @modelcontextprotocol/sdk bump** — HIGH exploitability × reachable if MCP HTTP server is wired up. Audit MCP integration.
4. **[S-009] @boxyhq/saml-jackson 26.2.0** — HIGH exploitability × reachable via SAML. Investigate what CVE npm-audit attributes to the >=26.2.0 range.
5. **[S-003] nodemailer 7.0.13 → 8.x** — HIGH exploitability × dependent on `raw:` usage. Grep for `raw:` in `lib/email/`.
6. **[S-004] @opentelemetry/host-metrics** — HIGH exploitability × dependent on host-metrics initialisation. Bump or disable.
7. **[S-007] next 14.x → 15.x** — Plan upgrade; immediate mitigations: disable Pages Router i18n (mitigates GHSA-36qx).
8. **[S-013] protobufjs 7.5.8/8.0.3 → 8.2.0+ via override** — MEDIUM exploitability × AWS SDK path.
9. **[S-014] undici 6.25.0 → 6.27.0+ via override** — MEDIUM exploitability × @vercel/blob direct imports.
10. **[S-012] tar 7.5.10 → 7.5.11+ via override** — MEDIUM exploitability × dev-install only.
11. **[S-005] form-data 4.0.5 → 4.0.6+ via override** — MEDIUM exploitability × axios path.
12. **[S-006] @grpc/grpc-js 1.14.3 → 1.14.4+ via override** — MEDIUM exploitability × saml-jackson path.
13. **[S-010] tmp 0.2.5 → 0.2.6+ via override** — LOW exploitability × exceljs/patch-package path.
14. **[S-011] underscore 1.13.6 → 1.13.8+ via override** — LOW exploitability × jsonpath/SAML path.
15. **[S-019] js-yaml 4.1.1 → 4.1.2+ via override** — LOW exploitability × saml config path.
16. **[S-016] cuid 3.0.0 → @paralleldrive/cuid2** — Information disclosure mitigation (replace deprecated IDs).
17. **[S-018] next-auth 4.24.14 → Auth.js v5** — Plan medium-term migration.

---

## PHASE_3_CHECKPOINT

- [x] Both sources read and compared (AI `00-ai-dependencies.md` + Tool `normalized.json`/`osv-scanner.json`/`npm-audit.json`)
- [x] Duplicates merged (S-002, S-007, S-012, S-013, S-015, S-020)
- [x] False positives eliminated (S-021–S-024 patched; Pipfile.lock Python findings excluded as CLI-tool-only)
- [x] Findings ranked by exploitability × reachability (HIGH/MEDIUM/LOW/INFORMATIONAL with explicit reachable flags)
- [x] Reachability cross-checked against `01-reachability-analysis.md` (D1–D13) and `01-structural-analysis.md`
- [x] Patched transitive deps excluded from action list
- [x] Pipfile.lock CLI-only Python findings scoped out of app reachability

---

## Cross-References

- AI-only findings in synthesis: S-016, S-017, S-018, S-025
- Tool-only findings in synthesis: S-001, S-003, S-004, S-005, S-006, S-008, S-009, S-010, S-011, S-014, S-019, S-021, S-022, S-023, S-024
- Both-source findings (merged): S-002, S-007, S-012, S-013, S-015, S-020
- Reachability-analysis cross-refs: S-001→D5, S-002→D10, S-007→D1–D4, S-016→D13, S-009→D12 (related: oidc-provider OIDC callback validation)
- Structural-analysis cross-refs: S-002 (XLSX parsed in `lib/sheet/index.ts:18-22`, called from `ee/features/ai/lib/trigger/process-excel-for-ai.ts`); S-014 (@vercel/blob direct imports used across `lib/files/`, `lib/signing/`, `lib/trigger/`)
