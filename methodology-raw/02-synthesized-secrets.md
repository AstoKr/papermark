# Synthesized Secrets Findings — `synthesize-secrets`

**Workflow:** whitebox-bug-finder (M6)
**Methodology:** M6 — Hardcoded / leaked secrets
**Codebase:** papermark (Next.js 14 App Router, multi-tenant SaaS)
**Date:** 2026-06-19

---

## Source Status

| Source | Status | Findings contributed |
|--------|--------|----------------------|
| AI manual analysis (`methodology-raw/00-ai-secrets.md`) | Run | 7 |
| gitleaks (`tools-raw/gitleaks.json`) | **not_installed** — tool unavailable in runner | 0 |
| trufflehog (`tools-raw/trufflehog.json`) | **not_installed** — tool unavailable in runner | 0 |
| `tools-raw/normalized.json` (gitleaks/trufflehog slice) | No entries (the `summary.tools_with_findings` list mentions `"gitleaks"`, but no actual gitleaks/trufflehog finding objects exist in the array — only npm-audit entries) | 0 |
| `tools-raw/git-log-security.txt` | Recent commits reviewed for secret-leak signals (`secret`/`token`/`credential`/`password`/`key`); no historical secret-commit events found | 0 |

**Net effect:** All synthesized findings originate from the AI manual analysis. Tools produced zero corroborating or contradictory data points, so cross-source dedup is a no-op. The synthesized list is the AI list re-ranked and re-graded against the live source.

---

## Dedup Notes

- AI ↔ tools: 0 matches (tools produced no findings).
- AI ↔ other AI reports: Two cross-references were already documented in `00-ai-secrets.md` itself:
  - **Finding 3 (MEDIUM, placeholder in `.env.example`)** ↔ `00-ai-configs.md` Finding 9 (MEDIUM, default secret values). Same root cause, different grading lens. Kept as one entry here, flagged as cross-ref.
  - **Finding 7 (LOW, Trigger.dev project ID in source)** ↔ `00-ai-configs.md` Finding 15 (LOW). Duplicate. Kept once here, flagged as cross-ref.
- All other AI findings are unique to the secrets lens.

---

## Synthesized Findings

Findings are ranked by exploitability (CRITICAL first), then by ease of chaining with public information.

### S-001 — `NEXTAUTH_SECRET` is reused as the Jackson DB encryption key with an empty-string fallback

- **Severity:** CRITICAL
- **Source:** AI-only (tools not installed; nothing to corroborate from gitleaks/trufflehog)
- **File:** `lib/jackson.ts:16-20` (key derivation), `lib/jackson.ts:68` (`db.encryptionKey`), `lib/jackson.ts:72` (`clientSecretVerifier`)
- **Type:** dual-use-secret / empty-fallback-key-derivation
- **Description:** BoxyHQ Jackson's AES-256-GCM `encryptionKey` is derived from `NEXTAUTH_SECRET` via `sha256(NEXTAUTH_SECRET || "").digest("base64url").substring(0, 32)`. If `NEXTAUTH_SECRET` is unset or empty, the key collapses to a known string. The same `NEXTAUTH_SECRET` is also passed as `clientSecretVerifier`, which validates HMACs on every Jackson-issued `code`.
- **Evidence (verified in source):**
  ```ts
  // lib/jackson.ts:16-20
  const encryptionKey = crypto
    .createHash("sha256")
    .update(process.env.NEXTAUTH_SECRET || "")
    .digest("base64url")
    .substring(0, 32);

  // lib/jackson.ts:59-74
  function getJacksonOptions(): JacksonOption {
    return {
      // ...
      db: {
        engine: "sql",
        type: "postgres",
        url: getJacksonDbUrl(),
        encryptionKey,
      },
      idpEnabled: true,
      scimPath: "/api/scim/v2.0",
      clientSecretVerifier: process.env.NEXTAUTH_SECRET as string,
    };
  }
  ```
- **Exploit chain (public-source pivot):**
  1. Read `my-superstrong-secret` from `.env.example` (public repo).
  2. Use it to derive the same `encryptionKey` (`sha256` truncation).
  3. Decrypt every SAML connection row, SCIM directory row, and OAuth token at rest in Jackson-owned tables.
  4. Use the same string as `clientSecretVerifier` to forge valid `code` parameters on `/api/auth/saml/token` (called from `lib/auth/auth-options.ts:144-149`) and `/api/auth/saml/authorize`, logging in as any user whose email is known.
- **Confidence:** HIGH (code path traced and verified; pattern matches an active attack chain).
- **Exploitability:** HIGH — requires only the published placeholder value and an authenticated SAML setup, no other preconditions.
- **Remediation:**
  1. Introduce dedicated `JACKSON_ENCRYPTION_KEY` (32 bytes) and `JACKSON_CLIENT_SECRET_VERIFIER` env vars.
  2. Fail fast at module load if either is unset: `if (!process.env.JACKSON_ENCRYPTION_KEY) throw new Error(...)`.
  3. Rotate all rows encrypted under the old derived key during rollover.

---

### S-002 — `NEXTAUTH_SECRET` is used as both the SAML NextAuth `clientSecret` and the Jackson OAuth `client_secret`

- **Severity:** HIGH
- **Source:** AI-only (tools not installed)
- **File:** `lib/auth/auth-options.ts:128` (SAML NextAuth provider), `lib/auth/auth-options.ts:149` (Jackson `saml-idp` CredentialsProvider)
- **Type:** dual-use-secret (third and fourth consumer of the same string)
- **Description:** The SAML NextAuth provider sets `clientSecret: process.env.NEXTAUTH_SECRET` and the Jackson OAuth exchange sets `client_secret: process.env.NEXTAUTH_SECRET!`. Combined with S-001, the same string now holds four responsibilities — NextAuth JWT signer, Jackson DB encryption key source, Jackson `clientSecretVerifier`, and SAML/Jackson OAuth `client_secret`.
- **Evidence (verified in source):**
  ```ts
  // lib/auth/auth-options.ts:126-129
  options: {
    clientId: "dummy",
    clientSecret: process.env.NEXTAUTH_SECRET as string,
  },

  // lib/auth/auth-options.ts:144-150
  const { access_token } = await oauthController.token({
    code: credentials.code,
    grant_type: "authorization_code",
    redirect_uri: getMainDomainUrl(),
    client_id: "dummy",
    client_secret: process.env.NEXTAUTH_SECRET!,
  });
  ```
- **Confidence:** HIGH.
- **Exploitability:** HIGH — single published placeholder unlocks all four consumers.
- **Remediation:** Generate a dedicated `SAML_CLIENT_SECRET` (random 32+ bytes). The `clientId: "dummy"` literal is BoxyHQ's convention and is not a secret; do not change it.

---

### S-003 — Placeholder values in `.env.example` are realistic enough to ship to a real environment

- **Severity:** MEDIUM
- **Source:** AI-only (gitleaks/trufflehog would have flagged if installed, since the values are committed)
- **File:** `.env.example:1` (`NEXTAUTH_SECRET=my-superstrong-secret`), `.env.example:38` (`HANKO_API_KEY=add-your-hanko-api-key`), `.env.example:39` (`NEXT_PUBLIC_HANKO_TENANT_ID=add-your-hanko-tenent-id`), `.env.example:68` (`NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret`)
- **Type:** default-credential / placeholder-leak
- **Cross-ref:** Same root cause as `00-ai-configs.md` Finding 9 (config-hygiene lens). One entry kept here, scored from a secret-leak perspective.
- **Evidence (verified in source):**
  ```bash
  # .env.example
  NEXTAUTH_SECRET=my-superstrong-secret
  HANKO_API_KEY=add-your-hanko-api-key
  NEXT_PUBLIC_HANKO_TENANT_ID=add-your-hanko-tenent-id
  NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret
  ```
  `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` flows into `lib/utils.ts:624-630` as the SHA-256-derived AES-256-CTR key for every link's document password — same derivation pattern as S-001.
- **Confidence:** HIGH.
- **Exploitability:** HIGH in any environment that has not rotated the placeholder. Document password ciphertexts become decryptable to anyone with read access to the public repo.
- **Remediation:**
  1. Replace all four placeholders with empty values plus a header comment: `# ⚠️ DO NOT use the placeholder values. Generate a fresh secret for every environment.`
  2. Add a fail-fast startup guard in `middleware.ts` and `lib/jackson.ts` that refuses `my-superstrong-secret` / `my-superstrong-document-secret`.

---

### S-004 — `NEXTAUTH_SECRET` is published as `my-superstrong-secret` and is a 4-purpose secret (cross-file)

- **Severity:** MEDIUM
- **Source:** AI-only
- **File:** `.env.example:1`, `lib/jackson.ts:18, 72`, `lib/auth/auth-options.ts:128, 149`, `lib/middleware/app.ts:56`
- **Type:** dual-purpose-secret-publication (architectural finding that subsumes S-001/S-002/S-003)
- **Description:** A single published string unlocks four independent security boundaries: NextAuth session JWT signing, Jackson DB encryption, Jackson OAuth `client_secret` verification, and SAML NextAuth provider `clientSecret`. Fixing the architectural problem (one secret, one purpose) is the same remediation as S-001 + S-002 + S-003.
- **Confidence:** HIGH.
- **Exploitability:** Trivially chained: read public placeholder → forge NextAuth session → log in as any user → access any team → access any dataroom → decrypt any link password → read any document.
- **Remediation:** Combined fix of S-001, S-002, S-003. Long-term: rotate every existing environment that may have inherited `my-superstrong-secret`.

---

### S-005 — `REVALIDATE_TOKEN` is passed as a URL query parameter (`?secret=...`) instead of an HTTP header

- **Severity:** MEDIUM
- **Source:** AI-only
- **Files:** `lib/api/links/revalidate.ts:20, 55`, `pages/api/revalidate.ts:14`, `lib/trigger/pdf-to-image-route.ts:343`, `lib/trigger/convert-pdf-direct.ts:333`, `lib/api/documents/process-document.ts:296`, `pages/api/links/index.ts:491`, `pages/api/links/[id]/index.ts:743`, `pages/api/links/[id]/archive.ts:102`, `pages/api/teams/[teamId]/enable-advanced-mode.ts:126`
- **Type:** secret-in-URL (leak vector)
- **Description:** Every internal caller serializes `REVALIDATE_TOKEN` into the URL query string. Well-known leak vectors apply (server logs, Referer headers, CDN logs, browser history, screenshots, stack traces).
- **Evidence (verified in source):**
  ```ts
  // lib/api/links/revalidate.ts:19-21
  await fetch(
    `${revalidateUrl}/api/revalidate?secret=${revalidateToken}&linkId=${link.id}&hasDomain=${link.domainId ? "true" : "false"}`,
  );

  // pages/api/revalidate.ts:14
  if (req.query.secret !== process.env.REVALIDATE_TOKEN) {
    return res.status(401).json({ message: "Invalid token" });
  }
  ```
- **Note:** `/api/revalidate` itself is a low-impact target (DoS / cache stampede amplifier), but the anti-pattern matters because the same code shape is used elsewhere and could trivially be extended to higher-impact tokens (e.g. `INTERNAL_API_KEY`).
- **Confidence:** HIGH (mechanism verified in source).
- **Exploitability:** LOW for `REVALIDATE_TOKEN` in isolation. MEDIUM as an indicator that future secrets could be put in URLs.
- **Remediation:** Move the token to `x-revalidate-secret` HTTP header on both producer and consumer; deprecate the `?secret=` query path.

---

### S-006 — `INTERNAL_API_KEY` is a single shared bearer token across 26 call sites with no per-job scoping

- **Severity:** LOW
- **Source:** AI-only (the grep count of 26 files is from a fresh scan against the working tree; AI report cited a conservative "7+" — superseded by the verified count)
- **Files:** `lib/trigger/pdf-to-image-route.ts:130, 222`, `lib/trigger/dataroom-change-notification.ts:269`, `lib/trigger/dataroom-upload-notification.ts:106`, `lib/files/get-file.ts:99`, `lib/api/notification-helper.ts:19, 51`, `pages/api/file/s3/get-presigned-get-url-proxy.ts:67`, plus 20 additional consumers in `pages/api/mupdf/`, `pages/api/links/download/`, `pages/api/jobs/`, and `ee/features/billing/cancellation/`
- **Type:** shared-credential / no-rotation-enforcement
- **Description:** The same bearer token is used by every Trigger.dev background job to call internal endpoints (PDF rendering, dataroom notifications, presigned URL generation, billing job orchestration). Any one consumer's token leak compromises the rest; no rotation mechanism or per-job sub-keys exist.
- **Confidence:** HIGH (26-file grep confirmed in source).
- **Exploitability:** MEDIUM — token is not currently observed in source, but the single-key design means a leak in any consumer compromises the full set of internal capabilities (DoS amplifier on PDF rendering, phishing pivot on dataroom notifications, arbitrary presigned S3 URL generation).
- **Remediation:**
  1. Per-job scoped tokens: `INTERNAL_API_KEY_PDF`, `INTERNAL_API_KEY_NOTIFICATIONS`, `INTERNAL_API_KEY_PRESIGN`.
  2. Document a rotation runbook in `SECURITY.md`.

---

### S-007 — Trigger.dev project ID is committed in `trigger.config.ts`

- **Severity:** LOW
- **Source:** AI-only
- **File:** `trigger.config.ts:7` (`project: "proj_plmsfqvqunboixacjjus"`)
- **Type:** info-disclosure (tenant identifier, not credential)
- **Cross-ref:** Duplicate of `00-ai-configs.md` Finding 15.
- **Description:** This is by design — Trigger.dev recommends committing the project ID. The credential is `TRIGGER_SECRET_KEY` (env-only). Worth noting for log-scrubbing hygiene: any log line containing the project ID paired with a leaked `TRIGGER_SECRET_KEY` becomes the full credential.
- **Confidence:** HIGH.
- **Exploitability:** Negligible by itself.
- **Remediation:** None required. Add a note in `SECURITY.md` that `TRIGGER_SECRET_KEY` is the sensitive value and must never be committed.

---

## Summary Table (ranked by exploitability)

| ID | Sev | Title | File(s) | Source | Confidence |
|----|-----|-------|---------|--------|------------|
| S-001 | CRITICAL | `NEXTAUTH_SECRET` reused as Jackson DB encryption key with empty fallback | `lib/jackson.ts:16-20, 68, 72` | AI-only | HIGH |
| S-002 | HIGH | `NEXTAUTH_SECRET` used as SAML `clientSecret` and Jackson OAuth `client_secret` | `lib/auth/auth-options.ts:128, 149` | AI-only | HIGH |
| S-003 | MEDIUM | Placeholder values in `.env.example` are realistic enough to ship | `.env.example:1, 38, 39, 68` | AI-only | HIGH |
| S-004 | MEDIUM | `NEXTAUTH_SECRET` is a 4-purpose secret published in the repo | cross-file (S-001/S-002/S-003) | AI-only | HIGH |
| S-005 | MEDIUM | `REVALIDATE_TOKEN` passed as URL query parameter | 9 files (see list) | AI-only | HIGH |
| S-006 | LOW | `INTERNAL_API_KEY` shared across 26 call sites with no scoping | 26 files | AI-only | HIGH |
| S-007 | LOW | Trigger.dev project ID committed in `trigger.config.ts` | `trigger.config.ts:7` | AI-only | HIGH |

---

## Tool-Output Gaps

- gitleaks and trufflehog were both unavailable in this runner (`"error": "not_installed"` in `tools-raw/gitleaks.json` and `tools-raw/trufflehog.json`). Re-running this synthesis in an environment where either tool is installed would likely produce additional corroborating findings on S-003 / S-007, and possibly additional low-severity findings for hardcoded test fixtures or third-party SDK defaults.
- `tools-raw/normalized.json` claims `tools_with_findings` includes `"gitleaks"`, but no gitleaks finding objects exist in the array. Treat the summary field as inconsistent with the underlying data and trust the per-finding objects (which contain only npm-audit entries).

## PHASE_3_CHECKPOINT

- [x] Both sources read and compared (AI run; tools `not_installed`)
- [x] Duplicates merged (no AI↔tool overlaps; two AI↔AI cross-refs flagged in S-003 and S-007)
- [x] False positives eliminated (each finding verified in source by reading the relevant file)
- [x] Findings ranked by exploitability (CRITICAL → LOW)