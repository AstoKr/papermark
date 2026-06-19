# Finding: DOM-XSS Sink Cluster — prismaDocument.name + domainJson.apexName

## Severity
MEDIUM (latent)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N = 7.5 (latent — both sinks are wrong in principle; the practical exploit window is bounded by upstream defenses, but the defense is at the wrong layer).

## Affected Code
- File: `components/documents/document-header.tsx`
- Line: 604 (`dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` inside a `contentEditable` `<h2>`)
- Sink type: `dangerouslySetInnerHTML` on user-controlled text

- File: `components/domains/domain-configuration.tsx`
- Line: 143 (`dangerouslySetInnerHTML={{ __html: text }}` inside `<p className="prose-sm ..." />`)
- Sink type: `dangerouslySetInnerHTML` on user-controlled text

## Description
This is the merged finding M-004 (per `methodology-raw/06-validated.md` M-004). Both sinks are `dangerouslySetInnerHTML` calls that render user-controlled text as raw HTML. The practical exploit windows are bounded by upstream defenses today:
- `prismaDocument.name` — every active write path runs `sanitizePlainText` (verified in `methodology-raw/01-sanitizer-bypass.md` N1)
- `domainJson.apexName` — Vercel's DNS-label charset constraint (RFC 1035) prevents HTML metacharacters in the value

But the **defense is at the wrong layer** for both. Any future write path that bypasses the upstream defense, or any future change to the upstream constraint, immediately produces stored XSS that fires in every team member's session.

The full call chain for `prismaDocument.name` is documented in `methodology-raw/01-structural-analysis.md` Finding 7. The full call chain for `domainJson.apexName` is documented in `methodology-raw/02-synthesized-sast.md` S-003.

## Proof of Concept

See F-009 (`prismaDocument.name` latent XSS) and F-014 (`domainJson.apexName` latent XSS) for the per-sink call chain and PoC. The cluster is reported as a single defense-in-depth finding because:
1. Both sinks share the same root cause: `dangerouslySetInnerHTML` on user-controlled text.
2. Both defenses are at the wrong layer.
3. Both are bypassed by a single future regression (a new write path, or a change to the upstream constraint).

## Impact
- **Latent stored XSS** — both sinks are wrong in principle; the practical exploit window is bounded by upstream defenses today.
- **Same blast radius** as F-009 / F-014 — session hijack, CSRF-style actions, exfiltration of all team data.
- **Defense-in-depth gap** — papermark's intentional security pattern is "sanitize at write, render as text." Both sinks break the convention by rendering as HTML.

## Evidence
```tsx
// components/documents/document-header.tsx:604
<h2
  ...
  dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
/>
```
```tsx
// components/domains/domain-configuration.tsx:143
const MarkdownText = ({ text }: { text: string }) => {
  return (
    <p
      className="prose-sm max-w-none ..."
      dangerouslySetInnerHTML={{ __html: text }}
    />
  );
};
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 7)
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-002, S-003)
- **M3 — Sanitizer-bypass** (`methodology-raw/01-sanitizer-bypass.md` N1, N5)
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` D11)

## Chain Potential
- **Chain 3 / Chain 8** (`methodology-raw/05-chains.md`) — the sink cluster is the DOM-XSS defense-in-depth finding M-004.

## Remediation
For both sinks, replace `dangerouslySetInnerHTML` with React text nodes:
```tsx
// components/documents/document-header.tsx:604
<h2 ...>{prismaDocument.name}</h2>

// components/domains/domain-configuration.tsx:139-146
const MarkdownText = ({ children }: { children: React.ReactNode }) => {
  return <p className="prose-sm max-w-none ...">{children}</p>;
};
```

React auto-escapes string children. The `contentEditable` attribute does not require `dangerouslySetInnerHTML` — it requires a DOM node whose `textContent` is editable.

Apply the same fix to **all** `dangerouslySetInnerHTML` call sites in the codebase. Audit at `methodology-raw/01-sanitizer-bypass.md` N6 lists the other 4 sinks. Each should be evaluated: is the input guaranteed to be plain text? If yes, replace with `{value}`. If no, add a sanitizer at the sink.
