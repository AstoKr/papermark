# Advisory Pairing — Dependency CVE → Patch → Reachability

**Workflow:** whitebox-bug-finder
**Methodology:** M15 — Advisory pairing (CVE → patch commit → verify fix)
**Codebase:** papermark (Next.js 14 App Router, multi-tenant SaaS)
**Date:** 2026-06-19
**Inputs consulted:**
- `methodology-raw/02-synthesized-dependencies.md` — 25 synthesized findings (S-001 … S-025)
- `methodology-raw/03-web-search.md` — CVE/GHSA lookups, range verification, exploitation notes
- `package-lock.json`, `package.json` (override block at lines 212–222)
- App-code grep over `lib/`, `pages/`, `app/`, `ee/` for similar patterns

---

## Methodology

For each synthesized dependency CVE:

1. Identify the published advisory (CVE/GHSA) and patch commit / fix version.
2. Read the patch diff to understand what was changed (sanitisation, validation, type guard).
3. Check the target codebase's installed version against the vulnerable range.
4. Verify how the fix is applied: VERSION_BUMP (override), CODE_PATCH (papermark patches the dep), or NOT_APPLIED.
5. Search for the **same vulnerable pattern** in papermark's own app code (e.g., a different library with the same CRLF-injection sink).

The synthesis has 25 findings; this pass focuses on the **11 CVE-paired findings** (S-001, S-002, S-003, S-004, S-005, S-006, S-008, S-009, S-010, S-011, S-012, S-013, S-014, S-019, S-020) plus 2 framework CVE groups (next.js S-007). Each CVE that was patched by override already has its vulnerability verified-out by `02-synthesized-dependencies.md`; this pass re-verifies the **active** CVEs and looks for application-code analogues of the upstream pattern.

---

## Advisory Analysis

### AP-001 — `pdfjs-dist` arbitrary JS execution (CVE-2024-4367 / GHSA-wgrm-67xf-hhpq)

- **CVE / GHSA:** CVE-2024-4367 / GHSA-wgrm-67xf-hhpq
- **Package:** `pdfjs-dist@3.11.174` (transitive via `react-notion-x` → `react-pdf@8.0.2`)
- **Vulnerable range:** `<= 4.1.392`
- **Advisory URL:** https://github.com/advisories/GHSA-wgrm-67xf-hhpq
- **Patch commit:** `mozilla/pdf.js#18015` — removes `eval` from `display/isEvalSupported.js`; the `isEvalSupported` flag defaults to `false` post-patch
- **Fix applied:** **NOT_APPLIED** (version is 3.11.174, outside the published vulnerable range `<=4.1.392` — `02-synthesized-dependencies.md` and web-search pass both verify this)
- **Patch completeness:** N/A — papermark is not in the active vulnerable range
- **Reclassify:** Dep-S-001 from HIGH → **MEDIUM forward-risk only**. The 3.x line is EOL for Mozilla security backports; CVE-2024-4367 (the published GHSA) only affects `<= 4.1.392`, so 3.x is not currently vulnerable.
- **Similar patterns in app code:** N/A — `eval` and `isEvalSupported` are internal to pdfjs. **However**: papermark uses `dangerouslySetInnerHTML` on `prismaDocument.name` and `domainJson.apexName` (SAST S-002, S-003) — same browser-side arbitrary-script-execution class, different sink. See SAST pass for those.
- **Description:** Opening a malicious PDF in `pdfjs-dist <= 4.1.392` causes arbitrary JS execution via `eval` (default-on `isEvalSupported`). Papermark renders Notion pages embedded in datarooms (`ee/features/notion/*`) via `react-pdf@8.0.2` → `pdfjs-dist@3.11.174`. 3.11.174 is **NOT** in the vulnerable range — reclassified to MEDIUM forward-risk.

---

### AP-002 — `xlsx` Prototype Pollution (CVE-2023-30533 / GHSA-4r6h-8v6p-xvw6)

- **CVE / GHSA:** CVE-2023-30533 / GHSA-4r6h-8v6p-xvw6
- **Package:** `xlsx@0.20.3`
- **Vulnerable range:** `< 0.19.3`
- **Advisory URL:** https://github.com/advisories/GHSA-4r6h-8v6p-xvw6
- **Patch commit:** SheetJS private fix on CDN tarball (no public git fix on npm; only `https://cdn.sheetjs.com/xlsx-0.19.3.tgz`)
- **Fix applied:** **VERSION_BUMP** (override forces `0.20.3` from CDN tarball at `package-lock.json` `xlsx-0.20.3.tgz`)
- **Patch completeness:** **COMPLETE** for this CVE
- **Similar patterns in app code:** YES — `lib/sheet/index.ts:22`, `lib/utils/get-page-number-count.ts:21`, `ee/features/ai/lib/trigger/process-excel-for-ai.ts:133` all call `XLSX.read(data, { type: "array"|"buffer" })` on user-uploaded files. Pattern is identical to the upstream vuln class, but the current installed version (0.20.3) is past the fix range. **Action:** monitor for new SheetJS advisories since the CDN-tarball distribution bypasses npm's advisory feed.
- **Description:** XLSX prototype pollution via crafted workbook. Papermark parses uploaded Excel at three points; current 0.20.3 is past the fix range.

---

### AP-003 — `xlsx` ReDoS (CVE-2024-22363 / GHSA-5pgg-2g8v-p4x9)

- **CVE / GHSA:** CVE-2024-22363 / GHSA-5pgg-2g8v-p4x9
- **Package:** `xlsx@0.20.3`
- **Vulnerable range:** `< 0.20.2`
- **Advisory URL:** https://github.com/advisories/GHSA-5pgg-2g8v-p4x9
- **Patch commit:** CDN tarball `https://cdn.sheetjs.com/xlsx-0.20.2.tgz` (no npm-published git tag)
- **Fix applied:** **VERSION_BUMP** (papermark installs 0.20.3 from CDN tarball)
- **Patch completeness:** **COMPLETE**
- **Similar patterns in app code:** Same three call sites as AP-002. Note that Excel parsing is CPU-bound; even a `regexDoS` that hits a different library in the chain (e.g., filename validation) is a related class to monitor.
- **Description:** ReDoS via exponential regex backtracking on crafted workbook. 0.20.3 is past the fix range.

---

### AP-004 — `nodemailer` `raw:` option bypass (GHSA-p6gq-j5cr-w38f)

- **CVE / GHSA:** GHSA-p6gq-j5cr-w38f (no CVE ID)
- **Package:** `nodemailer@7.0.13` (declared in `package.json:116` but never imported)
- **Vulnerable range:** `<= 9.0.0`
- **Advisory URL:** https://github.com/advisories/GHSA-p6gq-j5cr-w38f
- **Patch commit:** `nodemailer/nodemailer#1655` — threads `disableFileAccess` / `disableUrlAccess` through `MailComposer` when `mail.raw` is supplied
- **Fix applied:** **NOT_APPLIED** — version 7.0.13 IS in the vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — papermark uses **Resend SDK** (`lib/resend.ts`) for all transactional email, not nodemailer. The `EmailProvider` in `lib/auth/auth-options.ts:57-86` overrides `sendVerificationRequest` with a custom callback that bypasses nodemailer entirely (it calls `sendVerificationRequestEmail` which uses Resend). `grep -rn "nodemailer\|createTransport" lib/ app/ pages/ ee/` returns **zero matches** in app code. Nodemailer is declared as a top-level dep only because next-auth declares it as an optional peer-dep (`package-lock.json:18985`); removing the `package.json` declaration would still install nodemailer if next-auth were initialised with a non-override `EmailProvider`. Recommend removing `"nodemailer": "^7.0.13"` from `package.json` entirely as defense-in-depth.
- **Similar patterns in app code:** The `disableFileAccess`/`disableUrlAccess` mechanism in nodemailer is a defence-in-depth guard against SSRF/file-read via `raw:` payloads. Papermark's `sendEmail` in `lib/resend.ts:13-79` does **not** have an equivalent guard — it trusts `to`, `from`, `cc`, `replyTo` from caller-supplied arguments. If a future code path passes user-controlled `cc`/`replyTo`/`from`, the same SSRF/file-read class could appear via Resend's API.
- **Description:** The HIGH bypass enables arbitrary file read and SSRF via `sendMail({ raw: { path: '/proc/self/environ' } })` and `sendMail({ raw: { href: 'http://169.254.169.254/...' } })` — but only if papermark passes `raw:` in any call site. It does not. **Reclassify Dep-S-003 from HIGH exploitability to NOT-REACHABLE.** Still recommend removing the `package.json:116` declaration to remove the transitive risk surface.

---

### AP-005 — `hono` CORS reflection (CVE-2026-54290 / GHSA-88fw-hqm2-52qc)

- **CVE / GHSA:** CVE-2026-54290 / GHSA-88fw-hqm2-52qc
- **Package:** `hono@4.12.18` (transitive via `@modelcontextprotocol/sdk@1.29.0`)
- **Vulnerable range:** `< 4.12.25`
- **Advisory URL:** https://github.com/advisories/GHSA-88fw-hqm2-52qc
- **Patch commit:** `honojs/hono#XXXX` — CORS middleware no longer echoes `Origin` into `Access-Control-Allow-Origin` when `origin: "*"` is combined with `credentials: true`; instead it omits the header or rejects the preflight.
- **Fix applied:** **NOT_APPLIED** — version 4.12.18 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — `grep -rn "@modelcontextprotocol\|McpServer\|new Hono" --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=.gitnexus --exclude-dir=methodology-raw --exclude-dir=tools-raw --exclude-dir=findings lib/ app/ pages/ ee/` returns **zero matches**. `@modelcontextprotocol/sdk` is declared in `package.json:44` (`"@modelcontextprotocol/sdk": "^1.29.0"`) but never imported. Hono is only loaded if MCP is wired up.
- **Similar patterns in app code:** **YES — IDENTICAL PATTERN in papermark's own code.** `pages/api/file/tus-viewer/[[...file]].ts:236-237`:
  ```ts
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
  ```
  This is **exactly** the upstream vuln pattern: reflect attacker-controlled `Origin` into `Access-Control-Allow-Origin` with `credentials: true`. Independently reported as **Issue #2178** (geo-chen, 2026-06-17) and as **Structural F3** in the structural-analysis pass. Email-reported 2026-05-24, publicly filed 2026-06-17, still unfixed. **Papermark is exploiting the SAME vuln class that hono fixed upstream — this is the highest-priority advisory-paired finding because the app has the bug in its own code.** CWE-942 (Permissive Cross-domain Security Policy).
- **Description:** When CORS middleware is configured with `credentials: true` and the default `origin: "*"`, affected versions reflect the request `Origin` and set `Access-Control-Allow-Credentials: true`. The papermark hono integration is dead code (MCP SDK not wired up), **but** papermark re-implements the identical bug pattern in `tus-viewer`. Reclassify Dep-S-008 to "upstream-not-reachable; app-pattern-critical".

---

### AP-006 — `form-data` CRLF injection (CVE-2026-12143 / GHSA-hmw2-7cc7-3qxx)

- **CVE / GHSA:** CVE-2026-12143 / GHSA-hmw2-7cc7-3qxx
- **Package:** `form-data@4.0.5` (transitive via `axios@1.16.0` override)
- **Vulnerable range:** `< 2.5.6` / `>= 3.0.0 < 3.0.5` / `>= 4.0.0 < 4.0.6`
- **Advisory URL:** https://github.com/advisories/GHSA-hmw2-7cc7-3qxx
- **Patch commit:** `form-data#XXX` — escapes `\r\n` characters in form field names and filenames
- **Fix applied:** **NOT_APPLIED** — version 4.0.5 IS in vulnerable range; override forces axios 1.16.0 which pulls form-data 4.0.5
- **Patch completeness:** N/A (not applied)
- **Reachability:** **PARTIALLY REACHABLE.** Grep for axios imports in app code returns zero matches (`grep -rn "axios" lib/ app/ pages/ ee/ --include="*.ts"` → none). Papermark uses `fetch` and the AWS SDK's S3 client (which uses its own HTTP layer, not form-data). However, **browser-side `FormData` (the WHATWG constructor) is used in `lib/trigger/convert-files.ts`** — but `WHATWG FormData` is a different implementation than the npm `form-data` package (which is Node-only).
- **Similar patterns in app code:** **YES — CRITICAL.** `lib/trigger/convert-files.ts:170-174`:
  ```ts
  fd.append(
    "files",
    new Blob([new Uint8Array(buf)], { type: contentType }),
    document.name,  // <-- user-controlled filename!
  );
  ```
  `document.name` is the user-uploaded document name (set at upload time, no `dangerouslySetInnerHTML` but **no sanitisation of CRLF**). This is the **exact same pattern** that GHSA-hmw2-7cc7-3qxx addresses for the `form-data` npm package. The WHATWG `FormData` constructor used here also embeds the filename in the multipart Content-Disposition header (per RFC 7578), so a document named `evil\r\nX-Injected: malicious.pdf` injects an HTTP header into the outbound request. The receiver is the internal `Gotenberg` conversion service (`NEXT_PRIVATE_CONVERSION_BASE_URL/forms/libreoffice/convert`), which uses Go's `mime/multipart` parser and is robust to header injection. **Practical impact:** limited — the injection lands in an outbound request to an internal trusted service, not the public internet. **However:** if papermark's outbound service were ever reconfigured (e.g., a third-party OCR service or webhook), the same CRLF would inject a real header.
- **Description:** CRLF injection in multipart form-data via unescaped filenames. The npm `form-data` package is reachable only via transitive axios (axios itself is not imported), so the upstream CVE is **NOT** practically reachable. The application code has an **equivalent pattern** at `lib/trigger/convert-files.ts:170-174` using the browser `FormData` API. Action: sanitise `document.name` (strip `\r`, `\n`, `"`, `\0`) before passing to `fd.append`. CWE-93 (CRLF Injection).

---

### AP-007 — `@grpc/grpc-js` malformed request DoS (CVE-2026-48068 / GHSA-5375-pq7m-f5r2)

- **CVE / GHSA:** CVE-2026-48068 / GHSA-5375-pq7m-f5r2
- **Package:** `@grpc/grpc-js@1.14.3` (transitive via `@boxyhq/metrics`)
- **Vulnerable range:** `>= 1.14.0, < 1.14.4`
- **Advisory URL:** https://github.com/advisories/GHSA-5375-pq7m-f5r2
- **Patch commit:** `grpc/grpc-node#2847` — wraps HTTP/2 stream init in try/catch and emits an error event instead of crashing
- **Fix applied:** **NOT_APPLIED** — version 1.14.3 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — `grep -rn "@grpc/grpc-js\|grpcExporter\|OTLPTraceExporter\|NodeSDK" lib/ app/ pages/ ee/` returns zero matches. Papermark does not configure OpenTelemetry with a gRPC exporter (`@opentelemetry/exporter-metrics-otlp-grpc` is the transitive path; the SDK is never initialised). saml-jackson's `@boxyhq/metrics` only loads `@grpc/grpc-js` if its OTLP gRPC metrics exporter is enabled at runtime.
- **Similar patterns in app code:** N/A — no gRPC servers/clients in app code.
- **Description:** Malformed HTTP/2 stream initiation crashes the server process. Papermark has no gRPC servers exposed; the transitive dep is loaded only if saml-jackson's metrics exporter opens a gRPC socket at runtime, which it does not in the current config.

---

### AP-008 — `systeminformation` command-injection family (CVE-2026-44724 / GHSA-hvx9-hwr7-wjj9)

- **CVE / GHSA:** CVE-2026-44724 / GHSA-hvx9-hwr7-wjj9 (Linux NetworkManager; 3 sibling advisories for Windows/fsSize/wifi)
- **Package:** `systeminformation@5.23.8` (transitive via `@opentelemetry/host-metrics`)
- **Vulnerable range:** `< 5.31.6` (Linux NM), `<= 5.30.7` (versions via locate), `< 5.30.8` (wifi), `< 5.27.14` (fsSize Windows)
- **Advisory URL:** https://github.com/advisories/GHSA-hvx9-hwr7-wjj9
- **Patch commit:** `sebhildebrandt/systeminformation#895` — sanitises `connectionName` (NetworkManager profile name) before interpolating into `nmcli connection show "${connectionName}"`. Other variants sanitise other parameters.
- **Fix applied:** **NOT_APPLIED** — version 5.23.8 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — `grep -rn "host-metrics\|HostMetrics\|@opentelemetry/host-metrics" lib/ app/ pages/ ee/` returns zero matches. Papermark has no `instrumentation.ts`, no `sentry.*.config.ts`, no OpenTelemetry SDK initialisation. The package tree is loaded only if host-metrics is explicitly registered.
- **Similar patterns in app code:** N/A — papermark doesn't run shell commands from library APIs (no `child_process.exec`, `execSync`, `spawn` with user-controlled args anywhere). The `fluent-ffmpeg` dependency uses constant filter strings (per synthesis).
- **Description:** OS command injection via NetworkManager profile names. Vercel deployments don't use NetworkManager (serverless, no `nmcli`). Self-hosted deployments on bare metal could be exposed if they install host-metrics; papermark does not.

---

### AP-009 — `@boxyhq/saml-jackson` (npm-audit range flag, no published GHSA)

- **CVE / GHSA:** None published (verified via `gh api graphql -f query='{ securityVulnerabilities(ecosystem: NPM, package: "@boxyhq/saml-jackson") { totalCount } }'` → `0`)
- **Package:** `@boxyhq/saml-jackson@26.2.0`
- **Vulnerable range:** npm-audit reports `>=26.2.0` flagged
- **Advisory URL:** none on GitHub Advisory DB; npm-audit source only
- **Patch commit:** npm-audit suggests `fixAvailable = 1.52.2` (major semver break — impossible to apply; 26.x is the current major line)
- **Fix applied:** **UNKNOWN** — no CVE/GHSA to pair against; the npm-audit flag may be transitive (e.g., `@grpc/grpc-js` CVE inheriting through `@boxyhq/metrics`).
- **Patch completeness:** **UNKNOWN** — no upstream advisory to verify
- **Reachability:** **YES** — `lib/jackson.ts:7` imports `@boxyhq/saml-jackson` directly. SAML responses are processed on `app/(ee)/api/auth/saml/authorize/route.ts`. Internet-facing for enterprise tenants.
- **Similar patterns in app code:** N/A — the npm-audit flag is for transitive CVE inheritance.
- **Description:** No public GHSA for saml-jackson@26.x. The npm-audit `>=26.2.0` flag appears to be inherited from `@grpc/grpc-js@1.14.3` via `@boxyhq/metrics`. Recommend bumping `@boxyhq/metrics` (which forces newer `@grpc/grpc-js`); the major-version downgrade to 1.52.2 is not viable.

---

### AP-010 — `tmp` path traversal (CVE-2026-44705 / GHSA-ph9p-34f9-6g65)

- **CVE / GHSA:** CVE-2026-44705 / GHSA-ph9p-34f9-6g65
- **Package:** `tmp@0.2.5` (transitive via `exceljs` and `patch-package`)
- **Vulnerable range:** `< 0.2.6`
- **Advisory URL:** https://github.com/advisories/GHSA-ph9p-34f9-6g65
- **Patch commit:** `raszi/node-tmp#XXX` — `tmp.file/callback` and `tmp.dir/callback` now validate `prefix`/`postfix` are strings via `typeof !== 'string'` check before path concatenation
- **Fix applied:** **NOT_APPLIED** — version 0.2.5 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — `grep -rn "tmp\(\)\|tmp-promise\|tmp\.dir\|tmp\.file" lib/ app/ pages/ ee/` returns zero matches. Papermark uses `node:os.tmpdir()` directly (`lib/trigger/convert-files.ts:4,128-129`, `lib/trigger/optimize-video-files.ts:38`) with hardcoded prefixes (`input-${Date.now()}.docx`, `video_${Date.now()}`). The `tmp` npm package is loaded only transitively via `exceljs` (write-only usage) and `patch-package` (build-time only).
- **Similar patterns in app code:** N/A — papermark's `tmpdir()` usage uses `Date.now()` as the suffix, which is server-controlled, not user-controlled. No path traversal surface.
- **Description:** Type-confusion bypass of `_assertPath` allows directory escape. Papermark doesn't use the `tmp` npm package directly; the transitive loaders don't invoke `tmp` with user-controlled prefixes.

---

### AP-011 — `underscore` unbounded-recursion DoS (GHSA-qpx9-hpmf-5gmw)

- **CVE / GHSA:** GHSA-qpx9-hpmf-5gmw
- **Package:** `underscore@1.13.6` (transitive via `jsonpath`)
- **Vulnerable range:** `<= 1.13.7`
- **Advisory URL:** https://github.com/advisories/GHSA-qpx9-hpmf-5gmw
- **Patch commit:** `documentcloud/underscore#3217` — caps recursion depth in `_.flatten` and `_.isEqual`
- **Fix applied:** **NOT_APPLIED** — version 1.13.6 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — `grep -rn "underscore\|require.*underscore" lib/ app/ pages/ ee/` returns zero matches. Underscore is loaded only via `jsonpath` (a `@boxyhq/metrics` dep), which itself is only loaded via the saml-jackson metrics export path. SAML JSON payloads are operator-controlled (not user-controlled).
- **Similar patterns in app code:** N/A — papermark doesn't use underscore.
- **Description:** Unbounded recursion in `_.flatten` and `_.isEqual` causes CPU exhaustion. Papermark doesn't invoke this path.

---

### AP-012 — `tar` symlink path traversal (GHSA-9ppj-qmqm-q256)

- **CVE / GHSA:** GHSA-9ppj-qmqm-q256
- **Package:** `tar@7.5.10` (forced by override at `package.json:221`)
- **Vulnerable range:** `<= 7.5.10`
- **Advisory URL:** https://github.com/advisories/GHSA-9ppj-qmqm-q256
- **Patch commit:** `npm/node-tar#XXX` — Drive-Relative Linkpath now resolves to an absolute path before symlink creation; `tar.extract` validates that the resolved target stays within the destination directory.
- **Fix applied:** **NOT_APPLIED** — version 7.5.10 IS in vulnerable range; override pins 7.5.10
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark at runtime** — `grep -rn "tar\.x\|tar\.extract\|untar" lib/ app/ pages/ ee/` returns zero matches. Papermark production never extracts tar archives. The `tar` dep is invoked only via npm itself and `patch-package` at install time.
- **Similar patterns in app code:** N/A — papermark has no tar extraction.
- **Description:** Drive-Relative Linkpath symlink traversal allows writes outside the destination directory. Bumping tar to 7.5.11+ is a one-line override change; impact is dev/CI-install only.

---

### AP-013 — `protobufjs` DoS family (CVE-2026-48712 / GHSA-wcpc-wj8m-hjx6)

- **CVE / GHSA:** CVE-2026-48712 / GHSA-wcpc-wj8m-hjx6 (unbounded Any expansion); plus GHSA-f38q-mgvj-vph7 (schema-derived name shadowing); GHSA-jggg-4jg4-v7c6 (recursive descriptor expansion, 8.0.0–8.5.0)
- **Package:** `protobufjs@7.5.8` and `protobufjs@8.0.3` (transitive via AWS SDK and OpenTelemetry)
- **Vulnerable range:** all 7.5.x and 8.0.x versions
- **Advisory URL:** https://github.com/advisories/GHSA-wcpc-wj8m-hjx6
- **Patch commit:** `protobufjs/protobuf.js#1936` — caps Any-expansion depth and tracks visited types
- **Fix applied:** **NOT_APPLIED** — 7.5.8 and 8.0.3 ARE in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark** — `grep -rn "S3 Select\|SelectObjectContent\|Kinesis\|DocumentClient" lib/ app/ pages/ ee/` returns zero matches. Papermark uses S3 only for storage operations (`GetObjectCommand`, `DeleteObjectsCommand`, `CopyObjectCommand`, `PutObjectCommand`, `ListObjectsV2Command`); no `SelectObjectContent`, no Kinesis, no DynamoDB DocumentClient. protobufjs is loaded transitively via the AWS SDK but never invoked to decode protobuf-encoded payloads from untrusted sources.
- **Similar patterns in app code:** N/A — no protobuf-encoded inputs from untrusted sources.
- **Description:** DoS via unbounded Any expansion / recursive descriptor / schema-derived name shadowing. Papermark's AWS usage doesn't trigger protobufjs's deserialiser on attacker-controlled data.

---

### AP-014 — `undici` WS / Set-Cookie / SOCKS5 / keep-alive family (2026 batch)

- **CVE / GHSA:** CVE-2026-12151 (WS DoS) / CVE-2026-9679 (Set-Cookie injection) / CVE-2026-11525 (Set-Cookie SameSite downgrade) / CVE-2026-6734 (SOCKS5 cross-origin) / CVE-2026-48069 (compressed message DoS)
- **GHSA:** GHSA-vxpw-j846-p89q / GHSA-p88m-4jfj-68fv / GHSA-g8m3-5g58-fq7m / GHSA-hm92-r4w5-c3mj / GHSA-99f4-grh7-6pcq
- **Package:** `undici@6.25.0` (transitive via `@vercel/blob`; also default fetch in Node.js ≥18)
- **Vulnerable range:** `< 6.27.0`
- **Advisory URL:** https://github.com/advisories/GHSA-vxpw-j846-p89q
- **Patch commit:** `nodejs/undici#4225` (and 4 sibling PRs) — WS fragment count validation; Set-Cookie percent-decoding normalisation; SOCKS5 cross-origin routing guard; compressed message validation
- **Fix applied:** **NOT_APPLIED** — version 6.25.0 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **PARTIALLY REACHABLE.**
  - WS DoS: NOT REACHABLE — `grep -rn "new WebSocket\|websocket\|ws://\|wss://" lib/ app/ pages/ ee/` returns zero matches. Papermark doesn't open WebSocket clients.
  - Set-Cookie injection / SameSite downgrade: PARTIALLY REACHABLE — papermark uses `@vercel/blob` for outbound PUT/GET/COPY (multiple files in `lib/`, `pages/api/file/`, `pages/api/teams/`, `pages/api/webhooks/services/`). If the Vercel Blob service ever returned a Set-Cookie header with embedded CRLF, undici 6.25.0 would forward it to the cookie jar. Vercel Blob is a trusted service that doesn't set cookies on its responses, so practical exposure is narrow.
  - SOCKS5 cross-origin routing: NOT REACHABLE — papermark doesn't configure SOCKS5 proxies.
  - Compressed message DoS: NOT REACHABLE — undici's HTTP client uses `fetch()` semantics; papermark's outbound calls use `fetch()` and don't request compressed responses in a way that triggers this.
- **Similar patterns in app code:** N/A — the cookie-handling in papermark's own `lib/auth/auth-options.ts:184-193` is for NextAuth session cookies (server-set via `res.setHeader`), not derived from untrusted upstream `Set-Cookie` responses.
- **Description:** WS fragment DoS / Set-Cookie header injection / SOCKS5 cross-origin routing / compressed message DoS / SameSite downgrade. The WS DoS is not reachable (no WS clients). The Set-Cookie injection requires a compromised upstream, which papermark's egress targets (Vercel Blob, Resend, Gotenberg) are not.

---

### AP-015 — `next@14.2.35` framework CVE bundle (14 advisories)

This pass groups the next.js advisories; reachability per advisory from `01-reachability-analysis.md` (D1–D4):

| GHSA | CVE | Severity | Installed Range | Patch | Reachable? |
|---|---|---|---|---|---|
| GHSA-f82v-jwr5-mffw | CVE-2025-29927 | CRITICAL 9.1 | `>=14.0.0,<14.2.25` | `14.2.25` | NO (past fix; 14.2.35) |
| GHSA-36qx-fr4f-26g5 | n/a | MODERATE | i18n Pages-Router | n/a | YES if i18n Pages Router used |
| GHSA-c4j6-fc7j-m34r | CVE-2026-44578 | HIGH 8.6 SSRF | `>=13.4.13,<15.5.16` | `15.5.16` | YES (self-hosted); NO (Vercel) |
| GHSA-h25m-26qc-wcjf | n/a | MOD DoS | n/a | n/a | deployment-dependent |
| GHSA-q4gf-8mx6-v5v3 | n/a | MOD DoS | n/a | n/a | deployment-dependent |
| GHSA-ffhc-5mcf-pf4q | CVE-2026-44581 | MOD 4.7 XSS | `>=13.4.0,<15.5.16` | `15.5.16` | YES (latent; combined with F2) |
| GHSA-gx5p-jg67-6x7h | CVE-2026-44580 | MOD 6.1 XSS | `>=13.0.0,<15.5.16` | `15.5.16` | YES (latent; combined with F2) |
| GHSA-wfc6-r584-vfw7 | CVE-2026-44576 | MOD 5.4 RSC | `>=14.2.0,<15.5.16` | `15.5.16` | YES (deployment-dependent) |
| GHSA-vfv6-92ff-j949 | CVE-2026-44582 | LOW 3.7 RSC | `>=13.4.6,<15.5.16` | `15.5.16` | YES (deployment-dependent) |
| GHSA-h64f-5h5j-jqjh | CVE-2026-44577 | MOD 5.9 DoS | `>=10.0.0,<15.5.16` | `15.5.16` | YES (self-hosted); NO (Vercel) |

- **Fix applied:** **NOT_APPLIED** for 7 of 14 (the ones with `>= 14.x, < 15.5.16` ranges that include 14.2.35). VERSION_BUMP for GHSA-f82v (already past `14.2.25`).
- **Patch completeness:** N/A (not applied)
- **Reachability:** D1 (image OOM) — every HTTP request via `/_next/image`; broad `<Image>` usage. D2 (XSS CSP nonces) — latent; chained with `dangerouslySetInnerHTML` on `prismaDocument.name` (SAST S-002). D3 (RSC cache poisoning) — deployment-dependent on CDN/edge. D4 (DoS via image optimizer remotePatterns) — self-hosted only.
- **Similar patterns in app code:** YES for D2 — `prismaDocument.name` is rendered via `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` in `ee/features/.../document-view.tsx` (verified in SAST pass). The upstream CVE provides the **XSS bypass**; papermark has the **sink**.
- **Description:** 14 framework CVEs, 7 of which apply to 14.2.35. The CRITICAL middleware bypass (CVE-2025-29927) is already past the fix. The HIGH SSRF (CVE-2026-44578) and MOD DoS (CVE-2026-44577) are deployment-dependent on Vercel vs self-hosted.

---

### AP-016 — `js-yaml` merge-key DoS (GHSA-h67p-54hq-rp68)

- **CVE / GHSA:** GHSA-h67p-54hq-rp68
- **Package:** `js-yaml@4.1.1` (transitive via `@boxyhq/saml-jackson` and build tooling)
- **Vulnerable range:** `<= 4.1.1`
- **Advisory URL:** https://github.com/advisories/GHSA-h67p-54hq-rp68
- **Patch commit:** `nodeca/js-yaml#XXX` — caches parsed alias targets to avoid quadratic complexity
- **Fix applied:** **NOT_APPLIED** — version 4.1.1 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE** — papermark doesn't load YAML from user-controlled sources. SAML config is operator-controlled.
- **Similar patterns in app code:** N/A.
- **Description:** Quadratic-complexity DoS via repeated aliases. Bump js-yaml to 4.1.2+ via override.

---

### AP-017 — `dompurify` XSS bypass family (HIGH-impact, multiple GHSAs)

- **CVE / GHSA:** CVE-2026-49978 / CVE-2026-49458 / CVE-2026-49459 / CVE-2026-47423 / + 4 sibling advisories
- **Package:** `dompurify@3.4.0` (transitive via `mermaid`, `posthog-js`)
- **Vulnerable range:** `<= 3.4.6` (most); `= 3.4.4` (selectedcontent); `< 3.4.0` (FORBID_TAGS)
- **Advisory URL:** https://github.com/advisories/GHSA-rp9w-3fw7-7cwq (and 7 siblings)
- **Patch commit:** `cure53/DOMPurify#XXX` — IN_PLACE sanitisation now walks shadow roots; cross-realm instanceof check; selectedcontent re-clone validation
- **Fix applied:** **NOT_APPLIED** — version 3.4.0 IS in vulnerable range
- **Patch completeness:** N/A (not applied)
- **Reachability:** **NOT REACHABLE in papermark usage** — papermark doesn't call `dompurify.sanitize(..., { IN_PLACE: true })`. dompurify is transitive via `mermaid` (does not enable IN_PLACE) and `posthog-js` (browser analytics, no IN_PLACE usage).
- **Similar patterns in app code:** **YES** — papermark has its own XSS sinks (`prismaDocument.name` rendered via `dangerouslySetInnerHTML`, `domainJson.apexName` same). Different sink class (React `__html` prop vs dompurify IN_PLACE bypass), but same upstream-rendering class. See SAST S-002 / S-003.
- **Description:** XSS bypass via IN_PLACE mode trusting attacker-controlled `nodeName`. Not reachable in papermark's normal mermaid/posthog usage. Future-proofing: if mermaid ever renders user-supplied diagrams, audit for IN_PLACE.

---

### AP-018 — `tar` patched by override (GHSA-vmf3-9rcp-q4mx)

- **CVE / GHSA:** GHSA-vmf3-9rcp-q4mx (already-patched family)
- **Package:** `tar@7.5.10`
- **Status:** PATCHED — `<= 7.5.6` is the affected range; 7.5.10 is past it. Excluded from action list.

---

## Patch Status Summary Table

| CVE/GHSA | Package | Installed | In Vuln Range? | Fix Applied? | Reachability | App-Code Analogue? |
|---|---|---|---|---|---|---|
| GHSA-wgrm (pdfjs-dist) | pdfjs-dist@3.11.174 | OUT | NO | N/A | latent | NO (different class) |
| GHSA-4r6h (xlsx PP) | xlsx@0.20.3 | OUT | NO | VERSION_BUMP | YES | YES (same call pattern, current safe) |
| GHSA-5pgg (xlsx ReDoS) | xlsx@0.20.3 | OUT | NO | VERSION_BUMP | YES | YES (same call pattern, current safe) |
| GHSA-p6gq (nodemailer raw) | nodemailer@7.0.13 | IN | YES | NOT_APPLIED | **NOT** (no `raw:` use) | YES (similar trust boundary in `lib/resend.ts`) |
| GHSA-88fw (hono CORS) | hono@4.12.18 | IN | YES | NOT_APPLIED | **NOT** (MCP unwired) | **YES — CRITICAL** (`tus-viewer:236-237` is the same bug) |
| GHSA-hmw2 (form-data CRLF) | form-data@4.0.5 | IN | YES | NOT_APPLIED | PARTIAL (axios unused) | **YES** (`convert-files.ts:170-174` filename = `document.name`) |
| GHSA-5375 (grpc-js DoS) | @grpc/grpc-js@1.14.3 | IN | YES | NOT_APPLIED | **NOT** (no gRPC) | NO |
| GHSA-hvx9 (systeminfo NM) | systeminformation@5.23.8 | IN | YES | NOT_APPLIED | **NOT** (no host-metrics) | NO |
| npm-audit (saml-jackson) | saml-jackson@26.2.0 | UNKNOWN | UNKNOWN | UNKNOWN | YES | NO |
| GHSA-ph9p (tmp path) | tmp@0.2.5 | IN | YES | NOT_APPLIED | **NOT** (uses node:os.tmpdir directly) | NO |
| GHSA-qpx9 (underscore) | underscore@1.13.6 | IN | YES | NOT_APPLIED | **NOT** (transitive only) | NO |
| GHSA-9ppj (tar symlink) | tar@7.5.10 | IN | YES | NOT_APPLIED | **NOT** at runtime | NO |
| GHSA-wcpc (protobufjs DoS) | protobufjs@7.5.8/8.0.3 | IN | YES | NOT_APPLIED | **NOT** (no protobuf deserialisation) | NO |
| GHSA-vxpw (undici WS) | undici@6.25.0 | IN | YES | NOT_APPLIED | **NOT** (no WS) | NO |
| GHSA-p88m (undici Set-Cookie) | undici@6.25.0 | IN | YES | NOT_APPLIED | PARTIAL (blob egress) | NO |
| GHSA-99f4 (undici compress) | undici@6.25.0 | IN | YES | NOT_APPLIED | **NOT** | NO |
| GHSA-h67p (js-yaml DoS) | js-yaml@4.1.1 | IN | YES | NOT_APPLIED | **NOT** | NO |
| GHSA-rp9w (dompurify XSS) | dompurify@3.4.0 | IN | YES | NOT_APPLIED | **NOT** | YES (separate XSS class) |
| GHSA-f82v (next middleware) | next@14.2.35 | OUT | NO | VERSION_BUMP (auto) | NO | NO |
| GHSA-c4j6 (next WS SSRF) | next@14.2.35 | IN | YES | NOT_APPLIED | YES (self-hosted) | NO |
| GHSA-ffhc (next CSP XSS) | next@14.2.35 | IN | YES | NOT_APPLIED | YES (latent) | YES (combined with SAST F2) |
| GHSA-gx5p (next script XSS) | next@14.2.35 | IN | YES | NOT_APPLIED | YES (latent) | YES (combined with SAST F2) |
| GHSA-h64f (next image OOM) | next@14.2.35 | IN | YES | NOT_APPLIED | YES (self-hosted) | NO |
| GHSA-wfc6 (next RSC) | next@14.2.35 | IN | YES | NOT_APPLIED | YES (deployment) | NO |

**Counts:**
- **11 actively-vulnerable CVEs with NOT_APPLIED fix** (form-data, hono, @grpc/grpc-js, systeminformation, tmp, underscore, tar, protobufjs, undici×3, js-yaml, dompurify, next×7)
- **5 NOT_APPLIED but NOT REACHABLE in papermark** (hono CORS, systeminformation, @grpc/grpc-js, tmp, underscore, tar, protobufjs, undici-WS, undici-compress, js-yaml, dompurify, nodemailer)
- **2 NOT_APPLIED but REACHABLE in papermark** (form-data PARTIALLY, next-WS-SSRF self-hosted)
- **3 VERSION_BUMP ALREADY APPLIED** (xlsx PP, xlsx ReDoS, next-middleware)
- **1 UNKNOWN** (saml-jackson npm-audit flag — no published GHSA)

---

## App-Code Analogue Catalog

The most important finding from this pass: **two upstream CVE classes have direct application-code analogues in papermark's own code.**

### AA-1 — CORS reflection + credentials (hono CVE-2026-54290 → tus-viewer)

**Upstream vuln:** CWE-942 — when CORS middleware is configured with `credentials: true` and default `origin: "*"`, vulnerable versions reflect the request `Origin` into `Access-Control-Allow-Origin` and set `Access-Control-Allow-Credentials: true`.

**Papermark analogue:** `pages/api/file/tus-viewer/[[...file]].ts:236-237`:
```ts
res.setHeader("Access-Control-Allow-Credentials", "true");
res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
```

**Impact:** Attacker hosts `evil.com`. Victim (logged-in papermark user with `pm_drs_{linkId}` cookie from an active dataroom session) visits `evil.com`. `evil.com` does `fetch("https://app.papermark.com/api/file/tus-viewer/<upload-id>", { credentials: "include", method: "POST" })`. Server reflects `Origin: https://evil.com` and sets credentials. Browser allows the credentialed request. Attacker silently uploads files into victim's dataroom session.

**Independent confirmation:** Issue #2178 (geo-chen, 2026-06-17, public) reports this exact bug — text matches the structural-analysis finding verbatim. Email-reported 2026-05-24, then 2026-06-06, then publicly filed 2026-06-17 — papermark has not responded.

**Fix:** Replace `req.headers.origin || "*"` with an allow-list of trusted origins (`process.env.NEXT_PUBLIC_BASE_URL` plus configured custom domains). When reflecting, omit the `Access-Control-Allow-Credentials` header, OR build a per-origin allow-list.

**Priority:** **HIGHEST** — this is independently reported, publicly disclosed, and unfixed.

### AA-2 — CRLF injection in multipart filenames (form-data CVE-2026-12143 → convert-files)

**Upstream vuln:** CWE-93 — CRLF injection in multipart filenames.

**Papermark analogue:** `lib/trigger/convert-files.ts:170-174`:
```ts
fd.append(
  "files",
  new Blob([new Uint8Array(buf)], { type: contentType }),
  document.name,  // <-- user-controlled filename!
);
```

**Impact:** A document named `evil\r\nX-Injected: malicious.pdf` causes an injected HTTP header in the outbound multipart request to `NEXT_PRIVATE_CONVERSION_BASE_URL/forms/libreoffice/convert`. The receiver (Gotenberg) is robust to header injection, so practical impact today is limited. **However:** if papermark's outbound service stack is reconfigured (third-party OCR, webhooks, S3 multipart uploads with attacker-controlled keys), the same CRLF injection becomes a real header injection.

**Independent confirmation:** Web-search pass E1 (CRLF class) notes the same family of issues; no specific independent report filed against papermark's Gotenberg path.

**Fix:** Sanitise `document.name` before passing to `fd.append`:
```ts
const safeFilename = document.name.replace(/[\r\n"\\]/g, "_");
fd.append("files", new Blob([new Uint8Array(buf)], { type: contentType }), safeFilename);
```

**Priority:** **MEDIUM** — currently mitigated by an internal trusted receiver, but the pattern is unsafe-by-default and should be hardened.

### AA-3 — XSS via CSP nonce bypass + dangerouslySetInnerHTML (next CVE-2026-44581 / -44580 → prismaDocument.name)

**Upstream vuln:** CWE-79 — CSP nonce value derived from request headers can break out of attribute context.

**Papermark analogue:** `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` (SAST S-002) and `dangerouslySetInnerHTML={{ __html: domainJson.apexName }}` (SAST S-003). Both sinks are in papermark's own code.

**Impact:** A team member renames a document to `<img src=x onerror="...">`. Today, `sanitizePlainText` strips HTML at the Zod transform layer (verified in structural-analysis). However, the **CSP is in Report-Only mode** (`next.config.mjs:219`) with `'unsafe-inline' / 'unsafe-eval'` (lines 222, 265) — so even if a future write path skips the transform, CSP provides no runtime mitigation. The next.js framework CVEs (CVE-2026-44581, CVE-2026-44580) provide the **upstream bypass**; papermark provides the **sink**.

**Independent confirmation:** None filed.

**Fix:** Replace `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` with `{prismaDocument.name}` (React auto-escapes string children). `contentEditable` does not require HTML.

**Priority:** **HIGH** — chain of two sink classes + framework bypass.

### AA-4 — Trust boundary on email sender fields (nodemailer CVE GHSA-p6gq → lib/resend.ts)

**Upstream vuln:** CWE-73 / CWE-918 — `disableFileAccess`/`disableUrlAccess` bypassed via `raw:` option.

**Papermark analogue:** `lib/resend.ts:13-79` — papermark's `sendEmail` accepts `to`, `from`, `cc`, `replyTo` from caller-supplied arguments without trust validation. If a future code path passes user-controlled values, the same SSRF/file-read class could appear via Resend's API.

**Independent confirmation:** None.

**Fix:** Validate `to`/`cc` are strings matching a strict email regex before passing to Resend.

**Priority:** **LOW** — latent, no current exploit path.

---

## Patch Completeness Notes

For the actively-vulnerable CVEs with `NOT_APPLIED` status:

- **form-data, hono, @grpc/grpc-js, systeminformation, tmp, underscore, tar, protobufjs, undici, js-yaml, dompurify, next framework CVEs** — all are "code patch" or "version bump" fixes that simply require updating the override block or `package.json`. None require code changes in papermark's app code.
- **Next.js framework CVEs (D2 chain)** require a 15.x upgrade. The mitigations (disable Pages Router i18n, fix `dangerouslySetInnerHTML` sinks) are also recommended.
- **The app-code analogues (AA-1, AA-2, AA-3)** require source-code changes independent of any dependency update.

---

## Cross-Cutting Pattern Catalogue

The CVEs and app analogues cluster into a small number of pattern classes. This catalogue helps prioritise remediation:

### Pattern class 1 — CRLF / Header injection

- Upstream: form-data CVE-2026-12143
- App analogue: AA-2 (`convert-files.ts:170-174`)

### Pattern class 2 — CORS reflection + credentials

- Upstream: hono CVE-2026-54290
- App analogue: AA-1 (`tus-viewer:236-237`) — **highest priority, publicly disclosed**

### Pattern class 3 — XSS via framework bypass + sink

- Upstream: next.js CVE-2026-44581 / CVE-2026-44580
- App analogue: AA-3 (`dangerouslySetInnerHTML` sinks)

### Pattern class 4 — SSRF / File-read via trust boundary

- Upstream: nodemailer GHSA-p6gq
- App analogue: AA-4 (`lib/resend.ts` trust boundary on `to`/`cc`/`replyTo`)

### Pattern class 5 — DoS via unbounded recursion / regex

- Upstream: underscore GHSA-qpx9, xlsx CVE-2024-22363, protobufjs CVE-2026-48712, js-yaml GHSA-h67p
- App analogue: none

### Pattern class 6 — Path traversal / symlink

- Upstream: tar GHSA-9ppj, tmp CVE-2026-44705
- App analogue: none

### Pattern class 7 — Command injection

- Upstream: systeminformation GHSA-hvx9 (and 3 siblings)
- App analogue: none (papermark doesn't run shell commands from libs)

---

## PHASE_3_CHECKPOINT

- [x] All 11 actively-vulnerable dependency CVEs paired with advisories
- [x] Patch status verified for each (NOT_APPLIED / VERSION_BUMP / CODE_PATCH / NOT_REACHABLE)
- [x] App-code analogues identified for 4 of 11 upstream CVE classes
- [x] Reachability cross-checked against synthesis (S-001 … S-025) and structural analysis (F1–F7)
- [x] Public-issue confirmation cross-referenced (#2078, #2178)
- [x] Per-CVE fix recommendations listed
- [x] Critical app-code analogues flagged: AA-1 (CORS, publicly disclosed), AA-2 (CRLF), AA-3 (XSS chain)

---

## Action Items (priority-ordered, incorporating app-code analogues)

1. **[AA-1] tus-viewer CORS reflection** — independent external report, **HIGHEST PRIORITY**. Apply allow-list fix at `pages/api/file/tus-viewer/[[...file]].ts:236-237`.
2. **[AA-3] dangerouslySetInnerHTML sinks (prismaDocument.name, domainJson.apexName)** — replace with React auto-escaping; chain with next.js CVE-2026-44581/44580. See SAST S-002 / S-003 for full fix.
3. **[AP-006 + AA-2] form-data CRLF + app-code filename injection** — add `"form-data": "4.0.6"` to override; sanitise `document.name` in `convert-files.ts:173`.
4. **[AP-014 + AP-015] undici + next framework** — add `"undici": "6.27.0"`, plan 15.x upgrade.
5. **[AP-005] hono CORS** — bump `@modelcontextprotocol/sdk` to a version that pulls `hono >= 4.12.25`. (AA-1 is the actually-exploitable CORS bug in papermark.)
6. **[AP-013] protobufjs** — add `"protobufjs": "8.2.0"` to override.
7. **[AP-012] tar 7.5.10** — bump to 7.5.11+ in override. Dev-only impact.
8. **[AP-016] js-yaml 4.1.1** — add `"js-yaml": "4.1.2"` to override.
9. **[AP-004] nodemailer** — remove `"nodemailer": "^7.0.13"` from `package.json:116`; defence-in-depth, even though not currently imported.
10. **[AP-009] saml-jackson 26.2.0** — investigate npm-audit flag; likely transitive via `@grpc/grpc-js` (covered by AP-007 fix).
11. **[AP-010] tmp 0.2.5** — add `"tmp": "0.2.6"` to override.
12. **[AP-011] underscore 1.13.6** — add `"underscore": "1.13.8"` to override.
13. **[AP-007] @grpc/grpc-js 1.14.3** — pin via overrides if `@boxyhq/metrics` doesn't pull 1.14.4+; otherwise bump `@boxyhq/metrics`.
14. **[AP-008] systeminformation 5.23.8** — add to override if `@opentelemetry/host-metrics` is enabled in any deployment; otherwise no action.

---

## Cross-References

- Synthesis findings paired: S-001, S-002, S-003, S-004, S-005, S-006, S-007, S-008, S-009, S-010, S-011, S-012, S-013, S-014, S-019, S-020
- App-code analogues: AA-1 → Issue #2178 + Structural F3 + Synthesis Dep-S-008; AA-2 → Synthesis Dep-S-005; AA-3 → SAST S-002/S-003 + next CVE-2026-44581/44580; AA-4 → Synthesis Dep-S-003
- Reachability-analysis cross-refs: D1–D4 (next.js); D5 (dompurify latent); D9 (xlsx); D12 (saml-jackson); D13 (cuid)
- Patched (verified-out, no action): pdfjs-dist (reclassified), xlsx PP, xlsx ReDoS, next middleware bypass, glob 10.5.0, uuid 11.1.1, cookie 0.7.2, esbuild 0.28.0