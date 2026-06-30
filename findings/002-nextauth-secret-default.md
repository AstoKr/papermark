# Finding: NEXTAUTH_SECRET Default Value

## Severity
CRITICAL (self-hosted deployments only)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H = 10.0

## Affected Code
- **File:** `.env.example:1`
- **Value:** `NEXTAUTH_SECRET=my-superstrong-secret`

## Description
The `NEXTAUTH_SECRET` environment variable in `.env.example` is set to a static placeholder value `my-superstrong-secret`. This is the root cryptographic secret used across five security domains: NextAuth JWT signing (session forgery), SAML OAuth client secret, Jackson (SSO) encryption key derivation, HMAC-signed download/access tokens, and API token hashing.

Any self-hosted deployment that copies `.env.example` to production without changing this value inherits a publicly known secret from a publicly readable GitHub file. The published web search confirmed there is existing tooling (Embrace The Red) that can forge NextAuth JWE session cookies from a known `NEXTAUTH_SECRET`.

Vercel-hosted deployments using Vercel's system environment variables are NOT affected by this specific finding (they have auto-generated secrets), though they still have the key reuse problem documented separately.

## Proof of Concept
1. **Identify a self-hosted papermark instance** — scan for instances that have not overridden the default.
2. **Forge a NextAuth JWE session cookie** using the Embrace The Red cookie-minting toolkit with `my-superstrong-secret`:
   - The tool uses HKDF with the cookie name as salt (per NextAuth's JWE encryption scheme)
   - Forges a valid `next-auth.session-token` cookie for any user ID
   - Result: A valid session cookie that bypasses all authentication
3. **Set the forged cookie in the browser** and access papermark as any user. Access all team data, documents, datarooms, and settings.

## Impact
Complete cryptographic trust collapse on self-hosted deployments that did not override the default. The attacker can:
- Forge NextAuth JWT session cookies (impersonate any user)
- Forge SAML IdP responses (account takeover on SAML-configured instances)
- Decrypt SAML/SCIM encrypted data in the database
- Forge HMAC-signed download/access tokens (document access bypass)
- Compute API token hash preimages (API credential recovery)

## Evidence
```bash
# .env.example:1
NEXTAUTH_SECRET=my-superstrong-secret
```

The secret is referenced in:
- `lib/middleware/app.ts:56` — JWT token verification
- `lib/auth/auth-options.ts:128` — SAML OAuth clientSecret
- `lib/auth/auth-options.ts:149` — SAML IdP client_secret
- `lib/jackson.ts:16-20` — AES-256-GCM encryption key via SHA-256(NEXTAUTH_SECRET)
- `lib/signing/download-token.ts:19` — HMAC-SHA256 download token signing
- `lib/signing/access-token.ts:21` — HMAC-SHA256 access token signing
- `lib/api/auth/token.ts:12` — API token hash preimage component

## Methodology
- **First found by:** Secrets synthesis (02-synthesized-secrets.md S-001)
- **Confirmed by:** Web search for published exploitation tooling (03-web-secrets.md)
- **Extended by:** Config analysis (00-ai-configs.md C-008) added key reuse context
- **Missed by:** gitleaks and trufflehog (tools do not scan `.env.example` for placeholder credentials)

## Chain Potential
**Critical chain enabler.**
- **Chain 6 (Default Secret → Full Compromise):** Single default value enables session impersonation, token forgery, and data decryption. Estimated $10K-$25K bounty.
- **Chain 10 (SAML Response Forgery):** NEXTAUTH_SECRET reused as SAML clientSecret + `allowDangerousEmailAccountLinking` enables SAML-based account takeover. Estimated $8K-$18K bounty.

## Remediation
Replace the `.env.example` value with an empty string or a clear instruction to generate a unique secret. Add deployment documentation warning about the consequences of using the default value. Add a startup check that warns if `NEXTAUTH_SECRET` is set to the default value.
