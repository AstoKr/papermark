# Finding: `pdfjs-dist` 3.11.174 Arbitrary JavaScript Execution via Malicious PDF

## Severity
MEDIUM

## CVSS Estimate
CVE-2024-4367: CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:H = 8.8

## Affected Code
- **Package:** `pdfjs-dist@3.11.174` (transitive via `react-pdf`)
- **CVE:** CVE-2024-4367 / GHSA-wgrm-67xf-hhpq
- **File:** `components/documents/preview-viewers/preview-viewer.tsx` (renders PDFs with default `isEvalSupported: true`)
- **Fix:** Available in `pdfjs-dist@4.2.67+` — papermark has 3.11.174

## Description
Papermark ships `pdfjs-dist@3.11.174` via `react-pdf`. The PDF viewer renders user-uploaded PDFs using `<Document>` from react-pdf. The component does NOT set `isEvalSupported: false` (defaults to `true`). react-pdf 8.0.2 pins `pdfjs-dist@3.11.174` in its own dependencies.

CVE-2024-4367 allows arbitrary JavaScript execution via crafted PDF font glyph rendering. An attacker who uploads a malicious PDF can trigger XSS in the browser of any user who views the document. The patch (in pdfjs-dist 4.2.67) removes all `eval()`/`new Function()` usage for font glyph rendering.

EPSS score: 72.6% (99th percentile) — actively exploited in the wild.

## Proof of Concept
1. Craft a PDF with a malicious font that triggers the CVE-2024-4367 exploit path
2. Upload the PDF to papermark (requires authentication but any team member can upload)
3. When any team member views the document, `pdfjs-dist@3.11.174` processes the font using `eval()` or `new Function()` in `font_loader.js`
4. The XSS executes in the viewer page's origin context, enabling session token theft and data exfiltration

## Impact
An attacker who uploads a crafted PDF can execute arbitrary JavaScript in the browser of any user who views the document. This includes:
- Session token theft via `document.cookie`
- API calls on the victim's behalf
- Team data exfiltration
- UI manipulation for phishing within the trusted application context

## Evidence
- `package-lock.json` confirms `pdfjs-dist@3.11.174` as dependency of `react-pdf`
- `preview-viewer.tsx` renders PDFs using `react-pdf`'s `<Document>` component
- No `isEvalSupported: false` configuration override present
- CVE-2024-4367 confirmed via OSV-scanner and advisory pairing

## Methodology
- **First found by:** Dependency synthesis (02-synthesized-dependencies.md S-004)
- **Confirmed by:** OSV-scanner
- **Confirmed by:** Advisory pairing (03-advisory.md) — patch available in pdfjs-dist 4.2.67

## Chain Potential
**Chainable.**
- **Chain 2 (CORS + PDF):** CORS misconfiguration on TUS viewer (F-008) enables CSRF upload of malicious PDF to victim's dataroom. PDF.js XSS then hijacks the victim's session. Estimated $5K-$15K.

## Remediation
Set `isEvalSupported: false` in the PDF viewer configuration as an immediate workaround:
```typescript
const pdfOptions = { isEvalSupported: false };
```
For a permanent fix, bump `react-pdf` to a version that uses `pdfjs-dist@4.2.67+`. This may require upstream changes in react-pdf's dependency resolution (override via `package.json` or `overrides`).
