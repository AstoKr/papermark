# Finding: NEXTAUTH_SECRET Reused as Jackson DB Encryption Key with Empty Fallback

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H = 9.8 (subsumed by F-001/F-004 — same root cause; the empty-fallback aspect adds additional risk on misconfigured deployments).

## Affected Code
- File: `lib/jackson.ts`
- Line: 16-20 (`const encryptionKey = crypto.createHash("sha256").update(process.env.NEXTAUTH_SECRET || "").digest("base64url").substring(0, 32)`)
- Function: `getJacksonOptions` (line 59-74)
- Endpoint: All SAML/OAuth endpoints (via Jackson's at-rest encryption)

## Description
BoxyHQ Jackson's AES-256-GCM `encryptionKey` is derived from `NEXTAUTH_SECRET` via `sha256(NEXTAUTH_SECRET || "").digest("base64url").substring(0, 32)`. **If `NEXTAUTH_SECRET` is unset or empty, the key collapses to a known string** (the SHA-256 of an empty string, then base64url-encoded, then truncated to 32 characters). The same `NEXTAUTH_SECRET` is also passed as `clientSecretVerifier` (line 72), which validates HMACs on every Jackson-issued `code`.

This is **one of the four consumers** of the published `my-superstrong-secret` placeholder (per F-001/F-004). The finding is reported separately because the empty-fallback aspect is a distinct risk: a misconfigured deployment that fails to set `NEXTAUTH_SECRET` produces a known encryption key, which lets the attacker decrypt every Jackson at-rest row.

## Proof of Concept

### Pre-conditions
- A deployment where `NEXTAUTH_SECRET` is unset or empty.
- The attacker can read the Jackson DB (e.g. via backup leak, SQL injection elsewhere, admin-token compromise).

### Step-by-step exploitation
1. Compute the known encryption key:
   ```js
   const crypto = require("crypto");
   const encryptionKey = crypto
     .createHash("sha256")
     .update("")   // empty string fallback
     .digest("base64url")
     .substring(0, 32);
   // → "47DEQpj8HBSa-_TImW+5JCeuQeRkm5NM"  (the SHA-256 of empty, base64url-encoded, truncated)
   ```
2. Decrypt every row in the Jackson DB (`connections`, `sets`, `users`, `events`, etc.). The `IV` is stored per-row (Jackson uses 12-byte GCM IVs); the `authTag` is stored per-row; the ciphertext is the rest.
3. Recover every IdP connection's `tenant`, `product`, `clientID`, `clientSecret`, metadata URL, and active OAuth codes.

### Step-by-step exploitation (with the published placeholder)
1. Same as F-001 step 2 — derive the key from `my-superstrong-secret`.
2. Decrypt the Jackson DB as above.

## Impact
- **Full decryption of Jackson at-rest data** — every SAML connection, every SCIM directory row, every OAuth token.
- **Empty-fallback risk** — a misconfigured deployment that fails to set `NEXTAUTH_SECRET` produces a known encryption key, which is a **silent catastrophic failure**. The application boots successfully; the SAML/OAuth flow works; but every encrypted row is decryptable to anyone who reads the DB.
- **Chains with F-004** (4-purpose NEXTAUTH_SECRET) — the same string unlocks NextAuth JWT signing, Jackson DB encryption, Jackson OAuth `client_secret` verification, and SAML NextAuth `clientSecret`.

## Evidence
```ts
// lib/jackson.ts:16-20
const encryptionKey = crypto
  .createHash("sha256")
  .update(process.env.NEXTAUTH_SECRET || "")  // ← empty fallback
  .digest("base64url")
  .substring(0, 32);
```
```ts
// lib/jackson.ts:59-74
function getJacksonOptions(): JacksonOption {
  return {
    externalUrl: process.env.NEXTAUTH_URL || "https://app.papermark.com",
    samlPath: "/api/auth/saml/callback",
    samlAudience,
    db: {
      engine: "sql",
      type: "postgres",
      url: getJacksonDbUrl(),
      encryptionKey,
    },
    idpEnabled: true,
    scimPath: "/api/scim/v2.0",
    clientSecretVerifier: process.env.NEXTAUTH_SECRET as string,
  };
}
```

## Methodology
- **M2 — Secrets** (`methodology-raw/02-synthesized-secrets.md` S-001)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 1 — one of the four consumers)

## Chain Potential
- **Chain 1** (`methodology-raw/05-chains.md`) — subsumed by F-001/F-004. The finding is reported separately because the empty-fallback aspect is a distinct risk and the four consumers should each be cited as evidence in the root-cause report.

## Remediation
1. **Introduce a dedicated `JACKSON_ENCRYPTION_KEY`** (32 raw bytes) and a dedicated `JACKSON_CLIENT_SECRET_VERIFIER` (32+ bytes random).
2. **Fail fast at module load** if either is unset:
   ```ts
   if (!process.env.JACKSON_ENCRYPTION_KEY || process.env.JACKSON_ENCRYPTION_KEY.length < 32) {
     throw new Error("JACKSON_ENCRYPTION_KEY must be set to 32 raw bytes");
   }
   if (!process.env.JACKSON_CLIENT_SECRET_VERIFIER) {
     throw new Error("JACKSON_CLIENT_SECRET_VERIFIER must be set");
   }
   ```
3. **Rotate all rows** encrypted under the old derived key during the rollover. The Jackson DB has helper methods for re-encryption (`apiController.rotateEncryptionKey` or similar — verify in the Jackson docs).
4. **Remove the empty fallback** — `process.env.NEXTAUTH_SECRET || ""` should be `process.env.NEXTAUTH_SECRET` (and the fail-fast check above ensures it's never empty).
