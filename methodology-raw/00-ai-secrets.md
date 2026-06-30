# ai-analyze-secrets — Hardcoded Secret Recon

## Summary

Three findings: two placeholder secrets in `.env.example` that code silently accepts without guarding against default values, and one hardcoded project identifier. No actual secret material (live API keys, tokens, passwords, or private keys) was found committed to the source tree.

---

### NEXTAUTH_SECRET uses well-known default value

- severity: CRITICAL
- file: `.env.example`
- line: 1
- type: default-credential
- description: `NEXTAUTH_SECRET=my-superstrong-secret` is a static placeholder that code uses as the root cryptographic secret for the entire application. Unlike HANKO_API_KEY, which has a runtime guard that throws, every consumer of NEXTAUTH_SECRET treats a missing or default value as valid. A deployment that copies `.env.example` to `.env.production` without changing this value inherits a publicly known secret.
- evidence: `NEXTAUTH_SECRET=my-superstrong-secret`
- usage: This single value branches into five distinct security domains:
  1. **NextAuth JWT signing** — `lib/middleware/app.ts:56` passes it as `secret` to `getToken()`. An attacker with the known secret can forge arbitrary session JWTs (`next-auth.session-token` cookie), impersonating any user.
  2. **SAML Jackson client_secret** — `lib/auth/auth-options.ts:128,149` uses it as both `clientSecret` for the SAML OAuth provider and `client_secret` in the SAML-idp token exchange. Also used as `clientSecretVerifier` in `lib/jackson.ts:72`. An attacker can replay SAML token exchanges.
  3. **Jackson encryption key derivation** — `lib/jackson.ts:16-20` derives a 32-byte AES-256-GCM key via `SHA-256(NEXTAUTH_SECRET)`. All SAML/SCIM data encrypted at rest becomes decryptable.
  4. **HMAC-signed agreement tokens** — `lib/signing/download-token.ts:19` and `lib/signing/access-token.ts:21` use NEXTAUTH_SECRET as HMAC-SHA256 key for signed download and access cookies. Forgeable tokens bypass access-form restrictions for signed documents.
  5. **API token hashing** — `lib/api/auth/token.ts:12` uses `SHA-256(token + NEXTAUTH_SECRET)` for API token storage. With a known secret, stored token hashes become trivially pre-image attackable.
- valid: The `.env.example` value `my-superstrong-secret` is the intended development default. It is active whenever the deployer has not overridden it via `NEXTAUTH_SECRET` in the environment. Production instances on Vercel using Vercal's system env vars are likely safe; self-hosted instances that copy `.env.example` verbatim are vulnerable.

---

### NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY uses well-known default value

- severity: HIGH
- file: `.env.example`
- line: 68
- type: default-credential
- description: `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret` is a static example value for AES-256-CTR document password encryption. Same copy-paste risk as NEXTAUTH_SECRET.
- evidence: `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret`
- usage: `lib/utils.ts:614,628` — the value is hashed with SHA-256 and truncated to 32 bytes to produce an AES-256-CTR key for `encryptEncrpytedPassword()` and `decryptEncrpytedPassword()`. These functions protect document-level passwords stored in the database. If the default is unchanged, any document password encrypted with this key can be decrypted by an attacker who knows the source (anyone who reads the public `.env.example` file).
- valid: Active whenever the env var is not set. This is the same copy-paste risk vector as NEXTAUTH_SECRET.

---

### Trigger.dev project ID hardcoded in config

- severity: LOW
- file: `trigger.config.ts`
- line: 7
- type: hardcoded-secret
- description: `project: "proj_plmsfqvqunboixacjjus"` is a Trigger.dev project identifier embedded directly in the config file. This is a project reference, not a bearer token or API key — Trigger.dev uses `TRIGGER_SECRET_KEY` from the environment for actual authentication.
- evidence: `project: "proj_plmsfqvqunboixacjjus"`
- usage: Used by Trigger.dev SDK (`defineConfig`) to associate background job runs with the correct project. This is metadata, not a credential — it identifies which project runs belong to.
- valid: Active. The project ID is intentionally public-facing within Trigger.dev's model (analogous to an API path prefix, not a secret).

---

## Non-findings investigated

| Pattern | Result |
|---|---|
| `Bearer ` + hardcoded token | All Bearer usages reference `process.env.AUTH_BEARER_TOKEN` or `process.env.*` — no static tokens |
| `sk-` (Stripe/OpenAI keys) | None found in source |
| `pk_` (Stripe publishable keys) | None found in source |
| `BEGIN (RSA|EC) PRIVATE KEY` | None found |
| AWS access keys (`AKIA...`) | None found |
| Embedded credentials in URLs (`https://user:pass@host`) | Only a doc example `http://username:password@proxy.example.com:8080` in `.agents/skills/agent-browser/references/proxy-support.md` |
| HANKO_API_KEY default (`add-your-hanko-api-key`) | Code in `lib/hanko.ts:3-9` throws if unset — mitigation exists, though it only checks truthiness, not whether the value was changed from the placeholder |
| GitHub Actions secrets | `GITHUB_TOKEN` and `PERSONAL_ACCESS_TOKEN` reference `${{ secrets.* }}` — no hardcoded values |
| Client secrets (Google, LinkedIn, Slack) | All consumed from `process.env.*` — no hardcoded defaults |
| Test fixtures / seed files | No seed files or test fixtures with credentials found |
| Prisma migrations | No hardcoded secrets in migration files |
| Vercel Blob URLs in `next.config.mjs` | `yoywvlh29jppecbh.public.blob.vercel-storage.com` and `36so9a8uzykxknsu.public.blob.vercel-storage.com` — these are intentionally public blob storage endpoints (the `.public.` subdomain is explicit) |
| `clientId: "dummy"` in SAML config | Intentional — SAML Jackson uses a decoupled flow where `client_id` is unused |

## Risk summary

The two CRITICAL/HIGH findings share the same root cause: example defaults in `.env.example` that downstream code trusts without verification. These are deployment-time misconfiguration risks, not code-injection risks. The code is correct — the vulnerability is environmental.

The most dangerous path is NEXTAUTH_SECRET at the default, because a single known value unlocks token forgery, session impersonation, encrypted data decryption, and API token compromise — all from a value that lives in a public GitHub file.
