# Finding: `feedback/index.ts` — Unauthenticated Feedback Response Recording

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N = 5.3

## Affected Code
- **File:** `pages/api/feedback/index.ts:9-56`
- **Endpoint:** `POST /api/feedback`

## Description
This endpoint records feedback responses with no authentication. It validates that the `feedbackId` and `viewId`+`linkId` pair exist in the database, but any unauthenticated client with knowledge of valid IDs can submit arbitrary feedback responses. The `answer` field is stored as JSON data in the database without sanitization, enabling content injection.

## Proof of Concept
```bash
# No authentication required — just valid feedbackId and viewId
curl -X POST "https://papermark.app/api/feedback" \
  -H "Content-Type: application/json" \
  -d '{"feedbackId":"valid_feedback_id","viewId":"valid_view_id","answer":"injected data"}'
```

## Impact
An unauthenticated attacker can:
- Inject arbitrary feedback response data into the database
- Pollute analytics that team members view in their dashboards
- Potentially inject unsanitized content via the `answer` field into the JSON `data` column

## Evidence
```typescript
// pages/api/feedback/index.ts:9-56 — no auth
const { answer, feedbackId, viewId } = req.body;
// ... validation that feedbackId and viewId exist ...
await prisma.feedbackResponse.create({
  data: {
    feedbackId,
    viewId,
    data: { ...(feedback.data as { question: string; type: string }), answer },
  },
});
```

## Methodology
- **First found by:** Structural Analysis (01-structural-analysis.md) — Finding 6
- **Confirmed by:** SAST synthesis (02-synthesized-sast.md S-007)
- **Confirmed by:** Custom semgrep rule `papermark-no-auth-prisma-write` matched at line 46

## Chain Potential
**Chainable.**
- **Chain 3 (Zod Recon → PII Extraction):** After schema disclosure, attacker can precisely target this endpoint alongside the conversations and record_reaction endpoints for data injection.

## Remediation
Add authentication (NextAuth `getServerSession`) to validate the caller. If intended as a public endpoint, remove direct PostgreSQL write in favor of the analytics pipeline pattern used by `record_view.ts` and `record_click.ts`. Sanitize the `answer` field before storage.
