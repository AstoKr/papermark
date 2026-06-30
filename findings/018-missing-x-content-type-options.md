# Finding: Missing X-Content-Type-Options Header

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:L/A:N = 4.3

## Affected Code
- **File:** `next.config.mjs:190-312` (global headers block — header is absent)
- **Only instance:** `pages/api/agreements/signing/[agreementResponseId]/download.ts:211`

## Description
The `X-Content-Type-Options: nosniff` header is only set on a single agreement-signing download endpoint and is absent from the global headers configuration. Without this header, browsers may perform MIME-type sniffing, potentially interpreting a user-uploaded file as a different MIME type (e.g., treating a PDF as HTML).

## Proof of Concept
```bash
# Check global headers — no X-Content-Type-Options present
curl -sI https://papermark.app | grep -i x-content-type-options
# Empty — not set on most responses
```

## Impact
An attacker who uploads a document with crafted content and a manipulated Content-Type header could potentially cause browsers to render it as HTML/JavaScript, leading to XSS in the document viewer context. The risk is partially mitigated because uploaded documents are served from blob storage/CDN rather than the app origin, but any view rendering user-uploaded content without explicit MIME handling is affected.

## Evidence
Global headers at `next.config.mjs:190-312` do not include `X-Content-Type-Options`. Only one endpoint sets it:
```
pages/api/agreements/signing/[agreementResponseId]/download.ts:211:
  res.setHeader("X-Content-Type-Options", "nosniff");
```

## Methodology
- **First found by:** Config analysis only (00-ai-configs.md C-005)
- **Confirmed by:** Judge B

## Chain Potential
Standalone finding. Defense-in-depth gap.

## Remediation
Add to the global `/:path*` header block in `next.config.mjs`:
```javascript
{ key: "X-Content-Type-Options", value: "nosniff" }
```
