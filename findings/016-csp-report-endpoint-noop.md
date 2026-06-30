# Finding: CSP Report Endpoint is a No-Op

## Severity
MEDIUM (blocks detection)

## CVSS Estimate
N/A — detection-impairment finding (no direct CVSS)

## Affected Code
- **File:** `app/api/csp-report/route.ts:7`
- **Endpoint:** `POST /api/csp-report`

## Description
The CSP report ingestion endpoint at `/api/csp-report` receives POST requests from browsers but discards all reports. The `console.log` line that would record violations is commented out, and there is no storage, forwarding, or alerting logic. The infrastructure for CSP violation monitoring is wired up in `next.config.mjs` (via the `Report-To` header and `report-to csp-endpoint;` directive) but the actual ingestion handler is a no-op.

## Proof of Concept
```bash
# Send a CSP violation report
curl -X POST "https://papermark.app/api/csp-report" \
  -H "Content-Type: application/json" \
  -d '{"csp-report":{"document-uri":"https://papermark.app/","violated-directive":"script-src","blocked-uri":"https://evil.com/exploit.js"}}'

# Response: {"success": true}
# But the report is silently discarded — no log, no storage, no alert
```

## Impact
XSS exploitation generates no security alerts. CSP violation reports sent by browsers when the report-only CSP is violated are silently discarded. The security team has zero visibility into attempted or successful XSS exploitation. The reporting infrastructure exists declaratively but produces no actionable data.

## Evidence
```typescript
// app/api/csp-report/route.ts:3-15
export async function POST(request: Request) {
  const report = await request.json();
  // console.log("CSP Violation:", report);  // ← commented out
  // No storage, no forwarding, no alerting
  return NextResponse.json({ success: true });
}
```

## Methodology
- **First found by:** Config analysis only (00-ai-configs.md C-003)

## Chain Potential
**Detection-impairment amplifier.**
- **Chain 11 (CSP Report-Only → Undetectable XSS Worm):** CSP is not enforced (F-015) and violation reports are discarded (F-016). Combined, this means the stored XSS worm (F-003) can propagate indefinitely without triggering any automated security alert. This extends mean time to detection from hours/days to potentially indefinite.

## Remediation
Implement CSP report logging — write to a database, forward to an external SIEM/logging service, or at minimum enable the commented-out `console.log`. Consider alerting on patterns that indicate XSS probing.
