# Finding: Embed Route `frame-ancestors *` Allows Framing by Any Origin

## Severity
MEDIUM (HIGH when combined with F-003 stored XSS)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:U/C:L/I:L/A:N = 4.3 (standalone)
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:H = 8.2 (with F-003 XSS)

## Affected Code
- **File:** `next.config.mjs:259-271`
- **Route:** `/view/:path*/embed`
- **Header:** `Content-Security-Policy: frame-ancestors *;`

## Description
The `/view/:path*/embed` route sends `Content-Security-Policy: frame-ancestors *;` which allows any website to load the document viewer in an iframe. Unlike the main route CSP (which is report-only), this one is a real enforced header, so the `frame-ancestors *` directive is actively allowing cross-origin framing.

While the API layer validates embeddable domains via `isEmbeddableUrl()` in `lib/edge-config/embeddable-domains.ts`, the HTML page itself loads in any iframe before the API check runs. This means the UI-layer protection (CSP) does not match the API-layer protection.

## Proof of Concept
```html
<!-- Attacker-controlled page -->
<iframe src="https://papermark.app/view/<linkId>/embed" style="opacity: 0.1; position: absolute;">
</iframe>
<div class="overlay">
  <!-- Transparent overlay mimicking login form -->
  <input type="password" placeholder="Session expired, please re-enter password">
</div>
```

The embed page loads in the iframe because `frame-ancestors *` does not block it.

## Impact
An attacker can iframe the embed page on their own website and overlay UI elements to perform clickjacking attacks against document viewers. Combined with F-003 (stored XSS), the attacker can execute arbitrary JavaScript in the embed page's origin context, enabling credential theft via transparent overlay.

## Evidence
```javascript
// next.config.mjs:259-271
{
  source: "/view/:path*/embed",
  headers: [{
    key: "Content-Security-Policy",
    value: "... frame-ancestors *; ..."
  }]
}
```

## Methodology
- **First found by:** Config analysis only (00-ai-configs.md C-001)
- **Not found by:** Structural analysis or SAST tools

## Chain Potential
**Critical chain enabler for clickjacking.**
- **Chain 8 (Embed Clickjacking + XSS):** `frame-ancestors *` allows iframing the embed page, where F-003 stored XSS executes. Enables credential theft via transparent overlay. Combined chain estimated $5K-$12K.

## Remediation
Replace `frame-ancestors *` with `frame-ancestors 'self'` or a specific allowlist of embed domains. Consider reading the embeddable-domains from Edge Config at build time to generate the CSP value dynamically.
