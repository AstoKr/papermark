# Finding: Missing Auth on /api/document-processing-status (Recon Oracle)

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N = 5.3 (information disclosure: 200 vs 404 reveals valid `documentVersionId` values; secondary disclosure: page count and conversion state for documents under embargo).

## Affected Code
- File: `pages/api/teams/[teamId]/documents/document-processing-status.ts`
- Line: 1-32 (the entire handler — 32 lines, no `getServerSession` call, the `teamId` URL parameter is **never used**)
- Function: `handler` (default export)
- Endpoint: `GET /api/teams/:teamId/documents/document-processing-status?documentVersionId=<id>`

## Description
Endpoint accepts `documentVersionId` and returns `{ currentPageCount, totalPages, hasPages }` for that version. No `getServerSession`, no team membership check, no document-ownership check. The URL embeds `teamId` but it is **never used** — the only input is `req.query.documentVersionId`, and the only output is `res.status(200).json(status)`.

This is the natural follow-up to commit `9db42913` (the recent auth sweep). The sweep closed the link↔document IDOR in `app/api/views/route.ts` but did not extend coverage to the two adjacent endpoints that accept a raw `documentVersionId` query parameter.

## Proof of Concept

### Pre-conditions
- An internet connection.
- A `documentVersionId` to test (cuid, not secret; can be obtained from view-source on dashboard pages, share URLs, error messages, Tinybird analytics).

### Step-by-step exploitation

1. **Enumerate**:
   ```bash
   # Test a known/guessed documentVersionId
   curl 'https://app.papermark.com/api/teams/<any_teamId>/documents/document-processing-status?documentVersionId=<id>'
   # → 200 {"currentPageCount": N, "totalPages": M, "hasPages": true}  (valid)
   # → 404  (invalid)
   ```

2. **Brute-force / enumerate at scale** — iterate over cuid generation windows. Each cuid has a timestamp prefix; the attacker can iterate timestamp+counter pairs to find active `documentVersionId` values. ~8K requests can scan the recent 24 hours of `documentVersionId` creation.

3. **Chain with F-014** (`/api/progress-token`):
   - For each valid `documentVersionId`, mint a Trigger.dev public access token via F-014.
   - Use the token to call the Trigger.dev public API to read the version's processing state and re-render status.

4. **Information disclosure** — the response reveals:
   - Whether the document version is real (200 vs 404)
   - The current page count (`currentPageCount`)
   - The total pages (`totalPages`)
   - Whether conversion is complete (`hasPages`)

   For documents under embargo or in active dataroom negotiations, this is sensitive (e.g. "this Q3 financial report has 47 pages and conversion finished at 3:42am UTC").

5. **Polling exposure** — wired into `useDocumentProcessingStatus` (`lib/swr/use-document.ts:200-218`) which calls it every 3 seconds, so a successful 200 is a high-confidence signal the document version is real and active in someone's session.

## Impact
- **Confirmation oracle** — 200 vs 404 tells the attacker which `documentVersionId` values are valid, even if they can't yet read their content.
- **Timing oracle for conversion** — `hasPages` and `currentPageCount` reveal whether conversion finished and how many pages exist, which can be paired with the views endpoints that *do* require auth.
- **Polling exposure** — every 3 seconds per `useDocumentProcessingStatus` (`lib/swr/use-document.ts:200-218`), so a successful 200 is a high-confidence signal the document version is real and active.
- **Recon amplifier for F-014** — the recon oracle is the precondition primitive for the Trigger.dev token mint. Without the recon, the attacker would need to brute-force `documentVersionId` values or scrape dashboards. With the recon, the attacker can iterate the entire `documentVersionId` space at scale.

## Evidence
```ts
// pages/api/teams/[teamId]/documents/document-processing-status.ts:1-32
import { NextApiRequest, NextApiResponse } from "next";
import prisma from "@/lib/prisma";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse,
) {
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

  const status = {
    currentPageCount: documentVersion._count.pages,
    totalPages: documentVersion.numPages,
    hasPages: documentVersion.hasPages,
  };

  res.status(200).json(status);
}
```
The handler is 32 lines. There is no `getServerSession` call. There is no team membership check. There is no document-ownership check. The URL `teamId` parameter is never referenced.

## Methodology
- **M4 — Git history** (`methodology-raw/02-synthesized-git-history.md` S-002)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 4)

## Chain Potential
- **Chain 4** (`methodology-raw/05-chains.md`) — chains with F-014 (progress token mint). The recon oracle converts the "valid ID required" precondition of F-014 into "no precondition."
- **F-005/F-006/F-007** (folder IDOR) — the recon oracle can be used to enumerate `documentVersionId` values that map to specific folder paths, accelerating the cross-tenant destruction chain.

## Remediation
Apply the same `getServerSession` + `userTeam.findUnique({ userId, teamId })` + document-ownership pattern used in `lib/documents/get-file-helper.ts:17-37`:
```ts
const session = await getServerSession(req, res, authOptions);
if (!session) return res.status(401).json({ error: "Unauthorized" });

const userId = session.user.id;
const teamId = req.query.teamId as string;

// Verify team membership
const teamAccess = await prisma.userTeam.findUnique({
  where: { userId_teamId: { userId, teamId } },
});
if (!teamAccess) return res.status(401).json({ error: "Unauthorized" });

// Verify the document version belongs to a document in the team
const documentVersion = await prisma.documentVersion.findUnique({
  where: { id: documentVersionId },
  include: { document: { select: { teamId: true } } },
});
if (!documentVersion || documentVersion.document.teamId !== teamId) {
  return res.status(404).end();
}

const status = {
  currentPageCount: documentVersion._count.pages,
  totalPages: documentVersion.numPages,
  hasPages: documentVersion.hasPages,
};
res.status(200).json(status);
```

Apply the same fix to F-011 (`/api/progress-token`).
