# Finding: Missing Auth on /api/progress-token Mints Trigger.dev Public Tokens

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N = 5.3 (information disclosure + token issuance for any `documentVersionId`; chained with F-015 the impact is recon + token = tokenization of arbitrary document versions).

## Affected Code
- File: `pages/api/progress-token.ts`
- Line: 1-29 (the entire handler — 28 lines, no `getServerSession` call, no team membership check)
- Function: `handle` (default export)
- Endpoint: `GET /api/progress-token?documentVersionId=<id>`

## Description
`GET /api/progress-token?documentVersionId=…` calls `generateTriggerPublicAccessToken("version:" + documentVersionId)` and returns the resulting token. No `getServerSession`, no team membership check, no document-ownership check. Anyone who can guess or enumerate a `documentVersionId` (cuid, exposed in dashboards, share URLs, error messages, analytics feeds) receives a Trigger.dev public access token scoped to that version.

This is the natural follow-up to commit `9db42913` (the recent auth sweep) — the sweep closed the link↔document IDOR in `app/api/views/route.ts` but did not extend coverage to the two adjacent endpoints that accept a raw `documentVersionId` query parameter.

## Proof of Concept

### Pre-conditions
- A `documentVersionId` value (cuid, not secret — exposed in dashboard view-source, share URLs, error messages, Tinybird analytics feed). F-015 (recon oracle) demonstrates a public endpoint that confirms valid IDs.

### Step-by-step exploitation

1. **Obtain a seed `documentVersionId`** — many surfaces:
   - View-source on a dashboard page that renders a document version's status badge
   - Inspect a share URL (e.g. `https://app.papermark.com/view/<linkId>?version=<versionId>`)
   - Error messages that include the version ID
   - The Tinybird analytics feed (if the attacker has dashboard access)
   - The recon oracle at F-015 (`/api/document-processing-status?documentVersionId=<id>`) — 200 vs 404 confirms validity

2. **Confirm via F-015** (optional, but reduces noise):
   ```bash
   curl 'https://app.papermark.com/api/teams/<any>/documents/document-processing-status?documentVersionId=<id>'
   # → 200 {"currentPageCount": N, "totalPages": M, "hasPages": true} (valid)
   # → 404 (invalid)
   ```

3. **Mint a Trigger.dev public access token:**
   ```bash
   curl 'https://app.papermark.com/api/progress-token?documentVersionId=<id>'
   # → 200 {"publicAccessToken": "<trigger_token>"}
   ```

4. **Use the token** against the Trigger.dev public-API surface scoped to that version. The exact API surface (pages, conversion status, PDF re-render, AI chat context) is reachable from any browser with the token.

5. **Enumerate at scale** — repeat with timestamp-iteration over cuid generation windows to find active `documentVersionId` values in the wild. Each iteration step is two requests: one to F-015 (confirm valid), one to F-014 (mint token). ~8K requests can scan the recent 24 hours of `documentVersionId` creation.

## Impact
- **Token issuance for any `documentVersionId`** — the token is a Trigger.dev public access token scoped to that version. With the token, the attacker can call the Trigger.dev public API to read the version's processing state and re-render status.
- **Information disclosure** — the token + F-015 oracle together let the attacker enumerate which `documentVersionId` values are real and what their conversion state is (e.g. "the document is fully converted," "the document is being processed," "the document has 47 pages"). This is sensitive for documents under embargo or in active dataroom negotiations.
- **Recon amplifier** — the token is the "validity confirmation" primitive that other gated endpoints don't have. The chain with F-015 turns "blind guess at `documentVersionId`" into "confirmed valid `documentVersionId` + Trigger.dev token."

## Evidence
```ts
// pages/api/progress-token.ts:1-29
import { NextApiRequest, NextApiResponse } from "next";
import { generateTriggerPublicAccessToken } from "@/lib/utils/generate-trigger-auth-token";

export default async function handle(
  req: NextApiRequest,
  res: NextApiResponse,
) {
  if (req.method !== "GET") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const { documentVersionId } = req.query;

  if (!documentVersionId || typeof documentVersionId !== "string") {
    return res.status(400).json({ error: "Document version ID is required" });
  }

  try {
    const publicAccessToken = await generateTriggerPublicAccessToken(
      `version:${documentVersionId}`,
    );
    return res.status(200).json({ publicAccessToken });
  } catch (error) {
    console.error("Error generating token:", error);
    return res.status(500).json({ error: "Failed to generate token" });
  }
}
```
The handler is 28 lines. There is no `getServerSession` call. There is no team membership check. There is no document-ownership check. The only input is `req.query.documentVersionId`, and the only output is the generated token.

## Methodology
- **M4 — Git history** (`methodology-raw/02-synthesized-git-history.md` S-001)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 4)

## Chain Potential
- **Chain 4** (`methodology-raw/05-chains.md`) — chains with F-015 (recon oracle) for "any internet caller can enumerate valid `documentVersionId` values and mint a Trigger.dev token for each." The chain's bounty estimate is $8K-$20K.
- **F-001/F-004** (4-purpose NEXTAUTH_SECRET) — once an attacker mints a Trigger.dev token, they can use it to exfiltrate the document processing pipeline, which has access to the document's S3 file (and indirectly, the document password).

## Remediation
Either:
**(a) Require an authenticated session + team-membership check + document-ownership check** before issuing the token:
```ts
const session = await getServerSession(req, res, authOptions);
if (!session) return res.status(401).json({ error: "Unauthorized" });

const documentVersion = await prisma.documentVersion.findUnique({
  where: { id: documentVersionId },
  include: { document: { include: { team: { include: { users: { where: { userId: session.user.id } } } } } } },
});
if (!documentVersion || documentVersion.document.team.users.length === 0) {
  return res.status(404).json({ error: "Document version not found" });
}

const publicAccessToken = await generateTriggerPublicAccessToken(
  `version:${documentVersionId}`,
);
return res.status(200).json({ publicAccessToken });
```

**(b) Gate the endpoint to internal `INTERNAL_API_KEY` callers** and have the team-side code that needs the token call it server-side. This is cleaner — the progress poller on `lib/swr/use-document.ts:200-218` is already authenticated, so the team-side code can call this endpoint server-to-server with the `INTERNAL_API_KEY` and pass the token to the client.

Apply the same fix to F-015 (`/api/document-processing-status`).
