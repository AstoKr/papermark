# Finding: NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY Default Value

## Severity
HIGH (self-hosted only)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N = 7.5

## Affected Code
- **Source:** `.env.example:68`
- **Key derivation:** `lib/utils.ts:614-628`
- **Usage:** AES-256-CTR encryption/decryption of document-level passwords

## Description
`NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret` is a static example value for AES-256-CTR document password encryption, used in `lib/utils.ts:614-628` to derive the encryption key for `encryptEncrpytedPassword()` and `decryptEncrpytedPassword()`.

If unchanged from the default, any document password encrypted in the database is decryptable by anyone who knows the source (anyone who reads `.env.example` — a public GitHub file). The key derivation is unsalted SHA-256 (no KDF, no salt, no iteration count), truncated to 32 bytes for AES-256-CTR. This means:
1. The default key is publicly known
2. Even with a custom key, the deterministic SHA-256 derivation without a KDF makes the derived key weaker than it should be
3. AES-256-CTR provides no authentication — ciphertext tampering is undetectable

## Proof of Concept
1. Read the default value from `.env.example:68`: `my-superstrong-document-secret`
2. Derive the AES key: `SHA-256("my-superstrong-document-secret").substring(0, 32)`
3. Extract encrypted document passwords from the database
4. Decrypt using AES-256-CTR with the derived key and the stored IV (prepended to ciphertext)

## Impact
On self-hosted deployments using the default value, all document-level passwords are decryptable by anyone who knows the public default. Even on deployments with a custom key, the unsalted SHA-256 KDF means there is no computational barrier to offline brute-force if the key is ever partially compromised.

## Evidence
```typescript
// lib/utils.ts:614-628 — key derivation for document password encryption
// NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY from .env.example:68
// SHA-256 hash truncated to 32 bytes = AES-256-CTR key
// No salt, no iterations, no KDF
```

```bash
# .env.example:68
NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret
```

## Methodology
- **First found by:** Secrets synthesis (02-synthesized-secrets.md S-002)
- **Confirmed by:** Git history synthesis confirmed no override in tracked commits
- **Missed by:** gitleaks and trufflehog (tools do not scan `.env.example`)

## Chain Potential
**Chainable.**
- **Chain 6 (Default Secret → Full Compromise):** Combined with default NEXTAUTH_SECRET, enables complete cryptographic trust collapse. Document password decryption is one of 5 compromised security domains.

## Remediation
Replace the `.env.example` value with an empty string or a clear instruction to generate a unique key. Use a proper KDF (e.g., PBKDF2 or Argon2id) instead of raw SHA-256. Add an integrity check (HMAC or GCM) for authenticated encryption instead of CTR mode.
