# Finding: cuid@3.0.0 Deprecated Non-Cryptographic ID for Permission Rows

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N = 3.0 (information disclosure: the ID structure is partially predictable, allowing brute-force enumeration of permission row IDs).

## Affected Code
- File: `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts`
- Line: 91 (`id: cuid()` — generates a cuid for each permission row)
- Function: `handle` (POST/PUT branch)
- Endpoint: `POST /api/teams/:teamId/datarooms/:id/groups/:groupId/permissions`

## Description
`cuid@3.0.0` is a non-cryptographic ID generator. The cuid format includes a timestamp prefix and a counter, which makes the IDs **partially predictable** — an attacker who knows the approximate creation time of a permission row can enumerate candidate IDs.

The deprecation note in the cuid 3.x changelog explicitly states that cuid 2.x is the recommended replacement for new code; cuid 1.x and 3.x are considered legacy. The papermark codebase uses cuid 3.x for permission row primary keys.

## Proof of Concept

### Pre-conditions
- The attacker knows the approximate creation time of a target permission row.
- The attacker can submit a request to the permissions endpoint and observe the ID validation.

### Step-by-step exploitation
1. Attacker learns that a permission row was created at approximately 2026-06-19 14:32:15 UTC.
2. The cuid timestamp is encoded in the first 8 characters (base36-encoded seconds since the cuid epoch).
3. The attacker iterates over the counter range (typically 0-1000 for a given second) and the fingerprint range (4 characters, base36).
4. The attacker submits a request with the guessed ID and observes the response. A 200/201 confirms a valid ID; a 404/400 confirms invalidity.
5. **Impact**: the attacker can enumerate permission row IDs and then chain into the permissions API to read/modify them.

## Impact
- **Information disclosure** — permission row IDs are partially predictable.
- **No direct exploit** — the attacker still needs to bypass the auth/role gate to read/modify the permission row.
- **Forward-risk** — if a future endpoint accepts a `permissionId` query parameter without a team/role check, the predictable IDs amplify the impact.

## Evidence
```ts
// pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:91
const upsertRows: PermissionUpsertRow[] = Object.entries(permissions).map(
  ([itemId, itemPermissions]) => ({
    id: cuid(),  // ← non-cryptographic ID
    itemId,
    itemType: itemPermissions.itemType,
    canView: Boolean(itemPermissions.view),
    canDownload: Boolean(itemPermissions.download),
  }),
);
```

## Methodology
- **M3 — SBOM reachability** (`methodology-raw/03-sbom-reachability.md` Dep S-016)
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` §2.5 — cuid deprecation)

## Chain Potential
- None directly. The finding is a defense-in-depth concern.

## Remediation
Migrate to `cuid@2.x` (or `@paralleldrive/cuid2`) for new permission row IDs. The cuid 2.x format uses a different encoding that is **not** based on a timestamp prefix, making the IDs non-predictable.

For existing rows, the migration can be done incrementally:
- New rows use cuid 2.x
- Old rows keep their cuid 1.x/3.x IDs
- The schema's `id` column type is `String` (cuid IDs are strings), so no schema change is required

Alternatively, use `crypto.randomUUID()` (Node 14.17+, browser support is universal) for new permission row IDs. This produces a 128-bit UUIDv4 that is cryptographically random and non-predictable.
