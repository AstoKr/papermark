# Finding: Folder RENAME IDOR by folderId Only (Cross-Tenant)

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:H/A:N = 8.5 (any team member can rewrite any other team's folder name/icon/color — phishing pivot + dashboard integrity loss). Higher if combined with social engineering: 8.8.

## Affected Code
- File: `pages/api/teams/[teamId]/folders/manage/index.ts`
- Line: 70-80 (resource fetch with `where: { id: folderId }` only), 113-153 (transaction including the rename `tx.folder.update`)
- Function: `handle` (PUT branch)
- Endpoint: `PUT /api/teams/:teamId/folders/manage` (body contains `{ folderId, name, icon, color }`)

## Description
The folder RENAME endpoint is the **sibling of F-005** (the DELETE IDOR). It uses the same auth/role gates (caller must be a team member on the URL's `teamId`) but the resource fetch and the subsequent `prisma.folder.update` are both keyed by `folderId` only — the folder's actual `teamId` is never compared to the URL's `teamId`. Any team member on `teamA` can rename any folder in `teamB`, change its icon, and change its color.

The visual integrity of the victim's folder tree is arbitrarily mutable by anyone — including rename to "Confidential Q4 Earnings" (a phishing pivot that survives across the victim's normal sessions).

## Proof of Concept

### Pre-conditions
- Caller is any team member on any team `teamA` (the role gate is more permissive here than the DELETE handler — any team member can rename, not just ADMIN/MANAGER).
- The `folderId` of the victim's folder (cuid, not secret; same exposure surface as F-005).

### Step-by-step exploitation

1. **Attacker is a member of `teamA`** (their own personal team — membership is sufficient, no role required).

2. **Pick the target** — a folder in some other tenant's team. `folderId` from the same sources as F-005.

3. **Send the cross-tenant RENAME:**
   ```bash
   curl -X PUT \
     'https://app.papermark.com/api/teams/<teamA_id>/folders/manage' \
     -H 'Content-Type: application/json' \
     -H 'Cookie: __Secure-next-auth.session-token=<attacker_session_jwt>' \
     -d '{
       "folderId": "<teamB_folderId>",
       "name": "Confidential Q4 Earnings",
       "icon": "briefcase",
       "color": "red"
     }'
   ```

4. **Server response** is `200 OK` with the updated folder JSON. The handler:
   - passes `getServerSession` (line 21-25) — attacker is authenticated
   - passes `prisma.team.findUnique({ where: { id: teamId, users: { some: { userId } } } })` (line 55-68) — the URL teamId is the attacker's
   - calls `prisma.folder.findUnique({ where: { id: folderId } })` (line 70-84) — finds the victim's folder because `teamId` is not part of the key
   - runs the rename in a transaction (line 113-153):
     - descendant query at line 117-126 filters by `teamId` (the URL team) — so an attacker renaming a victim's folder at the **root** level succeeds, but the descendant subtree paths are not re-written (this query returns zero rows because the victim's descendants are in `teamB`)
     - the root rename `tx.folder.update({ where: { id: folderId }, data: updateData })` (line 147-152) **succeeds** keyed by `id` only

5. **Verification:** the victim's folder now displays the new name/icon/color in every team member's dashboard view. Subsequent folder navigation breaks for descendants (the path-based slug derived from the new name does not match the old descendant paths, so `prisma.folder.findUnique` lookups by `path` will miss). The folder itself is renamed; descendants are reachable only by direct `id` lookup.

## Impact
- **Dashboard integrity loss** across tenants: any team member on any team can rename any folder in any other team, change its icon, change its color. Visual integrity of the victim's folder tree is fully mutable.
- **Phishing pivot**: rename a victim's "Quarterly Reports" folder to "🔒 Confidential Q4 Earnings" with a red color and briefcase icon. A victim team member clicking on it (or a viewer sent a link) sees the renamed folder in their dashboard and may believe it is a high-value resource, increasing the success rate of follow-on social-engineering.
- **Denial of service**: rename a victim's folder to emoji-only (e.g. `📁📁📁`) — all slug-derived descendant path lookups break, and the victim's bulk operations on the subtree fail.
- **The visual change persists** across sessions — this is not a session-scoped attack.
- **Cascading damage**: because the descendant path rewrite filters by the URL `teamId` (not the folder's actual `teamId`), the rename only succeeds for the root. If the attacker re-runs with a body that re-renames a descendant folder (they would need the descendant's `folderId`), the cascade can be walked one folder at a time.

## Evidence
```ts
// pages/api/teams/[teamId]/folders/manage/index.ts:55-84
const team = await prisma.team.findUnique({
  where: {
    id: teamId,
    users: { some: { userId: userId } },  // ← URL teamId, NOT the folder's teamId
  },
});
if (!team) {
  res.status(401).end("Unauthorized");
  return;
}
const folder = await prisma.folder.findUnique({
  where: {
    id: folderId,                          // ← no teamId in the where clause
  },
  select: { name: true, path: true, icon: true, color: true },
});
```
```ts
// pages/api/teams/[teamId]/folders/manage/index.ts:147-153
return tx.folder.update({
  where: {
    id: folderId,                          // ← keyed by id only — no teamId guard
  },
  data: updateData,
});
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 2)
- **M2 — Web synthesis** (`methodology-raw/03-web-synthesis.md` extended the GH #2078 disclosure with the RENAME sibling)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 5)
- **M9 — Custom rules** (Folder namespace audit)

## Chain Potential
- **Chain 2 / Chain 5** (`methodology-raw/05-chains.md`) — composes with F-005/F-007/F-013 for cross-tenant destruction.
- **F-024** (welcomeMessage) — the renamed folder can be linked to a custom-domain URL that the attacker controls (e.g. by uploading a malicious document into the renamed folder and sending a share link), creating a high-confidence phishing pivot.

## Remediation
Same fix as F-005: extend the resource fetch to include `teamId` and verify it matches the URL's `teamId` before any mutation:
```ts
const folder = await prisma.folder.findUnique({
  where: { id: folderId },
  select: { name: true, path: true, icon: true, color: true, teamId: true },
});
if (!folder) return res.status(404).json({ message: "Folder not found" });
if (folder.teamId !== teamId) {
  return res.status(404).json({ message: "Folder not found" });
}
```
Apply the same fix to **all** `pages/api/teams/[teamId]/folders/manage/**` handlers. Audit trail: per `methodology-raw/01-structural-analysis.md` non-findings, the **dataroom folder-manage** sibling (`pages/api/teams/[teamId]/datarooms/[id]/folders/manage/[folderId]/index.ts:68-73`) correctly uses `id + dataroomId` and is therefore not vulnerable — but the **team** folder-manage namespace has not been fixed.
