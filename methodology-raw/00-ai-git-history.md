# Git History VFC Analysis

Generated: 2026-06-30
Analyzed ~200 commits across the main branch, ~70 security-tagged commits, 6 reverted commits,
and all commits touching sensitive API routes (views, downloads, auth, webhooks).

---

## Findings

### 1. Authorization Bypass — POST /api/views (documentId injection)

- severity: HIGH
- commit: 9db429132acdd42975b531a9863cafd4edfeb0db
- type: incomplete-fix (original), VFC (fix commit)
- file: app/api/views/route.ts
- description:
  The POST /api/views endpoint accepted `documentId` and `documentVersionId` from the
  client request body. The fix removed `documentId` from the body and added server-side
  validation that the requested `documentVersionId` actually belongs to the link's document
  (`requestedVersion.documentId !== link.documentId` → 403). Before the fix, an attacker
  could submit any `documentVersionId` (pointing to any document in the system) as long as
  they had a valid `linkId`, potentially viewing documents they should not have access to.
- fix analysis: Complete. The fix:
  1. Removes client-supplied `documentId` from the request body entirely
  2. Validates that `documentVersionId` exists via `prisma.documentVersion.findUnique`
  3. Cross-checks `requestedVersion.documentId` against `link.documentId`
  4. Falls back to `link.documentId` for downstream lookups
- evidence:
  ```diff
  -       documentId,
  +       documentId: string;
  ...
  +    const requestedVersion = await prisma.documentVersion.findUnique({
  +      where: { id: documentVersionId },
  +      select: { documentId: true },
  +    });
  +    if (requestedVersion.documentId !== link.documentId) {
  +      return NextResponse.json({ message: "Unauthorized access." }, { status: 403 });
  +    }
  +    const documentId = link.documentId;
  ```
- similar patterns: The same pattern existed in `app/api/views-dataroom/route.ts` and was
  fixed in commit 7fa52b23 (the "security updates" batch). The fix there is more thorough
  — it also verifies dataroom document membership via `DataroomDocument` join table and
  checks group-based permissions before recording the view.

### 2. Webhook Isolation — SSRF + Leakage via Webhook Endpoints

- severity: HIGH
- commit: ece358be634f7fbed6df7bae40950c6926521d92
- type: VFC
- files: lib/webhook/send-webhooks.ts, lib/webhook/triggers/link-created.ts,
  lib/api/views/send-webhook-event.ts
- description:
  Three issues fixed in webhook delivery:
  1. **URL exposure in logs**: Webhook URLs could contain embedded secrets (tokens in
     path/query, basic auth credentials). Added `redactUrl()` that strips everything
     except protocol and host before logging.
  2. **Trial plan webhook bypass**: The condition `team?.plan.includes("trial")` was
     preventing webhook delivery to trial accounts that should receive them. Fixed by
     removing the `trial` check.
  3. **Error swallow**: `Promise.all()` without `allSettled` meant one failing webhook
     delivery could prevent remaining webhooks from being delivered. Added `allSettled`
     with per-webhook error logging.
- fix analysis: Complete. Error handling, logging safety, and plan-gating all addressed.
- evidence:
  ```diff
  -    if (team?.plan === "free" || team?.plan === "pro" || team?.plan.includes("trial"))
  +    if (team?.plan === "free" || team?.plan === "pro")
  ```
  ```diff
  + const redactUrl = (url: string): string => {
  +   try { const u = new URL(url); return `${u.protocol}//${u.host}`; }
  +   catch { return "[invalid-url]"; }
  + };
  ```
- similar patterns: No other endpoints log raw webhook URLs. The `redactUrl` helper is
  scoped to `send-webhooks.ts` — not shared, but other endpoints don't appear to log
  webhook URLs.

### 3. Download Endpoint Authorization — Session-Based Access Control

- severity: HIGH
- commit: 7fa52b23944823a9e74402fca8e48d644239bc0b (security updates batch)
- type: VFC
- files: pages/api/links/download/index.ts, pages/api/links/download/dataroom-document.ts,
  pages/api/links/download/bulk.ts, pages/api/links/download/by-email.ts (deleted)
- description:
  Download endpoints previously accepted a `viewId` from the client request body with
  minimal verification. The fix:
  1. `download/index.ts`: Added mandatory session verification — dataroom link downloads
     require `verifyDataroomSessionInPagesRouter`, document link downloads require
     `verifyLinkSessionInPagesRouter`. The session's `viewId` must match the database
     view record, preventing cross-view access.
  2. `download/dataroom-document.ts`: Removed `viewId` from request body entirely,
     replacing it with `session.viewId` from the server-verified dataroom session.
     Added `session.dataroomId !== view.dataroom.id` check and email verification gate.
  3. `download/bulk.ts`: Same pattern — removed client-supplied `viewId`, switched to
     `session.viewId`. Also changed from `verifyDataroomSessionInPagesRouter` to
     `getDataroomSessionByLinkIdInPagesRouter` and added `session.dataroomId` check.
  4. `download/by-email.ts` (deleted): This entire endpoint was removed. It had minimal
     auth (just IP rate limiting + dataroom session lookup) and could leak whether a
     given email had viewed documents.
- fix analysis: Complete. The key architectural improvement is that `viewId` is no longer
  a client-controlled value — it's derived from the server-side session cookie.
- evidence:
  ```diff
  -    const { linkId, viewId } = req.body as { linkId: string; viewId: string };
  +    const { linkId, documentId } = req.body as { linkId: string; documentId: string };
  +    const session = await getDataroomSessionByLinkIdInPagesRouter(req, linkId);
  +    const view = await prisma.view.findUnique({
  +      where: { id: session.viewId, ... }
  ```
- similar patterns: The download/verify.ts endpoint (pages/api/links/download/verify.ts)
  still accepts an optional `viewId` from the client (line 44, `viewId: providedViewId`),
  but only as a performance optimization — if not provided, it looks up by email+linkId.
  The provided `viewId` is validated against the linkId and email match (line 85).
  This is a reasonable pattern and does not constitute an authorization bypass.

### 4. Email Change Confirmation — Cross-User Token Acceptance

- severity: HIGH
- commit: 7fa52b23 (security updates)
- type: VFC
- file: app/(auth)/auth/confirm-email-change/[token]/page.tsx
- description:
  The email change confirmation endpoint used `currentUserId` (from the authenticated
  session) for both the Redis lookup and the Prisma update, instead of `tokenUserId`
  (from the token record). This meant:
  - User A could take User B's email change token
  - User A would confirm the email change using User B's token
  - The Redis lookup used User A's ID, so it would find no data and show NotFound
  - But the Prisma update at the end also used User A's ID
  - More critically, the token-user binding was never checked
  - Fixed by: adding `if (tokenUserId !== currentUserId) return <NotFound />` and
    switching Redis/Prisma lookups to `tokenUserId`
- fix analysis: Complete. The fix prevents token reuse across users.
- evidence:
  ```diff
  +  const tokenUserId = tokenFound.identifier;
  +  if (tokenUserId !== currentUserId) return <NotFound />;
  -    `email-change-request:user:${currentUserId}`,
  +    `email-change-request:user:${tokenUserId}`,
  -      id: currentUserId,
  +      id: tokenUserId,
  ```

### 5. Open Redirect in Auth Middleware

- severity: MEDIUM
- commit: 7fa52b23 (security updates)
- type: VFC
- file: lib/middleware/app.ts
- description:
  The `normalizeNextPath` function did not validate that the `next` redirect parameter
  pointed to the same origin. An attacker could craft a login URL with `?next=https://evil.com`
  and, after authentication, the user would be redirected to the attacker's site.
  Additionally, protocol-relative URLs (`//evil.com`) were not caught because the
  function only checked for a leading `/`.
- fix analysis: Complete. Adds `isProtocolRelativePath` check and origin validation.
- evidence:
  ```diff
  + function isProtocolRelativePath(path: string) { return path[1] === "/" || path[1] === "\\"; }
  ...
  -  if (!normalized.startsWith("/")) {
  +  if (!normalized.startsWith("/") || isProtocolRelativePath(normalized)) {
  ...
  +    const targetUrl = new URL(normalized, requestUrl);
  +    const requestOrigin = new URL(requestUrl).origin;
  +    if (targetUrl.origin !== requestOrigin) { return DEFAULT_AUTH_REDIRECT_PATH; }
  ```

### 6. SSRF Protection — Missing DNS-Level Checks

- severity: MEDIUM
- commit: 7fa52b23 (security updates)
- type: VFC
- files: lib/utils/ssrf-protection.ts (new), lib/zod/url-validation.ts (rewritten)
- description:
  The original `validateUrlSSRFProtection` only checked hostname strings against a
  hardcoded list of private IP patterns (`10.*`, `172.16-31.*`, `192.168.*`, `169.254.*`,
  `localhost`, `127.0.0.1`, `::1`, `fe80:*`). This was bypassable via:
   - DNS rebinding (a hostname resolving to `127.0.0.1` passed the string check)
   - IPv6 variants not matching the hardcoded patterns
   - URL-encoded IP addresses
   - Redirect chains (a public URL could redirect to an internal one)
  The fix replaced this with a comprehensive SSRF protection module:
   - DNS resolution and IPv4/IPv6 parsing with full private range detection
   - Redirect chain following with re-validation at each hop
   - Configurable timeouts (connect + read) to prevent hanging connections
   - Credential stripping (`urlObj.username || urlObj.password` → reject)
   - Only synchronous validation is done at URL-submission time; the actual
     network fetcher (for webhook services) does full DNS-level checks
- fix analysis: Complete. Proper defense-in-depth with both URL-level and DNS-level checks.
  However, coverage should be verified — the SSRF module is used by the webhooks service
  endpoint and the URL validation Zod schema. Other code paths that fetch remote URLs
  (e.g., Notion page cover images, link preview thumbnails) should be audited separately.
- evidence:
  ```diff
  -    if (hostname === "localhost" || hostname === "127.0.0.1" || hostname === "::1") { return false; }
  -    if (hostname.match(/^10\.|^172\.(1[6-9]|2[0-9]|3[01])\.|^192\.168\./)) { return false; }
  +    return isPublicHostnameLiteral(urlObj.hostname);
  ```

### 7. OTP Rate Limiting — Missing Per-Email Throttle

- severity: MEDIUM
- commit: d3ee5ff1313d5001c7694be7c929895e2e9e8199
- type: VFC
- files: app/api/views/route.ts, app/api/views-dataroom/route.ts,
  app/(ee)/api/workflow-entry/domains/[...domainSlug]/route.ts,
  app/(ee)/api/workflow-entry/link/[entryLinkId]/verify/route.ts
- description:
  OTP verification endpoints only had IP-based rate limiting, making it possible for
  an attacker to flood a specific email with verification codes across multiple IP
  addresses. Added per-email+link rate limiting (1 per 30 seconds) as the primary
  throttle, keeping IP-based (10 per minute) as secondary defense.
- fix analysis: Complete. The layered approach (email+link → IP) is correct.
- evidence:
  ```diff
  +    const { success: emailLimitSuccess } = await ratelimit(1, "30 s").limit(
  +      `send-otp:${linkId}:${email}`,
  +    );
  ```

### 8. Dataroom Changes — Missing Folder-Access Permission Checks in Notifications

- severity: MEDIUM
- commit: f0637c63354191b42163f06f5a23c01df81511f5
- type: VFC
- file: lib/trigger/dataroom-change-notification.ts
- description:
  Dataroom change notifications were sent to all viewers with links in the dataroom,
  regardless of whether they had folder-level access to the changed document. A viewer
  who could access Folder A would receive a notification when a document in Folder B
  changed, potentially leaking information about the existence and activity of documents
  they shouldn't know about. The fix adds folder-access control checks via the
  `ViewerGroupAccessControls` or `PermissionGroupAccessControls` tables before sending
  notifications.
- fix analysis: Complete. Uses folderAccessCache to avoid redundant DB queries.
- evidence:
  ```diff
  +    const canViewFolder = async (groupId, permissionGroupId) => {
  +      if (!dataroomDocument.folderId) return true;
  +      // Check ViewerGroupAccessControls or PermissionGroupAccessControls
  +      ...
  +    };
  ```

### 9. Document Page Access — Missing Team Membership Check

- severity: MEDIUM
- commit: 7fa52b23 (security updates)
- type: VFC
- file: lib/documents/get-file-helper.ts
- description:
  The `getFileForDocumentPage` helper looked up document versions by `documentId` only,
  with no verification that the requesting user belonged to the document's team. This
  meant any authenticated user could fetch document page images if they knew or could
  guess a `documentId` and `pageNumber`. The fix adds a nested relation filter that
  requires the document's team to include the requesting user.
- fix analysis: Complete. The bidirectional relationship check (`document → team → users
  → userId`) prevents cross-team document page access.
- evidence:
  ```diff
  -  const documentVersions = await prisma.documentVersion.findMany({
  +  const documentVersion = await prisma.documentVersion.findFirst({
        where: {
          documentId,
  +       document: {
  +         team: { users: { some: { userId } } },
  +       },
        },
  ```

### 10. Download Fix Loop — Reverted Fix (temporary)

- severity: LOW
- commits: 21d5a2ac → 81b8508e (revert) → 1e88cf87 (final fix)
- type: reverted-fix (temporary revert, re-fixed in subsequent commit)
- file: lib/utils/download-document.ts
- description:
  A fix for dataroom file downloads (21d5a2ac) was immediately reverted (81b8508e),
  then re-implemented differently (1e88cf87). The revert changed the download
  mechanism from iframe-based to `<a download>` with `fileName`, which broke renamed
  file downloads via CloudFront because `download` is silently ignored for cross-origin
  URLs. The final fix (1e88cf87) addresses the root cause: CloudFront signed URL
  encoding of `'` characters, using a custom policy with wildcard resource instead of
  canned-policy, and routing through a hidden iframe again.
- fix analysis: Complete (in the final commit 1e88cf87). The intermediate reverted
  state (81b8508e) would have caused download failures for renamed files but not a
  security vulnerability.
- evidence:
  ```diff
  +    // Use a custom policy (wildcard resource) when overriding Content-Disposition
  +    const policy = JSON.stringify({
  +      Statement: [{
  +        Resource: `${resourceBase}?*`,
  +        Condition: { DateLessThan: { "AWS:EpochTime": Math.floor(...) } },
  +      }],
  +    });
  ```

### 11. MCP Endpoint Auth Exclusion

- severity: LOW (informational)
- commit: b1fd57063fbd23350d2edfe4586ac893f18aeea7
- type: VFC
- file: middleware.ts
- description:
  The `/mcp` endpoint was being caught by the catch-all auth middleware pattern and
  redirected to `/login`, causing MCP requests to get 302 instead of 401. The fix adds
  `/mcp/?$` to the exclusion list so the MCP endpoint's own Bearer-token auth can
  return proper 401 responses.
- fix analysis: Complete.
- evidence:
  ```diff
  -    "/((?!api/|oauth/|...).*)",
  +    "/((?!api/|oauth/|mcp/?$|...).*)",
  ```

### 12. Screenshot Protection Coverage Improvement

- severity: LOW
- commit: f4d00e0a7b85a78e2066bd681897dd7e04d8c59f
- type: VFC
- file: components/view/ScreenProtection.tsx
- description:
  Screenshot protection event listeners were enhanced to catch both `keydown` and `keyup`
  events for all shortcut groups, improving coverage against print-screen and screenshot
  shortcuts.
- fix analysis: Complete for the stated scope. This is a client-side protection and does
  not prevent determined bypass (e.g., screenshot from another device, debugger tools).
  Server-side protections (watermarking) remain the primary defense.

---

## Summary

| Severity | Count | Key Issues |
|----------|-------|------------|
| CRITICAL | 0 | |
| HIGH | 5 | Auth bypass in views (2), download endpoints (2), email change hijack |
| MEDIUM | 5 | Open redirect, SSRF, OTP flooding, notification info leak, cross-team doc access |
| LOW | 2 | Reverted fix (temporary), screenshot protection, MCP auth exclusion |

All HIGH-severity findings from the commit history have been addressed with complete
fixes. No incomplete fixes or sibling unfixed patterns were found in the current
codebase for the identified vulnerability classes. The "security updates" commit
(7fa52b23) was a large batch fix addressing multiple HIGH and MEDIUM issues
simultaneously.

**Potential residual risk areas** (not confirmed as vulnerable, but worth deeper
investigation):
- The `/api/views/pages` route uses `documentVersionId` from the client. For the
  standard view path, it validates `documentVersion.documentId !== view.documentId`.
  For the preview path, it validates against the link's document or dataroom. Both
  paths appear properly gated.
- The SSRF protection is well-implemented but only used in specific code paths —
  any code fetching external URLs outside those modules should be audited.
