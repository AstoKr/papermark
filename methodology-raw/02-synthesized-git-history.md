# Synthesized Git History — Merge of AI Recon, Tool Findings, and Structural Analysis

**Date:** 2026-06-30  
**Inputs:** `00-ai-git-history.md` (12 VFCs), `01-structural-analysis.md` (10 findings), `tools-raw/normalized.json` (226 findings), `tools-raw/git-log-security.txt` (23 recent commits)

---

## Merge Summary

| Source | Feed | Count | Key Categories |
|--------|------|-------|----------------|
| AI Git History | VFC analysis | 12 | Auth bypass, SSRF, OTP, open redirect, email hijack, etc. |
| Structural Analysis | Live code audit | 10 | Conversations zero-auth, XSS, CORS, unauthed endpoints, framework CVEs |
| Tools (gitleaks) | Secrets scan | 29 | Generic API key hashes in `.gitnexus/meta.json` — **all FALSE POSITIVES** |
| Tools (semgrep) | SAST | 25 | CORS misconfig, XSS sinks, XXE, GCM tag, ReDoS, format strings |
| Tools (osv-scanner) | Dep vulns | 113 | tornado, cryptography, Next.js, GitPython, idna, urllib3 |
| Tools (npm-audit) | Dep vulns | 59 | next, hono, nodemailer, undici, dompurify, protobufjs |

---

## Dedup & Cross-Reference

| Topic | AI VFC | Structural | Tools | Merge Result |
|-------|--------|-----------|-------|-------------|
| Auth bypass /api/views | ✅ Complete fix | — | — | Historical (fixed) |
| Webhook SSRF/logging | ✅ Complete fix | — | — | Historical (fixed) |
| Download endpoint auth | ✅ Complete fix | — | — | Historical (fixed) |
| Email change cross-user | ✅ Complete fix | — | — | Historical (fixed) |
| Open redirect middleware | ✅ Complete fix | — | — | Historical (fixed) |
| SSRF DNS-level protection | ✅ Complete fix | — | — | Historical (fixed) |
| OTP per-email throttling | ✅ Complete fix | — | — | Historical (fixed) |
| Dataroom notification ACL | ✅ Complete fix | — | — | Historical (fixed) |
| Document page team check | ✅ Complete fix | — | — | Historical (fixed) |
| Download fix loop | ✅ Complete fix | — | — | Historical (fixed) |
| MCP endpoint auth | ✅ Complete fix | — | — | Historical (fixed) |
| Screenshot protection | ✅ Complete fix | — | — | Historical (fixed) |
| **Conversations zero auth** | — | 📌 Finding 1 | — | **Active — CRITICAL** |
| **Stored XSS** | — | 📌 Finding 2 | semgrep: dangerouslySetInnerHTML | **Active — HIGH** |
| **CORS TUS viewer** | — | 📌 Finding 3 | semgrep: CORS misconfig | **Active — MEDIUM** |
| **progress-token unauthed** | — | 📌 Finding 4 | — | **Active — HIGH** |
| **record_reaction unauthed** | — | 📌 Finding 5 | — | **Active — MEDIUM** |
| **feedback unauthed** | — | 📌 Finding 6 | — | **Active — MEDIUM** |
| **revalidate secret query** | — | 📌 Finding 7 | — | **Active — MEDIUM** |
| **Next.js v14 DoS** | — | 📌 Finding 8 | osv+audit: Next.js vulns | **Active — HIGH** |
| **helpText XSS sink** | — | 📌 Finding 9 | semgrep: dangerouslySetInnerHTML | **Active — MEDIUM** |
| **XXE docx-sanitizer** | — | — | semgrep: XXE parse | **Active — LOW** |
| **ReDoS workflow engine** | — | — | semgrep: non-literal RegExp | **Active — LOW** |
| **GCM tag check (FP)** | — | — | semgrep: GCM no tag | **FALSE POSITIVE** |
| **Gitleaks meta.json (FP)** | — | — | gitleaks: 29 generic API keys | **FALSE POSITIVE** |
| **Dependency vulns** | — | — | osv+audit: 172 combined | Consolidated below |

---

## Surviving Findings

### S-001 — Conversations API — Zero Authentication on All Handlers

- **severity:** CRITICAL
- **source:** AI-missed / Tool-missed (structural analysis)
- **file:** `ee/features/conversations/api/conversations-route.ts` (lines 19, 77, 202, 258) + `pages/api/conversations/[[...conversations]].ts` (lines 4-9)
- **description:** All four route handlers (GET list, POST create, POST messages, POST notifications) accept `viewerId` from the client with zero server-side verification. No `getServerSession()`, no token check, no API key validation. Adjacent `team-conversations-route.ts` demonstrates the correct pattern with `getServerSession` on every handler. An attacker can enumerate conversations, inject messages, toggle notifications, and trigger Trigger.dev email notifications to team members.
- **evidence:** No `next-auth` imports in conversations-route.ts; all four handlers read `viewerId` from `req.query` / `req.body` without validation.
- **confidence:** HIGH (confirmed by direct source read)
- **exploitability:** HIGH — unauthenticated network access, no guessing required for `dataroomId`/`viewerId` if leaked via client-side

### S-002 — Stored XSS via `sanitizePlainText` Entity-Decode Ordering + CVE-2026-44990

- **severity:** HIGH
- **source:** AI-missed / Tool-partial (semgrep found `dangerouslySetInnerHTML` sinks)
- **files:** `lib/utils/sanitize-html.ts:12-19` (bug), `components/documents/document-header.tsx:604` (sink), `pages/api/teams/[teamId]/documents/[id]/update-name.ts:15` (entry), `lib/zod/url-validation.ts:207` (entry)
- **description:** `sanitizePlainText` calls `sanitizeHtml` first (stripping literal tags), then `decodeHTML` (decoding entities). Encoded payloads like `&lt;img src=x onerror=alert(1)&gt;` survive tag stripping then become real tags after decode. Amplified by CVE-2026-44990 (`<xmp>` bypass in sanitize-html 2.17.3). Rendered via `dangerouslySetInnerHTML` in `document-header.tsx`.
- **evidence:** `decodeHTML(sanitized)` runs after `sanitizeHtml(content, {allowedTags: []})` — ordering bug confirmed by source read.
- **confidence:** HIGH (confirmed by source read)
- **exploitability:** HIGH — any team member who can create/rename documents can execute JS in every team member's browser

### S-003 — Next.js v14 Unpatchable Denial of Service (CVE-2026-23864 / CVE-2026-23869)

- **severity:** HIGH
- **source:** AI-missed / Tool-confirmed (osv-scanner + npm-audit)
- **file:** Next.js 14.2.35 (EOL, no backport)
- **description:** Memory/CPU exhaustion via crafted RSC Flight protocol requests. No patch available — fixes only in v15.5.15+/v16.2.3+. Any App Router page is a potential target. `middleware.ts` excludes `/api/*` but App Router pages in `app/` are exposed.
- **evidence:** package.json → next@14.2.35; Next.js v14 EOL October 2025.
- **confidence:** HIGH (confirmed CVE + version match)
- **exploitability:** MEDIUM — unauthenticated, but requires crafting RSC payloads; repeat requests sustain DoS

### S-004 — `progress-token.ts` — Unauthenticated Trigger.dev Access Token Generation

- **severity:** HIGH
- **source:** AI-missed / Tool-missed
- **file:** `pages/api/progress-token.ts:4-27`
- **description:** GET `/api/progress-token?documentVersionId=xxx` returns a Trigger.dev public access token with zero authentication. Grants 15-minute read access to Trigger.dev runs tagged with `version:{documentVersionId}`. The same `generateTriggerPublicAccessToken` function is used from authenticated ee routes only.
- **evidence:** Full file read — no session check, no token validation, no rate limiting.
- **confidence:** HIGH (confirmed by source read)
- **exploitability:** MEDIUM — requires knowing a valid `documentVersionId` (CUID, not trivially guessable but leakable via client-side/logs)

### S-005 — CORS Origin Reflection on TUS Viewer Upload Endpoint

- **severity:** MEDIUM
- **source:** AI-missed / Tool-confirmed (semgrep: CORS misconfiguration)
- **file:** `pages/api/file/tus-viewer/[[...file]].ts:237`
- **description:** `Access-Control-Allow-Origin` set to `req.headers.origin || "*"` with `Access-Control-Allow-Credentials: true`. Any website can make authenticated cross-origin TUS upload requests on behalf of a logged-in user. Dataroom session validation is present but the CORS config undermines it.
- **evidence:** `res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*")` at line 237.
- **confidence:** HIGH (confirmed by source read + semgrep)
- **exploitability:** MEDIUM — requires victim to have an active session; attacker page can issue TUS uploads to victim's dataroom

### S-006 — `record_reaction.ts` — Unauthenticated Database Write + Viewer Data Leak

- **severity:** MEDIUM
- **source:** AI-missed / Tool-missed
- **file:** `pages/api/record_reaction.ts:17-42`
- **description:** POST endpoint creates `Reaction` records with no auth. Accepts `viewId` from body, no caller verification. Response includes full view metadata (`viewerEmail`, `documentId`, `dataroomId`, `linkId`, `teamId`). Unlike `record_view.ts`/`record_click.ts` (which go to Tinybird), this writes directly to PostgreSQL.
- **evidence:** Full file read — no getServerSession, no API key, no session check.
- **confidence:** HIGH (confirmed by source read)
- **exploitability:** HIGH — unauthenticated network access; viewer email enumeration; arbitrary DB writes

### S-007 — `feedback/index.ts` — Unauthenticated Feedback Response Recording

- **severity:** MEDIUM
- **source:** AI-missed / Tool-missed
- **file:** `pages/api/feedback/index.ts:9-56`
- **description:** POST endpoint records feedback responses with no auth. Validates `feedbackId` and `viewId`+`linkId` pair exist but any unauthenticated client with valid IDs can inject arbitrary feedback data. `answer` field stored unsanitized.
- **evidence:** Full file read — no authentication calls.
- **confidence:** HIGH (confirmed by source read)
- **exploitability:** MEDIUM — requires knowing a valid `feedbackId`+`viewId` pair

### S-008 — `revalidate.ts` — Shared Secret in URL Query Parameter

- **severity:** MEDIUM
- **source:** AI-missed / Tool-missed
- **file:** `pages/api/revalidate.ts:13-14`
- **description:** Revalidation endpoint authenticates via `?secret=xxx` query param. Exposes `REVALIDATE_TOKEN` to server access logs, Referer headers, and browser history. If leaked, attacker can force mass ISR revalidation of all team links (DoS via cache stampede) and enumerate links for any team.
- **evidence:** `req.query.secret !== process.env.REVALIDATE_TOKEN` comparison at line 14; bulk revalidation at lines 48-100.
- **confidence:** HIGH (confirmed by source read)
- **exploitability:** MEDIUM — requires token leakage; once leaked, trivial to exploit

### S-009 — `dangerouslySetInnerHTML` on `helpText` Props — Secondary XSS Sinks

- **severity:** MEDIUM
- **source:** AI-missed / Tool-confirmed (semgrep: dangerouslySetInnerHTML)
- **files:** `components/ui/form.tsx:145`, `components/account/upload-avatar.tsx:96`
- **description:** `Form` and `UploadAvatar` components render `helpText` via `dangerouslySetInnerHTML`. Currently used with mostly static content, but if any caller passes user-controlled data through `helpText`, stored XSS is possible. Broadens attack surface beyond `document-header.tsx`.
- **evidence:** `dangerouslySetInnerHTML={{ __html: helpText || "" }}` in both files.
- **confidence:** MEDIUM (potential, not confirmed active)
- **exploitability:** LOW — requires a caller to pass unsanitized user data through the `helpText` prop

### S-010 — `webhook-events.tsx` — `dangerouslySetInnerHTML` on Highlighted Code

- **severity:** LOW
- **source:** Tool-only (semgrep)
- **file:** `components/webhooks/webhook-events.tsx:97`
- **description:** Renders webhook event bodies via `dangerouslySetInnerHTML` with syntax-highlighted HTML. While `shiki` escapes HTML entities, the `dangerouslySetInnerHTML` wrapper means any shiki bypass or rendering edge case could enable XSS. Webhook event body content can include viewer-submitted data.
- **evidence:** `dangerouslySetInnerHTML={{ __html: highlightedCode }}` at line 97; `highlightedCode` from `shiki.codeToHtml(code)`;
- **confidence:** LOW (shiki-generated HTML is escaped; risk depends on shiki version)
- **exploitability:** LOW — requires bypass of shiki HTML escaping

### S-011 — XML XXE in `docx-sanitizer.py`

- **severity:** LOW
- **source:** Tool-only (semgrep)
- **file:** `ee/features/conversions/python/docx-sanitizer.py:146`
- **description:** Uses `xml.etree.ElementTree.parse()` to parse XML from user-uploaded DOCX files. Python's ET does not resolve external entities by default in modern versions, but defense-in-depth best practice is `defusedxml`. Called from `convert-files.ts` pipeline after Gotenberg conversion failure.
- **evidence:** `ET.parse(path)` at line 146; `import xml.etree.ElementTree as ET` at line 35.
- **confidence:** LOW (Python's ET is not vulnerable by default; defense-in-depth issue)
- **exploitability:** LOW — Python xml.etree.ElementTree does not resolve external entities by default

### S-012 — Non-Literal RegExp in Workflow Engine (ReDoS Risk)

- **severity:** LOW
- **source:** Tool-only (semgrep)
- **file:** `ee/features/workflows/lib/engine.ts:307`
- **description:** `new RegExp(condition.value as string)` where `condition.value` comes from team-admin-configured workflow conditions. Admin could inject a pathological regex causing ReDoS when processing viewer emails. Error handling catches exceptions but performance impact occurs before the throw.
- **evidence:** `return new RegExp(condition.value as string).test(normalizedEmail)` at line 307.
- **confidence:** MEDIUM (authenticated attacker only; impact bounded by workflow scope)
- **exploitability:** LOW — requires authenticated team admin access; ReDoS bounded by per-request timeout

---

## Consolidated Dependency Vulnerabilities (OSV-Scanner + npm-audit)

Major dependency groups with actionable risk (above baseline supply-chain noise):

### Dep-001 — Tornado (Python, 14 CVEs)
- **package:** tornado@6.0.4
- **risk:** HTTP smuggling, open redirect, CRLF injection, cookie attribute injection, credential leakage across cross-origin redirects, gzip bomb, multipart DoS
- **exploitability:** LOW-MEDIUM — uses in docx conversion context; limited exposure

### Dep-002 — Next.js (npm, 14 CVEs)
- **package:** next@14.2.35
- **risk:** Multiple XSS, DoS, cache poisoning, request smuggling, SSRF — see S-003 for the unpatchable DoS CVEs
- **exploitability:** HIGH — framework-level, affects all App Router pages

### Dep-003 — Nodemailer (npm, 6 CVEs)
- **package:** nodemailer@7.0.13
- **risk:** SMTP command injection (CRLF in EHLO/HELO, envelope.size, List-* headers), SSRF via raw option, TLS cert validation bypass
- **exploitability:** MEDIUM — affects email delivery subsystem

### Dep-004 — Hono (npm, 9 CVEs)
- **package:** hono@4.12.18
- **risk:** CORS wildcard with credentials, JWT auth bypass, path traversal, cookie injection, body-limit bypass on Lambda
- **exploitability:** LOW — uses in MCP sub-application only; limited user-facing exposure

### Dep-005 — Undici (npm, 4 CVEs)
- **package:** undici@6.25.0
- **risk:** HTTP response queue poisoning, header injection, SameSite downgrade, WebSocket DoS
- **exploitability:** LOW — HTTP client library used internally

### Dep-006 — DOMpurify (npm, 8 CVEs)
- **package:** dompurify@3.4.0
- **risk:** Multiple IN_PLACE mode bypasses, hook/ATTR pollution, template bypass
- **exploitability:** LOW — used server-side in sanitization pipeline; IN_PLACE mode not used

---

## Historical VFCs — Fix Completeness Verification

All 12 VFCs from the AI git-history analysis were verified for fix completeness. None had incomplete or partial fixes. Key results:

| VFC | Fix Scope | All Instances? | Unfixed Siblings? |
|-----|-----------|---------------|-------------------|
| Auth bypass /api/views (9db4291) | Removed client `documentId`, added server-side validation | ✅ Both `route.ts` and `views-dataroom/route.ts` fixed | None found |
| Webhook SSRF/logging (ece358b) | URL redaction, plan-gate fix, allSettled | ✅ All webhook delivery paths addressed | None found |
| Download auth (7fa52b2) | Server-session viewId, removed client viewId | ✅ All 4 download endpoints addressed (1 deleted) | download/verify.ts uses optional client viewId with server validation — acceptable |
| Email change token (7fa52b2) | tokenUserId vs currentUserId check | ✅ Redis + Prisma both switched to tokenUserId | None found |
| Open redirect (7fa52b2) | Protocol-relative + origin validation | ✅ normalizeNextPath fully gated | None found |
| SSRF DNS protection (7fa52b2) | Full DNS-level + redirect chain validation | ✅ SSRF module complete | Other URL-fetching paths (Notion covers, link previews) should be audited separately |
| OTP rate limiting (d3ee5ff) | Per-email+link throttle + IP backup | ✅ All 4 OTP endpoints fixed | None found |
| Dataroom notification ACL (f0637c6) | Folder-access control check | ✅ Notification path only; appropriate scope | None found |
| Document page access (7fa52b2) | Team membership relation filter | ✅ Single helper, single fix | None found |

---

## Eliminated Findings

### FP-001 — Gitleaks: Generic API Keys in `.gitnexus/meta.json` (29 findings)

All findings are SHA-256 hashes stored in gitnexus metadata for file integrity tracking. These are not secrets — they are deterministic hashes of source files. No real credential exposure.

### FP-002 — Semgrep: GCM Authentication Tag Length (slack/utils.ts)

Code at `lib/integrations/slack/utils.ts:110` manually validates auth tag length (lines 105-107) before calling `createDecipheriv`. The semgrep rule flags the missing `authTagLength` option but the code's manual validation achieves the same effect.

---

## Exploitability Ranking

| Rank | ID | Title | Severity | Auth Required | Network | Guess Required |
|------|-----|-------|----------|---------------|---------|----------------|
| 1 | S-001 | Conversations zero auth | CRITICAL | No | Yes | Low (leakable IDs) |
| 2 | S-002 | Stored XSS | HIGH | Yes (team) | Yes | No |
| 3 | S-004 | progress-token unauthed | HIGH | No | Yes | Medium (CUID) |
| 4 | S-003 | Next.js v14 DoS | HIGH | No | Yes | No |
| 5 | S-006 | record_reaction unauthed | MEDIUM | No | Yes | Low (viewId) |
| 6 | S-005 | CORS TUS viewer | MEDIUM | No (victim session) | Yes | No |
| 7 | S-007 | feedback unauthed | MEDIUM | No | Yes | Medium |
| 8 | S-008 | revalidate secret in query | MEDIUM | Token | Yes | Token leakage |
| 9 | S-009 | helpText XSS sinks | MEDIUM | Conditional | Via parent | Via parent |
| 10 | S-012 | ReDoS workflow engine | LOW | Yes (admin) | Yes | No |
| 11 | S-010 | webhook-events XSS sink | LOW | No | Conditional | Conditional |
| 12 | S-011 | XXE docx-sanitizer | LOW | No | Via upload | File-level |

---

## Key Patterns

1. **Auth gap by omission:** The conversations route (`conversations-route.ts`) mirrors the structure of `team-conversations-route.ts` but was simply never wired with auth. No VFC exists for this — it was missed entirely.

2. **Sanitizer ordering bug:** The `sanitizeHtml → decodeHTML` ordering in `sanitizePlainText` is a logic error, not a version vulnerability. No git commit ever addressed it.

3. **Public tracking endpoints becoming data-injection vectors:** `record_reaction.ts` and `feedback/index.ts` write directly to PostgreSQL without auth, unlike `record_view.ts`/`record_click.ts` which publish to Tinybird.

4. **No overlapping coverage:** The AI git-history VFCs (all historical, all fixed) and the structural analysis findings (all active, all unaddressed) are disjoint sets. The tools only partially overlapped with structural analysis (CORS, XSS sinks) and primarily found dependency vulns + false positives.

5. **EOL framework risk:** Next.js 14.2.35 reached EOL October 2025. No backports for CVE-2026-23864/23869. Only upgrade to v15.5.15+ or v16.2.3+ resolves these.

---

## Data Sources

- `00-ai-git-history.md` — 12 VFCs from ~200 commits analyzed
- `01-structural-analysis.md` — 10 findings from GitNexus + live code audit
- `tools-raw/normalized.json` — 226 findings (gitleaks 29, semgrep 25, osv-scanner 113, npm-audit 59)
- `tools-raw/git-log-security.txt` — 23 recent commits (partial overlap with VFCs)
- Live source reads: `conversations-route.ts`, `sanitize-html.ts`, `engine.ts`, `webhook-events.tsx`, `docx-sanitizer.py`, `slack/utils.ts`, `convert-files.ts`
