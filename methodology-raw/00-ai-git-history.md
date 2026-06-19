# AI Analysis: git history (M4 — ai-analyze-git-history)

**Worktree**: archon/task-whitebox-papermark
**Scope**: Security-relevant commits in last ~100 commits, plus similar-pattern audit of current HEAD.

---

## Summary

| # | Severity | Commit | Type | File |
|---|----------|--------|------|------|
| 1 | CRITICAL | 9db42913 | VFC (fix complete, similar patterns checked) | app/api/views/route.ts |
| 2 | HIGH | n/a (uncommitted) | similar-pattern (NEW finding) | pages/api/teams/[teamId]/documents/document-processing-status.ts |
| 3 | HIGH | n/a (uncommitted) | similar-pattern (NEW finding) | pages/api/progress-token.ts |
| 4 | MEDIUM | b1fd5706 | VFC (correctness fix) | middleware.ts |
| 5 | MEDIUM | b68c937b | VFC (input validation, mostly complete) | 9 files / lib/utils.ts |
| 6 | LOW | b68c937b | similar-pattern (unfixed) | components/datarooms/add-viewer-modal.tsx |
| 7 | LOW | 81b8508e / 1e88cf87 | reverted-fix (UI exposure, not vuln) | lib/utils/download-document.ts |
| 8 | INFO | 7fa52b23 | feature (proactive SSRF protection) | lib/utils/ssrf-protection.ts |

---

## Finding 1: Authorization Bypass on POST /api/views

- **Severity**: CRITICAL
- **Commit**: 9db42913 (`fix(api/views): close authorization bypass on POST /api/views`)
- **Type**: incomplete-fix (verified complete)
- **File**: `app/api/views/route.ts`
- **Author/Date**: Marc Seitz, 2026-05-14

### Description

The `POST /api/views` route accepted a `documentId` field from the request body and used it downstream (e.g. as `documentId` in `prisma.view.create` and `recordLinkView`). A link in Papermark is bound to a single document (`link.documentId`). A caller with a valid link id could submit any `documentId` they wished and have the resulting `View` row persisted against that document — bypassing the link↔document relationship. This is a classic **mass-assignment / IDOR** pattern that allowed one link's traffic to be misattributed to any document the caller chose (and confuses analytics, view history, and viewer↔document associations).

### Fix analysis

The fix:

1. **Removed `documentId` from the destructured body** — callers can no longer influence which document the view is recorded against.
2. **Added validation that `link.documentId` is set** — refuse dataroom-style links that lack a primary document id (returns 403).
3. **Added validation of `documentVersionId`** — fetches `documentVersion.documentId` from the database and asserts it equals `link.documentId` (returns 403 on mismatch, 404 if the version is unknown).
4. **Set `documentId = link.documentId`** after the checks — using the trusted database value, never the request body.

### Verification of fix completeness

- Lines 187–220 of current `app/api/views/route.ts` confirm the four checks are present and `documentId` is no longer read from the body.
- The earlier destructuring at lines 49–66 no longer contains `documentId`.
- Downstream uses at lines 735 (view.create) and 889 (recordLinkView) use the trusted `documentId` variable.

### Verification of fix correctness

The fix is correct: the body is treated as untrusted input, the database row is the only source of truth for the link↔document binding, and the `documentVersionId` lookup additionally prevents the (newer) attack of swapping versions of the same document.

### Similar patterns checked

I audited every other route that accepts a `documentVersionId` (or related version/document id) from the client:

| Route | Verdict | Evidence |
|-------|---------|----------|
| `app/api/views-dataroom/route.ts` | SAFE | Lines 1061–1102: requires `documentVersionId`, fetches `documentVersionAccess.documentId` from DB, then checks `prisma.dataroomDocument.findUnique({ dataroomId: link.dataroomId, documentId: documentVersionAccess.documentId })`. Uses DB-derived `documentId`, not body. |
| `app/api/views/pages/route.ts` | SAFE | Lines 81–123: both `viewId`-authenticated and `previewToken`-authenticated paths fetch `link.documentId` / `link.dataroomId` from DB and compare against the version's `documentId`. |
| `pages/api/teams/[teamId]/documents/[id]/versions/index.ts` | SAFE | Server-side route, team-membership gated. |
| `pages/api/teams/[teamId]/datarooms/[id]/documents/index.ts` | Server-side, team-gated |
| `pages/api/mupdf/convert-page.ts` | Gated by `INTERNAL_API_KEY`, uses signed URL from caller |
| `pages/api/progress-token.ts` | **MISSING AUTH — see Finding 3** |
| `pages/api/teams/[teamId]/documents/document-processing-status.ts` | **MISSING AUTH — see Finding 2** |
| `pages/api/teams/[teamId]/documents/agreement.ts` | Server-side, session + team membership gated |
| `app/(ee)/api/links/[id]/upload/route.ts` | Dataroom session gated; uses viewer-owned `documentUpload.documentId` |

The fix at 9db42913 is complete for the views endpoint. The closely related endpoints in the same domain (`views-dataroom`, `views/pages`) already follow the correct pattern.

### Evidence (commit diff highlights)

```diff
-      documentId,
       userId,
       documentVersionId,
       ...
+    if (!link.documentId) { return 403 }
+    if (!documentVersionId || typeof documentVersionId !== "string") { return 400 }
+    const requestedVersion = await prisma.documentVersion.findUnique({...})
+    if (!requestedVersion) { return 404 }
+    if (requestedVersion.documentId !== link.documentId) { return 403 }
+    const documentId = link.documentId;
```

### Similar patterns

None within the `/api/views*` family — the fix is fully applied. However, see Findings 2 & 3 for **related** missing-auth issues in adjacent endpoints.

---

## Finding 2: Missing authentication on document-processing-status

- **Severity**: HIGH
- **Commit**: n/a (uncommitted; present in current HEAD)
- **Type**: similar-pattern (NOT covered by any VFC)
- **File**: `pages/api/teams/[teamId]/documents/document-processing-status.ts`

### Description

The endpoint accepts a `documentVersionId` query parameter and returns `{ currentPageCount, totalPages, hasPages }` for the matching `DocumentVersion` row. It performs **no authentication** and **no team-membership check**. Anyone (even unauthenticated callers) can probe for the existence and processing state of any document version by guessing or scraping its id.

This endpoint is wired into a polling hook (`useDocumentProcessingStatus` in `lib/swr/use-document.ts:200–218`) that calls it every 3 seconds. The exposure path is therefore:

- The endpoint itself returns 200 to any caller that supplies a valid `documentVersionId`.
- A successful 200 confirms the document version exists and reveals its page count + processing status (whether pages have been generated yet).
- This information aids reconnaissance for the other endpoints (`/api/views`, `/api/views/pages`, `/api/views-dataroom`, `/api/jobs/get-thumbnail`) that *do* enforce auth — it gives an attacker a cheap oracle for "which version ids are valid in the system" and tells them when conversion finished.

### Fix analysis

Not yet fixed. The endpoint should:

1. Call `getServerSession(req, res, authOptions)` like other team-scoped routes.
2. Verify the caller is a member of the URL's `teamId`.
3. Verify the supplied `documentVersionId` belongs to a document owned by that team (the same pattern used in `lib/documents/get-file-helper.ts:17–37` for thumbnails).

### Evidence

```typescript
// pages/api/teams/[teamId]/documents/document-processing-status.ts
export default async function handler(req, res) {
  const { documentVersionId } = req.query as { documentVersionId: string };
  const documentVersion = await prisma.documentVersion.findUnique({
    where: { id: documentVersionId },
    select: { numPages: true, hasPages: true, _count: { select: { pages: true } } },
  });
  if (!documentVersion) return res.status(404).end();
  const status = {
    currentPageCount: documentVersion._count.pages,
    totalPages: documentVersion.numPages,
    hasPages: documentVersion.hasPages,
  };
  res.status(200).json(status);
}
```

No `getServerSession`, no team membership, no document-ownership check.

### Similar patterns

- `pages/api/jobs/get-thumbnail.ts` (sibling endpoint) **does** enforce session + team membership via `getFileForDocumentPage` (`lib/documents/get-file-helper.ts:17–37`). The processing-status endpoint should follow the same pattern.

---

## Finding 3: Missing authentication on progress-token

- **Severity**: HIGH
- **Commit**: n/a (uncommitted; present in current HEAD)
- **Type**: similar-pattern (NOT covered by any VFC)
- **File**: `pages/api/progress-token.ts`

### Description

`GET /api/progress-token?documentVersionId=…` mints a Trigger.dev public access token scoped to `version:${documentVersionId}` and returns it to the caller. It performs **no authentication** and **no team-membership check**. This means:

- Anyone who can guess or enumerate a `documentVersionId` can obtain a Trigger.dev public access token bound to that version.
- The token's eventual blast radius depends on what `generateTriggerPublicAccessToken` authorizes at Trigger.dev, but the fact that it is **unauthenticated and undocumented in the public API surface** is itself a smell. At minimum it is a confirmation oracle for valid `documentVersionId` values (matching Finding 2).

### Fix analysis

Not yet fixed. The endpoint should:

1. Require an authenticated session.
2. Verify the caller has access to the version's parent document (via the same `document: { team: { users: { some: { userId } } } }` pattern used in `lib/documents/get-file-helper.ts`).
3. Possibly fold into an authenticated flow with a short-lived signed token instead of a Trigger.dev access token, to avoid even relying on Trigger.dev authorization.

### Evidence

```typescript
// pages/api/progress-token.ts
export default async function handle(req, res) {
  if (req.method !== "GET") return res.status(405).json({ error: "Method not allowed" });
  const { documentVersionId } = req.query;
  if (!documentVersionId || typeof documentVersionId !== "string") {
    return res.status(400).json({ error: "Document version ID is required" });
  }
  try {
    const publicAccessToken = await generateTriggerPublicAccessToken(`version:${documentVersionId}`);
    return res.status(200).json({ publicAccessToken });
  } catch (error) {
    console.error("Error generating token:", error);
    return res.status(500).json({ error: "Failed to generate token" });
  }
}
```

### Similar patterns

- `lib/api/documents/process-document.ts`, `lib/trigger/pdf-to-image-route.ts`, `pages/api/teams/[teamId]/documents/[id]/update-link-url.ts` all gate operations by `isTrustedTeam(teamId)` or session checks. `progress-token.ts` is the odd one out.

---

## Finding 4: MCP middleware redirect (auth-bypass-by-misdirection)

- **Severity**: MEDIUM
- **Commit**: b1fd5706 (`fix: exclude /mcp from auth middleware so remote MCP endpoint returns 401`)
- **Type**: VFC (correctness)
- **File**: `middleware.ts`

### Description

The Next.js auth middleware's matcher regex caught `/mcp` and redirected unauthenticated callers to `/login`. But `/mcp` is a remote MCP endpoint that performs its **own** Bearer-token authentication and is supposed to return `401` for unauthenticated callers, not bounce them to a UI login page. A `Location: /login` redirect breaks non-browser MCP clients (e.g. Claude Desktop) that follow redirects with the wrong intent and end up logged-out instead of receiving the proper 401.

### Fix analysis

The fix adds `mcp/?$` (end-anchored to match only the exact `/mcp` and `/mcp/` paths) to the middleware's negative lookahead. The `$` anchor is important — it prevents matching unrelated future routes like `/mcp-oauth/*`.

### Verification of fix completeness

- Single-line change to the `config.matcher` regex.
- No other middleware exclusions or matcher patterns exist for `/mcp`.
- The end-anchor matches the comment ("End-anchored so it matches only that exact path, not /mcp-oauth/*").

### Verification of fix correctness

Correct. The middleware now lets `/mcp` fall through to its own Bearer-token handler, which returns a proper `401` for unauthenticated callers.

### Similar patterns checked

- Other middleware exclusions (`api/`, `oauth/`, `.well-known/`, `_next/`, `_static`, `vendor`, `_icons`, `_vercel`, `favicon.ico`, `sitemap.xml`, `robots.txt`) are all end-state endpoints that do their own auth or are static files. No other "API-shaped" route is being incorrectly bounced to `/login`.

### Evidence (commit diff)

```diff
-    "/((?!api/|oauth/|\.well-known/|_next/|_static|vendor|_icons|_vercel|favicon.ico|sitemap.xml|robots.txt).*)",
+    "/((?!api/|oauth/|mcp/?$|\.well-known/|_next/|_static|vendor|_icons|_vercel|favicon.ico|sitemap.xml|robots.txt).*)",
```

---

## Finding 5: Email/Domain list input validation (silent-save + multi-separator)

- **Severity**: MEDIUM
- **Commit**: b68c937b (`Accept comma/semicolon/newline in email + domain list inputs and block silent saves`)
- **Type**: VFC (input validation hardening)
- **Files**:
  - `lib/utils.ts` (new `validateList` helper)
  - `components/settings/global-block-list-form.tsx`
  - `components/settings/ignored-domains-form.tsx`
  - `components/links/link-sheet/allow-list-section.tsx`
  - `components/links/link-sheet/deny-list-section.tsx`
  - `components/links/link-sheet/index.tsx` (parent that aggregates validation state)
  - `components/links/link-sheet/link-options.tsx`
  - `components/datarooms/groups/add-member-modal.tsx`
  - `components/visitors/visitor-group-modal.tsx`

### Description

Email/domain list inputs in Papermark (block list, ignored domains, link allow/deny list, dataroom group members, visitor groups) previously:

- Only split on `\n`, silently ignoring entries pasted with `,` or `;`.
- When the user pasted invalid entries, those entries were **silently dropped** on save — the form looked successful but the user's intended entries never landed.
- In link-sheet, the save button was always enabled, so users could save a partially-broken allow/deny list without realizing.

This is both a UX bug (silent data loss) and a security-relevant input-validation gap: a user pasting `attacker.com, @blocked.com, valid@example.com` would only get `valid@example.com` saved, defeating the intent of the block/allow list.

### Fix analysis

The fix:

1. Introduced a single `validateList(list, mode)` helper in `lib/utils.ts` that:
   - Splits on `,`, `;`, `\n`, `\r`, `\t`.
   - Trims, lowercases, de-duplicates via a `Set`.
   - Returns `{ all, valid, invalid }` so callers can display the bad entries.
2. Migrated every form listed above to use `validateList` (or its back-compat sibling `sanitizeList`).
3. Added a centralized `setValidationError` aggregation in `link-sheet/index.tsx` so the Save button is disabled when **any** child section has invalid entries, and a Cmd+Enter shortcut is also gated on validity.
4. Switched UI text from "entries will be ignored" to "must be fixed before saving", making the failure mode explicit.

### Verification of fix completeness

All identified list-input forms were updated in the same commit, with the `validateList` helper supporting both `email` and `domain` modes plus a `both` mode for sections that accept either. Forms now show destructive-colored invalid entry lists and disable save. The link-sheet parent gates both the form submit and the keyboard shortcut on `hasValidationErrors`.

### Verification of fix correctness

- The regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` is reasonable but lenient — it accepts `a@b.c` and `evil@a.b`. This is the **same regex as before**, so behavior on already-valid entries is unchanged. Not a regression.
- The `LIST_SEPARATOR_REGEX = /[,;\n\r\t]+/` is correct.
- Dedup is via `Set` of lower-cased trimmed entries — correct.

### Similar patterns NOT covered

- **`components/datarooms/add-viewer-modal.tsx`** still uses the old pattern (`inputValue.split(",")`). It only filters invalid emails into a separate list rather than saving them — same silent-drop UX problem, but the downstream impact is limited because:
  - These emails are used to **send invitations**, not to enforce access control.
  - The toast already informs the user via "Fix invalid emails or domains" only in the (now-fixed) sibling modals; this one still has the older "Found one or more invalid email addresses" toast.
  - Severity: **LOW** (no security impact; only UX).
- **`components/profile-menu.tsx:113`** uses `session?.user?.email?.split("@")[0]` for display — not a list input, not in scope.

### Evidence (commit diff highlights)

```diff
+const LIST_SEPARATOR_REGEX = /[,;\n\r\t]+/;
+export const validateList = (list, mode) => {
+  // returns { all, valid, invalid } — see lib/utils.ts:653+
+};
+export const sanitizeList = (list, mode) => validateList(list, mode).valid;
```

The Save-button disable in `link-sheet/index.tsx`:

```diff
-<Button type="submit" loading={isSaving} onClick={(e) => handleSubmit(e, false)}>
+<Button type="submit" loading={isSaving} disabled={hasValidationErrors}
+        onClick={(e) => handleSubmit(e, false)}>
```

---

## Finding 6: Similar-pattern (unfixed) — add-viewer-modal still uses old split

- **Severity**: LOW (UX only, not security-relevant)
- **Commit**: n/a (unfixed sibling of b68c937b)
- **Type**: similar-pattern
- **File**: `components/datarooms/add-viewer-modal.tsx`

### Description

The "Invite Visitors" modal for datarooms still uses the legacy `inputValue.split(",")` pattern (line 97) and does not use the new `validateList` helper. It silently drops invalid emails instead of prompting the user to fix them.

### Recommendation

Migrate to the shared `validateList(inputValue, "email")` helper from `lib/utils.ts` so that:

- `;` and newline-pasted emails work.
- Invalid entries surface in a destructive-colored list before save.
- Save is disabled while invalid entries exist.

This is a follow-up to b68c937b that would close out the email-list UX gap project-wide.

---

## Finding 7: Reverted-fix dance around CloudFront signed URL

- **Severity**: LOW (UX/data-exposure, not a security vulnerability)
- **Commit pair**: 81b8508e (`revert fix`) followed by 1e88cf87 (`fix: cloudfront AccessDenied for downloads with renamed filename`)
- **Type**: reverted-fix (audit trail looks alarming but is benign)
- **File**: `lib/utils/download-document.ts`

### Description

The `lib/utils/download-document.ts` file saw a revert+re-apply dance on 2026-05-25 within 33 minutes of each other:

- 20:09 UTC — `81b8508e revert fix` reverts a previous change to use `<iframe>` for cross-origin downloads, with the comment "revert fix".
- 20:42 UTC — `1e88cf87 fix: cloudfront AccessDenied for downloads with renamed filename` re-applies the iframe approach *and* fixes the underlying root cause (CloudFront canned-policy signed URLs can't handle the `''` in RFC 5987 `filename*=UTF-8''…` because the signer and the browser disagree on percent-encoding of `'`).

### Why this is not a security vulnerability

- The iframe change prevents a CloudFront AccessDenied error page from being shown to the end user when a download fails. That's a UX improvement, not a security boundary.
- The actual fix in 1e88cf87 introduces a **wildcard-resource custom policy** so the signature is over the Policy JSON, sidestepping the URL-byte mismatch. This is a correct cryptographic fix.
- No CVE-class issue here — only "users saw an ugly error page" and "we reverted too eagerly then re-applied with a better fix."

### Recommendation

None. The end state (1e88cf87) is the correct one.

---

## Finding 8: Proactive SSRF protection (informational)

- **Severity**: INFO (defensive feature, no known exploit)
- **Commit**: 7fa52b23 (`feat: security updates`)
- **Type**: feature
- **Files**: `lib/utils/ssrf-protection.ts` (501 lines), `lib/zod/url-validation.ts`

### Description

The "feat: security updates" commit added a comprehensive SSRF protection module that:

- Validates URL hostnames as public (rejects localhost, link-local, private RFC1918, IPv6 ULA, etc.).
- Resolves DNS via `node:dns/promises` to catch DNS-rebinding and hostname-as-IP tricks.
- Performs HTTPS fetches with configurable redirect limits and timeouts.
- Exposes `fetchPublicHttpsUrlToBuffer` for use by routes that need to fetch external content.

It's wired into:

- `pages/api/webhooks/services/[...path]/index.ts` (webhook handler — `webhookFileUrlSchema` enforces SSRF rules)
- `lib/zod/url-validation.ts` (`validateUrlSSRFProtection`, used by `webhookFileUrlSchema`, `notionUrlUpdateSchema`, `linkUrlUpdateSchema`, `documentUploadSchema`)

### Verification of coverage

Searched for direct `await fetch(…)` calls on user-controllable URLs:

- `pages/api/mupdf/convert-page.ts:47` — `fetch(url)` where `url` is a signed S3 URL from the caller. **Gated by `INTERNAL_API_KEY`**, so this is internal-only.
- `lib/trigger/*` background tasks — operate on signed URLs created inside the app, not user-controlled.

No additional public-facing `fetch()` of user URLs was found that bypasses the SSRF protection module.

### Recommendation

None. This is well-applied defensive code; nothing to fix.

---

## Cross-cutting observations

1. **The team's recent security cadence is healthy** — multiple VFCs in the last 30 days (`9db42913`, `b1fd5706`, `b68c937b`, `7fa52b23`, `ece358be`, `970fb4cf`) plus a proactive SSRF module.
2. **Two adjacent endpoints (`document-processing-status.ts`, `progress-token.ts`) appear to have been missed** in the recent views/route.ts authorization sweep. They're the only routes accepting a `documentVersionId` query parameter that **don't** enforce session + team membership. **This is the most actionable finding in this report.**
3. **The link↔document binding fix at 9db42913 is correctly applied to all routes in the `/api/views*` family.** I checked `views-dataroom` and `views/pages`; both fetch the version's `documentId` from the database before trusting it.
4. **`add-viewer-modal.tsx` is the only email-list input that wasn't migrated** by `b68c937b`. Low priority (UX, not security).

## PHASE_3_CHECKPOINT

- [x] Findings written to `methodology-raw/00-ai-git-history.md`
- [x] Each finding has: file, line, severity, description, evidence
- [x] No duplicate findings (Findings 2 & 3 are distinct routes; Finding 1's "similar patterns" section lists them as the natural follow-up)