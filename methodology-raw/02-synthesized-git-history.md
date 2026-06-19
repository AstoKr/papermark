# Synthesized Findings: Git History (M4 — synthesize-git-history)

**Worktree**: archon/task-whitebox-papermark
**Date**: 2026-06-19
**Sources combined**:
- `methodology-raw/00-ai-git-history.md` — AI manual analysis
- `tools-raw/git-log-security.txt` — raw commit log
- `tools-raw/normalized.json` — no git-log-specific findings (tool output is just commit list)
- `methodology-raw/01-structural-analysis.md` — current code context (cross-checked)

---

## Phase 2 Summary

| Source | Findings contributed |
|--------|---------------------|
| AI manual analysis | 8 (F1–F8) |
| Tool output (git-log) | 0 specific findings — list of 50 commit messages only |
| **Dedup merge** | None required — tool did not flag findings to merge |
| **Verified against HEAD** | All 8 AI findings re-checked against current source (see Verification column) |

The `git-log-security.txt` tool is a commit-message extractor, not a vulnerability detector. It surfaces recent security-relevant commits (e.g., `9db42913 fix(api/views): close authorization bypass on POST /api/views`, `b1fd5706 fix: exclude /mcp from auth middleware…`, `b68c937b Accept comma/semicolon/newline…`, `7fa52b23 feat: security updates`) but does not flag patterns or call out unfixed adjacent code. All findings therefore originate from the AI manual analysis, with the tool output serving only as commit-existence corroboration.

---

## Synthesized Findings (ranked by exploitability)

### S-001 — Missing auth on `pages/api/progress-token.ts` mints Trigger.dev public access tokens

- **Severity**: HIGH
- **Source**: AI-only (tool did not flag)
- **File**: `pages/api/progress-token.ts:1-29`
- **Commit**: n/a (present in HEAD, not in git-log tool output)
- **Confidence**: HIGH (code re-read in HEAD, no auth present)
- **Exploitability**: HIGH

**Description**: `GET /api/progress-token?documentVersionId=…` calls `generateTriggerPublicAccessToken("version:" + documentVersionId)` and returns the resulting token. No `getServerSession`, no team membership check, no document-ownership check. Anyone who can guess or enumerate a `documentVersionId` (cuid, exposed in dashboards, share URLs, error messages, analytics feeds) receives a Trigger.dev public access token scoped to that version.

**Evidence (HEAD, lines 13-23)**:
```ts
const { documentVersionId } = req.query;
if (!documentVersionId || typeof documentVersionId !== "string") {
  return res.status(400).json({ error: "Document version ID is required" });
}
try {
  const publicAccessToken = await generateTriggerPublicAccessToken(
    `version:${documentVersionId}`,
  );
  return res.status(200).json({ publicAccessToken });
```

**Why this slipped past the recent auth sweep**: Commit `9db42913` closed the link↔document IDOR in `app/api/views/route.ts` but did not extend coverage to the two adjacent endpoints that accept a raw `documentVersionId` query parameter. This is the most actionable finding from the git-history pass — same root cause class as F1 (untrusted versionId ⇒ cross-resource confusion), just on a different blast-radius surface (Trigger.dev token issuance).

**Recommendation**: Either (a) require an authenticated session + team-membership check, plus a `prisma.documentVersion.findFirst({ where: { id, document: { team: { users: { some: { userId } } } } } })` ownership check, before issuing the token; or (b) gate the endpoint to internal `INTERNAL_API_KEY` callers and have the team-side code that needs the token call it server-side.

---

### S-002 — Missing auth on `pages/api/teams/[teamId]/documents/document-processing-status.ts` (information disclosure + recon oracle)

- **Severity**: HIGH
- **Source**: AI-only (tool did not flag)
- **File**: `pages/api/teams/[teamId]/documents/document-processing-status.ts:1-32`
- **Commit**: n/a (present in HEAD, not in git-log tool output)
- **Confidence**: HIGH (code re-read in HEAD, no auth present)
- **Exploitability**: MEDIUM (no direct data destruction, but valuable as a recon oracle for S-001 and other gated endpoints)

**Description**: Endpoint accepts `documentVersionId` and returns `{ currentPageCount, totalPages, hasPages }` for that version. No `getServerSession`, no team membership check, no document-ownership check. The URL embeds `teamId` but it is never used. Confirmed in HEAD — the handler is 32 lines, the only inputs are `req.query.documentVersionId`, the only output is `res.status(200).json(status)`.

**Evidence (HEAD, lines 9-22)**:
```ts
const { documentVersionId } = req.query as { documentVersionId: string };
const documentVersion = await prisma.documentVersion.findUnique({
  where: { id: documentVersionId },
  select: {
    numPages: true,
    hasPages: true,
    _count: { select: { pages: true } },
  },
});
if (!documentVersion) {
  return res.status(404).end();
}
```

**Impact**:
1. **Confirmation oracle**: 200 vs 404 tells an attacker which `documentVersionId` values are valid, even if they can't yet read their content.
2. **Timing oracle for conversion**: `hasPages` and `currentPageCount` reveal whether conversion finished and how many pages exist, which can be paired with the views endpoints that *do* require auth.
3. **Polling exposure**: Wired into `useDocumentProcessingStatus` (`lib/swr/use-document.ts:200-218`) which calls it every 3 seconds, so a successful 200 is a high-confidence signal the document version is real and active.

**Recommendation**: Apply the same `getServerSession` + `userTeam.findUnique({ userId, teamId })` + document-ownership pattern used in `lib/documents/get-file-helper.ts:17-37`.

---

### S-003 — Authorization-bypass mass-assignment on POST /api/views was correctly fixed (verified complete)

- **Severity**: CRITICAL (in the historical sense — bug is **FIXED** in HEAD)
- **Source**: Both (commit `9db42913` is in git-log tool output; AI analyzed the fix completeness)
- **File**: `app/api/views/route.ts:187-220` (fix location), `:735` and `:889` (downstream sinks)
- **Commit**: `9db42913 fix(api/views): close authorization bypass on POST /api/views`
- **Confidence**: HIGH (fix re-verified in HEAD)
- **Exploitability**: n/a — fixed

**Description**: Before the fix, the POST handler accepted `documentId` from the request body and used it downstream in `prisma.view.create` and `recordLinkView`. A caller with any valid link id could submit any `documentId` they wished, recording a view against a different document than the link was bound to (mass-assignment / IDOR).

**Fix re-verification in HEAD**:
- Line 187: `if (!link.documentId) return res.status(403)` — refuses dataroom-style links without primary document id.
- Line 201: `prisma.documentVersion.findUnique({ where: { id: documentVersionId }, select: { documentId: true } })` — fetches the trusted `documentId` from the version row.
- Line 213: `if (requestedVersion.documentId !== link.documentId) return res.status(403)` — version/document mismatch rejected.
- Line 220: `const documentId = link.documentId` — body-controlled `documentId` is no longer destructured; trusted DB value is used.
- Line 735 (view.create) and 889 (recordLinkView): both reference the trusted `documentId` variable.

**Similar patterns checked**:
- `app/api/views-dataroom/route.ts:1061-1102` — SAFE (uses DB-derived `documentId`).
- `app/api/views/pages/route.ts:81-123` — SAFE (both `viewId`-authenticated and `previewToken`-authenticated paths verify version's `documentId` against the link's `documentId` or `dataroomId`).

**Verdict**: Fix is complete for the `/api/views*` family. The natural follow-up would have been to also gate the two endpoints that S-001 and S-002 flag — they were missed.

---

### S-004 — Email/domain list input validation hardening (silent-save bug closed)

- **Severity**: MEDIUM (correctness + UX with security-adjacent impact)
- **Source**: Both (commit `b68c937b` is in git-log tool output; AI analyzed the fix coverage)
- **Files**: 9 files (see below); helper added in `lib/utils.ts`
- **Commit**: `b68c937b Accept comma/semicolon/newline in email + domain list inputs and block silent saves`
- **Confidence**: HIGH
- **Exploitability**: LOW (no direct security boundary crossed — UX/data-integrity only)

**Description**: List inputs in block list, ignored domains, link allow/deny list, dataroom group members, and visitor groups previously split on `\n` only, silently dropping entries pasted with `,` or `;` and silently discarding invalid entries on save. Fix introduces `validateList` helper and gates Save button + Cmd+Enter shortcut on validity.

**Files updated in `b68c937b`**:
- `lib/utils.ts` (new `validateList` helper)
- `components/settings/global-block-list-form.tsx`
- `components/settings/ignored-domains-form.tsx`
- `components/links/link-sheet/allow-list-section.tsx`
- `components/links/link-sheet/deny-list-section.tsx`
- `components/links/link-sheet/index.tsx`
- `components/links/link-sheet/link-options.tsx`
- `components/datarooms/groups/add-member-modal.tsx`
- `components/visitors/visitor-group-modal.tsx`

**Similar pattern NOT covered (LOW)**:
- `components/datarooms/add-viewer-modal.tsx:97` — still uses legacy `inputValue.split(",")` pattern. **Re-verified in HEAD** — the line is unchanged. Downstream impact limited: emails are used for invitation delivery, not access control; toast still informs user via older "Found one or more invalid email addresses" copy rather than the new destructive list. This is a follow-up to close the email-list UX gap project-wide.

---

### S-005 — MCP middleware redirect (auth-bypass-by-misdirection) was correctly fixed (verified complete)

- **Severity**: MEDIUM (correctness fix)
- **Source**: Both (commit `b1fd5706` is in git-log tool output; AI analyzed the fix)
- **File**: `middleware.ts:49` (matcher regex)
- **Commit**: `b1fd5706 fix: exclude /mcp from auth middleware so remote MCP endpoint returns 401`
- **Confidence**: HIGH
- **Exploitability**: n/a — fixed

**Description**: The Next.js auth middleware's matcher regex caught `/mcp` and redirected unauthenticated callers to `/login`. The `/mcp` remote MCP endpoint performs its **own** Bearer-token auth and must return `401` for unauthenticated callers, not bounce them to a UI login page (which breaks non-browser MCP clients like Claude Desktop).

**Fix re-verification in HEAD**: `middleware.ts:49` contains `mcp/?$` in the negative lookahead, end-anchored so it matches only `/mcp` and `/mcp/`, not unrelated future routes like `/mcp-oauth/*`. Comment at line 47 confirms the intent.

**Similar patterns checked**: All other middleware exclusions (`api/`, `oauth/`, `.well-known/`, `_next/`, `_static`, `vendor`, `_icons`, `_vercel`, `favicon.ico`, `sitemap.xml`, `robots.txt`) are end-state endpoints doing their own auth or are static files. No other API-shaped route is incorrectly bounced to `/login`.

---

### S-006 — Reverted-fix dance on `lib/utils/download-document.ts` (audit trail noise, not a vuln)

- **Severity**: LOW (informational — not a security vulnerability)
- **Source**: AI-only (the revert+reapply pattern is only visible via git-history inspection; the tool output lists both commits but doesn't flag them)
- **File**: `lib/utils/download-document.ts`
- **Commit pair**: `81b8508e revert fix` (2026-05-25 20:09 UTC) → `1e88cf87 fix: cloudfront AccessDenied for downloads with renamed filename` (2026-05-25 20:42 UTC)
- **Confidence**: HIGH
- **Exploitability**: n/a

**Description**: A revert+re-apply dance within 33 minutes looks alarming but is benign. The intermediate revert undid an `<iframe>` cross-origin download approach that was masking a CloudFront AccessDenied error page. The re-applied commit (`1e88cf87`) keeps the iframe approach AND fixes the underlying root cause: CloudFront canned-policy signed URLs can't handle the `''` in RFC 5987 `filename*=UTF-8''…` because signer and browser disagree on percent-encoding of `'`. The proper fix introduces a **wildcard-resource custom policy** so the signature is over the Policy JSON instead of URL bytes.

**Verdict**: No CVE-class issue. End state (1e88cf87) is correct.

---

### S-007 — Proactive SSRF protection module (informational — defensive feature, well-applied)

- **Severity**: INFO (no known exploit; well-applied defensive code)
- **Source**: Both (commit `7fa52b23 feat: security updates` is in git-log; AI audited the coverage)
- **Files**: `lib/utils/ssrf-protection.ts` (501 lines), `lib/zod/url-validation.ts`
- **Commit**: `7fa52b23 feat: security updates`
- **Confidence**: HIGH
- **Exploitability**: n/a — defensive

**Description**: New module that validates URL hostnames as public (rejects localhost, link-local, private RFC1918, IPv6 ULA, etc.), resolves DNS via `node:dns/promises` to catch DNS-rebinding and hostname-as-IP tricks, performs HTTPS fetches with configurable redirect limits and timeouts. Wired into `pages/api/webhooks/services/[...path]/index.ts`, `notionUrlUpdateSchema`, `linkUrlUpdateSchema`, `documentUploadSchema`, `webhookFileUrlSchema`.

**Coverage audit**: AI searched for direct `await fetch(…)` calls on user-controllable URLs:
- `pages/api/mupdf/convert-page.ts:47` — `fetch(url)` on caller-supplied signed S3 URL; gated by `INTERNAL_API_KEY` (internal-only).
- `lib/trigger/*` background tasks — operate on signed URLs created inside the app, not user-controlled.

No public-facing `fetch()` of user URLs bypasses the SSRF module. Note: this conclusion is from the AI's static search; a separate code-path audit (out of scope for M4) would be required to assert it absolutely.

---

## Cross-cutting observations

1. **The team's recent security cadence is healthy** — multiple VFCs in the last 30 days (`9db42913`, `b1fd5706`, `b68c937b`, `7fa52b23`, `ece358be`, `970fb4cf`) plus a proactive SSRF module.
2. **The two unauthenticated endpoints (S-001, S-002) appear to have been missed by the recent views/route.ts authorization sweep.** They are the only routes accepting a `documentVersionId` query parameter that don't enforce session + team membership. These are the **most actionable findings** in this report and should be tracked as the natural follow-up to commit `9db42913`.
3. **The link↔document binding fix at 9db42913 is correctly applied to all routes in the `/api/views*` family.** `views-dataroom` and `views/pages` both fetch the version's `documentId` from the database before trusting it.
4. **`add-viewer-modal.tsx` is the only email-list input that wasn't migrated by `b68c937b`.** Low priority (UX, not security).

## Tool-vs-AI delta

- **AI-only findings (no tool corroboration)**: S-001, S-002, S-006. The tool only lists commit messages; it did not flag any of these — the AI inferred S-001 and S-002 by reading current HEAD code and noting the asymmetry with the fixed `app/api/views/route.ts`.
- **Both (AI + tool)**: S-003, S-004, S-005, S-007 — commits appear in `git-log-security.txt`, AI analyzed completeness.
- **Tool-only findings**: None. The tool is a commit-extractor, not a vulnerability detector for this domain.

## PHASE_3_CHECKPOINT

- [x] Both sources read and compared (`00-ai-git-history.md`, `git-log-security.txt`)
- [x] Duplicates merged (none — tool had no flagged findings)
- [x] False positives eliminated (verified S-001, S-002 still unfixed in HEAD; S-003, S-005 fix still in place; S-004 unfixed sibling S-004 sub-finding confirmed)
- [x] Findings ranked by exploitability (S-001 and S-002 are the highest-priority actionable items; S-003 and S-005 are verified-fixed historicals)
