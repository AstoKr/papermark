# Finding: `revalidate.ts` — Shared Secret in URL Query Parameter

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N = 5.3

## Affected Code
- **File:** `pages/api/revalidate.ts:13-14`
- **Endpoint:** `GET /api/revalidate?secret=<token>&linkId=<id>`
- **Secret:** `REVALIDATE_TOKEN` from environment, passed via `req.query.secret`

## Description
The revalidation endpoint authenticates via `req.query.secret !== process.env.REVALIDATE_TOKEN`. Passing the shared secret as a URL query parameter exposes it to leakage through:
1. Server access logs (Vercel, CDN, reverse proxy)
2. `Referer` header when the page loads external resources
3. Browser history if triggered from client-side navigation

If the `REVALIDATE_TOKEN` is leaked, an attacker can force mass revalidation of all team links (DoS via cache stampede) and enumerate which links exist for a given team/document. The endpoint accepts `teamId`, meaning a leaked secret allows enumerating all links for a team.

## Proof of Concept
```bash
# Secret is passed as a query parameter — visible in logs
curl "https://papermark.app/api/revalidate?secret=TOKEN_HERE&linkId=link_cuid"
# Leaks via Referer header when page loads external resources
# Leaks into server access logs (Vercel, CDN, proxy)
```

## Impact
Secret leakage enables:
- Mass ISR cache invalidation for all links of a team (cache stampede DoS)
- Enumeration of which links exist for a given team or document
- Link existence oracle via response timing/status

## Evidence
```typescript
// pages/api/revalidate.ts:13-14 — secret in query param
if (req.query.secret !== process.env.REVALIDATE_TOKEN) {
  return res.status(401).json({ message: "Invalid token" });
}

// pages/api/revalidate.ts:48-100 — mass revalidation capability
if (teamId) {
  const allLinks = await prisma.link.findMany({ where: { teamId, isArchived: false, deletedAt: null } });
  for (const link of allLinks) { await res.revalidate(...); }
}
```

## Methodology
- **First found by:** Structural Analysis (01-structural-analysis.md) — Finding 7
- **Confirmed by:** SAST synthesis (02-synthesized-sast.md S-008)
- **Confirmed by:** Custom semgrep rule `papermark-secret-in-query-param` matched at line 14

## Chain Potential
**Chainable.**
- **Chain 12 (HSTS + Query Param Intercept):** Missing HSTS enables SSL stripping — attacker on same network intercepts revalidation URL and captures the secret. With the captured token, attacker performs mass link invalidation.

## Remediation
Move the authentication token from the query parameter to the `Authorization` header or a signed request mechanism:
```typescript
const authHeader = req.headers.authorization;
if (authHeader !== `Bearer ${process.env.REVALIDATE_TOKEN}`) {
  return res.status(401).json({ message: "Invalid token" });
}
```
