# Finding: X-Powered-By Header Leaks Application Information on Custom Domains

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N = 2.7

## Affected Code
- **File:** `lib/middleware/domain.ts:40-41`

## Description
The DomainMiddleware adds a custom `X-Powered-By` header with the value `"Papermark - Secure Data Room Infrastructure for the modern web"` on all rewritten custom-domain responses. This leaks the specific software stack to anyone probing custom domains, enabling targeted vulnerability searches.

## Proof of Concept
```bash
# Check response headers on a custom domain
curl -sI https://customer-custom-domain.com | grep -i x-powered-by
# X-Powered-By: Papermark - Secure Data Room Infrastructure for the modern web
```

## Impact
Minor information disclosure enabling targeted vulnerability search. An attacker scanning custom domains can fingerprint the backend software as Papermark. The main application already identifies itself via standard responses, so the additional disclosure on custom domains is marginal.

## Evidence
```typescript
// lib/middleware/domain.ts:37-43
return NextResponse.rewrite(url, {
  headers: {
    "X-Robots-Tag": "noindex",
    "X-Powered-By": "Papermark - Secure Data Room Infrastructure for the modern web",
  },
});
```

## Methodology
- **First found by:** Config analysis only (00-ai-configs.md C-009)

## Chain Potential
Standalone finding. Marginal contribution to attacker reconnaissance for targeted attacks.

## Remediation
Remove the custom `X-Powered-By` header or set it to a generic value that does not identify the application stack.
