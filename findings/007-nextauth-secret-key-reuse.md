# Finding: NEXTAUTH_SECRET Reused Across 5 Security Domains (inc. SAML clientSecret → Response Forgery)

## Severity
MEDIUM (standalone design weakness) / HIGH (with default secret, self-hosted)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:H/A:N = 5.3 (MEDIUM) standalone
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H = 8.2 (HIGH) with default secret

## Affected Code
- `lib/auth/auth-options.ts:128` — SAML OAuth clientSecret = `process.env.NEXTAUTH_SECRET`
- `lib/auth/auth-options.ts:149` — SAML IdP client_secret = `process.env.NEXTAUTH_SECRET`
- `lib/jackson.ts:16-20` — Jackson encryption key via SHA-256(NEXTAUTH_SECRET)
- `lib/signing/download-token.ts:19` — HMAC-SHA256 download token key
- `lib/signing/access-token.ts:21` — HMAC-SHA256 access token key
- `lib/api/auth/token.ts:12` — API token hash preimage component

## Description
The `NEXTAUTH_SECRET` environment variable is used as the encryption/secret key in at least 5 distinct security contexts. A single secret compromise (leak, default value, or server breach) undermines all 5 simultaneously.

The SAML-specific reuse is the most critical: NEXTAUTH_SECRET is used as both the SAML OAuth clientSecret (`auth-options.ts:128`) and the SAML IdP client_secret (`auth-options.ts:149`). Combined with `allowDangerousEmailAccountLinking: true` on the SAML provider, knowing NEXTAUTH_SECRET enables forging SAML assertions that auto-link to victim accounts without confirmation — a SAML response forgery attack independent of JWT cookie forging.

## Proof of Concept
On a self-hosted deployment with default NEXTAUTH_SECRET (`my-superstrong-secret`):
1. Forge a SAML OAuth authorization code for the victim's email using the known clientSecret
2. Exchange the authorization code for OAuth tokens using the known client_secret
3. NextAuth's `allowDangerousEmailAccountLinking: true` auto-links the SAML identity to the existing user
4. The attacker now has full account access

## Impact
- **Standalone (MEDIUM):** Single secret compromise breaks JWT signing, SAML auth, SSO encryption, HMAC tokens, and API tokens simultaneously. No defense in depth.
- **With default secret (HIGH):** Direct SAML response forgery enabling account takeover on self-hosted deployments with SAML configured.

## Evidence
```typescript
// lib/auth/auth-options.ts — SAML OAuth clientSecret
clientSecret: process.env.NEXTAUTH_SECRET as string;  // line 128

// lib/jackson.ts — encryption key derivation
.update(process.env.NEXTAUTH_SECRET || "").digest("base64url").substring(0, 32);  // line 18

// lib/signing/download-token.ts — download token signing
const secret = process.env.NEXTAUTH_SECRET;  // line 19

// lib/signing/access-token.ts — access token signing
const secret = process.env.NEXTAUTH_SECRET;  // line 21
```

## Methodology
- **First found by:** Config analysis (00-ai-configs.md C-008) identified the key reuse pattern
- **Confirmed by:** Structural analysis verified each usage site
- **Extended by:** F-011c (SAML clientSecret subset) absorbed into this finding during dedup

## Chain Potential
**Critical chain enabler.**
- **Chain 6 (Default Secret → Full Compromise):** Reuse amplifies the impact of the default secret from "session forgery" to "complete cryptographic trust collapse."
- **Chain 10 (SAML Response Forgery):** Reuse of NEXTAUTH_SECRET as SAML clientSecret enables SAML-based account takeover, independent of JWT cookie forging.

## Remediation
Use separate, dedicated secrets for each security context. Create `SAML_CLIENT_SECRET`, `JACKSON_ENCRYPTION_KEY`, `SIGNING_TOKEN_SECRET`, and `API_TOKEN_SECRET` environment variables and use them in their respective contexts instead of reusing `NEXTAUTH_SECRET`.
