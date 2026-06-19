# AI Static Application Security Testing (SAST) — Findings

**Methodology**: M5 (whitebox-bug-finder / ai-analyze-sast)
**Target**: papermark (worktree: `task-whitebox-papermark`)
**Date**: 2026-06-19

## Summary

Identified 6 findings across the codebase spanning insecure crypto, stored/DOM
XSS risk surfaces, raw SQL execution, and XXE in a Python helper. The codebase
shows strong SSRF protection (in `lib/utils/ssrf-protection.ts`), correct
sanitization of user-supplied document/link names at the API layer
(`sanitizePlainText` from `lib/utils/sanitize-html.ts`), and good random-number
hygiene (`randomInt` from `node:crypto` in `lib/utils/generate-otp.ts`). The
findings below are the residual issues where a safer primitive is available, a
dangerous API is reachable from user input, or a defense-in-depth gap exists.

| # | Severity | Title | File | Pattern |
|---|----------|-------|------|---------|
| 1 | HIGH | Insecure symmetric encryption: AES-CTR without integrity / authentication | `lib/utils.ts:595-622` | Insecure crypto |
| 2 | MEDIUM | DOM-XSS surface: `dangerouslySetInnerHTML` on document name | `components/documents/document-header.tsx:604` | XSS / unsafe sink |
| 3 | MEDIUM | DOM-XSS surface: `dangerouslySetInnerHTML` on domain-config instructions | `components/domains/domain-configuration.tsx:143` | XSS / unsafe sink |
| 4 | LOW | XXE-adjacent risk: `xml.etree.ElementTree.parse` on attacker-controlled DOCX | `ee/features/conversions/python/docx-sanitizer.py:146` | XXE / XML parsing |
| 5 | LOW | `$executeRawUnsafe` with template-built identifier (defense-in-depth gap) | `lib/folders/bulk-create.ts:51-60` | Raw SQL / unsafe sink |
| 6 | INFO | SAML XML parsing handled by third-party library (unverified XXE surface) | `app/(ee)/api/auth/saml/authorize/route.ts:7-77` | XXE (vendor) |

## Pattern checks performed (negative results)

These categories were searched and produced no findings worth raising:

- **Command injection (`eval`, `new Function`, `child_process`, `exec`)**: zero
  matches in `lib/`, `app/`, `pages/`, `ee/`, `components/`. The only hit for
  `exec` is `pipeline.exec()` on an ioredis pipeline (Redis client), which is
  not a command-injection vector.
- **`Math.random()` for security purposes**: every match is a UI/animation use
  (sidebar opacity, tag badge color, globe vertex angles). `generateOTP` in
  `lib/utils/generate-otp.ts:2-9` correctly uses `randomInt` from `node:crypto`
  with a comment citing the CodeQL `js/insecure-randomness` rule.
- **Deserialization of untrusted data (`yaml.load`, `pickle`, `node-serialize`)**:
  no matches. `JSON.parse` is used heavily but always against controlled Redis
  blobs, signed cookies, or schema-validated request bodies.
- **Path traversal on `fs.readFile` / `fs.writeFile` with user input**:
  no matches. The only `fs` usage outside `public/vendor/` is
  `lib/trigger/optimize-video-files.ts:38-41`, which builds paths from
  `os.tmpdir() + Date.now()` plus hardcoded file names.
- **S3 / webhook SSRF**: protected by `lib/utils/ssrf-protection.ts` which
  validates the URL hostname literally (`isPublicHostnameLiteral`), then
  resolves DNS and re-checks the resolved address before connecting, and
  re-validates each redirect target. Username/password in URLs is also
  rejected. Webhook consumer in
  `pages/api/webhooks/services/[...path]/index.ts:428,746` uses this protected
  fetcher.
- **`prisma.$queryRaw` / `$executeRaw` with user input**: every match uses
  `Prisma.sql` template literals where user input is bound via parameter
  placeholders, not concatenated. `permissions-sql.ts` documents this pattern
  explicitly and uses `Prisma.raw` only for an `AccessControlTable` union
  type that is closed (lines 38-44).

---

### Finding 1: Insecure symmetric encryption for document passwords (HIGH)

- **Severity**: HIGH
- **File**: `lib/utils.ts`
- **Line**: 595–622 (`generateEncrpytedPassword`); 624–651 (`decryptEncrpytedPassword`)
- **Function**: `generateEncrpytedPassword`, `decryptEncrpytedPassword`
- **Pattern**: Insecure crypto — AES-256-CTR without authentication
- **Source**: `req.body` or any caller of `generateEncrpytedPassword(password)`
- **Sink**: `crypto.createCipheriv("aes-256-ctr", encryptedKey, IV)` (line 618)
- **Taint path**: API handler receives password string → `generateEncrpytedPassword(password)` → `crypto.createCipheriv("aes-256-ctr", encryptedKey, IV)` → ciphertext stored in DB
- **Description**:
  Document passwords are encrypted with AES-256-CTR using a key derived from
  `process.env.NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` (SHA-256 → base64 → first
  32 chars) and a random 16-byte IV. The format is `iv:ciphertext` (hex).
  **There is no authentication tag.** CTR mode is a stream cipher: bit-flips in
  the ciphertext produce predictable bit-flips in the plaintext, and without an
  HMAC/GCM tag there is no way for `decryptEncrpytedPassword` to detect
  tampering. An attacker with database write access (e.g. a compromised
  admin token, a SQL injection elsewhere, or a backup leak) can:
  1. XOR the ciphertext with arbitrary padding to flip plaintext bits,
  2. Re-submit the modified blob, and
  3. The decryption will succeed and produce a different but valid password
     that the server will accept.
  This breaks password-based access control on link documents.

  For comparison, the Slack token path in `lib/integrations/slack/utils.ts:41-74`
  uses AES-256-GCM with a 12-byte IV and stores the auth tag — the correct
  pattern.
- **Evidence**:
  ```ts
  // lib/utils.ts:612-621
  const encryptedKey: string = crypto
    .createHash("sha256")
    .update(String(process.env.NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY))
    .digest("base64")
    .substring(0, 32);
  const IV: Buffer = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv("aes-256-ctr", encryptedKey, IV);
  let encryptedText: string = cipher.update(password, "utf8", "hex");
  encryptedText += cipher.final("hex");
  return IV.toString("hex") + ":" + encryptedText;
  ```
- **Recommendation**: Migrate to AES-256-GCM (`crypto.createCipheriv("aes-256-gcm", key, iv)`) with a 12-byte IV and store `getAuthTag()` alongside the IV/ciphertext, mirroring the Slack token implementation. On the decryption side, call `setAuthTag()` and surface any thrown error as "decryption failed" rather than a generic message. Also address the key-derivation weakness: `digest("base64").substring(0, 32)` truncates the SHA-256 to 32 ASCII characters (not 32 bytes), reducing the effective key space; the Slack path uses the full 32-byte digest and is the model to follow.

---

### Finding 2: DOM-XSS surface on `prismaDocument.name` (MEDIUM)

- **Severity**: MEDIUM
- **File**: `components/documents/document-header.tsx`
- **Line**: 604
- **Function**: `DocumentHeader` (default export)
- **Pattern**: XSS — `dangerouslySetInnerHTML` on user-controllable string
- **Source**: `prisma.document.update` on the document name field (driven by `pages/api/teams/[teamId]/documents/[id]/update-name.ts:79-82` and the `documentUploadSchema` in `lib/zod/url-validation.ts:202-213`).
- **Sink**: `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` (line 604)
- **Taint path**: Authenticated team member edits a document name in the `contentEditable` heading (handler at line 174-206) → POST `/api/teams/.../documents/.../update-name` → Zod `updateNameSchema` runs `sanitizePlainText` → stored in DB → re-rendered with `dangerouslySetInnerHTML` in any team member's document header.
- **Description**:
  The document name is rendered with `dangerouslySetInnerHTML` in the
  `contentEditable` `<h2>` element. All known write paths today run the value
  through `sanitizePlainText` (`lib/utils/sanitize-html.ts:12-20` uses
  `sanitize-html` with `allowedTags: []` and `allowedAttributes: {}`), so the
  current risk is **latent**: a stored XSS that becomes exploitable the moment
  any future write path forgets to sanitize. Two concrete risks:
  1. **Migration / backfill** of legacy records created before sanitization
     was added will retain HTML payloads. React's `dangerouslySetInnerHTML`
     will execute them when viewed.
  2. **Notion-derived document names** flow through the same Zod schema, so
     they're currently sanitized, but the title for a Notion page can be very
     long and contain Unicode; if a future refactor moves name extraction
     outside the Zod transform, sanitization is silently bypassed.
  The `<h2>` is also `contentEditable`, so the same user who could self-XSS
  via paste is also the user whose data gets rendered. That's why the
  *current* impact is bounded, but the unsafe sink should be removed
  regardless — render with `{prismaDocument.name}` (React auto-escapes).
- **Evidence**:
  ```tsx
  // components/documents/document-header.tsx:596-605
  <h2
    className="truncate …"
    ref={nameRef}
    contentEditable={true}
    onFocus={() => setIsEditingName(true)}
    onBlur={handleNameSubmit}
    onKeyDown={preventEnterAndSubmit}
    title="Click to edit"
    dangerouslySetInnerHTML={{ __html: prismaDocument.name }}
  />
  ```
- **Recommendation**: Replace `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` with `{prismaDocument.name}`. If `contentEditable` needs to display styled placeholder text, render an explicit child element rather than an HTML string. This eliminates the entire class of stored-XSS regressions for this field.

---

### Finding 3: DOM-XSS surface on domain-configuration instructions (MEDIUM)

- **Severity**: MEDIUM
- **File**: `components/domains/domain-configuration.tsx`
- **Line**: 143 (sink) and 96-101 (source)
- **Function**: `MarkdownText` (lines 139-146), `DomainConfiguration` default export
- **Pattern**: XSS — `dangerouslySetInnerHTML` on template-interpolated data
- **Source**: `domainJson.apexName` / `domainJson.name` (Vercel Domains API response for the user's verified custom domain), interpolated into an HTML literal at lines 96-101
- **Sink**: `dangerouslySetInnerHTML={{ __html: text }}` inside `MarkdownText` (line 143), reached via `<DnsRecord … instructions={…} />` which wraps `instructions` in `<MarkdownText text={instructions} />` (line 162)
- **Taint path**: User adds a custom domain to their Papermark team → Vercel API returns the domain object → `instructions` string is built as a template literal containing `domainJson.apexName` / `domainJson.name` (line 100) → wrapped in `<code>…</code>` tags and passed to `DnsRecord` → `MarkdownText` renders the result with `dangerouslySetInnerHTML`.
- **Description**:
  `instructions` is built by interpolating a Vercel-controlled domain name
  into a hard-coded HTML string (`<code>${domainJson.apexName}</code>`) and
  then handed to `MarkdownText`, which renders it with
  `dangerouslySetInnerHTML`. Today, Vercel enforces that `apexName` is a
  valid DNS label, so the practical exploitability is very low — DNS labels
  cannot contain `<`, `>`, or `"`. **But** the sink is wrong in principle:
  if any upstream (Vercel API, a future custom-domain feature, or a
  misconfigured Vercel project) ever returns a value with metacharacters,
  the payload executes in the team admin's session. The `DnsRecord` flow is
  the only consumer of this `MarkdownText` helper, so the fix is local.
- **Evidence**:
  ```tsx
  // components/domains/domain-configuration.tsx:96-101 (source)
  <DnsRecord
    instructions={`To configure your ${
      recordType === "A" ? "apex domain" : "subdomain"
    } <code>${
      recordType === "A" ? domainJson.apexName : domainJson.name
    }</code>, set the following ${txtVerification ? "records" : `${recordType} record`} on your DNS provider:`}
    …

  // components/domains/domain-configuration.tsx:139-146 (sink)
  const MarkdownText = ({ text }: { text: string }) => {
    return (
      <p
        className="prose-sm …"
        dangerouslySetInnerHTML={{ __html: text }}
      />
    );
  };
  ```
- **Recommendation**: Build the instructions with React nodes (`<code>{domainJson.apexName}</code>`) instead of an HTML string, and render `MarkdownText` as plain text. Alternatively, drop the `<code>` wrapper and render the domain name in a styled `<span>` outside the instruction line.

---

### Finding 4: XXE-adjacent risk in Python DOCX sanitizer (LOW)

- **Severity**: LOW
- **File**: `ee/features/conversions/python/docx-sanitizer.py`
- **Line**: 146
- **Function**: `_strip_numpages_fields` (line 122-176) — also see `sanitize_docx` (line 216-303) which performs the broader file walk.
- **Pattern**: XXE / XML parsing of attacker-controlled archive contents
- **Source**: A DOCX file uploaded by an end user (the file is unzipped to `tmp_dir` earlier in the function; `tree = ET.parse(path)` at line 146 opens `word/header*.xml` / `word/footer*.xml` from that directory).
- **Sink**: `ET.parse(path)` (line 146)
- **Taint path**: User uploads a DOCX via Papermark's normal upload flow → the conversion service unzips the DOCX to `tmp_dir` → the sanitizer iterates `word/header*.xml` / `word/footer*.xml` and parses each one with `ET.parse`.
- **Description**:
  `xml.etree.ElementTree.parse` does **not** expand external entities by
  default in modern Python (no DOCTYPE-external-entity XXE in CPython 3.x for
  `ET.parse`/`ET.fromstring`), so the **practical** XXE risk is low. However:
  1. The code is part of a security-sensitive sanitizer (the file name is
     `docx-sanitizer.py`), so any future maintainer who "hardens" the parser
     by switching to `lxml.etree.parse` (the default `lxml` is XXE-vulnerable)
     or by explicitly enabling `resolve_entities` on the parser would
     introduce a real XXE that can read local files (e.g. `file:///etc/passwd`,
     AWS instance metadata via `http://169.254.169.254/`) or cause a billion-laughs DoS.
  2. The XML files inside a DOCX come from the original document author — a
     hostile document author who is also a Papermark user can plant
     `<!DOCTYPE … SYSTEM "…">` declarations in headers/footers.
  3. The function processes XML files in-place and writes them back
     (`tree.write(path, …)` at line 172), so any expanded-entity content gets
     re-serialized — turning a one-time XXE into a persistent payload
     embedded back into the user's stored file.
- **Evidence**:
  ```python
  # ee/features/conversions/python/docx-sanitizer.py:145-147
  _register_all_namespaces(path)
  tree = ET.parse(path)
  root = tree.getroot()
  ```
- **Recommendation**: Use a parser that explicitly disables DTDs and external entities, e.g. `ET.iterparse(path, events=("start",), parser=ET.XMLParser(resolve_entities=False, no_network=True))`, or switch the whole sanitizer to `defusedxml.ElementTree` (a drop-in replacement that is hardened against XXE by default). Verify the same protection for `sanitize_docx` (lines 216-303) which also walks XML parts of the DOCX.

---

### Finding 5: `$executeRawUnsafe` with template-built identifier (LOW)

- **Severity**: LOW
- **File**: `lib/folders/bulk-create.ts`
- **Line**: 51, 54, 57-60
- **Function**: `withUniqueConstraintRetry`
- **Pattern**: Raw SQL execution (`$executeRawUnsafe`)
- **Source**: Internal caller passes `savepointName: \`bulk_main_folder_lvl_${depth}\`` (lines 365, 469); the template string is a numeric loop counter.
- **Sink**: `await tx.$executeRawUnsafe(\`SAVEPOINT "${savepointName}"\`)` (line 51)
- **Taint path**: `withUniqueConstraintRetry({ tx, savepointName, attempt })` is called from `bulkCreateMainDocsFolders` (line 363) and `bulkCreateDataroomFolders` (line 467) with a hardcoded string template. No user input currently flows into `savepointName`, but the function accepts it as a free-form `string` argument.
- **Description**:
  Prisma exposes `$executeRawUnsafe` for cases where the developer has already
  escaped/validated the SQL string. Here the call site is a Postgres
  `SAVEPOINT` statement with a quoted identifier, so any untrusted value in
  `savepointName` would be a SQL-injection sink — the double-quotes are not
  sufficient because the function does not escape embedded quotes. The
  current call sites pass a template `bulk_main_folder_lvl_${depth}` where
  `depth` is a loop integer, so the *practical* risk is zero. However:
  1. The pattern invites a future caller to pass a request-derived
     `savepointName`.
  2. The code comment "Defense-in-depth: API routes already cap via Zod…"
     (line 137) elsewhere in the file is a good example of the team
     valuing belt-and-suspenders defenses; this sink is the inverse.
  3. The whole call can be expressed with the parameterized
     `Prisma.sql\`SAVEPOINT …\`` form or the higher-level
     `tx.$executeRaw(Prisma.sql\`SAVEPOINT …\`)` form, which would make
     injection impossible at the type level.
- **Evidence**:
  ```ts
  // lib/folders/bulk-create.ts:50-60
  for (let i = 0; i <= MAX_INSERT_RETRIES; i++) {
    await tx.$executeRawUnsafe(`SAVEPOINT "${savepointName}"`);
    try {
      const result = await attempt();
      await tx.$executeRawUnsafe(`RELEASE SAVEPOINT "${savepointName}"`);
      return result;
    } catch (error) {
      await tx.$executeRawUnsafe(
        `ROLLBACK TO SAVEPOINT "${savepointName}"`,
      );
      await tx.$executeRawUnsafe(`RELEASE SAVEPOINT "${savepointName}"`);
      …
  ```
- **Recommendation**: Replace the four `$executeRawUnsafe` calls with `$executeRaw(Prisma.sql\`SAVEPOINT ${Prisma.raw(savepointName)}\`)` (using `Prisma.raw` only because the savepoint identifier cannot be bound as a parameter) and add a regex guard at the top of `withUniqueConstraintRetry` that rejects `savepointName` if it does not match `^[A-Za-z_][A-Za-z0-9_]*$`. Even simpler: drop the parameter and let `withUniqueConstraintRetry` build the savepoint name internally from a fixed prefix + the retry counter.

---

### Finding 6: SAML XML parsing handled by third-party library (INFO)

- **Severity**: INFO
- **File**: `app/(ee)/api/auth/saml/authorize/route.ts`
- **Line**: 7-77 (both `GET` and `POST` handlers)
- **Function**: route handlers
- **Pattern**: XXE / XML parsing of unverified input (vendor implementation)
- **Source**: `req.url` query string (`GET`) or `req.formData()` / `req.json()` body (`POST`) for the SAML `OAuthReq` payload
- **Sink**: `oauthController.authorize(requestParams)` / `oauthController.authorize(body)` from `@boxyhq/saml-jackson` (imported in `lib/jackson`)
- **Taint path**: Enterprise admin configures a SAML connection → end user starts SSO → GET `/api/auth/saml/authorize?…` → `oauthController.authorize` returns either a 302 redirect or an HTML form containing the SAML AuthnRequest → user posts back to the IdP with a SAMLResponse → … on the IdP side. The XML parser inside the `jackson` library is the actual sink.
- **Description**:
  This route does not parse XML itself — it delegates the SAML AuthnRequest
  and (eventually) the SAMLResponse to `@boxyhq/saml-jackson`'s
  `oauthController` and `samlController`. BoxyHQ's library has had past
  advisories around SAML signature validation and XML parsing (e.g. 2023
  CVE-2023-… advisories in the broader SAML/SSO ecosystem). The Papermark
  code does not perform any defense-in-depth such as capping `Content-Length`
  on the response form, validating the AuthnRequest's `RelayState` against
  an allowlist, or pinning a specific `jackson` patch version. This is
  flagged as INFO because the actual exploitability lives in the third-party
  library and is out of scope for a Papermark SAST pass; it is included so
  that the SAST ledger captures the trust boundary.
- **Evidence**:
  ```ts
  // app/(ee)/api/auth/saml/authorize/route.ts:7-25
  export async function GET(req: Request) {
    try {
      const { oauthController } = await jackson();
      const url = new URL(req.url);
      const requestParams = Object.fromEntries(
        url.searchParams.entries(),
      ) as unknown as OAuthReq;
      const { redirect_url, authorize_form } =
        await oauthController.authorize(requestParams);
      if (redirect_url) {
        return NextResponse.redirect(redirect_url, { status: 302 });
      } else if (authorize_form) {
        return new Response(authorize_form, {
          headers: { "Content-Type": "text/html; charset=utf-8" },
        });
      }
      …
  ```
- **Recommendation**: Track `@boxyhq/saml-jackson` (and `@boxyhq/saml-jackson-jose` / `@boxyhq/saml-jackson-binding`) for XXE-related advisories via Dependabot / Renovate. Add an integration test that submits a SAMLResponse with an embedded `<!DOCTYPE … SYSTEM "…">` entity and asserts the response is rejected. Document the SAML library version in the security review policy so future upgrades are reviewed for XML hardening.

---

## Phase 2 / Phase 3 checklists

- [x] Codebase analyzed for vulnerabilities in this domain
- [x] GitNexus queries executed (8 conceptual queries + 4 context lookups)
- [x] Findings documented with file paths and line numbers
- [x] Findings written to `methodology-raw/00-ai-sast.md`
- [x] Each finding has: file, line, severity, description, evidence
- [x] No duplicate findings
