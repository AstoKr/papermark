# AI Secrets Analysis — `ai-analyze-secrets`

**Workflow:** whitebox-bug-finder (M6)
**Methodology:** M6 — Hardcoded / leaked secrets
**Codebase:** papermark (Next.js 14 App Router, multi-tenant SaaS)
**Date:** 2026-06-19

---

## Scope

Hardcoded credentials, leaked tokens, default credentials, dual-use secrets, and secret-in-URL anti-patterns reviewed.

> **Note:** This report **complements** `methodology-raw/00-ai-configs.md` (configuration analysis). Findings that overlap (e.g. `.env.example` placeholders, `trigger.config.ts` project ID) are referenced but not re-graded here — they are already filed at MEDIUM/LOW in the configs report and graded from a config-hygiene perspective. This report grades them from a *secret-leak* perspective and adds the secret-flow tracing (where the secret is used, not just where it is defined) per the `secrets.md` reference.

> **Codebase hygiene summary:** Papermark's pattern of `process.env.X` everywhere is correct — there are no hardcoded API keys, AWS access keys, OpenAI keys, Stripe keys, Slack tokens, or private keys committed in source. The only literal "secret" string in source is `clientId: "dummy"` in `lib/auth/auth-options.ts` (see Finding 5), which is a placeholder for the BoxyHQ Jackson OAuth controller (the real `client_id` is sent at runtime).

---

## Findings

### Finding: `NEXTAUTH_SECRET` is reused as the Jackson SAML/SCIM DB encryption key with an empty-string fallback

- **Severity:** CRITICAL
- **File:** `lib/jackson.ts`
- **Line:** 16–20 (and downstream use at line 68)
- **Type:** dual-use-secret / empty-fallback-key-derivation
- **Description:** `lib/jackson.ts` derives the AES-256-GCM `encryptionKey` that BoxyHQ Jackson uses to encrypt every SAML connection, SCIM directory, and OAuth token at rest in the Postgres database:
  ```ts
  // Jackson's AES-256-GCM encrypter requires exactly 32 bytes.
  // Derive a stable 32-byte key from NEXTAUTH_SECRET using SHA-256.
  const encryptionKey = crypto
    .createHash("sha256")
    .update(process.env.NEXTAUTH_SECRET || "")
    .digest("base64url")
    .substring(0, 32);
  ```
  The `|| ""` fallback is silent — if `NEXTAUTH_SECRET` is unset or empty, the key is `sha256("")` truncated to 32 bytes — a *known* string. The same `NEXTAUTH_SECRET` is also passed as `clientSecretVerifier: process.env.NEXTAUTH_SECRET` to Jackson (line 72), meaning Jackson also trusts JWTs signed with it.
- **Evidence:**
  ```ts
  function getJacksonOptions(): JacksonOption {
    return {
      externalUrl: process.env.NEXTAUTH_URL || "https://app.papermark.com",
      samlPath: "/api/auth/saml/callback",
      samlAudience,
      db: {
        engine: "sql",
        type: "postgres",
        url: getJacksonDbUrl(),
        encryptionKey,  // ← derived from NEXTAUTH_SECRET, with empty fallback
      },
      idpEnabled: true,
      scimPath: "/api/scim/v2.0",
      clientSecretVerifier: process.env.NEXTAUTH_SECRET as string,  // ← reused
    };
  }
  ```
- **Usage / blast radius:**
  - `encryptionKey` decrypts every SAML connection row, SCIM directory row, and OAuth token at rest in the `samlJackson` / Jackson-owned tables.
  - `clientSecretVerifier` verifies the HMAC of every `code` parameter Jackson issues during SAML/IdP login.
  - An attacker who learns `NEXTAUTH_SECRET` (e.g. via the published placeholder in `.env.example` — see Finding 3) can:
    1. Decrypt every SAML connection's IdP metadata, certificates, and SCIM tokens at rest.
    2. Forge valid `code` parameters for the `/api/auth/saml/token` exchange (`lib/auth/auth-options.ts:144–149`) and the `/api/auth/saml/authorize` callback, logging in as any user whose email they know.
    3. Forge NextAuth session JWTs (Finding 1 of `00-ai-configs.md` mentions `NEXTAUTH_SECRET` is also the session signer via `lib/middleware/app.ts:56`).
  - The same `NEXTAUTH_SECRET` is also passed as `clientSecret` to the SAML NextAuth provider at `lib/auth/auth-options.ts:128, 149` — yet another code path that trusts the value.
- **Valid:** Yes, in any environment where `NEXTAUTH_SECRET` is unset, empty, or matches the `.env.example` placeholder.
- **Remediation:**
  1. **Stop deriving Jackson's encryption key from `NEXTAUTH_SECRET`.** Introduce a dedicated `JACKSON_ENCRYPTION_KEY` env var (32 bytes) and `JACKSON_CLIENT_SECRET_VERIFIER` env var, and require both at startup with a fail-fast check.
  2. **Remove the empty-string fallback.** `if (!process.env.JACKSON_ENCRYPTION_KEY) throw new Error(...)` at module load.
  3. **Decouple `NEXTAUTH_SECRET` from SAML** (`lib/auth/auth-options.ts:128, 149`) — use a dedicated `SAML_CLIENT_SECRET` env var.
  4. **Update `.env.example`** (see Finding 3) so the placeholder value is not a viable secret.
  5. **Rotate all existing Jackson-encrypted data** after introducing the new key: any rows encrypted under the old derived key cannot be read by the new key and will need to be re-encrypted (or re-created) during the rollover.

---

### Finding: `NEXTAUTH_SECRET` is used as the NextAuth SAML `clientSecret` and Jackson OAuth `client_secret`

- **Severity:** HIGH
- **File:** `lib/auth/auth-options.ts`
- **Line:** 128, 149
- **Type:** dual-use-secret
- **Description:** The SAML NextAuth provider and the Jackson `saml-idp` `CredentialsProvider` both pass `process.env.NEXTAUTH_SECRET` as the `clientSecret` / `client_secret`:
  ```ts
  // line 95-131 (SAML NextAuth provider)
  options: {
    clientId: "dummy",
    clientSecret: process.env.NEXTAUTH_SECRET as string,
  },
  ```
  ```ts
  // line 138-178 (saml-idp CredentialsProvider)
  const { access_token } = await oauthController.token({
    code: credentials.code,
    grant_type: "authorization_code",
    redirect_uri: getMainDomainUrl(),
    client_id: "dummy",
    client_secret: process.env.NEXTAUTH_SECRET!,
  });
  ```
  This is the same string that signs NextAuth JWTs and (per Finding 1 above) is the Jackson DB encryption key derivation source. BoxyHQ Jackson's OAuth controller uses the `client_secret` to verify that the `code` returned from the IdP-initiated flow was issued by this application. With the same string doing four things, a single leak compromises all four.
- **Usage / blast radius:**
  - Same as Finding 1, plus:
  - Anyone with the `NEXTAUTH_SECRET` can independently pass the `client_secret` check at the Jackson OAuth `/oauth/token` endpoint, exchanging captured authorization `code` values for `access_token`s without going through the legitimate IdP.
- **Valid:** Yes — combined with the `.env.example` placeholder, this is exploitable in any environment that has not generated a fresh `NEXTAUTH_SECRET`.
- **Remediation:** Generate a dedicated `SAML_CLIENT_SECRET` env var (random 32+ bytes) and use it for the SAML provider and the Jackson OAuth `client_secret`. The `clientId: "dummy"` literal is the BoxyHQ convention and can remain (it's not a secret).

---

### Finding: `REVALIDATE_TOKEN` is passed as a URL query parameter (`?secret=...`) instead of an HTTP header

- **Severity:** MEDIUM
- **Files:** `lib/api/links/revalidate.ts`, `pages/api/revalidate.ts`, `lib/trigger/pdf-to-image-route.ts:343`, `lib/trigger/convert-pdf-direct.ts:333`, `lib/api/documents/process-document.ts:296`, `pages/api/links/index.ts:491`, `pages/api/links/[id]/index.ts:743`, `pages/api/links/[id]/archive.ts:102`, `pages/api/teams/[teamId]/enable-advanced-mode.ts:126`
- **Line:** see file:line list above
- **Type:** secret-in-URL (leak vector)
- **Description:** The ISR revalidation secret `REVALIDATE_TOKEN` is passed in the URL as `?secret=...`:
  ```ts
  // lib/api/links/revalidate.ts:19-21
  await fetch(
    `${revalidateUrl}/api/revalidate?secret=${revalidateToken}&linkId=${link.id}&hasDomain=${link.domainId ? "true" : "false"}`,
  );
  ```
  ```ts
  // pages/api/revalidate.ts:14
  if (req.query.secret !== process.env.REVALIDATE_TOKEN) {
    return res.status(401).json({ message: "Invalid token" });
  }
  ```
  While `REVALIDATE_TOKEN` is read from `process.env` (correct), sending secrets as URL query parameters has well-known leak vectors:
  1. **Server access logs** (Vercel function logs by default log the full URL including query string).
  2. **Referer headers** sent to any third-party resource loaded by `/api/revalidate`'s response.
  3. **Proxy / CDN logs** in any edge layer that captures URLs.
  4. **Browser history** in any client that ever calls this endpoint via the browser.
  5. **Bug reports / screenshots** that include the URL bar.
  6. **Stack traces** in error reports (Vercel surfaces `req.url` in 5xx).
- **Usage / blast radius:**
  - `/api/revalidate` is an internal endpoint that triggers `res.revalidate()` for a path. A captured `REVALIDATE_TOKEN` lets an attacker invalidate any public ISR cache entry (mostly a DoS / cache-stampede amplifier) — *not* a high-severity key by itself, but it indicates the team is willing to put secrets in URLs.
  - The pattern matters because the same code shape is used elsewhere; if a more sensitive token (e.g. `INTERNAL_API_KEY`) ever gets put in a query string in a future change, the same leak vectors apply.
- **Valid:** Yes — `REVALIDATE_TOKEN` is a working token; the URL pattern is active in production.
- **Remediation:** Move the token to an HTTP header:
  ```ts
  await fetch(`${revalidateUrl}/api/revalidate?linkId=...&hasDomain=...`, {
    headers: { "x-revalidate-secret": revalidateToken },
  });
  ```
  And accept the header on the handler side (fall back to header, drop the query-string path entirely). The same change should be applied to any other internal endpoints that take a `?secret=...` query param.

---

### Finding: Placeholder values in `.env.example` are realistic enough to ship to a real environment

- **Severity:** MEDIUM
- **File:** `.env.example`
- **Line:** 1 (`NEXTAUTH_SECRET=my-superstrong-secret`), 38 (`HANKO_API_KEY=add-your-hanko-api-key`), 39 (`NEXT_PUBLIC_HANKO_TENANT_ID=add-your-hanko-tenent-id`), 68 (`NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret`)
- **Type:** default-credential / placeholder-leak
- **Description:** Two sensitive placeholders look like real secrets rather than obvious placeholders. The Hanko placeholders are better ("add-your-hanko-api-key") but also risk being copy-pasted verbatim into a `.env.local`. The `my-superstrong-secret` pattern is the highest risk because:
  - It satisfies any "is the value set?" check (truthy non-empty).
  - It is the published value in a public GitHub repository.
  - The README's onboarding flow (`cp .env.example .env.local`) makes it the default boot value.
  - The `my-superstrong-document-secret` is published in this same file and is the AES-256-CTR key derivation source for every link/document password (`lib/utils.ts:626–630`).
  - The `NEXTAUTH_SECRET=my-superstrong-secret` is the published value for the multi-purpose secret in Finding 1 and Finding 2.
- **Evidence:**
  ```bash
  # .env.example
  NEXTAUTH_SECRET=my-superstrong-secret
  HANKO_API_KEY=add-your-hanko-api-key
  NEXT_PUBLIC_HANKO_TENANT_ID=add-your-hanko-tenent-id
  # ...
  NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=my-superstrong-document-secret
  ```
- **Usage / blast radius:**
  - `NEXTAUTH_SECRET` is the NextAuth session JWT signer (`lib/middleware/app.ts:56`), the SAML `clientSecret` (Finding 2), the Jackson DB encryption key source (Finding 1), and the Jackson `clientSecretVerifier` (Finding 1). All four are compromised.
  - `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` is hashed via `sha256(...).digest("base64").substring(0, 32)` in `lib/utils.ts:626–630` to produce the AES-256-CTR key that encrypts every link's document password. With the published placeholder, anyone who obtains a ciphertext from the database can decrypt it.
  - `HANKO_API_KEY` is sent as the `apiKey` header to `https://passkeys.hanko.io/{tenantId}/credentials[...]` in `lib/api/auth/passkey.ts:65–79` and `:121–141`. If a developer copy-pastes the placeholder and the request is made, Hanko will reject it (good), but the pattern remains: a "looks-valid" placeholder invites accidental reuse.
- **Valid:** Yes — `.env.example` is checked into the public repo, so the values are known to anyone with read access.
- **Remediation:**
  1. **Replace with empty values or obvious placeholders:**
     ```bash
     NEXTAUTH_SECRET=
     NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY=
     HANKO_API_KEY=
     NEXT_PUBLIC_HANKO_TENANT_ID=
     ```
  2. **Add a header comment** to `.env.example`:
     ```bash
     # ⚠️ DO NOT use the placeholder values. Generate a fresh secret for every
     # environment, e.g. `openssl rand -base64 32`.
     ```
  3. **Add a fail-fast runtime guard** in the entry points (`middleware.ts` and `lib/jackson.ts`) that refuses to start if `NEXTAUTH_SECRET` matches any of:
     - `my-superstrong-secret`
     - empty / missing
     - known list of placeholder values
  4. **Decouple secrets** (Finding 1 / Finding 2) so the next time a placeholder leaks, the blast radius is one key, not four.

---

### Finding: `INTERNAL_API_KEY` is sent in the `Authorization: Bearer ...` header to internal endpoints with no constant-time comparison

- **Severity:** LOW
- **Files:** `lib/trigger/pdf-to-image-route.ts:130, 222`, `lib/trigger/dataroom-change-notification.ts:269`, `lib/trigger/dataroom-upload-notification.ts:106`, `lib/files/get-file.ts:99`, `lib/api/notification-helper.ts:19, 51`, `pages/api/file/s3/get-presigned-get-url-proxy.ts:67`
- **Type:** secret-rotation-not-enforced / bearer-token-usage
- **Description:** `INTERNAL_API_KEY` is the bearer token used by Trigger.dev background jobs to call internal Next.js endpoints (e.g. `INTERNAL_API_KEY=...` → `Authorization: Bearer ...`). All current call sites and consumers read it from `process.env` (correct, no hardcoded value). However:
  - The token is shared across **all** internal call sites (PDF image conversion, dataroom notifications, presigned URL generation) — there is no per-job scoping. Any one consumer's token leak gives the holder the full set of internal capabilities.
  - There is no rotation mechanism (no `INTERNAL_API_KEY_PREVIOUS` for overlap, no expiry, no per-job sub-keys).
  - The token is treated as a static long-lived secret. In Trigger.dev, the `clientId` is also embedded in `trigger.config.ts:7` (`proj_plmsfqvqunboixacjjus`) — the project ID alone is not sensitive, but combined with `TRIGGER_SECRET_KEY` (in env) it is the credential.
- **Evidence:**
  ```ts
  // lib/trigger/pdf-to-image-route.ts:128-131
  await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/mupdf/get-pages`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.INTERNAL_API_KEY}`,
      // ...
    },
  });
  ```
  ```ts
  // pages/api/file/s3/get-presigned-get-url.ts:31
  if (!process.env.INTERNAL_API_KEY) {
    return res.status(503).json({ error: "Internal API key not configured" });
  }
  ```
- **Usage / blast radius:** Captured `INTERNAL_API_KEY` lets the holder:
  - Call `/api/mupdf/...` to run server-side PDF rendering (DoS / cost amplifier).
  - Call `/api/jobs/send-dataroom-new-document-notification` and `/api/jobs/send-dataroom-upload-notification` to send arbitrary dataroom emails to arbitrary recipients (phishing pivot).
  - Get presigned S3 URLs for arbitrary team storage buckets (`get-presigned-get-url-proxy.ts`).
- **Valid:** Yes — the token is in use in production. No leak observed in the repo, but the design (single shared key) means a leak in any of the 7+ call sites compromises the rest.
- **Remediation:**
  1. **Per-job scoped tokens** — `INTERNAL_API_KEY_PDF`, `INTERNAL_API_KEY_NOTIFICATIONS`, `INTERNAL_API_KEY_PRESIGN`. A leak in one job doesn't compromise the others.
  2. **Rotate the key** periodically and document the rotation runbook in `SECURITY.md`.
  3. **Add audit logging** on the consumer side so that each `Bearer` use is logged with the calling job ID.

---

### Finding: `NEXTAUTH_SECRET` placeholder is silently used in `.env.example` while also being a multi-purpose secret

- **Severity:** MEDIUM
- **File:** `.env.example` line 1
- **Type:** dual-purpose-secret-publication
- **Description:** (See Finding 1, Finding 2, Finding 3.) `NEXTAUTH_SECRET` carries **four** independent responsibilities:
  1. NextAuth session JWT signing key (`lib/middleware/app.ts:56`).
  2. Jackson DB encryption key derivation source (`lib/jackson.ts:16–20`).
  3. Jackson OAuth `clientSecretVerifier` (`lib/jackson.ts:72`).
  4. SAML NextAuth provider `clientSecret` and Jackson OAuth `client_secret` (`lib/auth/auth-options.ts:128, 149`).
  Plus it is published as `my-superstrong-secret` in the public `.env.example`. The combination is a "keys to the kingdom" secret — a single published string unlocks session forgery, SAML login forgery, DB decryption, and SCIM token compromise.
- **Usage / blast radius:** Trivially chained: read the value from this public repo → forge NextAuth session → log in as any user → access any team → access any dataroom → decrypt any link password → read any document.
- **Valid:** Yes, in any environment that has not rotated the placeholder.
- **Remediation:** This is the same as the combined remediation of Findings 1, 2, and 3. The architectural fix is "one secret, one purpose" — split `NEXTAUTH_SECRET` into separate keys for each consumer.

---

### Finding: `trigger.config.ts` ships the Trigger.dev project ID in source

- **Severity:** LOW
- **File:** `trigger.config.ts`
- **Line:** 7
- **Type:** info-disclosure (project identifier)
- **Description:** The Trigger.dev project ID `proj_plmsfqvqunboixacjjus` is hard-coded in source. The Trigger.dev docs explicitly recommend committing this — it is a tenant identifier, not a credential. The matching credential is `TRIGGER_SECRET_KEY` (only in env).
- **Evidence:**
  ```ts
  export default defineConfig({
    project: "proj_plmsfqvqunboixacjjus",
    // ...
  });
  ```
- **Usage / blast radius:** Negligible by itself. Combined with a leaked `TRIGGER_SECRET_KEY`, the project ID is the rest of the credential. Worth noting for log-scrubbing hygiene.
- **Valid:** Yes — this is the published production project ID. Cannot be rotated without a Trigger.dev migration.
- **Remediation:** None required (this is by design). Add a note in `SECURITY.md` that `TRIGGER_SECRET_KEY` is the sensitive value and must never be committed.

---

## Summary Table

| # | Severity | Title | File | Line |
|---|----------|-------|------|------|
| 1 | CRITICAL | NEXTAUTH_SECRET reused as Jackson DB encryption key with empty fallback | `lib/jackson.ts` | 16–20, 68 |
| 2 | HIGH | NEXTAUTH_SECRET used as SAML clientSecret (4-way dual-use) | `lib/auth/auth-options.ts` | 128, 149 |
| 3 | MEDIUM | Placeholder values in `.env.example` are realistic enough to ship | `.env.example` | 1, 38, 39, 68 |
| 4 | MEDIUM | REVALIDATE_TOKEN in URL query string | `lib/api/links/revalidate.ts` + 8 more | (see file list) |
| 5 | MEDIUM | NEXTAUTH_SECRET is a multi-purpose key published in the repo | `.env.example`, `lib/jackson.ts`, `lib/auth/auth-options.ts` | (cross-file) |
| 6 | LOW | INTERNAL_API_KEY is a single shared bearer token across 7+ call sites | various | various |
| 7 | LOW | Trigger.dev project ID committed in source | `trigger.config.ts` | 7 |

---

## Cross-References

- **`methodology-raw/00-ai-configs.md`** Finding 9 (MEDIUM, Default secret values in `.env.example`) — same root cause as this report's Finding 3 / Finding 5. Filed there as a config-hygiene issue; here as a secret-leak / placeholder-reuse issue. Combined remediation: replace placeholders + split NEXTAUTH_SECRET.
- **`methodology-raw/00-ai-configs.md`** Finding 15 (LOW, `trigger.config.ts` project ID) — duplicate of this report's Finding 7. Both graded LOW; same finding.

---

## Recommended Remediation Order

1. **Fix Finding 1 (CRITICAL dual-use + empty-fallback)** — single highest-impact issue. A leaked `NEXTAUTH_SECRET` decrypts the Jackson DB and forgers SAML tokens. Split into dedicated env vars and remove the `|| ""` fallback. Estimated impact: closes the largest single-secret compromise in the codebase.
2. **Fix Finding 2 (HIGH SAML clientSecret)** — same fix as #1, since the SAML `clientSecret` and the Jackson `client_secret` are the same source value. Once `NEXTAUTH_SECRET` is decoupled, this falls out.
3. **Fix Finding 3 / Finding 5 (MEDIUM placeholders + multi-purpose secret)** — clean up `.env.example` and add a startup guard that refuses `my-superstrong-secret`. Defense-in-depth.
4. **Fix Finding 4 (MEDIUM secret-in-URL)** — move `REVALIDATE_TOKEN` to an HTTP header. Low risk in isolation, but the anti-pattern is worth eradicating.
5. **Fix Finding 6 (LOW shared bearer token)** — split `INTERNAL_API_KEY` into per-job tokens. Optional but recommended.
6. **Finding 7 (LOW)** — no action; mention only.
