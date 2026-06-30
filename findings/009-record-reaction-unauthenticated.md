# Finding: `record_reaction.ts` — Unauthenticated DB Write + Viewer Data Leak

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N = 5.3

## Affected Code
- **File:** `pages/api/record_reaction.ts:17-42`
- **Endpoint:** `POST /api/record_reaction`

## Description
This endpoint creates a `Reaction` record in PostgreSQL with no authentication or session validation. It accepts `viewId`, `pageNumber`, and `type` from the request body and does not verify the caller is authorized to perform this action. The endpoint includes the full `view` relation in the creation response (lines 30-41), leaking `documentId`, `dataroomId`, `linkId`, `viewerEmail`, `viewerId`, and `teamId` back to the caller.

Unlike `record_view.ts` and `record_click.ts` (which publish to Tinybird analytics pipeline), this endpoint writes directly to the PostgreSQL `Reaction` table, making it a data-injection vector.

## Proof of Concept
```bash
# No authentication required — anyone can create reactions and leak viewer data
curl -X POST "https://papermark.app/api/record_reaction" \
  -H "Content-Type: application/json" \
  -d '{"viewId":"any_view_id","pageNumber":1,"type":"like"}'
# Response leaks: viewerEmail, documentId, dataroomId, linkId, viewerId, teamId
```

## Impact
An unauthenticated attacker can:
- Inject arbitrary reaction data tied to any known `viewId`
- Enumerate valid `viewId` values by observing error responses
- Leak protected viewer metadata (`viewerEmail`, `viewerId`, `documentId`, `dataroomId`, `linkId`, `teamId`) via the `include` clause in the response
- Pollute analytics and reaction data that team members see in their dashboards

## Evidence
```typescript
// pages/api/record_reaction.ts:17-42 — no auth, creates DB record with viewer data leak
const { viewId, pageNumber, type } = req.body;
const reaction = await prisma.reaction.create({
  data: { viewId, pageNumber, type },
  include: {
    view: {
      select: {
        documentId: true,
        dataroomId: true,
        linkId: true,
        viewerEmail: true,
        viewerId: true,
        teamId: true,
      },
    },
  },
});
```

## Methodology
- **First found by:** Structural Analysis (01-structural-analysis.md) — Finding 5
- **Confirmed by:** SAST synthesis (02-synthesized-sast.md S-006)
- **Confirmed by:** Custom semgrep rule `papermark-no-auth-prisma-write` matched at line 24

## Chain Potential
**Chainable.**
- **Chain 1 (XSS Worm):** XSS payload calls this endpoint to extract viewer PII (viewerEmail) from discovered viewIds. Each POST leaks a viewer's email, document access, and team membership.
- **Chain 3 (Zod Recon → PII Extraction):** After Zod schema disclosure, attacker uses discovered viewIds to bulk-extract viewer email addresses.
- **Chain 12 (HSTS + Query Param):** With network position, attacker intercepts viewIds and uses this endpoint for PII extraction.

## Remediation
Add authentication (NextAuth `getServerSession`) to validate that the caller is authorized. If the endpoint is intended to be public (like `record_view.ts` and `record_click.ts`), remove the `include: { view }` clause that leaks protected viewer metadata, and consider routing through the analytics pipeline instead of direct PostgreSQL writes.
