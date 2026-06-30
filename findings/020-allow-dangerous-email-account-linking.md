# Finding: `allowDangerousEmailAccountLinking: true` on 3 OAuth Providers

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N = 5.3

## Affected Code
- **File:** `lib/auth/auth-options.ts:35` (Google provider)
- **File:** `lib/auth/auth-options.ts:55` (LinkedIn provider)
- **File:** `lib/auth/auth-options.ts:130` (SAML provider)

## Description
The `allowDangerousEmailAccountLinking: true` flag is set on Google, LinkedIn, and SAML providers. This allows NextAuth to automatically link a new OAuth account to an existing user if the email addresses match, without any confirmation step.

An attacker who can sign up with the victim's email on a provider the victim hasn't used yet could gain access to the victim's account. This is compounded by:
- NEXTAUTH_SECRET being reused as the SAML client secret (F-007), reducing attacker uncertainty about SAML flow parameters
- Rate limiter failing open (F-019), enabling brute-force attempts without throttling

## Proof of Concept
1. Identify a target victim's email address
2. Determine which OAuth provider the victim has NOT yet linked to their Papermark account
3. Register an account on that provider using the victim's email address
4. Initiate OAuth login through the unlinked provider
5. NextAuth's `allowDangerousEmailAccountLinking: true` automatically links the new provider account to the existing user if emails match
6. The attacker's session is now linked to the victim's Papermark account

## Impact
If an attacker controls any of the configured OAuth providers with the victim's email address, the attacker can sign in and automatically be linked to the existing Papermark account. This enables account takeover without requiring the victim's password.

## Evidence
```typescript
// lib/auth/auth-options.ts — three providers with the flag set
GoogleProvider({ ..., allowDangerousEmailAccountLinking: true }),     // line 35
LinkedInProvider({ ..., allowDangerousEmailAccountLinking: true }),   // line 55
{ id: "saml", ..., allowDangerousEmailAccountLinking: true },        // line 130
```

## Methodology
- **First found by:** Config analysis (00-ai-configs.md C-007)
- **Confirmed by:** Structural analysis via code read

## Chain Potential
**Chainable.**
- **Chain 9 (Rate Limit → OAuth Account Linking):** Rate limiter fail-open (F-019) + no auth rate limit (F-025) + allowDangerousEmailAccountLinking enables brute-force OAuth account takeover.
- **Chain 10 (SAML Response Forgery):** With default NEXTAUTH_SECRET (F-002) reused as SAML clientSecret (F-007), SAML allowDangerousEmailAccountLinking enables auto-linking forged SAML assertions.

## Remediation
Remove `allowDangerousEmailAccountLinking` where not needed, or implement an email verification step before linking accounts. Set it to `false` on providers where account linking confirmation is desired.
