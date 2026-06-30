# synthesize-secrets — Secrets Synthesis

## 1. AI Findings

| ID | File | Line | Severity | Description |
|---|---|---|---|---|
| A1 | `.env.example` | 1 | CRITICAL | `NEXTAUTH_SECRET=my-superstrong-secret` — default credential used across 5 security domains |
| A2 | `.env.example` | 68 | HIGH | `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret` — default key for AES-256-CTR document password encryption |
| A3 | `trigger.config.ts` | 7 | LOW | `project: "proj_plmsfqvqunboixacjjus"` — hardcoded Trigger.dev project identifier |

## 2. Tool Findings (gitleaks, trufflehog)

All 29 gitleaks findings are in `.gitnexus/meta.json` — a metadata index containing SHA-256 content hashes of source files. The `generic-api-key` rule is matching 64-character hex strings (e.g., `"27de475defb3345efa0d03fe092784b6e256c7913c353352e6bae131000a461b"`) in a JSON dictionary mapping filenames to integrity hashes. These are **not secrets**.

No trufflehog findings present.

**Verdict: 29/29 tool findings are false positives. Zero actionable tool findings for the secrets domain.**

## 3. AI-Only Findings (Missed by Tools)

A1, A2, and A3 were all missed by gitleaks and trufflehog. The tools do not scan `.env.example` for default/placeholder credentials, and the Trigger.dev project ID is not recognized as a secret pattern.

## 4. Tool-Only Findings (Missed by AI)

None. The gitleaks hits in `.gitnexus/meta.json` are false positives; AI correctly identified them as non-secrets in its non-findings table (line 67-68 of the AI recon, which lists investigated patterns).

## 5. Merge

No overlap exists between AI and tool findings. All three surviving findings are AI-only.

## 6. Verification of AI-Only Findings

- **A1 (NEXTAUTH_SECRET)**: Confirmed at `lib/middleware/app.ts:56` (`getToken({ req, secret: process.env.NEXTAUTH_SECRET })`), `lib/jackson.ts:16-20` (AES-256-GCM key derivation via `SHA-256(NEXTAUTH_SECRET)`), `lib/signing/download-token.ts:19` and `lib/signing/access-token.ts:21` (HMAC-SHA256 key), `lib/api/auth/token.ts:12` (API token hash preimage).
- **A2 (DOCUMENT_PASSWORD_KEY)**: Confirmed at `lib/utils.ts:614-628` — `SHA-256(NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY)` truncated to 32 bytes as AES-256-CTR key for `encryptEncrpytedPassword()` / `decryptEncrpytedPassword()`.
- **A3 (Trigger.dev project ID)**: Confirmed at `trigger.config.ts:7` — `project: "proj_plmsfqvqunboixacjjus"` in `defineConfig()`. Not a bearer credential.

## 7. Verification of Tool-Only Findings

All 29 gitleaks findings checked against `.gitnexus/meta.json` — confirmed as SHA-256 file integrity hashes, not secrets. Every matched line follows the pattern `<filename>: "<64-char-hex>"` inside a JSON file index. **False positive.**

## 8. False Positives Eliminated

- **All 29 gitleaks findings**: FALSE POSITIVE — `.gitnexus/meta.json` content hashes
- **A3 (Trigger.dev project ID)**: FALSE POSITIVE candidate, but retained as informational — the project ID is not a credential (Trigger.dev auth uses `TRIGGER_SECRET_KEY` from env), but hardcoding it couples config to a specific workspace and may leak internal project naming conventions.

## 9. Ranked Findings

---

### S-001 — NEXTAUTH_SECRET uses well-known default value

- severity: CRITICAL
- source: AI-only
- file: `.env.example:1`
- description: `NEXTAUTH_SECRET=my-superstrong-secret` is a static placeholder used as the root cryptographic secret across five security domains: NextAuth JWT signing (session forgery), SAML Jackson client_secret/encryption key (SAML token replay, encrypted data decryption), HMAC-signed download/access tokens (forgery bypass), and API token hashing (pre-image attack). Any deployment that copies `.env.example` to production without changing this value inherits a publicly known secret from a public GitHub file.
- evidence: `NEXTAUTH_SECRET=my-superstrong-secret`
- confidence: HIGH
- exploitability: A single known value unlocks session impersonation (any user), SAML/SCIM encrypted data decryption, signed document access bypass, and API credential recovery. Self-hosted deployments that follow the example verbatim are vulnerable. Vercel deployments using Vercal's system env vars are likely safe.

---

### S-002 — NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY uses well-known default value

- severity: HIGH
- source: AI-only
- file: `.env.example:68`
- description: `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret` is a static example value for AES-256-CTR document password encryption. Used in `lib/utils.ts:614-628` to derive the encryption key for `encryptEncrpytedPassword()` and `decryptEncrpytedPassword()`. If unchanged from default, any document password encrypted in the database is decryptable by anyone who knows the source (anyone who reads `.env.example`).
- evidence: `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret`
- confidence: HIGH
- exploitability: Self-hosted deployments that copy `.env.example` verbatim expose all document-level passwords. Less severe than S-001 because scope is limited to document passwords (not cross-system), but the key derivation is deterministic (SHA-256, no salt) and the key length is exactly 32 bytes.

---

### S-003 — Trigger.dev project ID hardcoded in config

- severity: LOW
- source: AI-only
- file: `trigger.config.ts:7`
- description: `project: "proj_plmsfqvqunboixacjjus"` is a Trigger.dev project identifier embedded in the config file. This is a project reference, not a bearer token or API key — Trigger.dev uses `TRIGGER_SECRET_KEY` from the environment for authentication.
- evidence: `project: "proj_plmsfqvqunboixacjjus"`
- confidence: MEDIUM (it is intentionally exposed but offers marginal recon value)
- exploitability: No direct exploit. The project ID is analogous to an API path prefix and is not a credential. An attacker could identify the Trigger.dev project for targeted reconnaissance.

---

## Summary

| Finding | Severity | Source | Type |
|---|---|---|---|
| S-001 - NEXTAUTH_SECRET default | CRITICAL | AI-only (gitleaks/trufflehog missed) | Default credential |
| S-002 - Document password key default | HIGH | AI-only | Default credential |
| S-003 - Trigger.dev project ID | LOW | AI-only | Hardcoded identifier |

**Tool findings eliminated as false positives: 29/29 (all gitleaks hits in `.gitnexus/meta.json`).**

The critical mass is S-001: a single `.env.example` value that, if unmodified, compromises session auth, SAML/SCIM encryption, signed document tokens, and API credential storage. S-002 is narrower (document passwords only) but shares the same root cause. Both are deployment misconfiguration risks, not code flaws.

**Consumed by:** `web-search-secrets`
