# Finding: No Rate Limiting on Authentication Endpoints

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N = 4.3 (defense-in-depth)

## Affected Code
- **File:** `pages/api/auth/[...nextauth].ts` (NextAuth configuration)

## Description
The NextAuth.js authentication endpoints do not implement rate limiting on login attempts, email verification requests, or password reset flows. This allows brute-force attacks against user credentials, email enumeration, and mass email-sending via the SMTP relay.

While the `ee/features/security/lib/ratelimit.ts` utility exists for rate limiting, it is not imported or used in the NextAuth configuration. The custom semgrep rule `papermark-nextauth-no-rate-limit` confirmed the absence of any rate limiter import.

## Proof of Concept
```bash
# Brute-force login attempts — no rate limiting
for password in $(cat passwords.txt); do
  curl -X POST "https://papermark.app/api/auth/callback/credentials" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "email=victim@example.com&password=$password"
done
# All requests are processed — no throttling
```

## Impact
Defense-in-depth gap. Enables:
- Brute-force credential attacks against login endpoints
- Email enumeration via timing or response differences
- Mass email-sending via password reset flows (if email provider relay is used)
- The impact is partially mitigated by the fail-open rate limiter (F-019) for other endpoints, but auth endpoints have no rate limiting at all

## Evidence
```typescript
// pages/api/auth/[...nextauth].ts:5
import NextAuth, { type NextAuthOptions } from "next-auth"
// No rate limiter imported — no @upstash/ratelimit, no express-rate-limit
```

Custom semgrep rule `papermark-nextauth-no-rate-limit` confirmed zero rate limiter imports in the NextAuth configuration file.

## Methodology
- **First found by:** AI SAST (00-ai-sast.md)
- **Confirmed by:** Custom semgrep rule `papermark-nextauth-no-rate-limit`
- **Confirmed by:** Structural analysis (01-structural-analysis.md)

## Chain Potential
**Chainable.**
- **Chain 9 (Rate Limit → OAuth Account Linking):** No rate limiting on auth endpoints + allowDangerousEmailAccountLinking (F-020) + rate limiter fail-open (F-019) enables brute-force OAuth account takeover.

## Remediation
Import the `checkRateLimit` utility from `ee/features/security/lib/ratelimit.ts` and apply rate limiting to credential-based authentication endpoints. At minimum, rate-limit failed login attempts per email/IP.
