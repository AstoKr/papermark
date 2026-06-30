# Finding: `progress-token.ts` — Unauthenticated Trigger.dev Token Generation

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N = 7.5

## Affected Code
- **File:** `pages/api/progress-token.ts:4-27`
- **Endpoint:** `GET /api/progress-token?documentVersionId=<id>`
- **Sink:** `lib/utils/generate-trigger-auth-token.ts:2-11` → `auth.createPublicToken()`

## Description
The progress token endpoint generates a Trigger.dev public access token for any `documentVersionId` passed as a query parameter, without any authentication or authorization check. The token grants read access to Trigger.dev runs tagged with `version:{documentVersionId}`. Anyone who can discover or guess a valid `documentVersionId` can obtain a 15-minute public access token. While CUIDs are not trivially guessable, they can be leaked through client-side code, referrer headers, logs, or browser history.

The same `generateTriggerPublicAccessToken` function is called from three authenticated EE routes (`monitor-token`, `retry-archive`, `freeze`) — all behind NextAuth session checks. This endpoint is the **only unguarded path** to the same function.

## Proof of Concept
```bash
# No authentication required — just a documentVersionId
curl -s "https://papermark.app/api/progress-token?documentVersionId=clx_example_cuid"
# Returns: {"publicAccessToken": "tr_trigger_dev_token"}
```

The returned token can then be used to query the Trigger.dev API:
```bash
curl -H "Authorization: Bearer tr_trigger_dev_token" \
  "https://api.trigger.dev/api/v1/runs?tag=version:clx_example_cuid"
```

## Impact
An attacker who discovers a `documentVersionId` obtains a 15-minute Trigger.dev public access token granting read access to runs for that document version. This exposes internal processing data, document conversion status, workflow execution metadata, and potentially sensitive data flowing through document processing workflows.

## Evidence
```typescript
// pages/api/progress-token.ts:4-27 — FULL FILE, zero auth
export default async function handle(req, res) {
  if (req.method !== "GET") return res.status(405).json({ error: "Method not allowed" });
  const { documentVersionId } = req.query;
  if (!documentVersionId || typeof documentVersionId !== "string")
    return res.status(400).json({ error: "Missing documentVersionId" });
  const publicAccessToken = await generateTriggerPublicAccessToken(`version:${documentVersionId}`);
  return res.status(200).json({ publicAccessToken });
}
```

```typescript
// lib/utils/generate-trigger-auth-token.ts:2-11 — token creation
export async function generateTriggerPublicAccessToken(tag: string) {
  return auth.createPublicToken({
    scopes: { read: { tags: [tag] } },
    expirationTime: "15m",
  });
}
```

## Methodology
- **First found by:** Structural Analysis (01-structural-analysis.md) — Finding 4
- **Confirmed by:** SAST synthesis (02-synthesized-sast.md S-003)
- **Missed by:** AI SAST and stock semgrep rules

## Chain Potential
**Chainable.**
- **Chain 5 (EOL Middleware Bypass):** After bypassing middleware via `.rsc` suffix, attacker enumerates document version IDs and calls this endpoint.
- **Chain 7 (CORS TUS → Token Leak):** CORS misconfiguration on TUS viewer allows reading upload metadata containing document version IDs, which feed into this endpoint.

## Remediation
Add authentication (NextAuth `getServerSession`) to the `progress-token.ts` handler. The function should only be callable by authenticated team members who have access to the associated document. Apply rate limiting as a defense-in-depth measure.
