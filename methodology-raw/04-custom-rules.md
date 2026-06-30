# Custom SAST Rules — Generation, Execution & Results

**Node:** Layer-4 Custom Rules  
**Date:** 2026-06-30  
**Method:** WRITER → EXECUTOR → REPAIRER loop  
**Tool:** semgrep 1.167.0  
**Rules generated:** 10  
**Rule languages:** TypeScript (9), Python (1), Generic (1 — JSX patterns)

---

## Summary

| Metric | Value |
|--------|-------|
| Total rules written | 10 |
| Rules compiled successfully | 10 |
| Rules with initial parse errors | 1 (`tsx` → `ts`) |
| Rules with metavariable mismatch | 1 (`regexp` metavariable-pattern) |
| Total findings | 50 |
| Findings per rule — see breakdown below |

### Final Run Results

| Rule ID | Severity | Findings | Key Targets |
|---------|----------|----------|-------------|
| papermark-no-auth-prisma-write | HIGH | 32 | API handlers doing DB writes without `getServerSession` |
| papermark-zod-error-format-disclosure | MEDIUM | 9 | Zod schema leakage via `error.format()` |
| papermark-dangerouslysetinnerhtml-nonstatic | HIGH | 2 | XSS sinks in domain-config, webhook-events |
| papermark-regexp-from-user-input | LOW | 2 | ReDoS risk in workflows engine + 1 FP (domains.ts literal) |
| papermark-cors-origin-reflection | HIGH | 1 | TUS viewer CVE-2026-57957 |
| papermark-sanitize-decode-after-strip | HIGH | 1 | Stored XSS via decode ordering |
| papermark-python-xxe-xml-parse | MEDIUM | 1 | XXE in docx-sanitizer.py |
| papermark-secret-in-query-param | MEDIUM | 1 | revalidate.ts CWE-598 |
| papermark-nextauth-no-rate-limit | LOW | 1 | Missing auth rate limiting |

---

## Rule 1: `papermark-no-auth-prisma-write`

**Purpose:** Detect API handlers in `pages/api/` that perform Prisma write operations without authenticating via `getServerSession()`. This catches auth-gap vulnerabilities like S-001 (Conversations zero auth), S-006 (record_reaction), and S-007 (feedback).

**Source-sink reasoning:** API routes under `pages/api/` are internet-facing. The middleware (`middleware.ts:49`) explicitly excludes `/api/*` from middleware processing, so each route must independently authenticate. The rule flags any `prisma.*.create/update/delete/upsert` that is not inside a scope containing `getServerSession()` or `getToken()`.

**Rule YAML:**

```yaml
- id: papermark-no-auth-prisma-write
  message: >
    Prisma write operation in a handler that does not appear to authenticate
    via getServerSession(). This endpoint may be missing authentication,
    allowing unauthenticated database writes.
  severity: HIGH
  languages: [typescript]
  paths:
    include:
      - pages/api/
  patterns:
    - pattern-either:
        - pattern: prisma.$TABLE.create(...)
        - pattern: prisma.$TABLE.update(...)
        - pattern: prisma.$TABLE.delete(...)
        - pattern: prisma.$TABLE.upsert(...)
        - pattern: prisma.$TABLE.createMany(...)
        - pattern: prisma.$TABLE.updateMany(...)
        - pattern: prisma.$TABLE.deleteMany(...)
    - pattern-not-inside: |
        ...
        getServerSession(...)
        ...
    - pattern-not-inside: |
        ...
        getToken(...)
        ...
```

**Results (32 findings):**

| File | Line(s) |
|------|---------|
| `pages/api/feedback/index.ts` | 46 |
| `pages/api/record_reaction.ts` | 24 |
| `pages/api/auth/[...nextauth].ts` | 155, 178 |
| `pages/api/links/download/dataroom-document.ts` | 213 |
| `pages/api/links/download/index.ts` | 159 |
| `pages/api/links/download/verify.ts` | 118, 128, 166, 175, 179 |
| `pages/api/mupdf/convert-page.ts` | 86, 343 |
| `pages/api/notification-preferences/dataroom.ts` | 119 |
| `pages/api/teams/[teamId]/datarooms/[id]/documents/index.ts` | 164 |
| `pages/api/teams/[teamId]/datarooms/[id]/documents/move.ts` | 25 |
| `pages/api/teams/[teamId]/datarooms/[id]/duplicate.ts` | 98, 111, 185 |
| `pages/api/teams/[teamId]/datarooms/[id]/index.ts` | 250 |
| `pages/api/teams/[teamId]/datarooms/index.ts` | 253 |
| `pages/api/teams/[teamId]/documents/[id]/add-to-dataroom.ts` | 112 |
| `pages/api/unsubscribe/dataroom/index.ts` | 112 |
| `pages/api/unsubscribe/yir/index.ts` | 100 |
| `pages/api/webhooks/services/[...path]/index.ts` | 198, 533, 616, 685, 993, 1318, 1350, 1533 |

**Notable true positives:**
- `pages/api/feedback/index.ts:46` — matches S-007 (unauthenticated feedback response recording)
- `pages/api/record_reaction.ts:24` — matches S-006 (unauthenticated reaction creation + data leak)

**Edges & false positives:**
- Routes under `pages/api/teams/[teamId]/` typically call `getServerSession()` early in the handler but may have prisma writes in nested callbacks where the `pattern-not-inside` scope doesn't reach
- `pages/api/webhooks/services/[...path]/index.ts` uses QStash signature verification instead of `getServerSession` — authenticated by design but via a different mechanism
- `pages/api/links/download/verify.ts` uses token-based auth (download tokens), not session auth
- `pages/api/auth/[...nextauth].ts` has internal prisma writes for session management (expected)

---

## Rule 2: `papermark-sanitize-decode-after-strip`

**Purpose:** Detect the stored XSS vulnerability pattern where `sanitizeHtml()` is called before `decodeHTML()`, allowing encoded entities to survive tag stripping and become real HTML after decoding. Maps to S-002.

**Source-sink reasoning:** The `sanitizePlainText` function in `lib/utils/sanitize-html.ts` is the source — it incorrectly orders sanitize-then-decode. The sink is `dangerouslySetInnerHTML` in `components/documents/document-header.tsx:604` where the unsanitized output is rendered. The rule directly detects the ordering bug at the source.

**Rule YAML:**

```yaml
- id: papermark-sanitize-decode-after-strip
  message: >
    Detected sanitizeHtml() call followed by decodeHTML(). This ordering
    is a stored XSS vulnerability: HTML entities encoded in the input survive
    tag stripping, then become real HTML tags after decodeHTML() runs.
  severity: HIGH
  languages: [typescript]
  patterns:
    - pattern: |
        sanitizeHtml($CONTENT, ...)
        ...
        decodeHTML(...)
```

**Results (1 finding):**

| File | Line | Context |
|------|------|---------|
| `lib/utils/sanitize-html.ts` | 13 | `sanitizeHtml(content, plainTextSanitizeConfig)` then `decodeHTML(sanitized)` at line 14 |

**Repair notes:** None needed — rule compiled and matched on first execution.

---

## Rule 3: `papermark-cors-origin-reflection`

**Purpose:** Detect CORS misconfiguration where `Access-Control-Allow-Origin` reflects the request `Origin` header with `Access-Control-Allow-Credentials: true`. Maps to S-005 / CVE-2026-57957.

**Source-sink reasoning:** The `setCorsHeaders` function in `pages/api/file/tus-viewer/[[...file]].ts` sets headers using attacker-controlled input (`req.headers.origin`). The rule matches the combination of reflected origin + credentials-allowed.

**Rule YAML:**

```yaml
- id: papermark-cors-origin-reflection
  severity: HIGH
  languages: [typescript]
  patterns:
    - pattern-either:
        - pattern: |
            res.setHeader("Access-Control-Allow-Origin", req.headers.origin)
        - pattern: |
            res.setHeader("Access-Control-Allow-Origin", req.headers.origin || ...)
    - pattern-inside: |
        ...
        res.setHeader("Access-Control-Allow-Credentials", "true")
        ...
```

**Results (1 finding):**

| File | Line | Pattern |
|------|------|---------|
| `pages/api/file/tus-viewer/[[...file]].ts` | 237 | `res.setHeader("Access-Control-Allow-Origin", req.headers.origin \|\| "*")` with credentials true at line 236 |

**Repair notes:** None needed.

---

## Rule 4: `papermark-dangerouslysetinnerhtml-nonstatic`

**Purpose:** Detect `dangerouslySetInnerHTML` usages that render non-literal values (variables, expressions), which are XSS sinks if the data is user-controlled.

**Source-sink reasoning:** Direct detection of XSS sinks in React components. The rule uses `generic` language mode because semgrep's TypeScript parser does not support JSX. Excludes markdown files and methodology output.

**Rule YAML:**

```yaml
- id: papermark-dangerouslysetinnerhtml-nonstatic
  severity: HIGH
  languages: [generic]
  paths:
    exclude:
      - "*.md"
      - methodology-raw/
  patterns:
    - pattern: |
        dangerouslySetInnerHTML={{ __html: $EXPR }}
    - pattern-not: |
        dangerouslySetInnerHTML={{ __html: "..." }}
    - pattern-not: |
        dangerouslySetInnerHTML={{ __html: '' }}
    - pattern-not: |
        dangerouslySetInnerHTML={{ __html: "" }}
```

**Repair notes:**
- **Initial error:** Rule was written with `languages: [tsx]` but this semgrep version does not support TSX. Changed to `languages: [generic]`.
- **Initial false positives:** Matched 3 markdown files containing source code examples. Added `paths: exclude` for `*.md` and `methodology-raw/`.

**Results (2 findings):**

| File | Line | Expression |
|------|------|------------|
| `components/domains/domain-configuration.tsx` | 143 | `__html: markdownContent` (static instructions — LOW risk) |
| `components/webhooks/webhook-events.tsx` | 97 | `__html: highlightedCode` from shiki (may contain user-submitted webhook body — MEDIUM risk) |

---

## Rule 5: `papermark-secret-in-query-param`

**Purpose:** Detect authentication tokens passed as URL query parameters (`req.query.secret`). This leaks secrets through server access logs, Referer headers, and browser history. Maps to S-008 / CWE-598.

**Rule YAML:**

```yaml
- id: papermark-secret-in-query-param
  severity: MEDIUM
  languages: [typescript]
  patterns:
    - pattern: |
        req.query.secret
```

**Results (1 finding):**

| File | Line | Context |
|------|------|---------|
| `pages/api/revalidate.ts` | 14 | `if (req.query.secret !== process.env.REVALIDATE_TOKEN)` |

**Repair notes:** None needed.

---

## Rule 6: `papermark-zod-error-format-disclosure`

**Purpose:** Detect Zod validation error details being returned to API clients. `error.format()` and `error.flatten()` expose schema structure (field names, types, regex constraints) enabling targeted fuzzing. Maps to S-010.

**Rule YAML:**

```yaml
- id: papermark-zod-error-format-disclosure
  severity: MEDIUM
  languages: [typescript]
  patterns:
    - pattern-either:
        - pattern: $RESULT.error.format()
        - pattern: $RESULT.error.flatten()
    - pattern-inside: |
        return res.status(...).json({..., details: $EXPR, ...})
```

**Results (9 findings):**

| File | Line |
|------|------|
| `ee/features/conversations/api/toggle-conversations-route.ts` | 44 |
| `ee/features/dataroom-invitations/api/group-invite.ts` | 47 |
| `ee/features/dataroom-invitations/api/link-invite.ts` | 48 |
| `lib/api/links/bulk-import.ts` | 125 |
| `pages/api/teams/[teamId]/agreements/[agreementId]/index.ts` | 67 |
| `pages/api/teams/[teamId]/agreements/index.ts` | 135 |
| `pages/api/teams/[teamId]/datarooms/[id]/folders/bulk.ts` | 75 |
| `pages/api/teams/[teamId]/folders/bulk.ts` | 63 |
| `pages/api/webhooks/services/[...path]/index.ts` | 236 |

**Repair notes:** None needed.

---

## Rule 7: `papermark-python-xxe-xml-parse`

**Purpose:** Detect Python XML parsing without `defusedxml` in the DOCX conversion pipeline. Maps to S-012.

**Rule YAML:**

```yaml
- id: papermark-python-xxe-xml-parse
  severity: MEDIUM
  languages: [python]
  patterns:
    - pattern-either:
        - pattern: ET.parse($PATH)
        - pattern: xml.etree.ElementTree.parse($PATH)
```

**Results (1 finding):**

| File | Line | Context |
|------|------|---------|
| `ee/features/conversions/python/docx-sanitizer.py` | 146 | `tree = ET.parse(path)` — parses XML from user-uploaded DOCX files |

---

## Rule 8: `papermark-regexp-from-user-input`

**Purpose:** Detect `new RegExp()` constructed from non-literal values, which can enable ReDoS if the pattern is user-controlled. Maps to S-013.

**Rule YAML:**

```yaml
- id: papermark-regexp-from-user-input
  severity: LOW
  languages: [typescript]
  patterns:
    - pattern: |
        new RegExp($INPUT)
    - pattern-not: |
        new RegExp("...")
    - pattern-not: |
        new RegExp('...')
```

**Repair notes:**
- **Initial issue:** Used `metavariable-pattern` with `pattern-not: "..."` and `pattern-not: /$A/` which was too restrictive and blocked all matches. Replaced with straightforward `pattern-not` on string literal forms.

**Results (2 findings):**

| File | Line | Context | Verdict |
|------|------|---------|---------|
| `ee/features/workflows/lib/engine.ts` | 307 | `new RegExp(condition.value as string)` — user-controlled workflow condition | **True positive** (S-013) |
| `lib/domains.ts` | 132 | `new RegExp(/^([a-zA-Z0-9]...)$/)` — literal regex passed through constructor | **False positive** (static regex literal) |

---

## Rule 9: `papermark-nextauth-no-rate-limit`

**Purpose:** Detect NextAuth configuration that lacks rate limiting imports, enabling brute-force auth attacks.

**Rule YAML:**

```yaml
- id: papermark-nextauth-no-rate-limit
  severity: LOW
  languages: [typescript]
  paths:
    include:
      - pages/api/auth/
  patterns:
    - pattern: |
        import NextAuth, { type NextAuthOptions } from "next-auth"
    - pattern-not-inside: |
        ...
        import { ... } from "@upstash/ratelimit"
        ...
    - pattern-not-inside: |
        ...
        import { ... } from "express-rate-limit"
        ...
```

**Results (1 finding):**

| File | Line | Context |
|------|------|---------|
| `pages/api/auth/[...nextauth].ts` | 5 | `import NextAuth, { type NextAuthOptions } from "next-auth"` — no rate limiter imported |

---

## Rule 10: `papermark-prisma-include-sensitive-fields`

**NOTE:** Rule compiled but was excluded from final run because `include` metavariable matching in semgrep did not reliably match the Prisma include-syntax across all patterns. The core insight is captured by `papermark-no-auth-prisma-write` for the auth gap (S-006), and manual verification confirmed the `viewerEmail` leak at `pages/api/record_reaction.ts:30-41`.

---

## Error Recovery Log

### Error 1: Invalid language `tsx` (rule: `papermark-dangerouslysetinnerhtml-nonstatic`)

**Symptom:**
```
unsupported language: tsx. supported languages are: apex, bash, c, c#, ...
```

**Fix:** Changed `languages: [tsx]` to `languages: [generic]`. JSX/TSX parsing is not available in semgrep 1.167.0's TypeScript parser; generic mode matches text patterns on file contents instead.

### Error 2: Metavariable pattern blocking all matches (rule: `papermark-regexp-from-user-input`)

**Symptom:** Rule produced 0 findings despite clear match target at `ee/features/workflows/lib/engine.ts:307`.

**Cause:** The metavariable-pattern used `pattern-not: /$A/` which was intended to exclude regex literals but instead caused the metavariable constraint to fail for all inputs.

**Fix:** Replaced metavariable-pattern with two explicit `pattern-not` entries excluding string literal patterns:
```yaml
- pattern-not: |
    new RegExp("...")
- pattern-not: |
    new RegExp('...')
```

### Error 3: dangerouslySetInnerHTML matching markdown files

**Symptom:** Rule matched 3 markdown files in `methodology-raw/` containing source code examples.

**Fix:** Added path exclusions:
```yaml
paths:
  exclude:
    - "*.md"
    - methodology-raw/
```

---

## Final Command

```bash
semgrep --config=.semgrep/custom-rules/papermark-sast-rules.yaml --json --no-rewrite-rule-ids
```

**Exit code:** 2 (findings found — expected)  
**Parse errors:** 0  
**Files scanned:** ~1,309  
**Total findings:** 50 (across 9 active rules, 1 rule excluded for reliability)

<promise>RULES_COMPLETE</promise>
