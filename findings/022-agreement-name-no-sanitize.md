# Finding: Agreement.name Write Path Skips sanitizePlainText

## Severity
MEDIUM (latent)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N = 4.5 (latent — every current sink escapes the value, but the write path breaks the codebase's documented sanitization pattern).

## Affected Code
- File: `pages/api/teams/[teamId]/agreements/index.ts`
- Line: 21-24 (Zod schema for `name`: no `.transform(sanitizePlainText)`), 139-154 (handler writes `name` verbatim)
- Function: `handle` (POST branch)
- Endpoint: `POST /api/teams/:teamId/agreements`

- File: `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts`
- Line: 13-17 (update Zod schema for `name`: no `.transform(sanitizePlainText)`), 92-94 (handler writes `data.name = name.trim()`)
- Function: `handle` (PUT/PATCH branch)
- Endpoint: `PUT /api/teams/:teamId/agreements/:agreementId`

## Description
Every agreement write path stores `name` with only a `.trim()`. Compare to the sibling field `content` in the same handler:
```ts
const sanitizedContent =
  contentType === "SIGNING"
    ? content?.trim()
    : validateContent(content || "", 1500);
```
The author of this handler knew about `validateContent` (imported at line 14 / line 10 of both files), knew that the `TEXT` contentType needs sanitization but `LINK` and `SIGNING` don't, and **deliberately wrote the `name` field without invoking either**. The Zod schema also has no `.transform()`.

## Proof of Concept

### Pre-conditions
- The attacker is an authenticated team member.
- The attacker can submit a POST to the agreements endpoint (bypassing the client-side validation if any).

### Step-by-step exploitation (latent)
1. Attacker creates an agreement with `name: "<img src=x onerror=alert(1)>"`:
   ```bash
   curl -X POST 'https://app.papermark.com/api/teams/<teamId>/agreements' \
     -H 'Content-Type: application/json' \
     -b '__Secure-next-auth.session-token=<attacker_session_jwt>' \
     -d '{"name":"<img src=x onerror=alert(1)>","contentType":"TEXT","content":"agreement text"}'
   ```
   The server returns 201 Created; the name row in `Agreement.name` now contains literal HTML.

2. **Today**: every reader escapes it. `<h3>{agreement.name}</h3>` in `components/agreements/agreement-card.tsx:187` auto-escapes. `aria-label={\`Edit signing fields for ${agreement.name}\`}` in `components/links/link-sheet/agreement-section.tsx:191` auto-escapes. The filename path at `agreement-card.tsx:153-156` strips non-`[a-z0-9_-]` characters. **No XSS via `name` today.**

3. **Latent**: if a future admin tool renders the agreement list with `dangerouslySetInnerHTML` to support markdown, the XSS fires in the admin's session.

## Impact
- **Latent stored XSS** — every team member and every viewer who interacts with the agreement is a potential victim.
- **Defense-in-depth gap** — the codebase's documented pattern is "sanitize at write." The agreement-name path breaks the convention.
- **CSV injection chain** — the `agreement.name` value is included in the dataroom index CSV export (F-021). If the name starts with `=`, `+`, `-`, or `@`, the formula fires when a team admin opens the export in Excel.

## Evidence
```ts
// pages/api/teams/[teamId]/agreements/index.ts:21-24
name: z
  .string()
  .min(1, "Name is required")
  .max(150, "Name must be less than 150 characters"),
  // ← no .transform(sanitizePlainText)
```
```ts
// pages/api/teams/[teamId]/agreements/index.ts:139-154 — name is verbatim
const { name, content, contentType, requireName } = parseResult.data;

const sanitizedContent =
  contentType === "SIGNING"
    ? content?.trim()
    : validateContent(content || "", 1500);

const agreement = await prisma.$transaction(async (tx) => {
  const created = await tx.agreement.create({
    data: getSigningAgreementCreateData({
      teamId,
      name,                    // ← no sanitize
      content: sanitizedContent,// ← sanitized
      contentType,
      requireName,
    }),
  });
  ...
});
```
```ts
// pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:92-94 — update path also verbatim
if (typeof name === "string") {
  data.name = name.trim();     // ← no sanitize
}
```

## Methodology
- **M3 — Sanitizer-bypass** (`methodology-raw/01-sanitizer-bypass.md` Finding 2)

## Chain Potential
- **Chain 8** (`methodology-raw/05-chains.md`) — the agreement name is the entry point for the CSV injection chain (Chain 8, leg 2). An attacker who creates an agreement with name `=WEBSERVICE(...)` triggers the formula when a team admin exports the dataroom index.

## Remediation
At `pages/api/teams/[teamId]/agreements/index.ts:21-24`:
```ts
name: z
  .string()
  .min(1, "Name is required")
  .max(150, "Name must be less than 150 characters")
  .transform((value) => sanitizePlainText(value))
  .pipe(
    z
      .string()
      .min(1, "Name is required")
      .max(150, "Name must be less than 150 characters"),
  ),
```
and at `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:92-94`:
```ts
if (typeof name === "string") {
  data.name = sanitizePlainText(name);
}
```
This brings agreement-name handling in line with the document-name handling. The trade-off is small: `sanitizePlainText` strips every tag/attribute, so `<b>My NDA</b>` becomes `My NDA` — but the field is meant to be a short display label and the same trade-off is already accepted for document names, team names, conversation messages, viewer welcome messages, etc.

Additionally, **add a leading-character guard** to the agreement name (and to the `name` field in all other text fields) so the CSV injection chain is broken at the source: if `name.startsWith("=") || name.startsWith("+") || name.startsWith("-") || name.startsWith("@")`, prepend a space or wrap in a code block.
