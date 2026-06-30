# Finding: Zod Validation Error Information Disclosure

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N = 4.3

## Affected Code
9 endpoints returning `error.format()` or `error.flatten()`:

| Endpoint | File | Line |
|----------|------|------|
| Webhook services | `pages/api/webhooks/services/[...path]/index.ts` | 236 |
| Team agreements | `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts` | 67 |
| Team agreements list | `pages/api/teams/[teamId]/agreements/index.ts` | 135 |
| Dataroom folders bulk | `pages/api/teams/[teamId]/datarooms/[id]/folders/bulk.ts` | 75 |
| Team folders bulk | `pages/api/teams/[teamId]/folders/bulk.ts` | 63 |
| EE conversations toggle | `ee/features/conversations/api/toggle-conversations-route.ts` | 44 |
| EE dataroom group invite | `ee/features/dataroom-invitations/api/group-invite.ts` | 47 |
| EE dataroom link invite | `ee/features/dataroom-invitations/api/link-invite.ts` | 48 |
| Bulk import links | `lib/api/links/bulk-import.ts` | 125 |

## Description
Several API endpoints return detailed Zod validation error messages to the client, including validation rules, expected values, and internal field names. The `error.format()` method produces a structured error object with `_errors` arrays and nested field details. The `error.flatten()` method returns a flat structure with field names and error messages.

This leaks:
- Schema structure (field names, types, nesting)
- Validation constraints (regex patterns, string lengths, enum values)
- Internal implementation details (nested object paths, union discrimination logic)

This information enables an attacker to craft precisely-valid requests, significantly reducing the effort needed to enumerate valid resource identifiers and bypass validation logic.

## Proof of Concept
```bash
# Send malformed request to any affected endpoint
curl -X POST "https://papermark.app/api/webhooks/services/webhook/create" \
  -H "Content-Type: application/json" \
  -d '{"invalid": true}'

# Response leaks schema structure:
# {
#   "_errors": [],
#   "url": { "_errors": ["Expected string, received boolean"] },
#   "eventType": { "_errors": ["Invalid enum value"] },
#   "secret": { "_errors": ["Required"] }
# }
```

## Impact
Enables targeted fuzzing and API recon by:
- Revealing field names, types, and expected formats
- Disclosing enum values and regex validation patterns
- Exposing nested schema structure for precise request crafting
- Reducing effort to find injection points and bypass validation

## Evidence
```typescript
// pages/api/webhooks/services/[...path]/index.ts:232-238
return res.status(400).json({
  error: "Invalid request body",
  details: validationResult.error.format(),  // leaks full schema structure
});
```

## Methodology
- **First found by:** AI SAST (00-ai-sast.md)
- **Confirmed by:** Structural analysis (01-structural-analysis.md)
- **Confirmed by:** Custom semgrep rule `papermark-zod-error-format-disclosure` matched 9 endpoints

## Chain Potential
**Chainable.**
- **Chain 3 (Zod Recon → PII Extraction):** Schema disclosure enables precise crafting of IDs for the conversations, record_reaction, and feedback zero-auth endpoints. The disclosed schema structure tells the attacker exactly what fields to enumerate.
- **Chain 12 (HSTS + Query Param):** Schema disclosure helps understand ID formats for revalidation enumeration.

## Remediation
Replace `error.format()` and `error.flatten()` with a sanitized error response that reveals only the minimal information needed for API consumers:
```typescript
return res.status(400).json({
  error: "Validation failed",
  message: "One or more fields are invalid",
});
```
Or sanitize the error output to strip schema structure details while keeping user-facing messages.
