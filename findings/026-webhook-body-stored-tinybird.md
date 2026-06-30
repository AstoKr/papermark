# Finding: Webhook Event Request/Response Body Stored in Tinybird

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:U/C:L/I:N/A:N = 2.7

## Affected Code
- **File:** `app/api/webhooks/callback/route.ts:43-44`

## Description
Webhook event request and response bodies are decoded from base64-encoded QStash payloads and stored in Tinybird. If a webhook destination returns data containing sensitive tokens (API keys, session tokens, credentials), those could be exposed to team members viewing webhook logs in the admin UI.

## Proof of Concept
1. Configure a webhook that points to a service that returns sensitive data in responses
2. The webhook response body is base64-decoded and stored in Tinybird analytics
3. Team members viewing webhook event logs in the admin UI can see the full response body

## Impact
If a configured webhook destination returns sensitive data in responses (e.g., OAuth tokens, API keys, internal service data), that information becomes visible to team members with access to webhook logs. Requires both a webhook returning sensitive data AND team member access to logs.

## Evidence
```typescript
// app/api/webhooks/callback/route.ts:43-44
// Decodes and stores webhook response data in Tinybird
```

## Methodology
- **First found by:** AI SAST (00-ai-sast.md)
- **Confirmed by:** Structural analysis
- **Listed as:** NC-002 in web synthesis (confirmed)

## Chain Potential
Standalone finding. Limited chain potential.

## Remediation
Add a field-level filtering mechanism for webhook data stored in Tinybird. Strip or mask fields that look like secrets (tokens, keys, passwords). Consider truncating response bodies to reduce storage of potentially sensitive data.
