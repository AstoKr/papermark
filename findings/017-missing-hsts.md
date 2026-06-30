# Finding: Missing HSTS Header

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N = 4.3

## Affected Code
- **File:** `next.config.mjs:190-312` (global headers block)
- **Scope:** All routes globally

## Description
The application does not set a `Strict-Transport-Security` header in any response. While Vercel's edge network may add HSTS at the platform level for `*.vercel.app` domains, custom domains and non-Vercel deployments receive no HSTS policy. Without HSTS, an attacker with a network position (e.g., on public Wi-Fi) can perform SSL stripping attacks against users accessing custom domains or direct deployments.

Grep confirms zero hits for `Strict-Transport-Security` across the entire codebase.

## Proof of Concept
```bash
# Check response headers — no Strict-Transport-Security present
curl -sI https://papermark.app | grep -i strict-transport-security
# Empty — no HSTS header
```

An attacker on the same Wi-Fi can use `sslstrip` to downgrade the victim's connection from HTTPS to HTTP. Without HSTS, the browser does not enforce HTTPS and accepts the downgraded HTTP response.

## Impact
Enables SSL stripping on custom domains and non-Vercel deployments:
- Users on HTTP connections or behind an active MITM receive no browser-enforced HTTPS upgrade
- First-time visitors or users who previously visited over HTTP can have their connections downgraded
- Amplifies F-011 (query-param secret interception) — without HSTS, the `REVALIDATE_TOKEN` in query parameters can be intercepted on downgraded connections

## Evidence
No `Strict-Transport-Security` header is present in the `headers()` function at `next.config.mjs:190-312`. Zero hits for `Strict-Transport-Security` across the entire codebase.

## Methodology
- **First found by:** Config analysis only (00-ai-configs.md C-004)

## Chain Potential
**Amplifier.**
- **Chain 12 (HSTS + Query Param Intercept):** Without HSTS, SSL stripping is possible on custom domains. An attacker with network position can downgrade HTTPS to HTTP and capture the `REVALIDATE_TOKEN` from revalidation URL query parameters (F-011). Enables mass link invalidation and enumeration.

## Remediation
Add to the global `/:path*` header block in `next.config.mjs`:
```javascript
{ key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" }
```
