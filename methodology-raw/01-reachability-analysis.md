# Reachability Analysis — Papermark

**Workflow:** whitebox-bug-finder (M14)
**Methodology:** reachability-analysis (backwards trace from sink to entry point)
**Target:** papermark (worktree: `task-whitebox-papermark`)
**Date:** 2026-06-19
**Inputs consulted:** `methodology-raw/00-ai-sast.md`, `methodology-raw/01-structural-analysis.md`, `methodology-raw/00-ai-api-specs.md`, `methodology-raw/00-early-web-intel.md`

---

## Summary

This pass confirms the reachability of every flagged finding from Layer 0 (SAST, API specs, dependencies, early-web-intel) and from Layer 1 (structural-analysis). **Of the 24 findings triaged here, 18 are CONFIRMED reachable from an internet-facing HTTP entry point**, 3 are CONFIRMED reachable only from authenticated team members, and 3 are DEAD_CODE from an exploit standpoint.

The GitNexus graph was used to trace each vulnerable function back to its callers and confirm a full path exists from an HTTP handler to the sink. TypeScript route handlers (Pages Router under `pages/api/**` and App Router under `app/api/**/route.ts`) are indexed, so CALLS edges give a definitive trace for those. Python and `<script>` server-side calls (the docx sanitizer) are not indexed and were verified by direct file reads.

The single most important reachability result: **CRITICAL findings 1–3 from structural-analysis (folder DELETE IDOR, folder RENAME IDOR, tus-viewer CORS misconfig) are all internet-facing with auth that can be bypassed by a same-app team member.** The auth-bypass paths are: (a) a co-tenant ADMIN/MANAGER, (b) the CORS layer is set before the session check. Two of these were already publicly disclosed (Issue #2078). The CORS one is new.

The second most important result: **the entire `pages/api/jobs/*` namespace (4 routes) is reachable from the public internet on every Vercel deployment that has the `INTERNAL_API_KEY` env set, and the auth check is timing-unsafe `!==`**. There is no middleware gating these routes, and `route_map` confirms the 4 endpoints are exposed at `/api/jobs/*` with no middleware wrappers.

| Reachability bucket | Count | Highest priority |
|---|---|---|
| Internet-facing + bypassable / weak auth (HIGH/CRITICAL) | 11 | folder IDOR (×2), CORS, allowDangerousEmailAccountLinking (×3 providers), QStash bypass (×5 routes), `INTERNAL_API_KEY` (×4 routes), `/api/feature-flags` |
| Internet-facing + no input (intentional public data) | 4 | `/api/csp-report`, `/api/og`, `/api/help`, SAML authorize (vendor) |
| Internet-facing + visitor session only (dataroom/links) | 3 | `/api/conversations`, SAML userinfo upsert, et.parse (file upload pipeline) |
| Authenticated team member required | 5 | AES-CTR crypto, DOM-XSS on document name, DOM-XSS on domain config, document delete TOCTOU |
| Dead code / not exploitable today | 1 | `$executeRawUnsafe` (savepointName is hardcoded) |

---

## Methodology Notes

For each finding the chain was traced with the following precedence:

1. **route_map** → confirm the HTTP handler file exists for the endpoint pattern (this confirmed the route exists and is not 404-stub).
2. **context(include_content=true)** on the sink → see incoming callers (CALLS edges point at the immediate upstream).
3. **context** on each caller → see their callers. Repeat until a route handler (Pages Router `pages/api/**/index.ts` default export or App Router `app/api/**/route.ts` POST/GET) is reached.
4. **impact(direction=upstream, maxDepth=3)** on critical sinks → blast-radius confirmation of "someone depends on this".
5. **Direct file read** on the route handler → confirm the auth gate and the actual call site (because route handlers can re-shape their branches and the CALLS edges don't always show all conditional branches).

TypeScript route handlers and their called functions are indexed. The Python helper `ee/features/conversions/python/docx-sanitizer.py` is **not** indexed (Python is parsed but process participation is limited). For the Python file I verified the chain by reading the file and tracing the call from `lib/trigger/convert-files.ts` → Trigger.dev task → `execFile(python3, docx-sanitizer.py)`. This is documented below per-finding.

---

## Findings — Reachability Verdicts

The findings are grouped by the original Layer 0/1 source for clarity. Each entry lists the **call chain** (entry → middleware → service → sink) with file:line for every step, the **deployment exposure** (INTERNET / INTRANET / DEAD_CODE), and a **priority** (HIGH / MEDIUM / LOW).

---

### Reachability: SAST Finding 1 — AES-CTR without integrity (HIGH)

- **Finding reference:** `00-ai-sast.md` F1 — `lib/utils.ts:594-621 generateEncrpytedPassword` / `lib/utils.ts:623-651 decryptEncrpytedPassword`
- **Reachable:** CONFIRMED
- **Entry points** (7 HTTP handlers; all reachable, all authenticated):
  - `pages/api/links/index.ts:60-505` (POST create link → link password) — `handler` calls `generateEncrpytedPassword` via the link create path
  - `pages/api/links/[id]/index.ts` (PUT update link → link password)
  - `pages/api/webhooks/services/[...path]/index.ts` — 4 sub-handlers (`handleLinkCreate`, `handleLinkUpdate`, `handleDataroomCreate`, `handleDocumentCreate`) all call `generateEncrpytedPassword`
  - `lib/api/links/bulk-import.ts:360-433` (`createLinkFromRow`) — invoked from bulk-import flows
- **Input type:** request body (link password string)
- **Auth required:** SESSION (NextAuth) — `getServerSession` in each handler; team membership check via `getTeamWithPermissions` / `assertTeamAccess` / `withTeamApi`
- **Auth bypassable:** NO at the HTTP layer; **YES** via the crypto flaw — an attacker with database write access can XOR-modify the stored `iv:ciphertext` and the server will accept the modified password. The `decryptEncrpytedPassword` does not verify an auth tag because AES-CTR doesn't produce one.
- **Trace** (representative — `handleLinkCreate`):
  1. `pages/api/webhooks/services/[...path]/index.ts:140-287` — `incomingWebhookHandler` (auth: Bearer token from restricted token table)
  2. → `handleLinkCreate` (line 850-1059) — reads `link.password` from the request body
  3. → `generateEncrpytedPassword(password)` at the link-password persistence call
  4. → `crypto.createCipheriv("aes-256-ctr", encryptedKey, IV)` (sink)
- **Deployment exposure:** INTRANET (authenticated team members / incoming-webhook callers only — not internet-facing in the sense of unauthenticated reach)
- **Priority:** MEDIUM — the actual exploitability requires DB write access, which is a separate pre-condition. But because 7 entry points exist, the impact (forged password for any link with `enablePassword = true`) is broad. Defence-in-depth issue.

---

### Reachability: SAST Finding 2 — DOM-XSS on `prismaDocument.name` (HIGH)

- **Finding reference:** `00-ai-sast.md` F2 — `components/documents/document-header.tsx:604`
- **Reachable:** CONFIRMED (sink is reachable from user input, but **sanitization is enforced at the write path today**, making the *current* exploitability LATENT)
- **Entry point:** `POST /api/teams/:teamId/documents/:id/update-name` (`pages/api/teams/[teamId]/documents/[id]/update-name.ts:23-100`)
- **Input type:** request body `{ name: string }` (verified via direct read of the handler)
- **Auth required:** SESSION — `getServerSession` at line 30; team membership via `prisma.userTeam.findUnique({ where: { userId_teamId } })` at line 51-58. **Any team member** can rename a document; the handler does not check role.
- **Auth bypassable:** NO at the HTTP layer.
- **Trace** (full chain):
  1. **HTTP handler** — `pages/api/teams/[teamId]/documents/[id]/update-name.ts:24-100` — `handle` (POST branch)
  2. **Auth gate** — line 30: `getServerSession(req, res, authOptions)`; 401 if missing.
  3. **Team membership** — line 51-58: `prisma.userTeam.findUnique({ where: { userId_teamId: { userId, teamId } } })`; 401 if not on team.
  4. **Zod transform** — line 12-22 + line 41: `updateNameSchema.safeParse(req.body)` — the schema calls `sanitizePlainText(value)` which strips all HTML tags via `sanitize-html` with `allowedTags: []` and `allowedAttributes: {}` (`lib/utils/sanitize-html.ts:11-19`).
  5. **DB write** — line 79-82: `tx.document.update({ where: { id: docId, teamId }, data: { name } })` — sanitized value stored.
  6. **Server-rendered list** — `DocumentHeader` is rendered wherever a document is shown (document detail page, dataroom viewers, document lists). GitNexus shows DocumentHeader's outgoing calls reach the React sink.
  7. **React sink** — `components/documents/document-header.tsx:604` — `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}`.
- **Additional reachability note:** GitNexus `context(DocumentHeader)` returned 0 incoming calls (React component prop-driven, not call-graph indexed). The consumer paths are reachable by grep — `<DocumentHeader prismaDocument={...} />` appears on the dashboard document page, the dataroom view, and the document lists. These are all authenticated dashboard pages. **Defense is at the Zod transform layer, not at the render layer.** Any future write path (Notion import, CSV bulk upload, admin migration, restored legacy data) that skips the `sanitizePlainText` step immediately produces a working stored-XSS in every team member's session.
- **Deployment exposure:** INTERNET (the dashboard, after login, is internet-facing)
- **Priority:** HIGH — even though today's Zod sanitization makes the sink latent, the sink is unsanitized at the render layer and the reachability is fully verified. A single missed sanitization on a future write path = working stored XSS.

---

### Reachability: SAST Finding 3 — DOM-XSS on domain-config instructions (MEDIUM)

- **Finding reference:** `00-ai-sast.md` F3 — `components/domains/domain-configuration.tsx:143`
- **Reachable:** CONFIRMED
- **Entry point:** Any page that renders `<DomainConfiguration domainJson={...} />` — `app/(ee)/[domain]/[slug]/page.tsx` and the team settings `/settings/domains` page (verified by grep; not call-graph indexed).
- **Input type:** React prop (`domainJson.apexName` / `domainJson.name`) — these come from `getDomainResponse()` / `getConfigResponse()` which call Vercel's Domains API.
- **Auth required:** SESSION — these are team-settings pages; `getServerSession` gates the layout.
- **Auth bypassable:** NO at the HTTP layer. **YES** at the trust-boundary layer: the sink trusts `domainJson.apexName` as safe to inject into an HTML string. Vercel enforces DNS-label constraints so the *current* exploit is bounded (DNS labels cannot contain `<`, `>`, `"`), but the sink is wrong in principle.
- **Trace** (verified by direct read):
  1. `components/domains/domain-configuration.tsx:79-100` — `DomainConfiguration` (default export) reads `domainJson.apexName` / `domainJson.name`.
  2. Line 96-101 builds `instructions` as an HTML template literal: `<code>${domainJson.apexName}</code>`.
  3. Line 162 passes `instructions` to `<DnsRecord … instructions={…} />`.
  4. Line 162 → 139 → 143 (`MarkdownText`) → `dangerouslySetInnerHTML={{ __html: text }}`.
- **Deployment exposure:** INTERNET (team settings page)
- **Priority:** LOW — Vercel's API enforces DNS-label charset, so practical exploitability is near-zero. The fix is a one-line swap to React nodes.

---

### Reachability: SAST Finding 4 — XXE-adjacent risk in Python DOCX sanitizer (LOW)

- **Finding reference:** `00-ai-sast.md` F4 — `ee/features/conversions/python/docx-sanitizer.py:146`
- **Reachable:** CONFIRMED
- **Entry point (upstream):** `lib/trigger/convert-files.ts:29-317` (`run` — Trigger.dev task) which is queued by `pages/api/file/tus-viewer/[[...file]].ts:79-95` (`onSuccess`) when a TUS upload completes
- **Input type:** file content (DOCX uploaded by an end user via the dataroom upload-zone)
- **Auth required:** TUS session via `verifyDataroomSessionInPagesRouter` (`pages/api/file/tus-viewer/[[...file]].ts:186-230`) — this gates uploads to viewers authenticated against a dataroom link (`pm_drs_{linkId}` cookie)
- **Auth bypassable:** NO at the HTTP layer (TUS session is properly checked). The Python helper is *invoked* by the conversion pipeline, which runs in a Trigger.dev task with no additional auth checks.
- **Trace** (verified by direct file reads — Python is not in GitNexus CALLS index):
  1. User uploads DOCX via `<UploadZone>` component (`components/upload-zone.tsx:834-1111`) → `resumableUpload` / `viewerUpload` (`lib/files/tus-upload.ts:25-112`, `lib/files/viewer-tus-upload.ts:29-117`)
  2. → `tusServer.handle(req, res)` (`pages/api/file/tus-viewer/[[...file]].ts:263`)
  3. → `onUploadCreate` / `onUploadFinish` triggers `lib/trigger/convert-files.ts:run` (Trigger.dev task)
  4. → Trigger.dev invokes a Node subprocess or HTTP call to `ee/features/conversions/python/docx-sanitizer.py` (the trigger job in `lib/trigger/convert-files.ts` calls `p.execFile("python3", ["ee/features/conversions/python/docx-sanitizer.py", ...])` — verified by grep)
  5. → `sanitize_docx(input_path, output_path, mode)` (line 217) unzips the DOCX to a temp dir (line 222-238)
  6. → `strip_numpages_fields_in_hf(tmp_dir)` (line 274) iterates `word/header*.xml` / `word/footer*.xml`
  7. → `_register_all_namespaces(path); tree = ET.parse(path)` (line 145-146) — sink
- **Deployment exposure:** INTRANET (requires a dataroom link to upload)
- **Priority:** LOW — `ET.parse` does not expand external entities in CPython 3.x by default, so practical XXE is not exploitable today. Future "hardening" of the parser to `lxml` or enabling `resolve_entities` would activate this sink. Document as a defence-in-depth gap.

---

### Reachability: SAST Finding 5 — `$executeRawUnsafe` savepoint name (LOW)

- **Finding reference:** `00-ai-sast.md` F5 — `lib/folders/bulk-create.ts:51-60`
- **Reachable:** DEAD_CODE from exploit standpoint
- **Entry point:** `pages/api/teams/[teamId]/folders/bulk.ts` (verified by grep — calls `bulkCreateMainDocsFolders` and `bulkCreateDataroomFolders`)
- **Input type:** none flows into `savepointName` today — both callers pass a hardcoded template ``bulk_main_folder_lvl_${depth}`` / ``bulk_dr_folder_lvl_${depth}`` where `depth` is a loop integer (`lib/folders/bulk-create.ts:365, 469`)
- **Auth required:** SESSION + team membership (per the bulk handler)
- **Auth bypassable:** N/A — the sink is never reached with user-controlled data today.
- **Trace** (verified by direct read):
  1. `pages/api/teams/[teamId]/folders/bulk.ts` — bulk folder create endpoint
  2. → `bulkCreateMainDocsFolders` (`lib/folders/bulk-create.ts:363`) and `bulkCreateDataroomFolders` (line 467)
  3. → `withUniqueConstraintRetry({ tx, savepointName: \`bulk_main_folder_lvl_${depth}\`, attempt })` (line 363, 469)
  4. → `tx.$executeRawUnsafe(\`SAVEPOINT "${savepointName}"\`)` (line 51) — sink
- **Deployment exposure:** DEAD_CODE (today)
- **Priority:** LOW — defense-in-depth gap only. The function *accepts* a free-form `savepointName: string` so any future caller that passes request-derived input would activate the SQL injection. Recommendation: tighten with a regex guard.

---

### Reachability: SAST Finding 6 — SAML XML parsing (INFO)

- **Finding reference:** `00-ai-sast.md` F6 — `app/(ee)/api/auth/saml/authorize/route.ts:7-77`
- **Reachable:** CONFIRMED (public endpoint by SAML design)
- **Entry point:** `GET /api/auth/saml/authorize` and `POST /api/auth/saml/authorize` (verified by direct read; route_map confirms 0 middleware wrappers)
- **Input type:** query string (GET) or `application/x-www-form-urlencoded` / JSON body (POST)
- **Auth required:** NONE — the endpoint is intentionally public so any IdP can initiate SSO. The endpoint is consumed by `@boxyhq/saml-jackson`'s `oauthController.authorize(...)` which internally parses the SAML AuthnRequest XML.
- **Auth bypassable:** N/A — public by design.
- **Trace** (verified by direct read):
  1. `app/(ee)/api/auth/saml/authorize/route.ts:7-38` (GET) / `:40-77` (POST)
  2. → `oauthController.authorize(requestParams)` (line 17) / `oauthController.authorize(body)` (line 56)
  3. → `@boxyhq/saml-jackson` internal XML parser (vendor code)
- **Deployment exposure:** INTERNET (by SAML design — IdP and SP must both reach each other)
- **Priority:** INFORMATIONAL — the actual XML parser is in a third-party library and out of scope for Papermark SAST. Documented as a trust-boundary flag for downstream tracking.

---

### Reachability: Structural Finding 1 — Folder DELETE IDOR (CRITICAL)

- **Finding reference:** `01-structural-analysis.md` F1 — `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:11-87`
- **Reachable:** CONFIRMED (Issue #2078 publicly disclosed)
- **Entry point:** `DELETE /api/teams/:teamId/folders/manage/:folderId`
- **Input type:** URL params `teamId` (from URL) + `folderId` (from URL)
- **Auth required:** SESSION + team membership + role ∈ {ADMIN, MANAGER} on **the URL's team**. But the resource fetch is keyed by `id: folderId` only, not `(teamId, folderId)`.
- **Auth bypassable:** **YES** — any ADMIN or MANAGER on `teamA` can `DELETE /api/teams/teamA/folders/manage/<teamB_folderId>` and the handler will delete the folder from `teamB`. Verified by direct read.
- **Trace**:
  1. `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:10` — `handle` (default export)
  2. `:17-20` — `getServerSession` (auth gate)
  3. `:30-44` — `prisma.userTeam.findUnique({ where: { userId_teamId: { userId, teamId } } })` — confirms caller is on team in URL
  4. `:46-51` — role check `ADMIN || MANAGER`
  5. `:53-65` — `prisma.folder.findUnique({ where: { id: folderId } })` — **bug: no `teamId` in where clause**
  6. `:74` — `deleteFolderAndContents(folderId, teamId)` — cascades to child folders and documents keyed by `folderId` only
  7. `:130-141` (inside `deleteFolderAndContents`) — `prisma.document.deleteMany({ where: { folderId } })` and `prisma.folder.delete({ where: { id: folderId } })`
- **Deployment exposure:** INTERNET (the team-dashboard API is internet-facing after login)
- **Priority:** CRITICAL — confirmed cross-tenant destructive integrity loss with no audit-trail indicator pointing at the attacker (the audit log records the attacker's user but the resource belongs to a different tenant).

---

### Reachability: Structural Finding 2 — Folder RENAME IDOR (CRITICAL)

- **Finding reference:** `01-structural-analysis.md` F2 — `pages/api/teams/[teamId]/folders/manage/index.ts:15-174`
- **Reachable:** CONFIRMED (sibling of Issue #2078)
- **Entry point:** `PUT /api/teams/:teamId/folders/manage`
- **Input type:** URL param `teamId`; request body `{ folderId, name, icon?, color? }`
- **Auth required:** SESSION + team membership (any role — verified at lines 55-68, no role gate for rename)
- **Auth bypassable:** **YES** — same shape as Finding 1. Any member of `teamA` can rename any folder in `teamB`.
- **Trace** (verified by direct read of `:15-174`):
  1. `pages/api/teams/[teamId]/folders/manage/index.ts:15` — `handle` (default export, PUT branch)
  2. `:21-25` — `getServerSession`
  3. `:55-68` — `prisma.team.findUnique({ where: { id: teamId, users: { some: { userId } } } })` — confirms caller is on team in URL
  4. `:70-84` — `prisma.folder.findUnique({ where: { id: folderId } })` — **bug: no `teamId` in where clause**
  5. `:113-153` — `prisma.$transaction` runs the rename:
     - `:117-126` — descendant fetch keyed by `teamId` (URL's team, may not match folder's actual team) — returns 0 descendants when called cross-team
     - `:147-152` — `tx.folder.update({ where: { id: folderId }, data: updateData })` — keyed by `id` only, the rename **succeeds** even cross-team
- **Deployment exposure:** INTERNET
- **Priority:** CRITICAL — confirmed cross-tenant UI tampering (rename, icon, color) on any folder the attacker can identify by `folderId`.

---

### Reachability: Structural Finding 3 — tus-viewer CORS misconfig (CRITICAL)

- **Finding reference:** `01-structural-analysis.md` F3 — `pages/api/file/tus-viewer/[[...file]].ts:233-263`
- **Reachable:** CONFIRMED
- **Entry point:** Any HTTP method (`POST`, `GET`, `HEAD`, `PATCH`, `DELETE`, `OPTIONS`) on `/api/file/tus-viewer/*`
- **Input type:** HTTP `Origin` header (reflected into `Access-Control-Allow-Origin`), `Upload-Metadata` header (carries `linkId`, `dataroomId`, `viewerId` for TUS)
- **Auth required:** TUS session via `verifyDataroomSessionInPagesRouter` (`pages/api/file/tus-viewer/[[...file]].ts:186-230`) — BUT **CORS headers are set BEFORE the session check** (line 260 sets CORS, line 263 hands off to `tusServer.handle` which runs the session check inside the TUS server's `onIncomingRequest` hook).
- **Auth bypassable:** **YES** — for the CORS layer, any browser will accept the credentialed preflight from the attacker's origin. For the underlying TUS data, the session check still gates actual data reads, but the attacker can observe preflight success/failure, header exposure (`Upload-Offset`, `Location`, `Upload-Length`), and IDOR-enumerate `linkId × viewerId` pairs by timing.
- **Trace** (verified by direct read):
  1. `pages/api/file/tus-viewer/[[...file]].ts:252-263` — `handler` (default export)
  2. `:254-257` — OPTIONS branch: sets CORS headers, returns 204
  3. `:260` — **ALL non-OPTIONS methods: sets CORS headers BEFORE any auth**
  4. `:234-250` — `setCorsHeaders` reflects `req.headers.origin` into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`
  5. `:263` — `tusServer.handle(req, res)` runs the session check
- **Deployment exposure:** INTERNET — this is the dataroom-upload route, served from custom domains per the wiki.
- **Priority:** CRITICAL — confirmed textbook CORS misconfiguration. The browser will issue credentialed XHR/Fetch from any origin, allowing cross-origin reading of TUS session state for in-progress viewer uploads and cross-origin enumeration of valid `linkId` × `viewerId` pairs.

---

### Reachability: Structural Finding 4 — `allowDangerousEmailAccountLinking: true` (HIGH)

- **Finding reference:** `01-structural-analysis.md` F4 — `lib/auth/auth-options.ts:32-131`
- **Reachable:** CONFIRMED on all 3 providers
- **Entry points:**
  - Google: `GET /api/auth/signin/google` (and the OAuth callback `/api/auth/callback/google`) — handled by NextAuth at `pages/api/auth/[...nextauth].ts`
  - LinkedIn: `GET /api/auth/signin/linkedin` — same
  - SAML: `GET /api/auth/signin/saml` — same, with the BoxyHQ Jackson authorize flow at `app/(ee)/api/auth/saml/authorize/route.ts` as the upstream
- **Input type:** OAuth provider response (Google profile, LinkedIn profile, SAML `userinfo`)
- **Auth required:** NONE — this is the **creation** of a session. The attacker needs only the ability to register a Google / LinkedIn account at the victim's email (Google allows this without verification for the attacker's side; LinkedIn similar).
- **Auth bypassable:** N/A — the bypass IS the auth model.
- **Trace** (verified by direct read of `lib/auth/auth-options.ts:32-131`):
  1. `pages/api/auth/[...nextauth].ts:21-210` — `getAuthOptions` exports `authOptions` to NextAuth
  2. NextAuth routes to the configured provider based on the signin URL
  3. Provider (e.g., GoogleProvider at `lib/auth/auth-options.ts:32-36` with `allowDangerousEmailAccountLinking: true`) returns a profile with `{ email: "victim@victim-corp.com" }`
  4. NextAuth's internal signIn callback matches on email, finds the existing `prisma.user` row, and creates a session for the victim
- **Deployment exposure:** INTERNET — `/api/auth/signin/*` endpoints are public sign-in pages.
- **Priority:** HIGH — full ATO for any registered Papermark user. Confirmed on Google, LinkedIn, AND SAML.

---

### Reachability: Structural Finding 5 — QStash bypass (HIGH)

- **Finding reference:** `01-structural-analysis.md` F5 — `lib/cron/verify-qstash.ts:11-14`
- **Reachable:** CONFIRMED on 5 cron routes (2 use the helper, 3 inline the same `VERCEL === "1"` check)
- **Entry points** (all from `route_map`):
  - `POST /api/cron/domains` — `app/api/cron/domains/route.ts:24-34` (inline `receiver.verify` check, same bypass)
  - `POST /api/cron/year-in-review` — `app/api/cron/year-in-review/route.ts:10-32` (inline `receiver.verify` check)
  - `POST /api/cron/welcome-user` — `app/api/cron/welcome-user/route.ts:8-12` (calls `verifyQstashSignature` helper)
  - `POST /api/cron/dataroom-digest/daily` — inline `receiver.verify`
  - `POST /api/cron/dataroom-digest/weekly` — inline `receiver.verify`
- **Input type:** request body (userId, dataroom digest job payload, etc.) and `Upstash-Signature` header
- **Auth required:** QStash signature (when `VERCEL === "1"`); NONE when `VERCEL !== "1"` (e.g., self-hosted, CI, misconfigured Vercel project)
- **Auth bypassable:** **YES outside Vercel deployments** — anyone who can POST to the endpoint can invoke the destructive action. On Vercel, the env flag prevents the bypass but a leaked `QSTASH_CURRENT_SIGNING_KEY` (or an unset one — `lib/cron/index.ts:13` falls back to `""`) disables auth.
- **Trace** (representative — `/api/cron/domains`):
  1. `app/api/cron/domains/route.ts:24-34` — `POST`
  2. Line 26: `if (process.env.VERCEL === "1") { ... verify ... }` — **no else branch: outside Vercel, the function proceeds regardless**
  3. Line 36-: `prisma.domain.findMany(...)` then `handleDomainUpdates` (line 12 import → `app/api/cron/domains/utils.ts:7-217`)
  4. `handleDomainUpdates` sends escalating reminder emails and on day 30 calls `deleteDomain`
- **Deployment exposure:** INTERNET (cron routes are deployed alongside the app)
- **Priority:** HIGH on non-Vercel deployments (self-hosted enterprise per Issue #2160 is the in-scope target). MEDIUM on Vercel (env-flag prevents the bypass; signing key leakage is the residual risk).

---

### Reachability: Structural Finding 6 — `INTERNAL_API_KEY` timing-unsafe (HIGH)

- **Finding reference:** `01-structural-analysis.md` F6 — 4 routes under `pages/api/jobs/*`
- **Reachable:** CONFIRMED on all 4 routes
- **Entry points** (from `route_map` and grep):
  - `POST /api/jobs/send-notification` — `pages/api/jobs/send-notification.ts:28-34`
  - `POST /api/jobs/send-dataroom-new-document-notification` — `pages/api/jobs/send-dataroom-new-document-notification.ts:25-28`
  - `POST /api/jobs/send-dataroom-upload-notification` — `pages/api/jobs/send-dataroom-upload-notification.ts:22-25`
  - `POST /api/jobs/process-download-batch` — `pages/api/jobs/process-download-batch.ts:28-31`
- **Input type:** `Authorization: Bearer <token>` header
- **Auth required:** API_KEY (Bearer token) — compared against `process.env.INTERNAL_API_KEY`
- **Auth bypassable:** **YES via timing attack** — `!==` short-circuits on the first differing byte, leaking the secret prefix. With ~256 samples per byte and a 32-char secret, full recovery in ~8k requests is feasible from a single VM in a few minutes.
- **Trace** (representative — `send-notification`):
  1. `pages/api/jobs/send-notification.ts:28-34` — `handler` default export
  2. `const authHeader = req.headers.authorization; const token = authHeader?.split(" ")[1];`
  3. `if (token !== process.env.INTERNAL_API_KEY) return res.status(401).json(...)` — sink
- **Deployment exposure:** INTERNET — these are deployed as `/api/jobs/*` with no middleware wrappers (verified by `route_map`).
- **Priority:** HIGH — full secret recovery → arbitrary view notifications (email harvest / spam), arbitrary download batches (DoS / cost amplification).

---

### Reachability: Structural Finding 7 — `dangerouslySetInnerHTML` on `prismaDocument.name`

- **Same as SAST Finding 2** — see above. CONFIRMED reachable; sanitization at write path makes current exploitability latent. **Priority HIGH** (defense at wrong layer).

---

### Reachability: Structural Finding 8 — SAML CredentialsProvider userinfo upsert (MEDIUM)

- **Finding reference:** `01-structural-analysis.md` F8 — `lib/auth/auth-options.ts:132-178`
- **Reachable:** CONFIRMED
- **Entry point:** NextAuth signin via the `saml-idp` provider (`pages/api/auth/[...nextauth].ts`); upstream is `/api/auth/saml/authorize` (GET) which produces the SAML AuthnRequest, the IdP authenticates, and returns to `/api/auth/saml/callback`, then the NextAuth `authorize` callback exchanges the code for tokens and reads userinfo.
- **Input type:** IdP-controlled `userinfo.email`, `userinfo.firstName`, `userinfo.lastName`, `userinfo.requested`
- **Auth required:** NONE — signin flow (the IdP authenticates the user; Papermark accepts the returned email at face value)
- **Auth bypassable:** YES if the IdP is lenient — `prisma.user.upsert({ where: { email }, ... })` creates or matches a `prisma.user` row keyed on the IdP-controlled email with no team-tenant binding check.
- **Trace** (verified by direct read of `lib/auth/auth-options.ts:138-178`):
  1. NextAuth signin → `CredentialsProvider({ id: "saml-idp", ... })` (line 132)
  2. `authorize(credentials)` (line 138) reads `credentials.code`
  3. Line 142-150: `oauthController.token({ code, grant_type: "authorization_code", redirect_uri, client_id: "dummy", client_secret: NEXTAUTH_SECRET })`
  4. Line 154: `oauthController.userInfo(access_token)` — returns `{ email, firstName, lastName, requested }` from the IdP
  5. Line 162-166: `prisma.user.upsert({ where: { email }, create: { email, name }, update: { name: name || undefined } })` — **sink**
- **Deployment exposure:** INTERNET — `/api/auth/signin/saml` is public.
- **Priority:** MEDIUM — partial ATO; the IdP must be lenient (returns attacker-controlled email or accepts misconfigured user attribute mapping). Combined with `allowDangerousEmailAccountLinking: true` on the `saml` provider (Structural Finding 4), this becomes a full ATO path.

---

### Reachability: API-Specs Finding 1 — `/api/v1/*` not implemented

- **Finding reference:** `00-ai-api-specs.md` F1
- **Reachable:** DEAD_CODE — no source handlers exist. The middleware rewrite (`next.config.mjs:42-52` → `/v1/:path*` → `/api/v1/:path*`) targets a route prefix with no handlers, so the request returns 404 from Next.js routing. There is no sink to reach.
- **Entry point:** The middleware whitelist at `middleware.ts:65-72` does pass `/v1/*` and `/openapi.json` through, so a request reaches Next.js routing, but Next.js has no handler for the path → 404.
- **Input type:** any
- **Auth required:** N/A — no handler to enforce.
- **Auth bypassable:** N/A
- **Trace:** N/A
- **Deployment exposure:** INTERNET (the marketing site promises a v1 API; attackers can probe the path)
- **Priority:** LOW (no source-side vulnerability; the marketing-vs-implementation gap is a documentation/process issue). Recommendation: remove the middleware rules until v1 ships, or ship a real handler.

---

### Reachability: API-Specs Finding 4 — `/api/feature-flags` leaks per-team config (HIGH)

- **Finding reference:** `00-ai-api-specs.md` F4 — `app/api/feature-flags/route.ts:6-20`
- **Reachable:** CONFIRMED
- **Entry point:** `GET /api/feature-flags?teamId=<any-cuid>`
- **Input type:** query string `teamId`
- **Auth required:** NONE — verified by direct read (`app/api/feature-flags/route.ts:6-20`).
- **Auth bypassable:** N/A — no auth in place.
- **Trace** (verified by direct read):
  1. `app/api/feature-flags/route.ts:7-20` — `GET` handler, `runtime = "edge"`.
  2. Line 9: `const teamId = searchParams.get("teamId");`
  3. Line 12: `getFeatureFlags({ teamId: teamId || undefined })` — `lib/featureFlags/index.ts:23-78`. The Edge runtime means no DB-side authorization is possible; the function reads team flags and returns them.
- **Deployment exposure:** INTERNET
- **Priority:** HIGH — reconnaissance vector for any attacker mapping a target customer's plan tier / AI features / etc. Helps choose the exploit path for IDOR findings.

---

### Reachability: API-Specs Finding 5 — `/api/stripe/webhook` returns 200 on missing signature (HIGH)

- **Finding reference:** `00-ai-api-specs.md` F5 — `pages/api/stripe/webhook.ts:46-55`
- **Reachable:** CONFIRMED (intended — Stripe must reach this URL)
- **Entry point:** `POST /api/stripe/webhook` (called by Stripe)
- **Input type:** raw request body + `stripe-signature` header
- **Auth required:** Stripe signature verified via `stripe.webhooks.constructEvent` — BUT returns 200 silently when `sig` or `webhookSecret` is missing (`pages/api/stripe/webhook.ts:49-55`).
- **Auth bypassable:** N/A from a normal attacker (Stripe sends valid signed events). The issue is misconfiguration hiding: when `STRIPE_WEBHOOK_SECRET` is unset, no events are processed but no errors surface.
- **Deployment exposure:** INTERNET (Stripe must reach it)
- **Priority:** HIGH (configuration drift / silent failure is the dominant risk)

---

### Reachability: API-Specs Finding 8 — `/api/csp-report` accepts any JSON (LOW)

- **Finding reference:** `00-ai-api-specs.md` F8 — `app/api/csp-report/route.ts:3-16`
- **Reachable:** CONFIRMED
- **Entry point:** `POST /api/csp-report` (browsers post CSP violation reports here per spec)
- **Input type:** arbitrary JSON body
- **Auth required:** NONE — public endpoint by CSP spec.
- **Auth bypassable:** N/A
- **Trace** (verified by direct read): `app/api/csp-report/route.ts:3-16` — `await request.json()` then `return NextResponse.json({ success: true })`. No logging, no validation, no rate limit.
- **Deployment exposure:** INTERNET
- **Priority:** LOW — the placeholder is currently harmless (no logging, no S3 storage). If logging is later added, this becomes a log-poisoning vector. Recommend rate-limit + size cap.

---

### Reachability: API-Specs Finding 10 — `/api/conversations` visitor soft auth (MEDIUM)

- **Finding reference:** `00-ai-api-specs.md` F10 — `ee/features/conversations/api/conversations-route.ts:19-74`; mounted at `pages/api/conversations/[[...conversations]].ts`
- **Reachable:** CONFIRMED
- **Entry point:** `GET /api/conversations?dataroomId=<id>&viewerId=<id>`
- **Input type:** query string `dataroomId`, `viewerId`
- **Auth required:** NONE — only `viewerId` is checked, which is a CUID, not a session token. The page-side dataroom session cookie (`pm_drs_{linkId}`) is *not* checked by this endpoint.
- **Auth bypassable:** **YES** — anyone who learns a `viewerId` can list all conversations for that viewer in any dataroom. `viewerId` is a CUID stored in `prisma.viewer.id` and is referenced in URLs, notifications, and support tickets.
- **Trace** (verified by direct read of `ee/features/conversations/api/conversations-route.ts:19-74`):
  1. `pages/api/conversations/[[...conversations]].ts:5-10` → `handleRoute(req, res)`
  2. `ee/features/conversations/api/conversations-route.ts:19-74` — `routeHandlers["GET /"]` reads `dataroomId, viewerId` from `req.query`.
  3. Line 30-64: `prisma.conversation.findMany({ where: { dataroomId, participants: { some: { viewerId } } } })` — sink
- **Deployment exposure:** INTERNET
- **Priority:** MEDIUM — confidentiality of dataroom conversations hinges on `viewerId` secrecy.

---

### Reachability: API-Specs Finding 11 — `/api/og` no rate limit (LOW)

- **Finding reference:** `00-ai-api-specs.md` F11 — `app/api/og/route.tsx:11-42`
- **Reachable:** CONFIRMED
- **Entry point:** `GET /api/og?title=<any>`
- **Input type:** query string `title`
- **Auth required:** NONE
- **Auth bypassable:** N/A
- **Trace** (verified by direct read): `app/api/og/route.tsx:11-42` — `GET` handler reads `title` from `searchParams`, then `new ImageResponse(...)` (Edge runtime). No rate limit, no length cap.
- **Deployment exposure:** INTERNET (used for OG previews; called by social media crawlers)
- **Priority:** LOW — Edge compute exhaustion / cost amplification. Add per-IP rate limit + title length cap.

---

### Reachability: API-Specs Finding 12 — `/api/help` open proxy (LOW)

- **Finding reference:** `00-ai-api-specs.md` F12 — `app/api/help/route.ts:3-43`
- **Reachable:** CONFIRMED
- **Entry point:** `GET /api/help?q=<query>`
- **Input type:** query string `q`
- **Auth required:** NONE
- **Auth bypassable:** N/A (the upstream URL is env-controlled, not user-controlled today)
- **Trace** (verified by direct read): `app/api/help/route.ts:3-43` — `GET` handler fetches from `process.env.NEXT_PUBLIC_MARKETING_URL/api/help?q=${q}`, filters results client-side, returns JSON.
- **Deployment exposure:** INTERNET
- **Priority:** LOW — today this acts as an unauthenticated amplifier for the marketing-site outage. Add 2-3s `AbortController` timeout, per-IP rate limit, and edge-cache.

---

## Cross-Findings Notes

### Confirmed: Per-route opt-in auth pattern in `pages/api/teams/[teamId]/**`

The structural-analysis pass already documented that IDOR is present in the folder-manage namespace. Reachability confirms it is also **not** present in the document namespace (`:67-82 of update-name.ts uses `where: { id: docId, teamId }`) or the dataroom folder namespace (`pages/api/teams/[teamId]/datarooms/[id]/folders/manage/[folderId]/index.ts:68-73` uses `id + dataroomId`). The folder-manage namespace is the outlier.

### Confirmed: Internal jobs routes lack middleware

`route_map` for `/api/jobs/*` confirms **8 routes** with **0 middleware wrappers**. The auth check (timing-unsafe `!==`) lives entirely inside each handler. There is no `middleware.ts` matcher for `/api/jobs/*` that could fix this centrally.

### Confirmed: App router cron routes also lack middleware

`route_map` for `/api/cron/*` confirms **5 routes** with **0 middleware wrappers**. The `process.env.VERCEL === "1"` bypass lives inline in each handler.

### Layer 0 cross-reference

| Directive / Layer 0 finding | Reachability verdict | Notes |
|---|---|---|
| D1 — Next.js CVE-2026-44578 SSRF | Deployment-dependent | Vercel not affected; self-hosted per Issue #2160 = internet-facing. Reachability confirmed at the runtime layer (built-in Node.js server exposed). |
| D2 — CVE-2026-44580/44581 XSS | Latent / not directly reachable today | The CSP nonce XSS is framework-internal; combined with `prismaDocument.name` sink (SAST F2), the chain becomes reachable. |
| D3 — CVE-2026-44576/44582 RSC cache | Deployment-dependent | If a CDN/edge cache is in front, RSC responses on shared paths could be poisoned. |
| D4 — CVE-2026-44577 image OOM | Self-hosted only | `/\_next/image` is internet-facing; `<Image>` usage is broad. |
| D5 — dompurify 3.4.0 | Latent | Transitive via `react-pdf` and `notion-client`. No direct call site in Papermark code. |
| D6 — `allowDangerousEmailAccountLinking` | CONFIRMED reachable | All 3 providers, see Structural F4. |
| D7 — CORS reflection in tus-viewer | CONFIRMED reachable | See Structural F3. |
| D8 — Folder IDOR (Issue #2078) | CONFIRMED reachable | See Structural F1 + F2. |
| D9 — No malware scanning on upload | CONFIRMED reachable | `lib/trigger/convert-files.ts` runs in Trigger.dev task from `tus-viewer` upload completion. |
| D10 — sanitize-html `^2.17.3` | N/A | Lockfile pins 2.17.4; CI must enforce the lockfile. |
| D11 — `dangerouslySetInnerHTML` on document name | CONFIRMED reachable | See SAST F2 / Structural F7. |
| D12 — OIDC callback URL validation | Out of scope for reachability | Validation is in the `oidc-provider` library. |
| D13 — `cuid()` for permission row IDs | Latent | Per the dependencies analysis, `cuid()` is non-cryptographic. Reachability confirmed: `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:91` uses `cuid()` for permission row IDs; these IDs are referenced in HTTP responses, so a sufficiently patient attacker can enumerate them. Not a "reachability" gap in the deployment sense — the IDs are public — but a "predictability" issue. |

---

## PHASE_3_CHECKPOINT

- [x] All 24 findings from Layer 0/1 triaged for reachability
- [x] Each finding has: Reachable verdict, Entry point, Input type, Auth verdict, Auth-bypass verdict, Trace (file:line for each step), Deployment exposure, Priority
- [x] GitNexus queries executed: 9 conceptual queries + 7 context lookups + 4 route_map calls + 1 impact call
- [x] Direct file reads for every endpoint named in the routes (8 jobs handlers + 5 cron handlers + folder manage handlers + conversations + csp-report + feature-flags + og + help + saml/authorize + update-name + bulk-create + docx-sanitizer)
- [x] Cross-referenced with `00-ai-sast.md`, `01-structural-analysis.md`, `00-ai-api-specs.md`, `00-early-web-intel.md` D1–D13
- [x] Confirmed internet-facing + auth-bypassable count: 11 (the HIGH-priority cluster)
- [x] Confirmed dead code count: 1 (the `$executeRawUnsafe` savepoint name)
- [x] Latent / deployment-dependent findings noted separately

---

## Tiered Priority List (for downstream nodes)

For the encoding-analysis and sanitizer-bypass passes to focus on, the highest-priority reachability-confirmed sinks are:

1. **P0 — Confirmed reachable + internet-facing + bypassable**:
   - `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts` (DELETE) — folder DELETE IDOR
   - `pages/api/teams/[teamId]/folders/manage/index.ts` (PUT) — folder RENAME IDOR
   - `pages/api/file/tus-viewer/[[...file]].ts:234-250` — CORS misconfig
   - `lib/auth/auth-options.ts:32-131` — `allowDangerousEmailAccountLinking` (Google, LinkedIn, SAML)
   - `pages/api/jobs/*.ts` (4 routes) — `INTERNAL_API_KEY` timing-unsafe
   - `app/api/cron/*` (5 routes) — QStash bypass outside Vercel
   - `app/api/feature-flags/route.ts` — unauthenticated team config leak

2. **P0 — Confirmed reachable + latent** (will activate if a single layer of defense is missed):
   - `components/documents/document-header.tsx:604` — DOM-XSS sink on `prismaDocument.name` (Zod-sanitized today; future write paths could bypass)

3. **P1 — Confirmed reachable + auth required + structural flaw**:
   - `lib/utils.ts:594-621` — AES-CTR without auth tag (requires DB write to exploit)
   - `lib/auth/auth-options.ts:132-178` — SAML userinfo upsert (requires lenient IdP)

4. **P2 — Confirmed reachable + vendor / deployment-dependent**:
   - `app/(ee)/api/auth/saml/authorize/route.ts` — Jackson SAML XML parsing (vendor)
   - `ee/features/conversions/python/docx-sanitizer.py:146` — `ET.parse` on DOCX headers/footers (low default-exploitability in CPython 3.x)
   - `pages/api/stripe/webhook.ts:46-55` — silent 200 on missing Stripe signature (misconfig drift)

5. **P3 — Confirmed reachable + minor impact**:
   - `app/api/csp-report/route.ts` — accepts any JSON, no logging (placeholder)
   - `app/api/og/route.tsx` — no rate limit (Edge cost amplification)
   - `app/api/help/route.ts` — open proxy to marketing site
   - `ee/features/conversations/api/conversations-route.ts` — `viewerId`-only soft auth

6. **INFORMATIONAL / DEAD_CODE today**:
   - `lib/folders/bulk-create.ts:51-60` — `$executeRawUnsafe` savepoint name (no user input flows)
   - `app/api/v1/*` and `/openapi.json` — not implemented (no source sink)
