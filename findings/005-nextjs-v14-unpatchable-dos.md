# Finding: Next.js v14 Unpatchable DoS (CVE-2026-23864 / CVE-2026-23869)

## Severity
HIGH

## CVSS Estimate
CVE-2026-23864: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H = 7.5
CVE-2026-23869: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H = 7.5

## Affected Code
- **Framework:** Next.js 14.2.35 (EOL since October 2025)
- **Type:** Framework-level RSC Flight protocol vulnerability
- **Scope:** All App Router pages accepting POST requests

## Description
Papermark runs on Next.js 14.2.35, which reached end-of-life in October 2025. Two unpatched CVEs affect the React Server Components (RSC) Flight protocol:

- **CVE-2026-23864 (BigInt Deserialization DoS):** Sending crafted HTTP POST requests with excessively large BigInt values in the RSC Flight protocol causes memory exhaustion on the server. CVSS 7.5.
- **CVE-2026-23869 (Cyclic Deserialization DoS / "React2DoS"):** Crafted cyclic Map/Set references in Flight protocol cause quadratic-time deserialization, pegging CPU for ~1 minute per request. CVSS 7.5.

No authentication is required for either attack. Any App Router page (in `app/`) accepts POST requests with RSC Flight protocol headers (`content-type: text/x-component`). The `middleware.ts:49` matcher excludes `/api/*` but does NOT exclude App Router pages from RSC Flight protocol handling.

**Patch status:** NO FIX AVAILABLE for Next.js 14.x. Both CVEs are fixed in React 19.0.5+ and Next.js 15.5.15+ / 16.2.3+. Next.js 14.x is EOL and will not receive backports.

## Proof of Concept
```bash
# Send crafted RSC Flight protocol POST to any App Router page
curl -X POST "https://papermark.app/view/some-link-id" \
  -H "content-type: text/x-component" \
  -d '{"largeBigInts": [...]}'  # Crafted to exhaust memory during deserialization
```

No authentication required. Single request causes ~1 minute of CPU/memory exhaustion. Repeated requests sustain a denial of service.

## Impact
An unauthenticated attacker can:
- Exhaust server memory with a single crafted request (CVE-2026-23864)
- Peg server CPU for ~1 minute per request (CVE-2026-23869)
- Sustain denial of service against the application with repeated requests
- Compound impact: both CVEs can be triggered simultaneously for worse outcomes

## Evidence
- `middleware.ts:49` — excludes `/api/`, but App Router pages remain exposed to RSC POST
- Papermark uses Next.js 14.2.35 (EOL since October 2025, confirmed via `package.json`)
- No `"use server"` directives found in App Router files, but the RSC Flight POST endpoint is always active at the framework level
- Web CVE intelligence confirmed no patch available for v14.x line
- Publicly accessible `app/` pages include viewer, auth, and marketing routes

## Methodology
- **First found by:** Web CVE intelligence (00-early-web-intel.md) identified both CVEs
- **Confirmed by:** Structural analysis (01-structural-analysis.md) confirmed reachability and EOL status
- **Confirmed by:** Advisory pairing (03-advisory.md) confirmed no fix available for v14.x

## Chain Potential
**Chainable.**
- **Chain 5 (EOL Middleware Bypass):** Unpatched DoS can be used to degrade service, potentially masking data exfiltration activity from other findings.
- **Chains with CVE-2026-23869:** Cyclic deserialization compounds with BigInt DoS for worse impact.

## Remediation
**Architectural:** Requires upgrading to Next.js 15.x+ (React 19.0.5+). This is a significant migration effort as Next.js 14.x to 15.x involves breaking changes. Short-term mitigations:
- Deploy a WAF rule to block requests with `content-type: text/x-component`
- Add middleware-level filtering for RSC-specific request patterns
- Monitor for excessive CPU/memory on App Router page POST requests
