# Finding: Trigger.dev Project ID Hardcoded in Config

## Severity
LOW

## CVSS Estimate
N/A — informational finding (not a credential)

## Affected Code
- **File:** `trigger.config.ts:7`
- **Value:** `project: "proj_plmsfqvqunboixacjjus"`

## Description
The Trigger.dev project identifier (`proj_plmsfqvqunboixacjjus`) is hardcoded directly in `trigger.config.ts`. This is a project reference, not a bearer token or API key — Trigger.dev uses `TRIGGER_SECRET_KEY` from the environment for authentication. However, hardcoding the project ID couples configuration to a specific workspace and provides marginal recon value to attackers.

## Proof of Concept
```bash
# Extract the project ID from the source code
grep -r "project:" trigger.config.ts
# project: "proj_plmsfqvqunboixacjjus"
```

## Impact
Marginal recon value. An attacker can identify the Trigger.dev project ID for targeted reconnaissance. Not a credential — Trigger.dev auth uses `TRIGGER_SECRET_KEY` from environment variables.

## Evidence
```typescript
// trigger.config.ts:7
project: "proj_plmsfqvqunboixacjjus"
```

## Methodology
- **First found by:** Secrets synthesis (02-synthesized-secrets.md S-003)
- **AI-only finding:** gitleaks and trufflehog missed it (not a credential pattern)
- **Confidence:** MEDIUM — intentionally exposed but offers marginal recon value

## Chain Potential
Standalone finding. No chain potential — not a credential, not exploitable.

## Remediation
Move the project ID to an environment variable for better separation of configuration from code:
```typescript
project: process.env.TRIGGER_PROJECT_ID,
```
