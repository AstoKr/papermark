# Web Synthesis — Cross-Reference & Exploitation (M2 final)

**Workflow:** whitebox-bug-finder
**Methodology:** M2 (web synthesis — final)
**Date:** 2026-06-19
**Inputs combined:**
- `methodology-raw/02-synthesized-secrets.md`
- `methodology-raw/02-synthesized-sast.md`
- `methodology-raw/02-synthesized-dependencies.md`
- `methodology-raw/02-synthesized-git-history.md`
- `methodology-raw/03-web-search.md`

**Goal:** For each synthesized finding, verify against public CVE / writeup / issue-tracker evidence from the web pass, add exploitation technique from the writeup where one exists, and rank by web-confirmed confidence × bounty potential.

---

## Confidence Tier Definitions

| Tier | Definition |
|---|---|
| **WEB-CONFIRMED** | Independent public disclosure (CVE / GHSA / GH issue / vendor advisory) matches the synthesized finding. Exploitation technique available from the writeup. |
| **PATTERN-CONFIRMED** | No papermark-specific CVE or writeup exists, but the class of bug is widely reported across the ecosystem (NextAuth documented footguns, generic timing-attack class, generic IDOR). The synthesis finding matches an established pattern. |
| **VENDOR-TRUST-BOUNDARY** | The vulnerable code is in a third-party library. Papermark's usage does or does not exercise the vulnerable API. |
| **UNCONFIRMED (novel)** | No public match. Synthesized finding may be a papermark-specific bug — potentially higher bounty value because no prior art exists. |

---

## 1. Web-Confirmed Findings (highest confidence × highest bounty potential)

### 1.1 WEB-CONFIRMED — Folder DELETE/RENAME IDOR (Synth Git S-001, S-002)

| | |
|---|---|
| **Synthesized finding** | Git S-001 (`pages/api/progress-token.ts` missing auth), Git S-002 (`document-processing-status.ts` missing auth), structural F1/F2 (Folder DELETE/RENAME IDOR) |
| **Public disclosure** | **GH Issue #2078 — Lighthouse Security Research, 2026-02-19, OPEN (4+ months)** |
| **URL** | https://github.com/mfts/papermark/issues/2078 |
| **Confirmed match** | EXACT — issue body text matches the structural pass at `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:53-55` (DELETE) and `pages/api/teams/[teamId]/folders/manage/index.ts:70-73` (RENAME). Issue reporter additionally flags two more endpoints the structural pass confirmed: `add-to-dataroom.ts:21-22` and `create-from-folder.ts:22-24`. |
| **Reporter verdict** | Public, no team response, no PR, no fix in 4+ months. Issue text includes the correct fix proposal (`{ id: folderId, teamId: teamId }`) and notes that adjacent endpoints (`folders/move.ts:113`, `documents/move.ts:58`, `webhooks/services/.../index.ts:511`) ARE correctly scoped. |
| **Exploitation technique** | E4 — Per-route opt-in IDOR in team-scoped APIs. Attacker creates free account, becomes ADMIN/MANAGER on Team A, enumerates `folderId` from Team B (via shared links / dashboards / error messages / cuid enumeration noted in Dep S-016), issues `DELETE /api/teams/teamA/folders/manage/<teamB_folderId>`. Handler validates attacker session + Team A membership + role, but fetches the folder by `id: folderId` only. Folder is found (cuids are globally unique), delete proceeds against Team B's folder. Cascade deletes Team B's child folders and documents. Audit log records attacker's user, but the resource belongs to a different tenant. |
| **Confidence** | **WEB-CONFIRMED** — independent external researcher reported the same bug class 4 months ago |
| **Bounty potential** | **HIGH** (multi-tenant destructive integrity loss; no audit-trail indicator pointing at the attacker; the CWE-639 IDOR pattern is widely bounty-eligible) |
| **Status** | UNFIXED — papermark has had 4 months to act and has not |

### 1.2 WEB-CONFIRMED — CORS Misconfiguration on `tus-viewer` (Synth Structural F3, Dep S-008)

| | |
|---|---|
| **Synthesized finding** | Structural F3 (tus-viewer CORS misconfig at `pages/api/file/tus-viewer/[[...file]].ts:234-237`), Dep S-008 (hono 4.12.18 transitive CORS reflection) |
| **Public disclosure** | **GH Issue #2178 — geo-chen, 2026-06-17, OPEN (2 days old)**, preceded by an email disclosure on 2026-05-24 (no response) and follow-up on 2026-06-06 (no response) |
| **URL** | https://github.com/mfts/papermark/issues/2178 |
| **Confirmed match** | EXACT — issue body text matches the structural pass verbatim: "Access-Control-Allow-Credentials: true is set alongside an Access-Control-Allow-Origin header that reflects whatever Origin value the browser sends." Same CVE class as hono CVE-2026-54290 (GHSA-88fw-hqm2-52qc). |
| **Reporter's attack scenario** | "An attacker who knows a valid dataroom link (a shared recipient, for example) can trick a victim viewer into visiting a malicious page. The malicious page then silently uploads arbitrary files to the victim's dataroom session, injecting unauthorized content into the dataroom." Identical to the structural pass's call chain. |
| **Exploitation technique** | E1 — CORS Origin reflection + Credentials (CWE-942). Attacker hosts `evil.com` with `<script>fetch("https://app.papermark.com/api/file/tus-viewer/<upload-id>", { credentials: "include", method: "POST" })</script>`. Victim's browser sends preflight OPTIONS with `Origin: https://evil.com`. Server reflects `Origin` into `Access-Control-Allow-Origin` and sets `Access-Control-Allow-Credentials: true`. Browser allows credentialed request, exposes response to `evil.com`. Same browser context with `pm_drs_{linkId}` cookie from an active dataroom session → attacker silently uploads files into the victim's dataroom session. |
| **Confidence** | **WEB-CONFIRMED** — same CORS bug independently reported via email (no response) then publicly filed 2 days before this whitebox pass. Same root cause as Hono CVE-2026-54290. |
| **Bounty potential** | **HIGH** (CORS + file-upload + dataroom-injection = multi-vector impact; same root cause class as PortSwigger-documented labs) |
| **Status** | UNFIXED — papermark has had 2 days; the structural pass independently verified the misconfig on 2026-06-19 |

### 1.3 WEB-CONFIRMED — `xlsx@0.20.3` (Synth Dep S-002) — NO active CVE, but in-risk forward-zone

| | |
|---|---|
| **Synthesized finding** | Dep S-002 (`xlsx@0.20.3` Prototype Pollution + ReDoS at `lib/sheet/index.ts:18-22` and `ee/features/ai/lib/trigger/process-excel-for-ai.ts:15`) |
| **Public disclosure** | CVE-2023-30533 / GHSA-4r6h-8v6p-xvw6 (`< 0.19.3`, PATCHED); CVE-2024-22363 / GHSA-5pgg-2g8v-p4x9 (`< 0.20.2`, PATCHED at 0.20.3) |
| **URL** | https://github.com/advisories/GHSA-4r6h-8v6p-xvw6 ; https://github.com/advisories/GHSA-5pgg-2g8v-p4x9 |
| **Confirmed match** | **NO (active CVE)** — Papermark's `xlsx@0.20.3` is past both vulnerable ranges. The synthesis was correct to call the past CVEs patched. However, the forward risk remains: SheetJS distributes fixes only via CDN tarball (`xlsx-0.20.3.tgz`); npm-audit cannot backport security patches; 0.20.3 line is approaching EOL. |
| **Exploitation technique** | E-CVE-2023-30533 — `XLSX.read(data, { type: "array" })` triggers prototype pollution when reading crafted files; CVE-2024-22363 — crafted workbook triggers exponential regex backtracking, CPU exhaustion. |
| **Confidence** | **PATTERN-CONFIRMED + VENDOR-TRUST-BOUNDARY** — no active CVE at 0.20.3, but the reachable code path (`lib/sheet/index.ts:18-22`, `process-excel-for-ai.ts:15`) processes untrusted workbooks uploaded to datarooms. The forward-risk window grows over time. |
| **Bounty potential** | **LOW** (no active CVE at installed version — not a "vulnerability" under most bounty scopes) — but flag for monitoring |
| **Status** | Informational — track SheetJS distribution-channel and EOL status |

### 1.4 WEB-CONFIRMED — `form-data@4.0.5` CRLF injection (Synth Dep S-005)

| | |
|---|---|
| **Synthesized finding** | Dep S-005 (form-data 4.0.5 CRLF injection via axios, transitive) |
| **Public disclosure** | CVE-2026-12143 / GHSA-hmw2-7cc7-3qxx |
| **URL** | https://github.com/advisories/GHSA-hmw2-7cc7-3qxx |
| **Severity** | HIGH, CVSS 8.7 |
| **Confirmed match** | **YES — version 4.0.5 IS in the vulnerable range** (`>= 4.0.0 < 4.0.6`) |
| **Exploitation technique** | Public PoC in advisory: `form.append('email"\r\nX-Injected: true\r\nfake="', 'user@example.com')` produces an injected `X-Injected: true` header. With `--<boundary>` sequences, attacker can append extra parts (e.g., `name="is_admin"`) that downstream parsers accept as legitimate. |
| **Reachable?** | Papermark's outbound S3 uploads use signed URLs (not multipart), so practical exposure is narrow. But the version is in the vulnerable range, so defense-in-depth fix warranted. |
| **Confidence** | **WEB-CONFIRMED** — active CVE at installed version |
| **Bounty potential** | **MEDIUM** (defense-in-depth; active CVE; narrow reachability) |
| **Status** | Unfixed; add `"form-data": "4.0.6"` to overrides |

### 1.5 WEB-CONFIRMED — `undici@6.25.0` family of advisories (Synth Dep S-014)

| | |
|---|---|
| **Synthesized finding** | Dep S-014 (undici 6.25.0 — WS DoS, Set-Cookie injection, SOCKS5 cross-origin routing, compressed message DoS) |
| **Public disclosure** | CVE-2026-12151 / GHSA-vxpw-j846-p89q (WS fragment DoS); CVE-2026-9679 / GHSA-p88m-4jfj-68fv (Set-Cookie header injection); CVE-2026-11525 / GHSA-g8m3-5g58-fq7m (Set-Cookie SameSite downgrade); CVE-2026-6734 / GHSA-hm92-r4w5-c3mj (SOCKS5 cross-origin routing); CVE-2026-48069 / GHSA-99f4-grh7-6pcq (compressed message DoS) |
| **Severity** | HIGH (WS DoS, SOCKS5); MODERATE (Set-Cookie injection); LOW (SameSite downgrade) |
| **Vulnerable range** | `< 6.27.0` (and 7.x/8.x ranges) |
| **Confirmed match** | **YES — 6.25.0 IS in the `< 6.27.0` vulnerable range** |
| **Exploitation technique** | Each advisory independently reachable on different code paths. WS DoS: send fragmented WS frames that bypass the fragment-count limit. Set-Cookie injection: percent-decoding bypass injects additional Set-Cookie headers. |
| **Reachable?** | YES — undici is the default fetch implementation in Next.js when running on Node.js, AND is transitively pulled via `@vercel/blob` (DIRECTLY IMPORTED in app code: `lib/signing/mirror.ts`, `lib/files/copy-file.ts`, `lib/trigger/export-visits.ts`, `pages/api/webhooks/services/[...path]/index.ts`). |
| **Confidence** | **WEB-CONFIRMED** — five independent GHSAs all active at installed version |
| **Bounty potential** | **HIGH** (active CVEs in the production HTTP path; broad reachability via @vercel/blob direct imports) |
| **Status** | Unfixed; bump to `undici@6.27.0+` via override |

### 1.6 WEB-CONFIRMED — `tmp@0.2.5` path traversal (Synth Dep S-010)

| | |
|---|---|
| **Synthesized finding** | Dep S-010 (tmp 0.2.5 path traversal via exceljs and patch-package) |
| **Public disclosure** | CVE-2026-44705 / GHSA-ph9p-34f9-6g65 |
| **Severity** | HIGH |
| **Confirmed match** | **YES — 0.2.5 IS in the `< 0.2.6` vulnerable range** |
| **Exploitation technique** | Type-confusion bypass of `_assertPath` allows directory escape via non-string `prefix`/`postfix`/`template` arguments. |
| **Reachable?** | INDIRECT — exceljs uses tmp for write-temp-file flows only (papermark only writes); patch-package runs only at install time. |
| **Confidence** | **WEB-CONFIRMED** — active CVE at installed version |
| **Bounty potential** | **LOW** (write-only call site reduces practical impact, but defense-in-depth fix is trivial — add to overrides) |
| **Status** | Unfixed; add `"tmp": "0.2.6"` to overrides |

### 1.7 WEB-CONFIRMED — `@grpc/grpc-js@1.14.3` malformed request DoS (Synth Dep S-006)

| | |
|---|---|
| **Synthesized finding** | Dep S-006 (@grpc/grpc-js 1.14.3 — malformed request DoS via @boxyhq/metrics) |
| **Public disclosure** | CVE-2026-48068 / GHSA-5375-pq7m-f5r2 |
| **Severity** | HIGH, CVSS 7.5 |
| **Confirmed match** | **YES — 1.14.3 IS in the `>= 1.14.0, < 1.14.4` vulnerable range** |
| **Exploitation technique** | "An invalid incoming HTTP/2 stream initiation can cause a server process to crash." Affects all servers created using `@grpc/grpc-js`. |
| **Reachable?** | INDIRECT — only on SAML response processing paths where @boxyhq/metrics opens a gRPC socket. |
| **Confidence** | **WEB-CONFIRMED** — active CVE at installed version |
| **Bounty potential** | **MEDIUM** (DoS only, narrow reachability) |
| **Status** | Unfixed; bump `@grpc/grpc-js` to 1.14.4+ via `@boxyhq/metrics` override |

### 1.8 WEB-CONFIRMED — `protobufjs@7.5.8 / 8.0.3` DoS family (Synth Dep S-013)

| | |
|---|---|
| **Synthesized finding** | Dep S-013 (protobufjs DoS — unbounded Any expansion, schema-derived name shadowing, recursive descriptor expansion) |
| **Public disclosure** | CVE-2026-48712 / GHSA-wcpc-wj8m-hjx6 (unbounded Any); CVE-2026-54269 / GHSA-f38q-mgvj-vph7 (schema-derived names); CVE-2026-45740 / GHSA-jggg-4jg4-v7c6 (recursive descriptor) |
| **Severity** | HIGH 7.5 (unbounded Any); MODERATE (others) |
| **Confirmed match** | **YES — both 7.5.8 and 8.0.3 are in the `all 7.5.x and 8.0.x` vulnerable range** (patched 8.2.0) |
| **Reachable?** | INDIRECT — transitive via AWS SDK and OpenTelemetry. Reachable if AWS service returns maliciously-crafted protobuf (S3 Select, Kinesis). |
| **Confidence** | **WEB-CONFIRMED** — three active CVEs at installed versions |
| **Bounty potential** | **MEDIUM** (DoS only, narrow reachability) |
| **Status** | Unfixed; bump protobufjs to 8.2.0+ via override |

### 1.9 WEB-CONFIRMED — `systeminformation@5.23.8` command injection family (Synth Dep S-004)

| | |
|---|---|
| **Synthesized finding** | Dep S-004 (systeminformation 5.23.8 command injection — Linux NM, versions via locate, wifi retry, fsSize Windows) |
| **Public disclosure** | CVE-2026-44724 / GHSA-hvx9-hwr7-wjj9 (Linux NM, CVSS 7.8); CVE-2026-26318 / GHSA-5vv4-hvf7-2h46 (locate); CVE-2026-26280 / GHSA-9c88-49p5-5ggf (wifi retry); CVE-2025-68154 / GHSA-wphj-fx3q-84ch (fsSize Windows) |
| **Vulnerable range** | `>= 4.17.0, <= 5.31.5` (Linux NM); `<= 5.30.7` (locate); `< 5.30.8` (wifi); `< 5.27.14` (Windows) |
| **Confirmed match** | **YES — 5.23.8 IS in the Linux NM vulnerable range** |
| **Exploitation technique** | PoC from advisory: `nmcli connection add type dummy ifname si-nmghsa0 con-name 'si-ghsa$(id>/tmp/proof)$(env>/tmp/proof)'` → activate → call `require('./lib').networkInterfaces()` → arbitrary shell commands execute. Root cause: `networkInterfaces()` builds `nmcli connection show "${connectionName}"` via `execSync()`; `connectionName` is the unsanitized NetworkManager connection profile name. |
| **Reachable?** | DEPENDENT — only if `@opentelemetry/host-metrics` is initialised in the otel SDK config. Action: grep `instrumentation.ts` / `sentry.*.config.ts` for `host-metrics` registration. |
| **Confidence** | **WEB-CONFIRMED** — four active CVEs, one in range |
| **Bounty potential** | **HIGH** if host-metrics is registered (RCE via attacker-controlled NM profile name on Linux server) |
| **Status** | Unfixed; bump `@opentelemetry/host-metrics` to >=0.38.1 (forces systeminformation 5.27.14+) or disable |

### 1.10 WEB-CONFIRMED — `nodemailer@7.0.13` `raw:` option bypass (Synth Dep S-003)

| | |
|---|---|
| **Synthesized finding** | Dep S-003 (nodemailer 7.0.13 raw option bypass — GHSA-p6gq-j5cr-w38f) |
| **Public disclosure** | GHSA-p6gq-j5cr-w38f |
| **Severity** | HIGH, CVSS 7.1 |
| **Vulnerable range** | `<= 9.0.0` |
| **Confirmed match** | **DEPENDENT** — 7.0.13 IS in the `<= 9.0.0` range, but exploitability requires `sendMail({ raw: ... })` call sites with attacker-controlled content. Action: grep `lib/email/`, `pages/api/email/` for `raw:`. |
| **Exploitation technique** | Two paths from advisory: (1) Arbitrary file read: `sendMail({ raw: { path: '/proc/self/environ' } })` — nodemailer reads the file and sends it as the message body, bypassing `disableFileAccess: true`. (2) Full-response SSRF: `sendMail({ raw: { href: 'http://169.254.169.254/latest/meta-data/' } })` — server-side fetch delivers the entire response body in the email, bypassing `disableUrlAccess: true`. |
| **Confidence** | **WEB-CONFIRMED + DEPENDENT** — active CVE, reachability pending grep |
| **Bounty potential** | **HIGH** if `raw:` is used (full file read or AWS metadata exfiltration via email) |
| **Status** | Unfixed; grep first, then bump to nodemailer 9.0.1+ |

### 1.11 WEB-CONFIRMED — Hono CORS reflection CVE-2026-54290 (Synth Dep S-008)

| | |
|---|---|
| **Synthesized finding** | Dep S-008 (hono 4.12.18 — CORS reflection when `origin: "*"` + `credentials: true`) |
| **Public disclosure** | CVE-2026-54290 / GHSA-88fw-hqm2-52qc |
| **Severity** | HIGH, CVSS 7.1 |
| **Confirmed match** | **YES — 4.12.18 IS in the `< 4.12.25` vulnerable range** (same root cause as the independently-reported tus-viewer CORS bug, GH Issue #2178) |
| **Exploitation technique** | When CORS middleware is configured with `credentials: true` and `origin: "*"`, affected versions reflect the request `Origin` and set `Access-Control-Allow-Credentials: true`. Preflight succeeds for every origin including `null`. Any third-party page a logged-in user visits can read the app's cookie-authenticated endpoints and perform state-changing requests. |
| **Reachable?** | DEPENDENT — only if MCP integration wires hono into a runtime HTTP server. |
| **Confidence** | **WEB-CONFIRMED + DEPENDENT** — active CVE in transitive dep |
| **Bounty potential** | **HIGH** if MCP is wired (same root cause as the confirmed tus-viewer bug) |
| **Status** | Unfixed; bump `@modelcontextprotocol/sdk` to force hono >=4.12.25 |

### 1.12 WEB-CONFIRMED — `sanitize-html@2.17.4` lockfile-pinned but `package.json` declares vulnerable range

| | |
|---|---|
| **Synthesized finding** | Sec S-003 (placeholder values in `.env.example`); `sanitize-html` declared as `^2.17.3` in `package.json:35` while lockfile pins 2.17.4 |
| **Public disclosure** | CVE-2026-44990 / GHSA-rpr9-rxv7-x643 — XSS via `xmp` raw-text passthrough |
| **Severity** | **CRITICAL, CVSS 9.3** |
| **Vulnerable range** | `= 2.17.3` only |
| **Confirmed match** | **LATENT** — lockfile-pinned to 2.17.4 (PATCHED), but `package.json:35` `^2.17.3` allows clean `npm install` to resolve to 2.17.3 (vulnerable). |
| **Exploitation technique** | XSS via `xmp` raw-text passthrough. |
| **Confidence** | **WEB-CONFIRMED + LATENT** — CVE matches the package.json range exactly; only the lockfile prevents active exploit |
| **Bounty potential** | **MEDIUM** (latent — only exploitable if lockfile is regenerated; defense-in-depth fix is to tighten to `~2.17.4`) |
| **Status** | Latent; tighten `sanitize-html` declaration to `~2.17.4` |

### 1.13 WEB-CONFIRMED — Next.js CVE-2026-44578 (SSRF via WebSocket upgrades) + CVE-2026-44577 (image-optimizer OOM) — self-hosted exposure

| | |
|---|---|
| **Synthesized finding** | Dep S-007 (Next.js 14.2.35 framework CVE bundle, reachability D1/D4) |
| **Public disclosure** | CVE-2026-44578 / GHSA-c4j6-fc7j-m34r (HIGH, CVSS 8.6 SSRF); CVE-2026-44577 / GHSA-h64f-5h5j-jqjh (MODERATE, CVSS 5.9 image-optimizer OOM); CVE-2026-44581 / GHSA-ffhc-5mcf-pf4q (XSS via CSP nonces); CVE-2026-44580 / GHSA-gx5p-jg67-6x7h (XSS in beforeInteractive scripts); CVE-2026-44576 / GHSA-wfc6-r584-vfw7 + CVE-2026-44582 (RSC cache poisoning); plus older CVE-2025-29927 (CRITICAL 9.1, middleware bypass) which Papermark's 14.2.35 is past. |
| **Vulnerable range** | `>= 13.4.13, < 15.5.16` (and 16.x) |
| **Confirmed match** | **YES — 14.2.35 IS in the vulnerable range** for the 2026-batch CVEs |
| **Reachable?** | **DEPLOYMENT-DEPENDENT** — Vercel-hosted Papermark is NOT affected (Vercel applies automatic patches). Self-hosted enterprise deployments per GH Issue #2160 ARE exposed. |
| **Exploitation technique** | CVE-2026-44578: Self-hosted apps using the built-in Node.js server can be made to proxy requests to arbitrary internal/external destinations (incl. cloud metadata endpoints). CVE-2026-44577: `/_next/image` fetches local images entirely into memory without size limit. |
| **Confidence** | **WEB-CONFIRMED + DEPLOYMENT-DEPENDENT** |
| **Bounty potential** | **HIGH for self-hosted** deployments (per Issue #2160, papermark is self-hostable and enterprise users do this) |
| **Status** | Vercel: patched automatically. Self-hosted: requires Next.js 15.5.16 upgrade or manual patching |

### 1.14 WEB-CONFIRMED — Next.js XSS CVEs chain with SAST S-002 / S-003 (DOM-XSS)

| | |
|---|---|
| **Synthesized finding** | SAST S-002 (`dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` at `components/documents/document-header.tsx:604`), SAST S-003 (`domainJson.apexName` at `components/domains/domain-configuration.tsx:143`) |
| **Public disclosure** | CVE-2026-44581 / GHSA-ffhc-5mcf-pf4q (Next.js XSS via CSP nonces); CVE-2026-44580 / GHSA-gx5p-jg67-6x7h (Next.js XSS in beforeInteractive scripts) |
| **Severity** | MODERATE (4.7 and 6.1 individually) — but chains to HIGH when combined with the SAST sinks |
| **Confirmed match** | **CHAIN CONFIRMED** — Next.js CVE amplifies the SAST finding. Papermark's CSP is in **Report-Only** mode (`next.config.mjs:219`) and includes `'unsafe-inline' / 'unsafe-eval'` (lines 222, 265), so the CSP provides no runtime mitigation even if the CVE were patched. |
| **Exploitation technique** | E7 — Stored XSS via `dangerouslySetInnerHTML` on user-controlled data. Team member renames document to `<img src=x onerror="fetch('/api/teams/.../settings/billing', { method: 'POST', body: ... })">`. (Today) `sanitizePlainText` strips the tag at the Zod transform — sink is safe. (Future) Any new write path (Notion import, bulk CSV upload, restored legacy data) skips the transform → on every subsequent dashboard render, the XSS fires in every team member's session. NextAuth JWT is `httpOnly: true` → direct cookie theft not possible, but **authenticated mutation** on behalf of the victim IS possible. |
| **Confidence** | **WEB-CONFIRMED + PATTERN-CONFIRMED** — Next.js CVEs are confirmed; SAST chain is pattern-confirmed via known XSS sink class |
| **Bounty potential** | **HIGH** (CSP Report-Only + `'unsafe-inline'` + sink-at-render-layer is the textbook stored-XSS chain) |
| **Status** | Latent today; the sink is exploitable the moment any write path skips `sanitizePlainText` |

### 1.15 WEB-CONFIRMED — `next-auth@4.24.14` past all known patches (NO active CVE)

| | |
|---|---|
| **Synthesized finding** | Dep S-018 (next-auth 4.24.14 patch-gap family) |
| **Public disclosure** | CVE-2023-27490 / GHSA-7r7x-4c4q-c4qf (PKCE/state, PATCHED at 4.20.1); CVE-2022-31093, CVE-2022-29214, CVE-2022-24858 (callbackUrl open redirect, PATCHED at 4.x); GHSA-5jpx-9hw9-2fx4 (Email misdelivery, PATCHED at 4.24.12) |
| **Confirmed match** | **NO (active CVE)** — Papermark's 4.24.14 is past all known patch ranges. |
| **Confidence** | **WEB-CONFIRMED + NOT-REACHABLE** |
| **Bounty potential** | **N/A** |
| **Status** | Not a bug — patched |

---

## 2. Pattern-Confirmed Findings (no specific CVE, but documented pattern class)

### 2.1 PATTERN-CONFIRMED — `allowDangerousEmailAccountLinking: true` (Synth Structural F4)

| | |
|---|---|
| **Synthesized finding** | Structural F4 (Google/LinkedIn/SAML OAuth providers enable `allowDangerousEmailAccountLinking: true`) |
| **Public disclosure** | None specific — NextAuth explicitly documents the option as "DANGEROUS: This option is dangerous. Only use it when you really need it..." |
| **Exploitation technique** | E2 — OAuth provider email matching → session assigned to victim user. Attacker registers a Google account at `email = victim@victim-corp.com` (Google allows attacker-side creation depending on domain verification). Attacker navigates to `https://app.papermark.com/api/auth/signin/google` and completes the OAuth flow. Google returns profile `{ id, email: "victim@...", verified_email: true }`. NextAuth's internal signIn callback (no override in `lib/auth/auth-options.ts`) matches on email, finds existing `prisma.user` row, creates a session for the victim. Attacker is now logged in as the victim; team-scoped `getServerSession` checks pass. |
| **Confidence** | **PATTERN-CONFIRMED** — NextAuth documented footgun; widely reported across SaaS (typical HackerOne US payout: $5k-$15k) |
| **Bounty potential** | **HIGH** — undocumented ATO vector; the highest-impact unfixed finding class |
| **Status** | Unfixed; the documented NextAuth mitigation is to add a `signIn` callback that returns `false` when `account.provider !== "credentials"` and `existingUser.email !== profile.email` is false (or similar per-provider checks). |

### 2.2 PATTERN-CONFIRMED — AES-256-CTR without auth tag (Synth SAST S-001)

| | |
|---|---|
| **Synthesized finding** | SAST S-001 (AES-256-CTR without integrity protection on document passwords at `lib/utils.ts:595-622`) |
| **Public disclosure** | None specific to papermark. NIST SP 800-38A §5.5.2 explicitly warns CTR has no integrity. |
| **Exploitation technique** | E3 — Crypto malleability on AES-CTR ciphertext (CWE-310). XOR-flip ciphertext bit → forge plaintext character (or NUL out bytes). Attacker has DB write access (via separate compromise, backup leak, or SQLi). Attacker writes `iv + ciphertext` such that `decrypt(key, iv, ciphertext)` produces a known plaintext byte. Without auth tag, `decryptEncrpytedPassword` returns the forged plaintext silently. Server accepts the forged password on `Link.password` verification. |
| **Internal comparison primitive** | The Slack token path (`lib/integrations/slack/utils.ts:41-74`) uses AES-256-GCM with 12-byte IV and stores the auth tag — the correct pattern to mirror. |
| **Confidence** | **PATTERN-CONFIRMED** — generic cryptographic guidance; well-known class |
| **Bounty potential** | **MEDIUM** (threat model requires DB write access; the secondary defect — `digest("base64").substring(0, 32)` truncates to 32 ASCII chars not 32 bytes — reduces effective key space and embeds charset bias, independently worth flagging) |
| **Status** | Unfixed; migrate to AES-256-GCM with `getAuthTag()` / `setAuthTag()` |

### 2.3 PATTERN-CONFIRMED — QStash signature bypass on non-Vercel deployments (Synth Structural F5)

| | |
|---|---|
| **Synthesized finding** | Structural F5 (QStash signature verification short-circuited by env flag in `lib/cron/verify-qstash.ts:12-14`) |
| **Public disclosure** | None specific — env-flag bypass is a common misconfig pattern in Next.js routes. |
| **Exploitation technique** | E6 — Cron auth bypass. Attacker identifies self-hosted papermark deployment (or CI / staging where `VERCEL !== "1"`). Attacker POSTs to `https://target.papermark.com/api/cron/domains` (or 4 other cron routes) without any `Upstash-Signature` header. `lib/cron/verify-qstash.ts:12-14` returns early with no error. Handler proceeds to destructive action: `handleDomainUpdates` sends escalating reminder emails and on day 30 calls `deleteDomain`. `/api/cron/welcome-user` sends welcome emails to arbitrary userIds. Mass email-bombing + domain deletion = high-impact DoS + spam-amplification. |
| **Confidence** | **PATTERN-CONFIRMED** — generic env-flag bypass class |
| **Bounty potential** | **HIGH for self-hosted deployments** |
| **Status** | Unfixed; make QStash verification unconditional + use a real middleware matcher so the dev escape hatch can't accidentally be bypassed |

### 2.4 PATTERN-CONFIRMED — INTERNAL_API_KEY timing-unsafe comparison (Synth Structural F6)

| | |
|---|---|
| **Synthesized finding** | Structural F6 (`!==` comparison leaks secret byte-by-byte across 4 jobs routes) |
| **Public disclosure** | None specific — standard CWE-208 timing-attack class. |
| **Exploitation technique** | E5 — Timing attack on shared-secret bearer tokens. Attacker on a network path that can measure response latency (cloud-to-cloud is sufficient). For each byte position i, measures latency for 256 candidates and selects the candidate with the longest response time. `!==` short-circuits on first differing byte → matched prefix adds deterministic latency increase. For a 32-char random key: ~256 samples per byte × 32 bytes = ~8k requests for full recovery. |
| **Internal comparison primitive** | The codebase has the correct pattern at `ee/features/billing/cancellation/api/automatic-unpause-route.ts:43-50` — apply it to the 4 jobs routes. |
| **Confidence** | **PATTERN-CONFIRMED** — generic CWE-208 class |
| **Bounty potential** | **LOW-MEDIUM** (requires network-latency measurement capability; the API key is shared across 26 routes per Sec S-006, magnifying impact once recovered) |
| **Status** | Unfixed; replace `!==` with `crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))` |

### 2.5 PATTERN-CONFIRMED — Dual-use / placeholder secret architecture (Synth Sec S-001/S-002/S-003/S-004)

| | |
|---|---|
| **Synthesized finding** | Sec S-001 (`NEXTAUTH_SECRET` reused as Jackson DB encryption key); Sec S-002 (same secret as SAML `clientSecret` and Jackson OAuth `client_secret`); Sec S-003 (placeholder values in `.env.example` are realistic enough to ship); Sec S-004 (single string unlocks four boundaries) |
| **Public disclosure** | None specific — generic dual-purpose-secret anti-pattern. The placeholder value `my-superstrong-secret` is published in the public repo. |
| **Exploitation technique** | Read `my-superstrong-secret` from `.env.example` (public repo). Derive the same `encryptionKey` (`sha256` truncation in `lib/jackson.ts:16-20`). Decrypt every SAML connection row, SCIM directory row, and OAuth token at rest in Jackson-owned tables. Use the same string as `clientSecretVerifier` to forge valid `code` parameters on `/api/auth/saml/token` and `/api/auth/saml/authorize`, logging in as any user whose email is known. |
| **Confidence** | **PATTERN-CONFIRMED + PUBLISHED-VALUE** — the secret is in the public repo; the architectural defect subsumes S-001 through S-004 |
| **Bounty potential** | **CRITICAL** (single published placeholder unlocks NextAuth JWT signer + Jackson DB encryption + Jackson OAuth client_secret verification + SAML NextAuth provider clientSecret. Trivial exploit chain. The published placeholder is the canonical "deployment that forgot to rotate" vulnerability class.) |
| **Status** | Unfixed; introduce dedicated `JACKSON_ENCRYPTION_KEY`, `JACKSON_CLIENT_SECRET_VERIFIER`, `SAML_CLIENT_SECRET`; fail-fast startup guard |

### 2.6 PATTERN-CONFIRMED — REVALIDATE_TOKEN in URL (Synth Sec S-005)

| | |
|---|---|
| **Synthesized finding** | Sec S-005 (REVALIDATE_TOKEN passed as URL query parameter in 9 files) |
| **Public disclosure** | None specific — generic secret-in-URL leak vector (server logs, Referer headers, CDN logs, browser history). |
| **Confidence** | **PATTERN-CONFIRMED** |
| **Bounty potential** | **LOW** (low-impact target — DoS / cache stampede amplifier — but the anti-pattern matters because the same code shape is used elsewhere and could trivially be extended to higher-impact tokens like `INTERNAL_API_KEY`) |
| **Status** | Unfixed; move to `x-revalidate-secret` HTTP header |

### 2.7 PATTERN-CONFIRMED — Shared INTERNAL_API_KEY across 26 routes (Synth Sec S-006)

| | |
|---|---|
| **Synthesized finding** | Sec S-006 (single bearer token across 26 call sites with no per-job scoping or rotation mechanism) |
| **Public disclosure** | None specific — generic shared-credential anti-pattern. |
| **Confidence** | **PATTERN-CONFIRMED** |
| **Bounty potential** | **MEDIUM** (any one consumer's token leak compromises the rest; combined with F6 timing-attack recovery, single-secret blast radius is large) |
| **Status** | Unfixed; per-job scoped tokens (`INTERNAL_API_KEY_PDF`, `_NOTIFICATIONS`, `_PRESIGN`) + documented rotation runbook |

### 2.8 PATTERN-CONFIRMED — Missing auth on `/api/progress-token` and `/api/document-processing-status` (Synth Git S-001 / S-002)

| | |
|---|---|
| **Synthesized finding** | Git S-001 (Trigger.dev public access token minting without auth at `pages/api/progress-token.ts:13-23`); Git S-002 (document-processing-status recon oracle at `pages/api/teams/[teamId]/documents/document-processing-status.ts:9-22`) |
| **Public disclosure** | **NONE** — these endpoints appear to have been missed by the recent auth sweep at commit `9db42913` (which closed the `/api/views` link↔document IDOR but did not extend to adjacent endpoints accepting `documentVersionId` query parameter) |
| **Exploitation technique** | E-S-001 — Anyone who can guess or enumerate a `documentVersionId` (cuid, exposed in dashboards, share URLs, error messages, analytics feeds — see Dep S-016 for cuid predictability note) receives a Trigger.dev public access token scoped to that version. The recon oracle (S-002) tells an attacker 200 vs 404 which `documentVersionId` values are valid; `hasPages` and `currentPageCount` reveal conversion state. Polling exposure: wired into `useDocumentProcessingStatus` (`lib/swr/use-document.ts:200-218`) which calls every 3 seconds. |
| **Confidence** | **UNCONFIRMED (novel)** — no prior public report |
| **Bounty potential** | **HIGH** — newly discovered by this whitebox pass; missed by papermark's own recent auth sweep; the same root cause class as the unfixed folder IDOR (F1/F2) but on a different blast-radius surface (Trigger.dev token issuance). Novel finding, no dup risk. |
| **Status** | Unfixed; natural follow-up to commit `9db42913` — apply the same `getServerSession` + team-membership + document-ownership pattern |

### 2.9 PATTERN-CONFIRMED — DOM-XSS surface via `dangerouslySetInnerHTML` on user-controlled data (Synth SAST S-002/S-003)

| | |
|---|---|
| **Synthesized finding** | SAST S-002 (`prismaDocument.name`); SAST S-003 (`domainJson.apexName`) |
| **Exploitation technique** | See E7 in §1.14 — chained with Next.js CVEs and Report-Only CSP with `'unsafe-inline'` / `'unsafe-eval'` |
| **Confidence** | **PATTERN-CONFIRMED** — Pervasive XSS sink class; the CSP provides no runtime mitigation (Report-Only + unsafe-inline). |
| **Bounty potential** | **HIGH** (chains with §1.14) |
| **Status** | Latent today; replace `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` with `{prismaDocument.name}` |

### 2.10 PATTERN-CONFIRMED — XXE-adjacent Python DOCX sanitizer (Synth SAST S-004)

| | |
|---|---|
| **Synthesized finding** | SAST S-004 (`xml.etree.ElementTree.parse` at `ee/features/conversions/python/docx-sanitizer.py:146`) |
| **Public disclosure** | None specific — generic CWE-611 XXE class. CPython 3.x default config is safe today. |
| **Confidence** | **PATTERN-CONFIRMED + LATENT** |
| **Bounty potential** | **LOW** (today not exploitable; risk grows if maintainer switches to `lxml.etree.parse` or enables `resolve_entities`) |
| **Status** | Unfixed; switch to `defusedxml.ElementTree` (drop-in, hardened) |

### 2.11 PATTERN-CONFIRMED — `$executeRawUnsafe` savepoint name (Synth SAST S-005)

| | |
|---|---|
| **Synthesized finding** | SAST S-005 (`await tx.$executeRawUnsafe` at `lib/folders/bulk-create.ts:51-60`) |
| **Public disclosure** | None specific — generic SQL injection class. |
| **Confidence** | **PATTERN-CONFIRMED + DEAD_CODE** (today no user input flows into `savepointName`; latent risk if future caller passes request-derived string) |
| **Bounty potential** | **LOW** |
| **Status** | Unfixed; replace with `Prisma.sql\`SAVEPOINT ${Prisma.raw(savepointName)}\`` + regex guard |

---

## 3. Vendor-Trust-Boundary Findings (third-party trust boundary)

### 3.1 VENDOR-TRUST-BOUNDARY — `@boxyhq/saml-jackson@26.2.0` npm-audit flag

| | |
|---|---|
| **Synthesized finding** | Dep S-009 (npm-audit flags HIGH severity for range `>=26.2.0` with `fixAvailable = "1.52.2"`) |
| **Public disclosure** | 0 published GitHub Security Advisories for the npm package `@boxyhq/saml-jackson` (verified via `gh api graphql`). No CVE/GHSA matches the flagged range. |
| **Reachable?** | YES — directly imported at `lib/jackson.ts:7`, processes SAML assertions on `app/(ee)/api/auth/saml/authorize/route.ts`. |
| **Investigation** | The underlying CVE may be inherited via `@boxyhq/metrics` transitive deps (the @grpc/grpc-js CVE GHSA-5375-pq7m-f5r2 is one such). The major-version downgrade to 1.52.2 is not viable. |
| **Confidence** | **VENDOR-TRUST-BOUNDARY + UNCONFIRMED** — npm-audit flags range but no public CVE matches; investigation required |
| **Bounty potential** | **MEDIUM** (SAML is internet-facing for enterprise tenants; the npm-audit flag may be a transitive CVE that gets fixed by bumping a sub-dep) |
| **Status** | Investigate which specific transitive CVE npm-audit attributes; if it's `@grpc/grpc-js`, the fix is to bump `@boxyhq/metrics` |

### 3.2 VENDOR-TRUST-BOUNDARY — SAML XML parsing (Synth SAST S-006)

| | |
|---|---|
| **Synthesized finding** | SAST S-006 (SAML XML parsing delegated to `@boxyhq/saml-jackson` at `app/(ee)/api/auth/saml/authorize/route.ts:7-77`) |
| **Public disclosure** | BoxyHQ saml-jackson: 0 published GHSAs. |
| **Confidence** | **VENDOR-TRUST-BOUNDARY + INFO** |
| **Bounty potential** | **INFO** — papermark does not perform its own XML parsing or hardening |
| **Status** | Track via Dependabot/Renovate; add an integration test that submits a SAMLResponse with `<!DOCTYPE … SYSTEM "…">` and asserts rejection |

---

## 4. Unconfirmed (Novel) Findings — High Bounty Potential

### 4.1 NOVEL — Missing auth on `/api/progress-token` (Synth Git S-001)

See §2.8 — no prior public report; this is the most actionable novel finding.

### 4.2 NOVEL — Missing auth on `/api/teams/[teamId]/documents/document-processing-status` (Synth Git S-002)

See §2.8 — no prior public report; recon oracle for the S-001 chain.

### 4.3 NOVEL — `allowDangerousEmailAccountLinking: true` (Synth Structural F4)

See §2.1 — NextAuth documented footgun, no papermark-specific report. Highest-impact unfixed ATO vector in this codebase.

### 4.4 NOVEL — QStash env-flag bypass on non-Vercel deployments (Synth Structural F5)

See §2.3 — generic env-flag bypass pattern; papermark-specific exploit chain (mass email-bombing + domain deletion) is novel.

### 4.5 NOVEL — INTERNAL_API_KEY timing-unsafe across 4 routes (Synth Structural F6)

See §2.4 — combined with Sec S-006 (shared key across 26 routes), the single-secret blast radius after timing-attack recovery is novel.

---

## 5. Confidence Tier Summary

| Tier | Count | Notes |
|---|---|---|
| **WEB-CONFIRMED + ACTIVE CVE** | 10 | form-data, undici, tmp, @grpc/grpc-js, protobufjs, systeminformation, nodemailer (dependent), hono (dependent), sanitize-html (latent), Next.js CVE bundle |
| **WEB-CONFIRMED + INDEPENDENT DISCLOSURE** | 2 | Folder IDOR (#2078), CORS misconfig (#2178) |
| **WEB-CONFIRMED + NOT-REACHABLE** | 4 | xlsx (forward-risk only), next-auth (past patches), dompurify (no IN_PLACE usage), Next.js CVE-2025-29927 (past patch) |
| **PATTERN-CONFIRMED** | 11 | allowDangerousEmailAccountLinking, AES-CTR, QStash bypass, timing-unsafe, dual-use secret, secret-in-URL, shared INTERNAL_API_KEY, missing auth (×2), DOM-XSS, XXE, $executeRawUnsafe |
| **VENDOR-TRUST-BOUNDARY** | 2 | saml-jackson npm-audit flag, SAML XML parsing |
| **UNCONFIRMED (novel)** | 5 | All Pattern-Confirmed items where papermark-specific exploit chain is novel |

---

## 6. Final Ranking — Bounty Potential × Web Confidence

| Rank | Finding | Synthesis ID | Confidence | Bounty | Notes |
|---|---|---|---|---|---|
| 1 | **Dual-use `NEXTAUTH_SECRET` published as `my-superstrong-secret`** | Sec S-001/S-002/S-003/S-004 | PATTERN-CONFIRMED + PUBLISHED-VALUE | **CRITICAL** | Single published string unlocks NextAuth JWT + Jackson DB encryption + Jackson OAuth client_secret verification + SAML clientSecret |
| 2 | **`allowDangerousEmailAccountLinking: true`** | Structural F4 | PATTERN-CONFIRMED | **HIGH** | Highest-impact unfixed ATO vector; documented NextAuth footgun |
| 3 | **Missing auth on `/api/progress-token`** | Git S-001 | UNCONFIRMED (novel) | **HIGH** | Missed by recent auth sweep; same root cause class as unfixed folder IDOR |
| 4 | **CORS misconfig on `tus-viewer` (independently reported #2178)** | Structural F3 + Dep S-008 | WEB-CONFIRMED (independent disclosure) | **HIGH** | Independently reported 2 days before this pass; same root cause as hono CVE |
| 5 | **Folder DELETE/RENAME IDOR (independently reported #2078)** | Structural F1/F2 + Git S-001/S-002 | WEB-CONFIRMED (independent disclosure) | **HIGH** | Open 4+ months; no fix; cross-tenant destructive integrity loss |
| 6 | **DOM-XSS chain with Next.js CVEs** | SAST S-002/S-003 + Next.js CVE-2026-44581/44580 | WEB-CONFIRMED + PATTERN-CONFIRMED | **HIGH** | CSP Report-Only + `'unsafe-inline'` + sink-at-render-layer |
| 7 | **`undici@6.25.0` family of 5 GHSAs** | Dep S-014 | WEB-CONFIRMED + ACTIVE CVE | **HIGH** | Directly imported via @vercel/blob; production HTTP path |
| 8 | **`systeminformation@5.23.8` Linux NM command injection** | Dep S-004 | WEB-CONFIRMED + ACTIVE CVE | **HIGH** if host-metrics enabled | RCE via attacker-controlled NM profile name |
| 9 | **`nodemailer@7.0.13` `raw:` option bypass** | Dep S-003 | WEB-CONFIRMED + DEPENDENT | **HIGH** if `raw:` is used | Full file read or AWS metadata exfiltration via email |
| 10 | **Next.js CVE-2026-44578 (SSRF) + CVE-2026-44577 (image OOM) — self-hosted exposure** | Dep S-007 | WEB-CONFIRMED + DEPLOYMENT-DEPENDENT | **HIGH** for self-hosted | Per Issue #2160, self-hosted enterprise users ARE exposed |
| 11 | **QStash env-flag bypass** | Structural F5 | PATTERN-CONFIRMED | **HIGH** for self-hosted | Mass email-bombing + domain deletion |
| 12 | **Missing auth on `/api/document-processing-status`** | Git S-002 | UNCONFIRMED (novel) | **HIGH** | Recon oracle; missed by recent auth sweep |
| 13 | **`hono@4.12.18` CORS reflection** | Dep S-008 | WEB-CONFIRMED + DEPENDENT | **HIGH** if MCP is wired | Same root cause as tus-viewer; transitive via MCP SDK |
| 14 | **`form-data@4.0.5` CRLF injection** | Dep S-005 | WEB-CONFIRMED + ACTIVE CVE | **MEDIUM** | Defense-in-depth; narrow reachability |
| 15 | **`@grpc/grpc-js@1.14.3` malformed request DoS** | Dep S-006 | WEB-CONFIRMED + ACTIVE CVE | **MEDIUM** | DoS only; narrow reachability via SAML |
| 16 | **`protobufjs@7.5.8/8.0.3` DoS family** | Dep S-013 | WEB-CONFIRMED + ACTIVE CVE | **MEDIUM** | DoS only; narrow reachability via AWS SDK |
| 17 | **AES-256-CTR without auth tag on document passwords** | SAST S-001 | PATTERN-CONFIRMED | **MEDIUM** | Threat model requires DB write access; comparison primitive exists in same codebase |
| 18 | **`sanitize-html@2.17.3` latent via `^2.17.3` declaration** | Sec S-003 | WEB-CONFIRMED + LATENT | **MEDIUM** | Lockfile pins 2.17.4 (patched); package.json allows clean install to 2.17.3 (vulnerable) |
| 19 | **Shared `INTERNAL_API_KEY` across 26 routes** | Sec S-006 | PATTERN-CONFIRMED | **MEDIUM** | Any one consumer's leak compromises the rest; combined with F6 timing-attack recovery |
| 20 | **`xlsx@0.20.3` forward-risk** | Dep S-002 | WEB-CONFIRMED + FORWARD-RISK | **LOW** | No active CVE; SheetJS distribution via CDN tarball; future EOL concern |
| 21 | **REVALIDATE_TOKEN in URL** | Sec S-005 | PATTERN-CONFIRMED | **LOW** | Low-impact target; anti-pattern matters as a code-shape precedent |
| 22 | **INTERNAL_API_KEY timing-unsafe comparison** | Structural F6 | PATTERN-CONFIRMED | **LOW-MEDIUM** | Requires network-latency measurement; combined with shared-key blast radius |
| 23 | **`tmp@0.2.5` path traversal** | Dep S-010 | WEB-CONFIRMED + ACTIVE CVE | **LOW** | Write-only call site; easy fix |
| 24 | **XXE-adjacent Python DOCX sanitizer** | SAST S-004 | PATTERN-CONFIRMED + LATENT | **LOW** | CPython 3.x default safe today |
| 25 | **`$executeRawUnsafe` savepoint name** | SAST S-005 | PATTERN-CONFIRMED + DEAD_CODE | **LOW** | Today no user input flows into the parameter |
| 26 | **`@boxyhq/saml-jackson@26.2.0` npm-audit flag** | Dep S-009 | VENDOR-TRUST-BOUNDARY | **MEDIUM** | Investigate transitive CVE attribution |

---

## 7. Key Bounty-Reporting Insights

### 7.1 Novel vs Dup Risk

- **#2078 (Folder IDOR)** and **#2178 (CORS)** are public duplicates of structural F1/F2 and F3. Reporting them now risks being marked a "dupe" by the papermark team. Recommend focusing on the novel findings instead.
- **Git S-001/S-002** (missing auth on `/api/progress-token` and `/api/document-processing-status`) are **NEWLY DISCOVERED by this whitebox pass** and missed by the recent auth sweep at commit `9db42913`. **No prior report** — novel and high-value.
- **Structural F4** (allowDangerousEmailAccountLinking) is a **documented NextAuth footgun** with no papermark-specific prior report. The papermark-specific exploit chain (Google Workspace + no `signIn` callback override) is novel.
- **Sec S-001/S-002/S-003/S-004** (dual-use `NEXTAUTH_SECRET` published as `my-superstrong-secret`) is a **published-placeholder exploit chain** — the placeholder is in the public repo and the architectural defect (single string for four boundaries) makes the chain trivial.

### 7.2 Where to Focus the Bounty Report

1. **Lead with the dual-use secret chain** (Sec S-001/S-002/S-003/S-004) — single published placeholder unlocks four boundaries. Highest-impact, fully self-contained PoC, and the published value is in the public repo.
2. **Second: the OAuth email account linking ATO** (Structural F4) — highest-impact unfixed ATO vector.
3. **Third: the newly discovered missing-auth endpoints** (Git S-001/S-002) — missed by recent auth sweep; novel and no dup risk.
4. **Fourth: the self-hosted-relevant findings** (CVE-2026-44578 SSRF, CVE-2026-44577 image OOM, QStash bypass) — Issue #2160 confirms self-hosted enterprise users are a target.

### 7.3 Findings to NOT Report (Dup Risk)

- Folder DELETE/RENAME IDOR (covered by #2078)
- CORS misconfig on `tus-viewer` (covered by #2178)
- xlsx 0.20.3 (no active CVE)
- next-auth 4.24.14 (past all known patches)
- dompurify (no IN_PLACE usage)
- Node-tar symlink traversal (dev-install only)
- Patched transitive deps

---

## PHASE_3_CHECKPOINT

- [x] All synthesized findings cross-referenced with web results
- [x] Each finding marked as WEB-CONFIRMED / PATTERN-CONFIRMED / VENDOR-TRUST-BOUNDARY / UNCONFIRMED (novel)
- [x] Exploitation techniques added from web research (CVE writeups, advisory PoCs, GitHub issue reports)
- [x] Findings ranked by web-confirmed confidence × bounty potential (26 entries in final ranking)
- [x] Novel vs dup risk assessed for each finding (recommend novel-first reporting)
- [x] Bounty-reporting priorities ordered by impact × novelty

---
