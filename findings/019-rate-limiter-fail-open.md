# Finding: Rate Limiter Fails Open When Redis is Unavailable

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N = 5.3

## Affected Code
- **File:** `ee/features/security/lib/ratelimit.ts:43-44`
- **Usage:** Auth rate limiting, billing rate limiting, view-recording rate limiting

## Description
The `checkRateLimit` function catches all errors from the Upstash rate limiter and returns `{ success: true, error: "Rate limiting unavailable" }`, which callers treat as a successful rate-limit pass. If Redis is down (network partition, outage, or DoS against the Redis endpoint), all rate-limited endpoints become unthrottled.

This is an availability vs. security tradeoff: legitimate users are not blocked during Redis issues, but the fail-open enables attackers to bypass all rate limits during Redis unavailability.

## Proof of Concept
1. Cause or observe a Redis outage (e.g., saturate Upstash rate limit endpoint, exploit a network partition)
2. The `checkRateLimit` function catches the error:
   ```typescript
   } catch (error) {
     console.error("Rate limiting error:", error);
     return { success: true, error: "Rate limiting unavailable" }; // fail open
   }
   ```
3. All rate-limited endpoints now return `success: true` for every request
4. Perform unlimited brute-force attempts against auth endpoints, unlimited billing operations, unlimited view-recording requests

## Impact
An attacker who can cause or observe a Redis outage can bypass all rate limits. This enables:
- Unlimited brute-force attempts against auth endpoints (amplified by F-025 — no auth rate limiting)
- Unlimited billing operations
- Unlimited view-recording requests
- When combined with F-020 (allowDangerousEmailAccountLinking), enables brute-force OAuth account linking

## Evidence
```typescript
// ee/features/security/lib/ratelimit.ts:31-46
export async function checkRateLimit(limiter, identifier) {
  try {
    const result = await limiter.limit(identifier);
    return { success: result.success, remaining: result.remaining };
  } catch (error) {
    console.error("Rate limiting error:", error);
    return { success: true, error: "Rate limiting unavailable" }; // ← fail open
  }
}
```

## Methodology
- **First found by:** Config analysis (00-ai-configs.md C-006)
- **Confirmed by:** Structural analysis

## Chain Potential
**Critical chain enabler.**
- **Chain 9 (Rate Limit → OAuth Account Linking):** Redis-down rate limit bypass + no auth rate limiting (F-025) + allowDangerousEmailAccountLinking (F-020) enables brute-force OAuth account takeover. Estimated $4K-$10K.

## Remediation
Change the error path to fail closed by default:
```typescript
return { success: false, error: "Rate limiting unavailable" };
```
Or implement a circuit-breaker that preserves the last-known rate-limit state during Redis outages.
