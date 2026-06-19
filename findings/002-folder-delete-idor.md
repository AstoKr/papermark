# Finding: Folder DELETE IDOR by folderId Only (Cross-Tenant)

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:N = 9.6 (any ADMIN/MANAGER on any team can delete any folder in any other team).

## Affected Code
- File: `pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts`
- Line: 53-65 (resource fetch with `where: { id: folderId }` only), 89-142 (`deleteFolderAndContents` cascades by `folderId`)
- Function: `handle` (DELETE branch) and `deleteFolderAndContents`
- Endpoint: `DELETE /api/teams/:teamId/folders/manage/:folderId`

## Description
The folder DELETE endpoint verifies the caller is an ADMIN/MANAGER on **the team specified in the URL**, but the resource fetch is keyed by `id` (folderId) only — the folder's actual `teamId` is never compared to the URL's `teamId`. A user who is ADMIN/MANAGER on `teamA` can call `DELETE /api/teams/teamA/folders/manage/<teamB_folderId>` and the handler will delete the folder (and all its child folders, documents, and S3 file versions) from `teamB` despite the caller having no membership in `teamB`.

This is the publicly disclosed papermark Issue **#2078**, confirmed in source.

## Proof of Concept

### Pre-conditions
- Caller is ADMIN or MANAGER on any team `teamA` (e.g. their own personal team).
- Attacker has a `folderId` belonging to a folder in any other team `teamB`. The `folderId` is a cuid; it appears in dashboard HTML, share URLs, error messages, and the analytics Tinybird feed. It is **not secret**.

### Step-by-step exploitation

1. **Create the attacker's "burner" team** by signing up for papermark and creating a team `teamA`. The attacker becomes ADMIN of `teamA`.

2. **Pick the target** — a folder in some other tenant's team. The `folderId` is exposed in:
   - The victim's dashboard view-source (if the attacker has been sent a share link)
   - The Tinybird analytics feed (for any team member)
   - Server error messages (e.g. when a link is malformed)
   - The 200 vs 404 response of the recon oracle at `/api/teams/<any>/documents/document-processing-status?documentVersionId=<id>` (F-015) — the documentVersionId can be correlated to the parent folderId via the cuid timestamp+counter sequence

3. **Send the cross-tenant DELETE:**
   ```bash
   curl -X DELETE \
     'https://app.papermark.com/api/teams/<teamA_id>/folders/manage/<teamB_folderId>' \
     -H 'Cookie: __Secure-next-auth.session-token=<attacker_session_jwt>' \
     -H 'X-CSRF-Token: <csrf_token>'
   ```
   Note: the `teamId` in the URL is the attacker's own team, and the request is authenticated as the attacker. The handler's auth/role gates pass (the attacker is ADMIN of `teamA`).

4. **Server response** is `204 No Content`. The handler:
   - passes `getServerSession` (line 17-20) — attacker is authenticated
   - passes the `prisma.userTeam.findUnique({ where: { userId_teamId: { userId: attacker, teamId: teamA } } })` check (line 30-44) — the URL teamId is the attacker's
   - passes the role check (line 46-51) — attacker is ADMIN
   - calls `prisma.folder.findUnique({ where: { id: folderId } })` (line 53-65) — finds the victim's folder because `teamId` is NOT part of the key
   - calls `deleteFolderAndContents(folderId, teamA)` (line 74) — recursively deletes the victim's folder, all child folders, all documents in those folders, and S3 file versions

5. **Verification:** the victim's folder (and all its descendants + documents) is gone. S3 file versions are also deleted. Audit log records the attacker's user.id and `teamId: teamA`, but the destroyed resource belongs to `teamB`.

## Impact
- **Cross-team integrity loss**: any ADMIN or MANAGER on any team can delete any folder in any other team. The destroy cascades to child folders, documents, and S3 file versions.
- **No data confidentiality crossed** (this is an integrity / availability issue, not a read).
- **Audit log noise**: the audit log records the attacker's user, not the team context of the resource. The `teamId` in the audit log entry is the URL's `teamId` (the attacker's burner), not the resource's actual `teamId`. Forensic trail is misleading.
- **Chains with F-013 (SAML ATO)** to reach any team's folders without needing a pre-existing account on the target team. The SAML upsert creates a `prisma.user` row from any email an IdP returns; the attacker then needs only an ADMIN/MANAGER role on any team (easy: their own personal team) to delete the victim's folders.

## Evidence
```ts
// pages/api/teams/[teamId]/folders/manage/[folderId]/index.ts:30-65
const teamAccess = await prisma.userTeam.findUnique({
  where: {
    userId_teamId: {
      userId: userId,
      teamId: teamId,         // ← URL teamId, NOT the folder's teamId
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
    id: folderId,            // ← no teamId in the where clause
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
Cross-check (sibling endpoints that are **NOT** vulnerable because they use a compound key):
- `pages/api/teams/[teamId]/folders/[...name].ts:51-62` — `teamId_path` (compound)
- `pages/api/teams/[teamId]/datarooms/[id]/folders/manage/[folderId]/index.ts:68-73` — `id + dataroomId` (compound)

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 1)
- **M2 — Web synthesis** (`methodology-raw/03-web-synthesis.md` confirmed via GH Issue #2078)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 5)
- **M9 — Custom rules** (Folder namespace pattern audit, `methodology-raw/04-custom-rules.md`)

## Chain Potential
- **Chain 2 / Chain 5** (`methodology-raw/05-chains.md`) — chains with F-013 (SAML upsert) and F-005/F-006/F-007 to reach "ATO + cross-tenant folder destruction."
- **F-015** (recon oracle) provides the folderId enumeration primitive.
- **F-024** (welcomeMessage) provides the post-ATO phishing pivot.

## Remediation
Change the resource fetch to include `teamId` and verify it matches the URL's `teamId` before any mutation:
```ts
const folder = await prisma.folder.findUnique({
  where: { id: folderId },
  select: { teamId: true, _count: { select: { documents: true, childFolders: true } } },
});
if (!folder) return res.status(404).json({ message: "Folder not found" });
if (folder.teamId !== teamId) {
  return res.status(404).json({ message: "Folder not found" });
  // or res.status(403) — but 404 is preferred to avoid leaking folder existence
}
```
Alternatively, add a Prisma compound unique constraint on `(id, teamId)` and use `prisma.folder.delete({ where: { id_teamId: { id: folderId, teamId } } })`. The current Prisma schema has `@unique` on `(teamId, path)` but not on `(id, teamId)` — verify in `prisma/schema.prisma` and add if absent.

Apply the same fix to **all** `pages/api/teams/[teamId]/folders/manage/**` handlers (see F-006 for the rename sibling and F-007 for the add-to-dataroom sibling).
