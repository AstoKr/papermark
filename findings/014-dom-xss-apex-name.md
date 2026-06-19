# Finding: DOM-XSS Surface on domainJson.apexName (DNS-Label Constrained)

## Severity
MEDIUM (latent)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N = 5.0 (latent — the sink is wrong, but Vercel's DNS-label charset enforcement on `apexName` / `name` constrains the practical exploit window today).

## Affected Code
- File: `components/domains/domain-configuration.tsx`
- Line: 96-101 (interpolates `domainJson.apexName` / `domainJson.name` into a template literal), 139-146 (`MarkdownText` component renders with `dangerouslySetInnerHTML={{ __html: text }}`)
- Function: `DomainConfiguration` (parent), `MarkdownText` (sink component)
- Endpoint: n/a — server-rendered React component on `/settings/domains`

## Description
`instructions` is built by interpolating `domainJson.apexName` / `domainJson.name` into an HTML literal (`<code>${...}</code>`), then handed to `MarkdownText`, which renders the result with `dangerouslySetInnerHTML`. The sink is wrong in principle: if any upstream (Vercel API, a future custom-domain feature, a misconfigured Vercel project) ever returns a value with metacharacters, the payload executes in the team admin's session.

The `domainJson.apexName` / `domainJson.name` values come from the Vercel API (`lib/domains/vercel-api.ts` or similar — verify in the codebase). Vercel's domain values are constrained by DNS-label charset (RFC 1035) — labels cannot contain `<`, `>`, or `"`. So the practical exploit window is **closed today**, but the sink is wrong in principle, and a future custom-domain feature or a misconfigured Vercel project could return values with metacharacters.

## Proof of Concept

### Pre-conditions (today, no exploit)
- The attacker would need to control the `apexName` or `name` value returned by the Vercel API. Vercel's API enforces DNS-label charset, so the value cannot contain HTML metacharacters.

### Latent exploit (if Vercel API behavior changes)
1. Attacker registers a domain with a name that Vercel's API returns with HTML metacharacters (hypothetical — Vercel would need to weaken its input validation, or a custom-domain provider would need to be added to papermark).
2. Attacker visits `/settings/domains` on papermark.
3. The `DomainConfiguration` component renders the DNS configuration instructions with `MarkdownText` and the `apexName` value.
4. The XSS fires in the team admin's session.

## Impact
- **Latent stored XSS** — if the upstream ever returns a value with metacharacters, the sink fires.
- **Same blast radius as F-009 (latent XSS on prismaDocument.name)** — session hijack, CSRF-style actions, exfiltration of team data.
- **Today: no practical exploit** — Vercel DNS-label charset is enforced upstream.

## Evidence
```tsx
// components/domains/domain-configuration.tsx:96-101
<DnsRecord
  instructions={`To configure your ${
    recordType === "A" ? "apex domain" : "subdomain"
  } <code>${
    recordType === "A" ? domainJson.apexName : domainJson.name
  }</code>, set the following ${txtVerification ? "records" : `${recordType} record`} on your DNS provider:`}
  ...
/>
```
```tsx
// components/domains/domain-configuration.tsx:139-146
const MarkdownText = ({ text }: { text: string }) => {
  return (
    <p
      className="prose-sm max-w-none prose-code:rounded-md prose-code:bg-gray-100 prose-code:p-1 prose-code:font-mono prose-code:text-[.8125rem] prose-code:font-medium prose-code:text-gray-900"
      dangerouslySetInnerHTML={{ __html: text }}    // ← unsafe sink
    />
  );
};
```

## Methodology
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-003)
- **M3 — Sanitizer-bypass** (`methodology-raw/01-sanitizer-bypass.md` N5)
- **M1 — Reachability** (`methodology-raw/01-reachability-analysis.md` F3 — confirmed, near-zero practical exploit)

## Chain Potential
- **Chain 3 / Chain 8** (`methodology-raw/05-chains.md`) — the same sink cluster (`prismaDocument.name` + `domainJson.apexName`) is the DOM-XSS defense-in-depth finding M-004.

## Remediation
Build `instructions` with React nodes instead of an HTML string, and render `MarkdownText` as plain text:
```tsx
<DnsRecord
  instructions={
    <>
      To configure your {recordType === "A" ? "apex domain" : "subdomain"}{" "}
      <code>{recordType === "A" ? domainJson.apexName : domainJson.name}</code>,
      set the following {txtVerification ? "records" : `${recordType} record`} on your DNS provider:
    </>
  }
  ...
/>
```

Alternatively, drop the `<code>` wrapper and render the domain name in a styled `<span>` outside the instruction line:
```tsx
const MarkdownText = ({ children }: { children: React.ReactNode }) => {
  return <p className="prose-sm max-w-none">{children}</p>;
};
```

Apply the same fix to **all** `dangerouslySetInnerHTML` call sites in the codebase (see `methodology-raw/01-sanitizer-bypass.md` N6 for the full list).
