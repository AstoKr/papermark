# Synthesized SAST — Papermark

**Workflow:** whitebox-bug-finder
**Methodology:** M5 (synthesize-sast)
**Date:** 2026-06-19
**Target:** papermark (worktree: `task-whitebox-papermark`)

---

## Synthesis Inputs

| Source | File | Findings |
|--------|------|----------|
| AI manual SAST | `methodology-raw/00-ai-sast.md` | 6 findings (1 HIGH, 2 MEDIUM, 2 LOW, 1 INFO) |
| Tool: semgrep | `tools-raw/semgrep.json` | **not installed** — `{"tool": "semgrep", "error": "not_installed"}` |
| Tool: normalized.json (semgrep slice) | `tools-raw/normalized.json` | 0 semgrep entries (file contains only `npm-audit` findings) |
| Confirmed by `tools-manifest.json` | — | `semgrep` listed under `skipped` (along with `gitleaks`, `osv-scanner`, `syft`, `trufflehog`) |

**Net result of synthesis: all 6 findings are AI-only.** No merging was required
because no tool independently rediscovered any of them. Each AI finding has been
verified against the source files (spot-checked) and cross-referenced against the
structural, reachability, sanitizer-bypass, and encoding analyses. The synthesis
therefore **promotes confidence to "MEDIUM"** for every finding (only one source
of detection), but the exploitation-context judgment still uses the full call-
chain picture from the structural / reachability / sanitizer-bypass passes.

---

## Cross-reference matrix

| AI finding | Structural (`01-structural-analysis.md`) | Reachability (`01-reachability-analysis.md`) | Sanitizer-bypass (`01-sanitizer-bypass.md`) | Encoding (`01-encoding-analysis.md`) |
|------------|------------------------------------------|------------------------------------------------|----------------------------------------------|--------------------------------------|
| F1 — AES-CTR w/o auth tag | — | Reachability F1: 7 entry points confirmed | — | — |
| F2 — DOM-XSS `prismaDocument.name` | Finding 7: full source→sink chain | Reachability F2: LATENT (Zod transform sanitizes) | N1: every known write path runs `sanitizePlainText` | — |
| F3 — DOM-XSS `domainJson.apexName` | — | Reachability F3: confirmed, near-zero practical exploit (Vercel DNS-label charset) | N5: same; only hardcoded callers today | — |
| F4 — XXE-adjacent Python DOCX sanitizer | — | Reachability F4: TUS upload → Trigger.dev task → `execFile("python3", …)` → `ET.parse`; **CPython 3.x does not expand external entities by default** | — | — |
| F5 — `$executeRawUnsafe` savepoint name | — | Reachability F5: **DEAD_CODE today** — `savepointName` is `bulk_main_folder_lvl_${depth}` with `depth` a loop integer | — | — |
| F6 — SAML vendor XML parser | (related: Finding 8 — SAML `CredentialsProvider` upsert from IdP-influenceable flag, separate finding) | Reachability F6: PUBLIC endpoint by SAML design | — | — |

---

## Ranking rationale (exploitability)

The ranking below uses the reachability verdicts:

1. **HIGH = any reachable bug today** (even if the practical attack needs a
   pre-condition, the bug is real and the code is reachable).
2. **MEDIUM = latent or constrained** (reachable but the current input does not
   trigger it; or the trust boundary is enforced by a downstream constraint that
   is correct but brittle).
3. **LOW = dead-code sink today / third-party trust boundary / info-only**.

The tool side would normally contribute independent confirmation and bump
confidence to HIGH; here, with no tool output, every finding sits at **MEDIUM
confidence** by default.

---

## Synthesized Findings (ranked by exploitability, highest first)

---

### S-001 — AES-256-CTR without integrity protection on document passwords

- **Severity:** HIGH
- **Source:** AI-only (semgrep not installed; no tool confirmation)
- **File:** `lib/utils.ts:595-622` (`generateEncrpytedPassword`), `lib/utils.ts:624-652` (`decryptEncrpytedPassword`)
- **Confidence:** MEDIUM (single source); exploitation context corroborated by reachability analysis (7 entry points, all reachable)
- **Exploitability:** HIGH (in the threat model where DB write access is already obtained — e.g. via backup leak, SQL injection elsewhere, or admin-token compromise — the lack of auth tag turns a passive DB read into an active forgery)
- **Description:** Document passwords are encrypted with `crypto.createCipheriv("aes-256-ctr", …)` (line 618). CTR mode is a stream cipher and produces no authentication tag; `decryptEncrpytedPassword` does not call `setAuthTag()` because none exists. An attacker who can write to the `Link.password` column can XOR-flip arbitrary bits of the plaintext and the server will accept the forged password on next view. This breaks password-gated link-document access.
- **Reachability:** 7 entry points confirmed reachable from authenticated team members and incoming-webhook callers (`pages/api/links/index.ts:60-505`, `pages/api/links/[id]/index.ts`, `pages/api/webhooks/services/[...path]/index.ts` ×4 sub-handlers, `lib/api/links/bulk-import.ts:360-433`). Reachability analysis verdict: "INTRANET" deployment exposure, MEDIUM priority.
- **Comparison primitive:** the Slack token path (`lib/integrations/slack/utils.ts:41-74`) uses AES-256-GCM with a 12-byte IV and stores the auth tag — the correct pattern to mirror.
- **Secondary defect (key derivation):** `digest("base64").substring(0, 32)` truncates the SHA-256 output to 32 ASCII characters (not 32 bytes), reducing the effective key space and embedding charset bias (only base64 alphabet). Slack path uses the full 32-byte digest.
- **Recommendation:** Migrate to AES-256-GCM with `getAuthTag()` persisted alongside the ciphertext; on decryption call `setAuthTag()` and surface errors as "decryption failed". Use the full 32-byte SHA-256 digest as the key.

---

### S-002 — DOM-XSS surface on `prismaDocument.name` (latent stored XSS)

- **Severity:** MEDIUM
- **Source:** AI-only (semgrep not installed); independently cross-referenced in structural-analysis F7, reachability F2, sanitizer-bypass N1
- **File:** `components/documents/document-header.tsx:604`
- **Confidence:** MEDIUM (single SAST source but four passes independently confirm the call chain and sanitization surface)
- **Exploitability:** MEDIUM (sink is unsanitized at render; **today** every active write path runs `sanitizePlainText`, so the exploit window is closed — but the defense is at the wrong layer)
- **Description:** `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` is rendered inside a `contentEditable` `<h2>`. Every active write path (`pages/api/teams/[teamId]/documents/[id]/update-name.ts:13-22`, `documentUploadSchema` in `lib/zod/url-validation.ts:202-213`, `app/(ee)/api/links/[id]/upload/route.ts:210`) runs the value through `sanitizePlainText` (`lib/utils/sanitize-html.ts:12-20`) which calls `sanitize-html` with `allowedTags: []` / `allowedAttributes: {}` and then `decodeHTML(...).normalize("NFC")`. Today's payload is plain text and the sink is safe.
- **Residual risk:** any future write path that forgets to call `sanitizePlainText` (Notion import, bulk CSV upload, admin migration, restored legacy data) immediately produces stored XSS that fires in every team member's session. The `<h2>` is `contentEditable`, so the self-XSS vector (paste HTML, then submit) is also live — same user, but it does demonstrate the sink is unsafe.
- **Reachability:** INTERNET exposure (dashboard, after login); any team member (no role gate on rename) can trigger the chain.
- **Recommendation:** Replace `dangerouslySetInnerHTML={{ __html: prismaDocument.name }}` with `{prismaDocument.name}`. React auto-escapes string children. `contentEditable` does not require an HTML string — it needs a DOM node whose `textContent` is editable. If a styled placeholder is needed, render an explicit `<span>` child.

---

### S-003 — DOM-XSS surface on `domainJson.apexName` in `MarkdownText`

- **Severity:** MEDIUM
- **Source:** AI-only (semgrep not installed); independently cross-referenced in reachability F3, sanitizer-bypass N5
- **File:** `components/domains/domain-configuration.tsx:143` (sink); 96-101 (source)
- **Confidence:** MEDIUM (single SAST source but three passes confirm; **mitigated by upstream trust constraint**)
- **Exploitability:** LOW (Vercel's API enforces DNS-label charset — labels cannot contain `<`, `>`, or `"`, so the practical exploit window is closed today)
- **Description:** `instructions` is built by interpolating `domainJson.apexName` / `domainJson.name` into an HTML literal (`<code>${...}</code>`), then handed to `MarkdownText`, which renders the result with `dangerouslySetInnerHTML`. The sink is wrong in principle: if any upstream (Vercel API, a future custom-domain feature, a misconfigured Vercel project) ever returns a value with metacharacters, the payload executes in the team admin's session.
- **Reachability:** INTERNET (team settings pages); `getServerSession` gates the layout.
- **Recommendation:** Build `instructions` with React nodes (`<code>{domainJson.apexName}</code>`) instead of an HTML string, and render `MarkdownText` as plain text. Alternatively, drop the `<code>` wrapper and render the domain name in a styled `<span>` outside the instruction line.

---

### S-004 — XXE-adjacent risk in Python DOCX sanitizer (`ET.parse`)

- **Severity:** LOW
- **Source:** AI-only (semgrep not installed); reachability F4 confirms the upload → Trigger.dev task → `execFile("python3", …)` → `ET.parse` chain
- **File:** `ee/features/conversions/python/docx-sanitizer.py:146` (inside `_strip_numpages_fields`); also relevant at lines 216-303 (`sanitize_docx`)
- **Confidence:** MEDIUM (single SAST source; reachability confirmed; **practical XXE NOT exploitable in CPython 3.x default config**)
- **Exploitability:** LOW today; latent risk only if a future maintainer switches to `lxml.etree.parse` (default XXE-vulnerable) or enables `resolve_entities` on the parser
- **Description:** `xml.etree.ElementTree.parse` does not expand external entities in CPython 3.x by default, so the practical XXE risk is low today. The file is, however, part of a security-sensitive sanitizer, and the XML payload inside a DOCX header/footer comes from the original document author — a hostile author who is also a Papermark user can plant `<!DOCTYPE … SYSTEM "…">`. The function also re-serializes the parsed XML back to disk (`tree.write(path, …)`), so any expanded-entity content gets re-persisted into the user's stored file.
- **Reachability:** INTRANET (requires a dataroom link to upload); chain is `UploadZone` → `tusServer.handle` → `onUploadFinish` → Trigger.dev → `execFile(python3, docx-sanitizer.py, …)` → unzipped DOCX → `ET.parse(word/header*.xml)`.
- **Recommendation:** Use `ET.iterparse(path, events=("start",), parser=ET.XMLParser(resolve_entities=False, no_network=True))` or switch the whole sanitizer to `defusedxml.ElementTree` (drop-in replacement, hardened against XXE by default). Verify the same protection in `sanitize_docx` (lines 216-303).

---

### S-005 — `$executeRawUnsafe` savepoint name (defense-in-depth gap)

- **Severity:** LOW
- **Source:** AI-only (semgrep not installed); reachability F5 confirms `DEAD_CODE` from exploit standpoint today
- **File:** `lib/folders/bulk-create.ts:51, 54, 57-60` (inside `withUniqueConstraintRetry`)
- **Confidence:** MEDIUM (single source; reachability confirms no current exploit path)
- **Exploitability:** LOW today (no user input flows into `savepointName`; both call sites pass ``bulk_main_folder_lvl_${depth}`` / ``bulk_dr_folder_lvl_${depth}`` where `depth` is a loop integer — `lib/folders/bulk-create.ts:365, 469`); latent SQL-injection sink if a future caller passes a request-derived string
- **Description:** `await tx.$executeRawUnsafe(\`SAVEPOINT "${savepointName}"\`)` uses string concatenation with double-quoted SQL identifiers. The current callers pass `bulk_main_folder_lvl_${depth}` where `depth` is a numeric loop counter, so practical risk is zero. The function signature, however, accepts `savepointName: string` as free-form input — a future caller that passes request-derived data would activate a SQL injection that the double-quote wrapping does not protect against (no escaping of embedded `"`).
- **Recommendation:** Replace the four `$executeRawUnsafe` calls with `$executeRaw(Prisma.sql\`SAVEPOINT ${Prisma.raw(savepointName)}\`)` (using `Prisma.raw` only because savepoint identifiers cannot be bound as parameters) and add a regex guard `^[A-Za-z_][A-Za-z0-9_]*$` at the top of `withUniqueConstraintRetry`. Even simpler: drop the parameter and build the savepoint name internally from a fixed prefix + the retry counter.

---

### S-006 — SAML XML parsing handled by third-party `@boxyhq/saml-jackson`

- **Severity:** INFO
- **Source:** AI-only (semgrep not installed); reachability F6 confirms the public-by-design endpoint
- **File:** `app/(ee)/api/auth/saml/authorize/route.ts:7-77`
- **Confidence:** MEDIUM (single source; the actual XML parser is in vendor code, out of scope for Papermark SAST)
- **Exploitability:** INFO (the trust boundary is third-party; Papermark does not perform its own XML parsing or hardening)
- **Description:** The route delegates SAML AuthnRequest / SAMLResponse to `@boxyhq/saml-jackson`'s `oauthController` / `samlController`. BoxyHQ's library has had past advisories around SAML signature validation and XML parsing. Papermark does not perform defense-in-depth (no `Content-Length` cap on response form, no `RelayState` allowlist, no `jackson` patch pinning).
- **Reachability:** INTERNET (public by SAML design — IdP and SP must both reach each other).
- **Note:** `npm-audit` (separate domain) flags `@boxyhq/saml-jackson` transitively affected by `@boxyhq/metrics` (HIGH) — that is a dependency-version concern tracked in the synthesized-dependencies pass, not here.
- **Recommendation:** Track `@boxyhq/saml-jackson` for XXE / SAML advisories via Dependabot / Renovate. Add an integration test that submits a SAMLResponse with `<!DOCTYPE … SYSTEM "…">` and asserts rejection. Document the SAML library version in the security review policy.

---

## False-positive elimination

After spot-checking each finding against the actual source (`lib/utils.ts:595-622`,
`lib/folders/bulk-create.ts:51-60`, `lib/folders/bulk-create.ts:365, 469`,
`components/documents/document-header.tsx:604`, `components/domains/domain-configuration.tsx:96-101, 143`,
`ee/features/conversions/python/docx-sanitizer.py:145-147`), all 6 AI findings
reproduce verbatim. None are false positives in the strict sense; however, the
*exploitability* of several is bounded by upstream defenses that the AI pass
identified and the reachability / sanitizer-bypass passes independently
verified:

- **S-002** is rendered safe today by `sanitizePlainText` running on every active
  write path; the *sink* is still wrong, and the *latent* exploit window is the
  finding.
- **S-003** is rendered safe today by Vercel's DNS-label charset enforcement on
  `apexName` / `name`; the *sink* is still wrong, and the *latent* exploit
  window is the finding.
- **S-004** is rendered safe today by CPython 3.x's default `ET.parse`
  configuration (no external-entity expansion); the *sink* is still wrong, and
  the *latent* exploit window is the finding (and it is amplified by the
  re-serialization step).
- **S-005** is rendered safe today because `savepointName` is never
  user-controlled in the only two callers; the *sink* is still wrong, and the
  *latent* exploit window is the finding.
- **S-006** is a vendor trust boundary, not a Papermark SAST finding in the
  exploitable sense.

These are not false positives; they are correctly characterized as **latent**
or **constrained** sinks.

---

## Deduplication

No duplicates between sources (only one source). No duplicates within the AI
set. No duplicates across the structural / reachability / sanitizer-bypass /
encoding cross-references either — those passes cite the same findings by ID
(SAST F1 → Reachability F1 → Synthesized S-001, etc.) without re-counting them
as new bugs.

---

## Tool-side gap

`semgrep` was not installed in the environment (`tools-raw/semgrep.json:
{"tool": "semgrep", "error": "not_installed"}`). The normalized output contains
zero semgrep entries — only `npm-audit` dependency findings (out of scope for
SAST). This means:

1. **Confidence cannot exceed MEDIUM** for any synthesized finding, regardless
   of how thorough the AI analysis is.
2. **Tool-only finding column is empty.** No additional bugs to investigate.
3. To raise confidence to HIGH on any of the 6 findings, re-run `semgrep` (with
   the `p/security-audit` and `p/javascript` rulesets at minimum) and the
   `p/python` ruleset for `ee/features/conversions/python/docx-sanitizer.py`.
   Expected high-signal rules: `node.lang.security.audit.crypto-node-aes`,
   `javascript.lang.security.audit.react.no-danger`,
   `python.lang.security.audit.xml.etree`,
   `javascript.lang.security.audit.detect-non-literal-regexp`,
   `javascript.lang.security.audit.detect-non-literal-fs-filename` (negative).

---

## PHASE_3_CHECKPOINT

- [x] Both sources read and compared (semgrep returns `not_installed`; AI analysis loaded with 6 findings)
- [x] Duplicates merged (N/A — single source)
- [x] False positives eliminated (none — all 6 reproduce in source; severity adjusted where defenses bound practical exploitability)
- [x] Findings ranked by exploitability (HIGH → INFO; see S-001 → S-006)

---

## Output ledger

| ID | Title | Severity | Source | File:Line |
|----|-------|----------|--------|-----------|
| S-001 | AES-256-CTR without integrity protection on document passwords | HIGH | AI-only | `lib/utils.ts:595-622` |
| S-002 | DOM-XSS surface on `prismaDocument.name` (latent stored XSS) | MEDIUM | AI-only | `components/documents/document-header.tsx:604` |
| S-003 | DOM-XSS surface on `domainJson.apexName` (latent; DNS-label constrained) | MEDIUM | AI-only | `components/domains/domain-configuration.tsx:143` |
| S-004 | XXE-adjacent risk in Python DOCX sanitizer | LOW | AI-only | `ee/features/conversions/python/docx-sanitizer.py:146` |
| S-005 | `$executeRawUnsafe` savepoint name (defense-in-depth gap) | LOW | AI-only | `lib/folders/bulk-create.ts:51-60` |
| S-006 | SAML XML parsing handled by `@boxyhq/saml-jackson` | INFO | AI-only | `app/(ee)/api/auth/saml/authorize/route.ts:7-77` |

Cross-references for downstream passes:
- Encoding-analysis F1-F3 (CSV injection) are out of scope for SAST but should be merged into the synthesized-encoding ledger.
- Structural-analysis F1-F3 (CRITICAL IDOR, CORS) and F4-F8 are out of scope for SAST — they belong to the synthesized-structural ledger.
- Sanitizer-bypass N1-N6 are non-findings that document safe patterns; they confirm S-002 and S-003 are latent (not active) today.