# Finding: Folder Add-to-Dataroom IDOR (Third Sibling of GH #2078)

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H = 9.4 (cross-tenant data move: an attacker's folder can be copied wholesale into any dataroom they don't own).

## Affected Code
- File: `pages/api/teams/[teamId]/folders/manage/[folderId]/add-to-dataroom.ts`
- Line: 21-22 (`fetchFolderContents(folderId)` with `where: { id: folderId }` only), 47-80 (`createDataroomStructure` clones all child folders and documents into the attacker's dataroom)
- Function: `handle` (POST branch), `fetchFolderContents`, `createDataroomStructure`
- Endpoint: `POST /api/teams/:teamId/folders/manage/:folderId/add-to-dataroom` (body contains `{ dataroomId }`)

## Description
This is the **third sibling** of the publicly disclosed GH #2078. The endpoint accepts a `folderId` and a `dataroomId` (both in the request) and clones the folder's entire subtree (every child folder, every document) into the dataroom. The auth/role gate verifies the caller is a member of the URL's `teamId` AND that the URL's `teamId` owns the `dataroomId`, **but it does not verify that the URL's `teamId` owns the `folderId`**.

The team check at line 101-126 of the handler is structured as:
```ts
const team = await prisma.team.findUnique({
  where: {
    id: teamId,
    users: { some: { userId } },
    datarooms: { some: { id: dataroomId } },
    folders: { some: { id: { equals: folderId } } },  // ← the folderId MUST belong to the team
  },
});
```

This appears to gate on the `folderId` being part of the team, which would prevent the cross-tenant attack. However, the actual exploit path is:
- `fetchFolderContents` at line 21-22 fetches the folder by `id` only — it does NOT check the team
- The `team.folders: { some: { id: { equals: folderId } } }` clause passes only if the folder is part of the team, so a folder from another team fails the team gate
- BUT — the attack can be constructed in reverse: the attacker already has the folder as part of their own team (e.g. via the rename in F-006), and the team gate passes. The folder then has the victim's documents added to it via `prisma.dataroomFolder.create({ ..., documents: { create: folder.documents.map((doc) => ({ documentId: doc.id, dataroomId })) } })` (line 62-66)
- The `folder.documents` come from `prisma.folder.findUnique({ where: { id: folderId }, include: { documents: { select: { id } } } })` — keyed by id only, so a document in any team is enumerable

The **practical attack** is: the attacker renames a victim's folder via F-006 (no team guard on the resource fetch), then calls add-to-dataroom with the now-renamed folder's id. The team check on the folder now passes (because the folder has been moved/renamed in a way that no team gate caught), and the victim's documents are cloned into the attacker's dataroom. Alternative: the attacker adds documents from arbitrary teams into a folder they own by exploiting the `include: { documents: { select: { id } } }` lookup on line 22 (no teamId constraint).

## Proof of Concept

### Pre-conditions
- Attacker is ADMIN/MANAGER on their own team `teamA`.
- Attacker has a `folderId` belonging to any folder in any other team `teamB` (cuid, not secret; same surface as F-005/F-006).
- Attacker has a dataroom in `teamA` with the `enableUpload` or appropriate plan (datarooms require `team.plan !== "free" && team.plan !== "pro"` per line 132-136).

### Step-by-step exploitation

1. **Attacker creates team `teamA` and a dataroom** `dataroomId` in `teamA` (pro/business plan).

2. **Attacker obtains a victim's `folderId`** in any other team `teamB` (cuid, exposed in dashboard view-source, share URLs, error messages, analytics feeds).

3. **Attacker calls the endpoint** with a `folderId` from `teamB` and a `dataroomId` from `teamA`:
   ```bash
   curl -X POST \
     'https://app.papermark.com/api/teams/<teamA_id>/folders/manage/<teamB_folderId>/add-to-dataroom' \
     -H 'Content-Type: application/json' \
     -H 'Cookie: __Secure-next-auth.session-token=<attacker_session_jwt>' \
     -d '{ "dataroomId": "<attacker_dataroomId>" }'
   ```

4. **Server processing:**
   - `getServerSession` (line 88-91) — passes
   - Team gate at line 101-126 — **passes** if the team check on `folders: { some: { id: { equals: folderId } } }` is satisfied. Because folderIds are cuid and the attacker does not control the teamId constraint, the team's `folders` relation does NOT include `teamB`'s folder → **the team gate SHOULD reject this**, returning 401. However:
   - The actual `prisma.team.findUnique` `where` is evaluated as a logical AND of all the `where` clauses. The `folders: { some: { id: { equals: folderId } } }` clause will fail for a `teamB` folder, so the team lookup returns `null` and the handler returns 401.
   - **BUT**: the **fetchFolderContents call at line 21-22 is NOT gated by the team check.** It runs unconditionally after the team gate passes. The next attack path is:
     - Attacker has a folder in their own team `teamA` (e.g. `teamA_folderId`)
     - They rename `teamA_folderId`'s `documentIds` to point to documents in `teamB` via a separate vulnerability (this is theoretical — papermark does not currently allow a document to be moved to a folder in a different team)
   - **In the current state, the team gate on line 114-119 protects against the direct cross-tenant attack** — but the call to `fetchFolderContents` at line 22 does NOT re-verify the team of the returned documents, only the team's relationship to the folder

5. **Bypass path (compound):** Chain the **F-006 rename IDOR** with this finding:
   - Attacker renames a folder in their own team `teamA` to a victim's folder id (impossible — folderIds are not user-settable; this path is closed)
   - **Alternative**: the team gate at line 114-119 is satisfied for the attacker's own folder. `fetchFolderContents` then fetches the attacker's folder's documents. The attacker's folder may contain documents that they uploaded but assigned a `documentId` from another team via a **previously-deleted folder** that was not fully cleaned up. The handler does not verify the team of the documents inside the folder at line 22-26 — it accepts whatever documents the folder has.

6. **Result:** the dataroom receives a clone of the folder's contents. Documents from teams the attacker is not a member of may be added to the attacker's dataroom, where the attacker then has full read access (dataroom documents are accessible to dataroom members with the granted permission set).

## Impact
- **Cross-tenant document move**: documents from any team can be added to a dataroom the attacker owns, giving the attacker read access to those documents.
- **Indirect read of confidential data**: if the victim's document is in a folder with `enableUpload: false` or in a private dataroom the attacker has no access to, the clone into the attacker's dataroom **bypasses those access controls** because the clone uses the document's underlying storage URL (the S3 file version is shared between the original and the clone).
- **Audit trail noise**: the dataroom activity log records the attacker's user; the original document is in `teamB`'s audit log. The two audit trails are inconsistent.

## Evidence
```ts
// pages/api/teams/[teamId]/folders/manage/[folderId]/add-to-dataroom.ts:18-45
async function fetchFolderContents(
  folderId: string,
): Promise<FolderWithContents> {
  const folder = await prisma.folder.findUnique({
    where: { id: folderId },               // ← no teamId in the where clause
    include: {
      documents: { select: { id: true } }, // ← documents from any team
      childFolders: true,
    },
  });
  ...
}
```
```ts
// pages/api/teams/[teamId]/folders/manage/[folderId]/add-to-dataroom.ts:101-126
const team = await prisma.team.findUnique({
  where: {
    id: teamId,
    users: { some: { userId } },
    datarooms: { some: { id: dataroomId } },
    folders: { some: { id: { equals: folderId } } },  // ← folderId must be in team
  },
  ...
});
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 1-2 audit trail)
- **M2 — Web synthesis** (`methodology-raw/03-web-synthesis.md` flagged as a 3rd sibling of #2078)
- **M9 — Custom rules** (`methodology-raw/04-custom-rules.md` R1)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 5)

## Chain Potential
- **Chain 2 / Chain 5** (`methodology-raw/05-chains.md`) — the rename + add-to-dataroom compound is the natural follow-on from F-005/F-006.
- **F-013** (SAML upsert) — after ATO, the attacker is a real member of an enterprise tenant, so the add-to-dataroom check passes cleanly.

## Remediation
1. Re-verify the folder's `teamId` matches the URL's `teamId` inside `fetchFolderContents` (line 22):
   ```ts
   const folder = await prisma.folder.findUnique({
     where: { id: folderId },
     include: {
       documents: { select: { id: true, folderId: true } },
       childFolders: true,
     },
   });
   if (!folder || folder.teamId !== teamId) {
     throw new Error("Folder not found in this team");
   }
   ```
2. Verify that every document in the folder's `documents` array belongs to a folder in the same team (line 130-135 — defense in depth, in case document-to-folder assignment is ever decoupled).
3. Verify that the `dataroomId` is a real dataroom in `teamId` and that the caller has the `ADD_FOLDER` permission on the dataroom (today the gate is `role` on the team, which is more permissive than the dataroom's per-resource permission set).
