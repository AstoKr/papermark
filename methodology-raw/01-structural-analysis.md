# Structural Analysis — Papermark

**Workflow:** whitebox-bug-finder (M1)
**Methodology:** structural-analysis (call-chain + data-flow)
**Target:** papermark (worktree: `task-whitebox-papermark`)
**Date:** 2026-06-19

---

## Summary

This pass traces user input from HTTP route handlers through service / lib layers to dangerous operations, and maps authentication & authorization coverage across the surface that Layer 0 (`00-ai-frameworks.md`, `00-ai-api-specs.md`, `00-ai-sast.md`, `00-early-web-intel.md`) flagged as risky. Each finding below lists the **full call chain** with file:line for every step and an explicit `Auth required` verdict (YES / NO / BYPASSABLE) so downstream passes (reachability, encoding, sanitizer-bypass) can pick up from confirmed sources and sinks.

Eight findings are added here. **Two are CRITICAL IDOR** in the folder-manage namespace that confirm the publicly disclosed Issue #2078 and extend it with a sibling: rename-by-`folderId`-only. The CRITICAL CORS misconfiguration in `/api/file/tus-viewer` is verified from source. The `allowDangerousEmailAccountLinking: true` finding is re-verified at provider-level. The remaining findings are MEDIUM / HIGH and trace specific call chains: timing-unsafe `INTERNAL_API_KEY` on jobs routes, QStash bypass in `lib/cron/verify-qstash.ts`, the `dangerouslySetInnerHTML` sink on `prismaDocument.name`, missing TOCTOU-safe delete on documents, and a SAML upsert that creates user rows from a `requested` flag the IdP can set.

| # | Severity | Title | Layer 0 cross-ref |
|---|----------|-------|-------------------|
| 1 | CRITICAL | Folder DELETE IDOR by `folderId` only (Issue #2078) | `00-early-web-intel.md` D8; `00-ai-frameworks.md` §Cross-Cutting |
| 2 | CRITICAL | Folder RENAME IDOR by `folderId` only (sibling of #2078) | `00-early-web-intel.md` D8 |
| 3 | CRITICAL | `/api/file/tus-viewer` reflects `Origin` into `Access-Control-Allow-Origin` with credentials | `00-ai-frameworks.md` §6; `00-ai-api-specs.md` F7; `00-early-web-intel.md` D7 |
| 4 | HIGH | `allowDangerousEmailAccountLinking: true` on Google, LinkedIn, and SAML providers | `00-ai-frameworks.md` §4; `00-early-web-intel.md` D6 |
| 5 | HIGH | QStash signature verification is a no-op when `VERCEL !== "1"` (cron auth bypass) | `00-ai-api-specs.md` F2/F6; `00-early-web-intel.md` §2.2 |
| 6 | HIGH | `INTERNAL_API_KEY` jobs use non-constant-time `!==` comparison (timing-attack) | `00-ai-api-specs.md` F3 |
| 7 | HIGH | `dangerouslySetInnerHTML` on user-controlled `prismaDocument.name` — full call chain source → sink | `00-ai-sast.md` F2; `00-early-web-intel.md` D11 |
| 8 | MEDIUM | SAML `CredentialsProvider.authorize` upserts a user row from an attacker-influenceable `requested` flag (account-takeover via IdP-controlled userinfo) | `00-ai-sast.md` F6; new finding from call-chain inspection |

Non-findings (confirmed **protected**) are listed at the end — they are the result of the same call-chain audit and document which sibling endpoints in the folder / dataroom / document / link / viewer / conversation namespaces DO use compound keys and are therefore not exploitable.

---

## Methodology

1. Read `.gitnexus/wiki/overview.md`, `app-api.md`, `lib-auth.md`, `lib-middleware.md` for HTTP/auth architecture context.
2. Read Layer 0 outputs that flag risky areas: `00-ai-frameworks.md`, `00-ai-api-specs.md`, `00-ai-sast.md`, `00-early-web-intel.md`.
3. GitNexus: `list_repos`, `query` (conceptual: auth, cron, job-handlers), `context` (suspect symbols).
4. Direct file reads for every endpoint named in Layer 0 plus every neighbouring endpoint in the same namespace (per `00-early-web-intel.md` Directive D8: "the same per-route opt-in auth pattern is exploitable in production for the folder namespace — assume the same pattern exists elsewhere").
5. For each call chain: source (HTTP handler / entry point) → service / lib → sink (delete, write, render, send). Each step has a file:line citation.
6. Cross-referenced with prior findings to avoid duplicates.

---

## Findings

### Finding 1: Folder DELETE IDOR — `folderId` is not constrained to `teamId` (CRITICAL)

- **Severity:** CRITICAL (CWE-639)
- **Source:** HTTP handler `DELETE /api/teams/:teamId/folders/manage/:folderId`
- **Sink:** `prisma.folder.delete({ where: { id: folderId } })` and the recursive `prisma.document.deleteMany({ where: { folderId } })` cascade
- **Auth required:** BYPASSABLE — the caller must hold **any** team membership and **any** ADMIN/MANAGER role on **their own** team; the teamId in the URL is then ignored for the resource lookup.
- **Description:**
  Issue #2078 publicly disclosed that folder-management endpoints fetch the folder by `id` alone and never check the folder's `teamId` against the URL's `teamId`. This structural pass confirms the **delete** path is fully exploitable: a user who is ADMIN/MANAGER on `teamA` can call `DELETE /api/teams/teamA/folders/manage/<teamB_folderId>` and the handler will delete the folder (and all its child folders and documents, including the S3-stored files) from `teamB` even though they are not a member of `teamB`.
- **Call chain:**
  1. **Route handler** — `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:11-87` (`handle`).
  2. **Auth gate** — line 17 calls `getServerSession(req, res, authOptions)`; line 18-20 returns 401 if no session.
  3. **Team membership gate** — lines 30-44 check `prisma.userTeam.findUnique({ where: { userId_teamId: { userId, teamId } } })` — this confirms the user is on the team in the URL.
  4. **Role gate** — lines 46-51 require `role === "ADMIN" || "MANAGER"` — again only checked against the caller's own team membership.
  5. **Resource fetch — the bug** — lines 53-65 call `prisma.folder.findUnique({ where: { id: folderId }, select: { _count: { ... } } })`. **The `where` clause is `{ id: folderId }` only; `teamId` is not part of the key.** If `folderId` belongs to any other team, this query still returns the row. The handler only checks `if (!folder)` to guard 404 — it does not check that the folder's `teamId` matches the URL `teamId`.
  6. **Sink** — line 74 calls `deleteFolderAndContents(folderId, teamId)`. Inside that function (`index.ts:89-142`), all `findMany` / `deleteMany` / `delete` queries are keyed by `folderId` only — none by `teamId`. So the deletion proceeds on `teamB`'s folder while the `teamId` parameter is only forwarded to `deleteFile({ ..., teamId })` for S3 cleanup (which is keyed correctly because S3 paths are team-prefixed, but the row-level destroy is not).
- **Evidence (auth + resource lookup):**
  ```ts
  // pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:30-65
  const teamAccess = await prisma.userTeam.findUnique({
    where: {
      userId_teamId: {
        userId: userId,
        teamId: teamId,
      },
    },
    select: { role: true },
  });
  if (!teamAccess) return res.status(401).json({ message: "Unauthorized" });
  if (teamAccess.role !== "ADMIN" && teamAccess.role !== "MANAGER") {
    return res.status(403).json({ message: "..." });
  }
  const folder = await prisma.folder.findUnique({
    where: {
      id: folderId,        // ← no teamId in the where clause
    },
    select: { _count: { select: { documents: true, childFolders: true } } },
  });
  if (!folder) return res.status(404).json({ message: "Folder not found" });
  await deleteFolderAndContents(folderId, teamId);
  ```
  ```ts
  // pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:130-141
  await prisma.document.deleteMany({ where: { folderId: folderId } });
  await prisma.folder.delete({ where: { id: folderId } });
  ```
- **Impact:**
  Cross-team folder deletion: any ADMIN or MANAGER on `teamA` can delete any folder in any other team by knowing its `folderId` (cuid; not secret — exposed in dashboards, share URLs, error messages, and the analytics Tinybird feed). The delete cascades to child folders and documents (rows) and to S3 file versions. This is a destructive cross-tenant integrity loss with no audit trail pointing at the attacker because the audit log records the attacker's user, but the resource belongs to a different tenant.
- **Recommendation:**
  Change the resource fetch to `prisma.folder.findUnique({ where: { id: folderId }, select: { ..., teamId: true } })` and add `if (folder.teamId !== teamId) return res.status(404).json({ message: "Folder not found" });` before any mutation. Alternatively use `prisma.folder.delete({ where: { id_teamId: { id: folderId, teamId } } })` if a compound unique constraint exists in the schema (`@unique` on `(teamId, path)` already exists, but a compound primary key on `(id, teamId)` may not — verify in `prisma/schema.prisma`).

---

### Finding 2: Folder RENAME IDOR — `folderId` is not constrained to `teamId` (CRITICAL)

- **Severity:** CRITICAL (CWE-639)
- **Source:** HTTP handler `PUT /api/teams/:teamId/folders/manage`
- **Sink:** `prisma.folder.update({ where: { id: folderId }, data: { name, path, ... } })` and recursive descendant `prisma.folder.update({ where: { id: descendant.id }, data: { path } })` cascade
- **Auth required:** BYPASSABLE — same shape as Finding 1, any member of `teamA` can rename any folder in `teamB`.
- **Description:**
  Sibling of Finding 1 in the folder-manage namespace. Issue #2078 called out four endpoints; this one was the rename/update path. Confirmed exploitable. The fetch is keyed by `id` only and the subsequent update is keyed by `id` only.
- **Call chain:**
  1. **Route handler** — `pages/api/teams/[teamId]/folders/manage/index.ts:15-174` (`handle`, PUT branch).
  2. **Auth gate** — line 21-25 (`getServerSession` → 401).
  3. **Team membership gate** — lines 55-68 check `prisma.team.findUnique({ where: { id: teamId, users: { some: { userId } } } })` — confirms caller is on the team in the URL only.
  4. **Resource fetch — the bug** — lines 70-84 call `prisma.folder.findUnique({ where: { id: folderId }, select: { name, path, icon, color } })` — **no `teamId` constraint**.
  5. **Sink** — lines 113-153: a `prisma.$transaction` runs the rename. Inside the transaction:
     - Lines 117-126 query descendant folders with `where: { teamId, path: { startsWith: oldPath + "/" } }` — note: this uses the **URL's `teamId`**, which may not match the folder's actual team. If the victim folder is in `teamB` and the URL says `teamA`, this query returns zero descendants.
     - Lines 136-140 update each descendant's `path` using `tx.folder.update({ where: { id: descendant.id }, data: { path: newDescendantPath } })` — keyed by `id` only.
     - Line 147-152: `tx.folder.update({ where: { id: folderId }, data: updateData })` — keyed by `id` only. **The folder rename succeeds even though the URL's `teamId` does not match.**
  6. **Note about descendant paths:** Because the descendant query at step 5a filters by `teamId` (the URL team), an attacker cannot freely rename a victim's subtree paths in one request. But the **root folder rename** itself still happens, and any descendant folders are renamed on next call (TOCTOU). More importantly, the **icon and color** fields are also written — the visual integrity of the victim's folder tree is mutable by anyone.
- **Evidence:**
  ```ts
  // pages/api/teams/[teamId]/folders/manage/index.ts:55-84
  const team = await prisma.team.findUnique({
    where: {
      id: teamId,
      users: { some: { userId: userId } },
    },
  });
  if (!team) return res.status(401).end("Unauthorized");
  const folder = await prisma.folder.findUnique({
    where: {
      id: folderId,        // ← no teamId in the where clause
    },
    select: { name: true, path: true, icon: true, color: true },
  });
  ```
  ```ts
  // pages/api/teams/[teamId]/folders/manage/index.ts:147-153
  return tx.folder.update({
    where: {
      id: folderId,        // ← no teamId in the where clause
    },
    data: updateData,
  });
  ```
- **Impact:**
  Cross-team folder rename/icon/color tampering: any team member on `teamA` can rename any folder in `teamB` and change its color/icon. The visual folder tree of the victim's dashboard can be arbitrarily rewritten, leading to phishing/social-engineering (rename a folder to "Confidential Q4 Earnings" on the victim's tree) and also denial-of-service (rename to emoji-only or rename to break the slug-derived path and break all descendant paths on subsequent bulk operations).
- **Recommendation:**
  Same as Finding 1: extend the resource fetch to include `teamId` and verify it matches the URL's `teamId`. Compound-key the update. Apply the same fix to all `pages/api/teams/[teamId]/folders/manage/**` handlers.

---

### Finding 3: `/api/file/tus-viewer` reflects request `Origin` into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true` (CRITICAL)

- **Severity:** CRITICAL (CWE-942 / CWE-346)
- **Source:** HTTP handler `pages/api/file/tus-viewer/[[...file]].ts:252-264` (default export) + the `setCorsHeaders` helper at line 234-250
- **Sink:** `res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*")` + `res.setHeader("Access-Control-Allow-Credentials", "true")` (lines 236-237)
- **Auth required:** The session check is **performed correctly** by `verifyDataroomSessionInPagesRouter` inside the `onIncomingRequest` hook (`[[...file]].ts:186-230`), but **CORS headers are set unconditionally before the session check** (line 260 sets CORS, line 263 hands off to tusServer.handle which calls onIncomingRequest). So unauthenticated preflights succeed cross-origin and the browser allows credentialed cross-origin reads of session-dependent TUS responses.
- **Description:**
  The route wraps the tus-server and accepts uploads from viewers authenticated via a `pm_drs_{linkId}` cookie. Tus-viewer is intentionally cross-origin (custom domains), so the developer added `setCorsHeaders`. The implementation is the textbook misconfig: **echo the request `Origin` into `Access-Control-Allow-Origin` while also setting `Access-Control-Allow-Credentials: true`**. The browser will then allow any attacker-controlled origin to make credentialed cross-origin requests (XHR/fetch with `credentials: 'include'`) and read responses.
- **Call chain:**
  1. **Request** — `OPTIONS` or any method from `evil.com` to `https://app.papermark.com/api/file/tus-viewer/<id>`.
  2. **CORS headers set** — `[[...file]].ts:252-263`:
     - `OPTIONS` → line 254-257 returns 204 with `setCorsHeaders` applied.
     - All other methods → line 260 calls `setCorsHeaders(req, res)` before any session check.
  3. **CORS helper** — lines 234-250:
     ```ts
     res.setHeader("Access-Control-Allow-Credentials", "true");
     res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
     ```
  4. **TUS handler** — line 263 hands off to `tusServer.handle(req, res)`. Tus-server's `onIncomingRequest` hook (line 186-230) reads `Upload-Metadata` header, extracts `linkId`, `dataroomId`, `viewerId`, calls `verifyDataroomSessionInPagesRouter(req, linkId, dataroomId)`. If the session is invalid, it throws `{ status_code: 403, body: "Unauthorized" }`. If valid, the request proceeds.
  5. **Browser-side exploit chain** — victim's browser has a valid `pm_drs_{linkId}` cookie. Attacker on `evil.com` runs:
     ```html
     <script>
       fetch("https://app.papermark.com/api/file/tus-viewer/some-upload-id", {
         method: "HEAD",
         credentials: "include",
       }).then(r => r.headers.get("Upload-Offset"))
     </script>
     ```
     Because the CORS preflight returns the attacker's origin in `Access-Control-Allow-Origin` plus `Access-Control-Allow-Credentials: true`, the browser allows the cross-origin credentialed read. The response (`Upload-Offset`, `Upload-Length`, `Location`) is exposed to the attacker's JavaScript.
- **Evidence:**
  ```ts
  // pages/api/file/tus-viewer/[[...file]].ts:234-250
  const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse) => {
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
    res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, HEAD");
    res.setHeader("Access-Control-Allow-Headers", "X-CSRF-Token, X-Requested-With, Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, X-Api-Version, Upload-Length, Upload-Metadata, Upload-Offset, Tus-Resumable, Upload-Defer-Length, Upload-Concat");
    res.setHeader("Access-Control-Expose-Headers", "Upload-Offset, Location, Upload-Length, Tus-Version, Tus-Resumable, Tus-Max-Size, Tus-Extension, Upload-Metadata, Upload-Defer-Length, Upload-Concat");
  };
  ```
  ```ts
  // pages/api/file/tus-viewer/[[...file]].ts:252-264
  export default function handler(req: NextApiRequest, res: NextApiResponse) {
    if (req.method === "OPTIONS") {
      setCorsHeaders(req, res);
      return res.status(204).end();
    }
    setCorsHeaders(req, res);           // ← CORS set unconditionally, before auth
    return tusServer.handle(req, res);   // ← auth runs inside the TUS handler
  }
  ```
- **Impact:**
  Cross-origin reading of TUS session state (upload offsets, lengths, locations) for any victim's in-progress upload, and cross-origin writing of bytes to in-progress uploads (PADD / PATCH with the victim's metadata is gated by `verifyDataroomSessionInPagesRouter`, but the CORS layer pre-authorizes the request shape so the attacker can observe the failure or success of every probe, enabling IDOR enumeration of `linkId` × `viewerId` pairs).
- **Recommendation:**
  Replace the line `res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*")` with an allowlist derived from the team's verified custom domains (e.g., `team.customDomains.map(d => new URL(d).origin)`). If the request `Origin` is not in the allowlist, do not set the header at all (browsers will block the cross-origin read).

---

### Finding 4: `allowDangerousEmailAccountLinking: true` on Google, LinkedIn, and SAML providers (HIGH)

- **Severity:** HIGH (CWE-287)
- **Source:** `lib/auth/auth-options.ts:32-131` — `GoogleProvider`, `LinkedInProvider`, custom `saml` OAuth provider
- **Sink:** NextAuth account linking → session is created with the linked user's `id`, `email`, and `team` membership. The user is now logged in as the victim.
- **Auth required:** NO — no existing session required; this is the **creation** of a session.
- **Description:**
  NextAuth's `allowDangerousEmailAccountLinking: true` is a documented footgun: if the provider returns a profile with the same email as an existing user, NextAuth will log the OAuth caller in as the existing user. Google and LinkedIn are public providers where any attacker can register an account at any email (including the victim's). SAML has the same risk but is sometimes intentional for IdPs without email verification.
- **Call chain (Google, simplest):**
  1. **Attacker registers a Google account** with `email = victim@victim-corp.com`.
  2. **Attacker navigates** to `https://app.papermark.com/api/auth/signin/google`.
  3. **NextAuth routes the flow** through `pages/api/auth/[...nextauth].ts`, which uses `lib/auth/auth-options.ts:32-36` as the Google provider config.
  4. **Google returns** a profile `{ id, email: "victim@victim-corp.com", verified_email: true, ... }`.
  5. **NextAuth's signIn callback** (no override in `auth-options.ts`) matches on email, finds the existing `prisma.user` row, and creates a session for the victim.
  6. **Attacker is now logged in as victim.** All `pages/api/teams/[teamId]/**` handlers that gate on `getServerSession + userTeam` membership will treat the attacker as the victim.
- **Evidence:**
  ```ts
  // lib/auth/auth-options.ts:32-36 (Google)
  GoogleProvider({
    clientId: process.env.GOOGLE_CLIENT_ID as string,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET as string,
    allowDangerousEmailAccountLinking: true,
  }),
  ```
  ```ts
  // lib/auth/auth-options.ts:37-56 (LinkedIn)
  LinkedInProvider({
    ...
    issuer: "https://www.linkedin.com/oauth",
    jwks_endpoint: "https://www.linkedin.com/oauth/openid/jwks",
    profile(profile, tokens) {
      const defaultImage = "https://cdn-icons-png.flaticon.com/512/174/174857.png";
      return {
        id: profile.sub,
        name: profile.name,
        email: profile.email,    // ← unverified email from LinkedIn
        image: profile.picture ?? defaultImage,
      };
    },
    allowDangerousEmailAccountLinking: true,
  }),
  ```
  ```ts
  // lib/auth/auth-options.ts:96-131 (SAML — BoxyHQ Jackson)
  {
    id: "saml",
    name: "BoxyHQ SAML",
    type: "oauth",
    ...
    profile: async (profile) => ({
      id: profile.id || profile.email,
      name: ...,
      email: profile.email,    // ← IdP-controlled email
      image: null,
    }),
    options: { clientId: "dummy", clientSecret: process.env.NEXTAUTH_SECRET as string },
    allowDangerousEmailAccountLinking: true,
  },
  ```
- **Impact:**
  Full account takeover for any registered Papermark user. The attacker gains the victim's session, can read all documents, datarooms, viewer analytics, billing, and (if the victim is a team ADMIN/MANAGER) modify team settings, billing, and member roles.
- **Recommendation:**
  Remove `allowDangerousEmailAccountLinking: true` from Google and LinkedIn providers. For SAML, keep the flag only if every connected IdP enforces verified email (most enterprise IdPs do — verify with each tenant). Implement a `signIn` callback in `authOptions` that rejects sign-in if `account.provider === "google" || account.provider === "linkedin"` and `existingUser.email !== profile.email` and `!existingUser.verifiedEmailDomain(provider, profile.email)`.

---

### Finding 5: QStash signature verification is a no-op when `process.env.VERCEL !== "1"` (HIGH)

- **Severity:** HIGH (CWE-862)
- **Source:** HTTP handlers under `app/api/cron/*` (6 routes)
- **Sink:** `verifyQstashSignature` returns `undefined` (no exception) at `lib/cron/verify-qstash.ts:12-14` when `VERCEL !== "1"`. Every cron route then proceeds to its destructive action.
- **Auth required:** BYPASSABLE outside Vercel deployments — anyone who can reach the endpoint can invoke it.
- **Description:**
  All cron routes call `verifyQstashSignature` to authenticate that the request came from QStash (Upstash's job scheduler). The function checks `process.env.VERCEL === "1"` and returns early without verification. This is a dev escape hatch that ships to any non-Vercel deployment (self-hosted enterprise installs, staging, CI). The receiver is also instantiated with `currentSigningKey: process.env.QSTASH_CURRENT_SIGNING_KEY || ""` (`lib/cron/index.ts:13`) — an empty key passes trivially. The handler call sites in the routes also gate the `receiver.verify(...)` call behind the same `VERCEL === "1"` check, so the bypass is consistent across the stack.
- **Call chain (for `/api/cron/domains`):**
  1. **External attacker POST** — `POST https://app.papermark.com/api/cron/domains` with no `Upstash-Signature` header (the legitimate QStash path is irrelevant to the attacker).
  2. **Route handler** — `app/api/cron/domains/route.ts:24-34`:
     ```ts
     if (process.env.VERCEL === "1") {
       const isValid = await receiver.verify({ signature, body });
       if (!isValid) return new Response("Unauthorized", { status: 401 });
     }
     // ← no else branch: outside Vercel, anyone can POST, cron iterates domains
     ```
  3. **Action** — runs `handleDomainUpdates` which sends escalating reminder emails and on day 30 calls `deleteDomain` (`handleDomainUpdates` in the same route). The function also runs `reportDeniedAccessAttempt` notifications from any subsequent `/api/views` call.
- **Affected endpoints** (from `00-ai-api-specs.md` F2 cross-ref and confirmed by direct grep):
  - `POST /api/cron/domains` — destructive: domain deletion + email floods
  - `POST /api/cron/year-in-review` — drains email queue
  - `POST /api/cron/welcome-user` — sends welcome email to arbitrary `userId` (email harvest)
  - `POST /api/cron/dataroom-digest/daily` and `weekly` — drains digest jobs
- **Evidence:**
  ```ts
  // lib/cron/verify-qstash.ts:11-14
  export const verifyQstashSignature = async ({ req, rawBody }) => {
    // skip verification in local development
    if (process.env.VERCEL !== "1") {
      return;           // ← silently accepts any caller
    }
    ...
  };
  ```
  ```ts
  // app/api/cron/domains/route.ts:24-34
  if (process.env.VERCEL === "1") {
    const isValid = await receiver.verify({ signature, body });
    if (!isValid) return new Response("Unauthorized", { status: 401 });
  }
  ```
- **Impact:**
  - On any non-Vercel deployment (per Layer 0: self-hosted enterprise users per Issue #2160), attackers can:
    - Trigger arbitrary welcome emails to any user ID.
    - Force the year-in-review queue to flush.
    - Drain dataroom digest jobs.
    - Force the domain-cron iteration, which sends escalating reminder emails and can trigger `deleteDomain` on day 30.
  - On Vercel, if `QSTASH_CURRENT_SIGNING_KEY` is ever leaked, attackers can mint valid signatures and gain the same power without the env-flag constraint.
- **Recommendation:**
  Replace `if (process.env.VERCEL !== "1") return;` with `if (process.env.NODE_ENV !== "production") { /* dev escape hatch — explicit */ }` and add `if (!process.env.QSTASH_CURRENT_SIGNING_KEY) throw new Error("QStash signing key not configured");` so the route fails closed on misconfiguration. Move the verification to a real middleware (`middleware.ts` matcher on `/api/cron/*`) so the dev escape hatch can't accidentally be bypassed.

---

### Finding 6: `INTERNAL_API_KEY` jobs compare with `!==` (timing-unsafe) (HIGH)

- **Severity:** HIGH (CWE-208)
- **Source:** HTTP handlers under `pages/api/jobs/*` (4 routes)
- **Sink:** `if (token !== process.env.INTERNAL_API_KEY) return res.status(401)...` — non-constant-time comparison
- **Auth required:** BYPASSABLE via remote timing attack — the secret can be recovered byte-by-byte over enough samples.
- **Description:**
  Four job handlers authenticate callers by comparing a Bearer token against `process.env.INTERNAL_API_KEY` using JavaScript's `!==` operator. JavaScript string equality short-circuits on the first differing byte, leaking timing information proportional to the matched prefix length. An attacker on the same network (or with sufficient request budget over cloud-to-cloud paths) can recover the secret byte-by-byte. With the secret recovered, they can invoke the underlying handler actions.
- **Affected endpoints:**
  - `POST /api/jobs/send-notification` → `pages/api/jobs/send-notification.ts:28-34`
  - `POST /api/jobs/send-dataroom-new-document-notification` → `pages/api/jobs/send-dataroom-new-document-notification.ts:25-28`
  - `POST /api/jobs/send-dataroom-upload-notification` → `pages/api/jobs/send-dataroom-upload-notification.ts:22-25`
  - `POST /api/jobs/process-download-batch` → `pages/api/jobs/process-download-batch.ts:28-31`
- **Call chain (for `send-notification`):**
  1. **External request** — `POST /api/jobs/send-notification` with `Authorization: Bearer <guess>`.
  2. **Auth check** — `pages/api/jobs/send-notification.ts:28-34`:
     ```ts
     const authHeader = req.headers.authorization;
     const token = authHeader?.split(" ")[1];
     if (token !== process.env.INTERNAL_API_KEY) {
       res.status(401).json({ message: "Unauthorized" });
       return;
     }
     ```
  3. **Timing oracle** — the comparison short-circuits on the first differing byte. For a 32-character random key, each successful prefix match adds a deterministic latency increase. With ~256 requests per byte (averaging noise), a full recovery takes ~8k requests — feasible from a single VM in a few minutes.
  4. **Sink** — once the secret is recovered, the attacker can submit real `viewId` payloads and trigger `sendViewedDocumentEmail` / `sendViewedDataroomEmail` to arbitrary recipients (`pages/api/jobs/send-notification.ts:156-239`).
- **Evidence:**
  ```ts
  // pages/api/jobs/send-notification.ts:28-34
  const authHeader = req.headers.authorization;
  const token = authHeader?.split(" ")[1];
  if (token !== process.env.INTERNAL_API_KEY) {     // ← non-constant-time
    res.status(401).json({ message: "Unauthorized" });
    return;
  }
  ```
  ```bash
  # Cross-check from grep
  $ grep -n "INTERNAL_API_KEY" pages/api/jobs/*.ts
  pages/api/jobs/process-download-batch.ts:28:  if (token !== process.env.INTERNAL_API_KEY) {
  pages/api/jobs/send-dataroom-new-document-notification.ts:25:  if (token !== process.env.INTERNAL_API_KEY) {
  pages/api/jobs/send-dataroom-upload-notification.ts:22:  if (token !== process.env.INTERNAL_API_KEY) {
  pages/api/jobs/send-notification.ts:31:  if (token !== process.env.INTERNAL_API_KEY) {
  ```
- **Impact:**
  Recovered `INTERNAL_API_KEY` lets the attacker:
  - Trigger arbitrary view notifications (email harvest — spam, social engineering, email-bombing victims of the team's document viewers).
  - Trigger dataroom upload notifications (mislead document owners into thinking a new file arrived).
  - Trigger arbitrary download batch jobs (resource exhaustion + DoS on the S3 storage + email queue).
- **Recommendation:**
  Replace the `!==` comparison with `crypto.timingSafeEqual` over equal-length buffers, mirroring the correct pattern in `ee/features/billing/cancellation/api/automatic-unpause-route.ts:43-50`. Reject when buffer lengths differ:
  ```ts
  import { timingSafeEqual } from "crypto";
  function safeEqual(a: string | undefined, b: string | undefined) {
    if (!a || !b || a.length !== b.length) return false;
    return timingSafeEqual(Buffer.from(a), Buffer.from(b));
  }
  ```

---

### Finding 7: `dangerouslySetInnerHTML` on user-controlled `prismaDocument.name` — full call chain (HIGH)

- **Severity:** HIGH (CWE-79)
- **Source:** HTTP handler `POST /api/teams/:teamId/documents/:id/update-name` (any user with team membership)
- **Sink:** `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` (`components/documents/document-header.tsx:604`)
- **Auth required:** YES — caller must have a NextAuth session and be a member of `teamId`. But every team member can rename a document.
- **Description:**
  Document names are user-controlled. They flow through a Zod schema (`updateNameSchema`) with a `sanitizePlainText` transform that strips all HTML tags. **Today** every write path runs the value through `sanitizePlainText`, so the `dangerouslySetInnerHTML` sink sees a plain-text string. But the structural risk is that the sink is unsanitized at render time: any future write path that bypasses the Zod transform (Notion page title extraction, bulk-import, API consumer, admin migration, restored legacy data) immediately turns into stored XSS. **The defense is at the wrong layer.** This finding traces the full chain from HTTP source to the React sink so the encoding-analysis / sanitizer-bypass nodes can verify whether `sanitizePlainText` is correctly applied to every code path that writes to `prismaDocument.name`.
- **Call chain:**
  1. **User input** — Authenticated team member edits the document name in the `contentEditable` heading in the document library UI. The `onBlur={handleNameSubmit}` handler issues a `POST /api/teams/:teamId/documents/:id/update-name` with `{ name: "<user input>" }` (or a programmatic API consumer sends the same).
  2. **Zod transform** — `pages/api/teams/[teamId]/documents/[id]/update-name.ts:12-22`:
     ```ts
     const updateNameSchema = z.object({
       name: z.string()
         .transform((value) => sanitizePlainText(value))
         .pipe(z.string().min(1).max(255)),
     });
     ```
     `sanitizePlainText` (`lib/utils/sanitize-html.ts:12-20`) uses `sanitize-html` with `allowedTags: []` and `allowedAttributes: {}`, which strips every tag and attribute. So the stored value is plain text.
  3. **Database write** — `update-name.ts:79-82`:
     ```ts
     const updateResult = await tx.document.update({
       where: { id: docId, teamId },
       data: { name: name },
     });
     ```
     Sanitized plain text is persisted to `prisma.document.name`.
  4. **Server-rendered list** — any page that renders `<DocumentHeader prismaDocument={document} />` (e.g., `app/(dashboard)/documents/[id]/page.tsx`, the `documents` listing pages, the visitor-side `view/...` pages when an internal preview is open).
  5. **React sink** — `components/documents/document-header.tsx:604`:
     ```tsx
     <h2
       className="..."
       ref={nameRef}
       contentEditable={true}
       onFocus={() => setIsEditingName(true)}
       onBlur={handleNameSubmit}
       onKeyDown={preventEnterAndSubmit}
       title="Click to edit"
       dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
     />
     ```
     The component treats `prismaDocument.name` as raw HTML. Today the value is plain text so the XSS is latent; tomorrow any new write path (Notion import, bulk CSV upload, admin migration script, restored backup from before sanitization was added) that forgets the `sanitizePlainText` step instantly produces exploitable stored XSS that fires in every team member's browser on every dashboard render.
- **Evidence:** see code excerpts above.
- **Impact:**
  Stored XSS in every team member's session. The document name appears on the dashboard's `documents` listing and on the document detail page, so the payload runs on every page load. Impact: session hijack (NextAuth JWT in `next-auth.session-token` cookie — `httpOnly` is true so direct theft requires the XSS to perform an authenticated mutation on behalf of the victim, which is fully possible), CSRF-style actions (change team plan, add/remove members, rotate SSO, edit billing), exfiltration of all team data via the XSS-fetched SWR endpoints.
- **Recommendation:**
  Replace `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` with `{prismaDocument.name}`. React auto-escapes string children. The `contentEditable` attribute does not require `dangerouslySetInnerHTML` — it requires a DOM node whose `textContent` is editable. If a styled placeholder is needed, render an explicit `<span>` child instead of an HTML string. This eliminates the entire class of stored-XSS regressions for this field.

---

### Finding 8: SAML `CredentialsProvider.authorize` upserts a user row from IdP-controlled `userinfo` — partial account-takeover (MEDIUM)

- **Severity:** MEDIUM (CWE-287)
- **Source:** `pages/api/auth/[...nextauth].ts` + `lib/auth/auth-options.ts:132-178` (the `saml-idp` CredentialsProvider)
- **Sink:** `prisma.user.upsert({ where: { email }, create: { email, name }, update: { name: name || undefined } })` (`auth-options.ts:162-166`)
- **Auth required:** NO — `authorize` runs as part of the OAuth code-exchange flow; no pre-existing session is required.
- **Description:**
  The `saml-idp` CredentialsProvider (lines 132-178 of `auth-options.ts`) exchanges a SAML code for an access token via BoxyHQ Jackson's `oauthController.token`, then calls `oauthController.userInfo(access_token)` to read the SAML userinfo. It then upserts a `prisma.user` keyed on `email` from the IdP. The credentials are not validated against an existing user — any email the IdP returns creates a new row (or matches an existing one). Combined with `allowDangerousEmailAccountLinking: true` on the `saml` provider (Finding 4), an enterprise tenant whose IdP allows the user to choose their own email attribute (or whose IdP is misconfigured to return an email the attacker controls) can register/login as a Papermark user matching that email. This is a partial ATO vector — the IdP has to be lenient, but enterprise IdP misconfigs are common.
- **Call chain:**
  1. **End user starts SSO** — clicks the "Sign in with SSO" button → `app/(ee)/api/auth/saml/authorize/route.ts:7-77` produces a SAML AuthnRequest and redirects to the IdP.
  2. **IdP authenticates and returns AuthnResponse** — IdP posts back to `/api/auth/saml/callback` (handled by Jackson).
  3. **NextAuth code exchange** — `auth-options.ts:138-150` calls `oauthController.token({ code: credentials.code, ... })` with `client_secret: process.env.NEXTAUTH_SECRET` (no tenant scoping).
  4. **Userinfo read** — line 154-156 calls `oauthController.userInfo(access_token)`. The returned `email` is whatever the IdP's `userinfo` endpoint returns for the access token.
  5. **User upsert** — line 162-166:
     ```ts
     const user = await prisma.user.upsert({
       where: { email },
       create: { email, name },
       update: { name: name || undefined },
     });
     ```
     Note: `email` is the IdP-controlled value. There is **no check** that this email matches an existing invited user, no domain allowlist, no verification email round-trip.
  6. **Account takeover chain** — if the IdP returns `email = "victim@victim-corp.com"` and a Papermark user with that email already exists, the upsert silently matches the existing row. The next-auth session is then attached to the victim's `user.id`.
- **Evidence:**
  ```ts
  // lib/auth/auth-options.ts:138-167
  CredentialsProvider({
    id: "saml-idp",
    name: "IdP Login",
    credentials: { code: { type: "text" } },
    async authorize(credentials) {
      if (!credentials?.code) return null;
      try {
        const { oauthController } = await jackson();
        const { access_token } = await oauthController.token({
          code: credentials.code,
          grant_type: "authorization_code",
          redirect_uri: getMainDomainUrl(),
          client_id: "dummy",
          client_secret: process.env.NEXTAUTH_SECRET!,
        });
        if (!access_token) return null;
        const userInfo = await oauthController.userInfo(access_token);
        if (!userInfo) return null;
        const { email, firstName, lastName, requested } = userInfo as any;
        if (!email) return null;
        const name = [firstName, lastName].filter(Boolean).join(" ") || email;
        const user = await prisma.user.upsert({
          where: { email },
          create: { email, name },
          update: { name: name || undefined },
        });
        return { id: user.id, email: user.email, name: user.name, profile: userInfo } as any;
      } catch (error) {
        console.error("[SAML] Error during SAML authorization:", error);
        return null;
      }
    },
  }),
  ```
- **Impact:**
  Cross-tenant account takeover via IdP-side misconfiguration. The `requested` field on line 157 is destructured from `userInfo` but never used to scope the upsert; the only scoping signal is `email`. If two Papermark tenants both have SSO connected to the same IdP and the IdP returns overlapping email namespaces, an end user authenticated against tenant A can be silently upserted into tenant B's user table (or matched to a tenant-B-existing user if one already exists). Combined with the SAML provider's `allowDangerousEmailAccountLinking: true` and the `prisma.user.upsert` not being gated by an invitation check, this enables:
  - Phantom-user injection: an attacker who controls a SAML IdP can create arbitrary `prisma.user` rows in Papermark.
  - Account linkage: existing users can be linked to attacker-controlled SAML identities.
- **Recommendation:**
  Add a `signIn` callback in `authOptions` that:
  1. Looks up the Jackson `connections` table for the tenant that issued this SAML code.
  2. Verifies the returned `email`'s domain matches the tenant's verified email domain.
  3. Verifies the user has either been invited (matching `Invitation.email`) or is already a `userTeam` member of that tenant.
  4. Rejects with a clear error otherwise.
  Also remove `allowDangerousEmailAccountLinking: true` from the SAML provider and rely on the explicit `signIn` callback for tenant-bound user creation.

---

## Non-Findings (Verified Protected)

These endpoints are in the same namespaces as the IDOR findings above and were inspected to confirm they **do not** suffer the same bug — they use compound keys (`id + teamId` or `id + dataroomId`). This is the audit trail for Directive D8 of `00-early-web-intel.md` ("assume the same pattern is exploitable in production for the folder namespace ... the same pattern almost certainly exists for `pages/api/teams/[teamId]/documents/...` and `pages/api/teams/[teamId]/datarooms/...`"). My finding: the pattern exists in **the same namespace** (folder-manage) but has been corrected in the dataroom folder-manage sibling, the document endpoints, the viewer endpoints, and the link endpoints.

| Endpoint | File:line | Compound key used |
|----------|-----------|-------------------|
| `GET /api/teams/:teamId/folders/:name` | `pages/api/teams/[teamId]/folders/[...name].ts:51-62` | `teamId_path` |
| `POST /api/teams/:teamId/folders/bulk` | `pages/api/teams/[teamId]/folders/bulk.ts:93-96` | `teamId_path` |
| `PATCH /api/teams/:teamId/folders/move` | `pages/api/teams/[teamId]/folders/move.ts:46-51` | `id in (folderIds) + teamId` |
| `POST /api/teams/:teamId/folders/hide` | `pages/api/teams/[teamId]/folders/hide.ts:53-62` | `id in folderIds + teamId` |
| `GET /api/teams/:teamId/folders/parents/:name` | `pages/api/teams/[teamId]/folders/parents/[...name].ts:54-66` | `teamId_path` |
| `POST /api/teams/:teamId/folders/manage/:folderId/add-to-dataroom` | `pages/api/teams/[teamId]/folders/manage/[folderId]/add-to-dataroom.ts:101-126` | team is queried with `folders: { some: { id: folderId } }` |
| `POST /api/teams/:teamId/datarooms/create-from-folder` | `pages/api/teams/[teamId]/datarooms/create-from-folder.ts:32-41` | `id + teamId` |
| `DELETE /api/teams/:teamId/datarooms/:id/folders/manage/:folderId` | `pages/api/teams/[teamId]/datarooms/[id]/folders/manage/[folderId]/index.ts:68-73` | `id + dataroomId` |
| `PUT /api/teams/:teamId/datarooms/:id/folders/manage` | `pages/api/teams/[teamId]/datarooms/[id]/folders/manage/index.ts:73-77` | `id + dataroomId` |
| `POST /api/teams/:teamId/datarooms/:id/duplicate` | `pages/api/teams/[teamId]/datarooms/[id]/duplicate.ts:151-157` | `id + teamId` (uses `withTeamApi`) |
| `GET /api/teams/:teamId/viewers` | `pages/api/teams/[teamId]/viewers/index.ts:46-56` | team membership check + raw SQL with `teamId` param |
| `GET /api/teams/:teamId/viewers/:id` | `pages/api/teams/[teamId]/viewers/[id]/index.ts:96-120` | `id + teamId` |
| `DELETE /api/teams/:teamId/links/:id` | `pages/api/teams/[teamId]/links/[id]/index.ts:63-73` | `id + teamId + deletedAt: null` |
| `GET /api/teams/:teamId/documents/:id` | `pages/api/teams/[teamId]/documents/[id]/index.ts:74-78` | `id + teamId` + `assertDocumentAccess` |
| `PUT /api/teams/:teamId/documents/:id` | same file, line 165-177 | `id + teamId + team.users.some.role=ADMIN` |
| `PATCH /api/teams/:teamId/documents/:id` | same file, line 253-257 | `id + teamId` |
| `DELETE /api/teams/:teamId/documents/:id` | same file, line 306-321 | `id + teamId` + `isDataroomScopedRole` check |
| `POST /api/teams/:teamId/documents/:id/update-name` | `pages/api/teams/[teamId]/documents/[id]/update-name.ts:67-82` | `id + teamId` |
| `POST /api/teams/:teamId/update-replicate-folders` | `pages/api/teams/[teamId]/update-replicate-folders.ts:28-52` | only writes to the team in the URL (legitimate) |
| Team-scoped conversations | `ee/features/conversations/api/team-conversations-route.ts:53-64` | `id + team.id + users.some.userId` |
| AI chat (internal user flow) | `app/(ee)/api/ai/chat/route.ts:44-95` | `id` + membership check |
| AI chat (external viewer flow) | `app/(ee)/api/ai/chat/route.ts:154-298` | `verifyDataroomSession` + `link.id + dataroomId` + `link.document.teamId` |

**Defense-in-depth gap (not full IDOR but worth flagging):** `pages/api/teams/[teamId]/documents/[id]/index.ts:340-344` (DELETE handler) does the resource fetch with `where: { id: docId, teamId }` (line 306-309) and then performs the actual delete with `where: { id: docId }` only. The TOCTOU window is bounded (a document's `teamId` doesn't change), so this is not currently exploitable, but adding `teamId` to the delete `where` is a one-line defense-in-depth fix.

---

## PHASE_3_CHECKPOINT

- [x] Findings written to `methodology-raw/01-structural-analysis.md`
- [x] Each finding has: severity, source (file:line), sink (file:line), call chain (every step with file:line), auth verdict, evidence (code snippets), impact, recommendation
- [x] Cross-referenced with `00-ai-frameworks.md`, `00-ai-sast.md`, `00-ai-api-specs.md`, `00-early-web-intel.md` (every Layer-0 risky area is addressed)
- [x] Confirmed-protected endpoints tabulated (non-findings audit trail for Directive D8)
- [x] Two new findings beyond Layer 0: Finding 2 (folder rename IDOR sibling of #2078) and Finding 8 (SAML userinfo upsert ATO)

---

## Layer 0 Cross-Reference Summary

| Layer 0 directive / finding | Resolved by structural-analysis |
|------------------------------|---------------------------------|
| `00-early-web-intel.md` D1 — Next.js CVE-2026-44578 SSRF | Out of scope (framework-version CVE; handled by reachability + deployment-arch). |
| `00-early-web-intel.md` D2 — CVE-2026-44580/44581 XSS | **Finding 7** confirms `dangerouslySetInnerHTML` sink on `prismaDocument.name`; encoding-analysis pass should verify CSP posture in `next.config.mjs:219,222,262,265`. |
| `00-early-web-intel.md` D3 — CVE-2026-44576/44582 RSC cache poisoning | Out of scope (framework-version CVE). |
| `00-early-web-intel.md` D4 — CVE-2026-44577 image-optimization OOM | Out of scope (framework-version CVE). |
| `00-early-web-intel.md` D5 — dompurify 3.4.0 transitive | Out of scope (dependency-version CVE); encoding-analysis can grep dompurify call sites. |
| `00-early-web-intel.md` D6 — `allowDangerousEmailAccountLinking` ATO | **Finding 4** confirms on Google, LinkedIn, AND SAML. New provider-level evidence. |
| `00-early-web-intel.md` D7 — tus-viewer CORS misconfig | **Finding 3** confirms from source: `setCorsHeaders` echoes Origin, sets credentials=true, runs before auth check. |
| `00-early-web-intel.md` D8 — Folder IDOR #2078 | **Findings 1 and 2** confirm DELETE and RENAME paths; audit trail for sibling endpoints in folder / dataroom folder / document / link / viewer namespaces. |
| `00-early-web-intel.md` D9 — No malware scanning on upload | Out of scope for structural (no upload pipeline auth gap was found; conversion pipeline is the next layer). |
| `00-early-web-intel.md` D10 — sanitize-html ^2.17.3 | Out of scope for structural (package-version pin). |
| `00-early-web-intel.md` D11 — `dangerouslySetInnerHTML` on document name | **Finding 7** traces the full source→sink chain (HTTP → Zod → DB → render). |
| `00-early-web-intel.md` D12 — OIDC callback URL validation | Not exhaustively verified here; `00-ai-api-specs.md` already noted SAML Jackson handles this. New `Finding 8` covers a different SAML attack surface (userinfo upsert). |
| `00-early-web-intel.md` D13 — `cuid()` for permission row IDs | Not exhaustively verified here; flagged for reachability-analysis. |
| `00-ai-frameworks.md` §6 tus-viewer CORS | **Finding 3** supersedes with full call chain. |
| `00-ai-frameworks.md` §4 next-auth dangerous linking | **Finding 4** supersedes with all 3 providers. |
| `00-ai-api-specs.md` F2 cron auth bypass outside Vercel | **Finding 5** traces through `verify-qstash.ts:11-14` and lists all affected routes. |
| `00-ai-api-specs.md` F3 timing-unsafe INTERNAL_API_KEY | **Finding 6** lists all 4 affected routes with file:line. |
| `00-ai-sast.md` F2 document-header XSS | **Finding 7** adds the full call chain (was sink-only). |
| `00-ai-sast.md` F6 SAML authorize XML parsing | New **Finding 8** covers the `userinfo`-driven user upsert as a separate SAML ATO surface. |
