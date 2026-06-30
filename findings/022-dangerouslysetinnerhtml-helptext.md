# Finding: `dangerouslySetInnerHTML` on `helpText` Props — Latent XSS Sinks

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:L/A:N = 4.3 (potential, if exploitable)

## Affected Code
- **File:** `components/ui/form.tsx:145`
- **File:** `components/account/upload-avatar.tsx:96`
- **Pattern:** `dangerouslySetInnerHTML={{ __html: helpText || "" }}`

## Description
Two shared components use `dangerouslySetInnerHTML` to render the `helpText` prop. If any caller passes user-controlled data through `helpText`, it will render unsanitized HTML. Currently, all callers pass hardcoded static strings. No active exploit path exists today, but this is a latent XSS sink that broadens the blast radius beyond `document-header.tsx` (F-003).

## Proof of Concept
No active exploit path — all current callers pass static strings. If a future code change feeds user data through `helpText`, XSS results:
```typescript
// Hypothetical dangerous usage — not currently present
<Form helpText={userControlledData} />
```

## Impact
Latent quality finding. If any future code change feeds user-controlled data into the `helpText` prop of `Form` or `UploadAvatar`, stored XSS is possible. This broadens the XSS blast radius beyond `document-header.tsx`.

## Evidence
```typescript
// components/ui/form.tsx:142-146
{typeof helpText === "string" ? (
  <p className="text-sm text-muted-foreground transition-colors"
     dangerouslySetInnerHTML={{ __html: helpText || "" }} />
) : helpText}
```

```typescript
// components/account/upload-avatar.tsx:93-97
{typeof helpText === "string" ? (
  <p className="text-sm text-muted-foreground transition-colors"
     dangerouslySetInnerHTML={{ __html: helpText || "" }} />
) : helpText}
```

## Methodology
- **First found by:** SAST synthesis via semgrep (02-synthesized-sast.md S-009)
- **Confirmed by:** Structural analysis (01-structural-analysis.md Finding 9) traced all callers — all hardcoded strings
- **Also identified by:** Custom semgrep rule `papermark-dangerouslysetinnerhtml-nonstatic`

## Chain Potential
- **Chain 1 (XSS Worm):** If any future code change routes user data through `helpText`, this becomes an additional XSS sink for the XSS worm.

## Remediation
Replace `dangerouslySetInnerHTML` with standard React rendering that escapes HTML. If HTML rendering is required for specific use cases (e.g., markdown), use a dedicated markdown renderer that sanitizes output.
