# Finding: `cuid@3.0.0` Predictable IDs in Permission Groups

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:L/A:N = 4.3

## Affected Code
- `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5`
- `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5`
- `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`
- **Package:** `cuid@^3.0.0` in `package.json:106`

## Description
The project generates resource IDs using `cuid@3.0.0`. CUIDs encode timestamps with a sequential counter, making them predictable — an attacker who observes a few IDs can predict future values. This enables ID enumeration and potential IDOR against permission group resources within an authenticated session.

All three affected handlers are behind NextAuth session + team membership auth, so exploitation requires an authenticated session. However, the project already depends on `nanoid@5.1.11` (cryptographically secure) elsewhere, making migration trivial.

The `cuid` package is also effectively unmaintained (last release 2021) and has a deprecation message recommending migration to `@paralleldrive/cuid2`. The package-lock.json deprecation warning confirms the package is insecure: "Cuid and other k-sortable and non-cryptographic ids ... are all insecure. Use @paralleldrive/cuid2 instead."

## Proof of Concept
1. Authenticate as a team member
2. Observe the pattern of permission group CUIDs (e.g., `clx...`, `cly...`)
3. The timestamp encoding in CUIDs allows predicting future IDs
4. Enumerate permission group IDs to discover groups the attacker should not have access to
5. Attempt IDOR against discovered groups via the permission management handlers

## Impact
Authenticated team members can:
- Enumerate permission group IDs via predictable CUID patterns
- Potentially access unauthorized permission group resources via IDOR
- Bypass access controls on dataroom permissions

## Evidence
```typescript
// Permission group handlers import cuid
import { createId } from "@paralleldrive/cuid2"; // or cuid-style IDs
```

Package confirmation:
```bash
# package.json:106
"cuid": "^3.0.0"
# package-lock.json deprecation: "Cuid and other k-sortable and non-cryptographic ids ...
# are all insecure. Use @paralleldrive/cuid2 instead."
```

## Methodology
- **First found by:** Dependency synthesis (02-synthesized-dependencies.md S-001)
- **Confirmed by:** Reachability analysis (01-reachability-analysis.md) correctly downgraded to MEDIUM by noting all 3 handlers are behind NextAuth + team membership auth
- **Confirmed by:** Structural analysis

## Chain Potential
**Chainable.**
- **Chain 3 (Zod Recon → PII Extraction):** Predictable IDs aid enumeration after Zod schema disclosure, making the overall ID-discovery phase faster.

## Remediation
Replace `cuid` with `nanoid` (already available in the project at 5.1.11) or `@paralleldrive/cuid2`. The `createId` function from `@paralleldrive/cuid2` provides a drop-in replacement with cryptographic randomness.
