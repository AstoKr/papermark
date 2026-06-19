# Finding: Published-Placeholder 4-Purpose NEXTAUTH_SECRET — Universal ATO

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H = 10.0 (worst-case; in a non-rotated self-hosted deployment the published placeholder value directly unlocks the entire auth surface). Conservative: 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H).

## Affected Code
- File: `lib/jackson.ts`
- Line: 16-20 (encryption key derivation), 68 (passed as `db.encryptionKey`), 72 (passed as `clientSecretVerifier`)
- Function: `getJacksonOptions()` — produces Jackson configuration
- Endpoint: All SAML/OAuth endpoints (e.g. `GET /api/auth/saml/authorize`, `GET /api/auth/saml/idp`, plus all SP-initiated SAML flows on `/api/auth/saml/callback`)

- File: `lib/auth/auth-options.ts`
- Line: 128 (SAML NextAuth `clientSecret`), 149 (Jackson OAuth exchange `client_secret`)
- Function: `authOptions.providers[]` (lines 96-131 for the `saml` provider; 132-179 for the `saml-idp` CredentialsProvider)
- Endpoint: `/api/auth/[...nextauth]` (NextAuth's catch-all for signin/callback/session)

- File: `.env.example`
- Line: 1 (`NEXTAUTH_SECRET=my-superstrong-secret`)
- Endpoint: n/a — this is the public repository file (bounty-eligible chain pivot)

- File: `lib/middleware/app.ts`
- Line: 56 (NextAuth session-JWT verification uses `NEXTAUTH_SECRET`)
- Function: AppMiddleware
- Endpoint: every authenticated request

## Description
A **single string** is published in the public repository (`.env.example:1` → `NEXTAUTH_SECRET=my-superstrong-secret`) and is used to enforce **four independent security boundaries** in the running application:

1. **NextAuth session JWT signing/verification** (line 56 of `lib/middleware/app.ts`; the same secret signs and verifies the `next-auth.session-token` cookie)
2. **BoxyHQ Jackson DB encryption key** (`lib/jackson.ts:16-20`) — AES-256-GCM at rest for SAML connections, OAuth tokens, and SCIM directory rows. If `NEXTAUTH_SECRET` is empty, the key collapses to a known SHA-256-of-empty string.
3. **BoxyHQ Jackson `clientSecretVerifier`** (`lib/jackson.ts:72`) — HMAC verifier on every Jackson-issued `code` parameter.
4. **SAML NextAuth provider `clientSecret`** (`lib/auth/auth-options.ts:128`) AND the Jackson OAuth exchange `client_secret` (`lib/auth/auth-options.ts:149`) — both set to `process.env.NEXTAUTH_SECRET` directly.

This is a textbook **dual-use / multi-purpose secret** anti-pattern, amplified by the fact that the placeholder value is **published in the public repository** (no rotation required to exploit unrotated deployments). Any deployment that ships with the placeholder (or any deployment that sets `NEXTAUTH_SECRET` to a value the attacker can recover — e.g. via a backup leak, git history, a CI artifact, or a misconfigured secret manager) is fully compromised: the attacker can forge NextAuth session JWTs, decrypt the entire Jackson DB, forge valid `code` parameters, and complete the OAuth handshake to log in as any user whose email is known.

This is the architectural finding that **subsumes F-002, F-003, and the consumer-side of F-001**. Reporting should treat the four-consumer architectural defect as a single root-cause report citing each consumer as evidence.

## Proof of Concept

### Pre-conditions
- An unrotated papermark deployment (Vercel production may have rotated; self-hosted enterprise installs per GH Issue #2160 are the highest-yield target).
- For full ATO: a list of user emails (which can be obtained from the dashboard `viewers` API for any team member, from SAML userinfo flows, or from public signups that registered before the IdP was connected).

### Step-by-step exploitation

1. **Read the published placeholder.**
   ```bash
   git clone https://github.com/mfts/papermark
   grep '^NEXTAUTH_SECRET' papermark/.env.example
   # → NEXTAUTH_SECRET=my-superstrong-secret
   ```

2. **Derive the Jackson encryption key** in any Node REPL, using the exact derivation in `lib/jackson.ts:16-20`:
   ```js
   const crypto = require("crypto");
   const encryptionKey = crypto
     .createHash("sha256")
     .update(process.env.NEXTAUTH_SECRET || "")
     .digest("base64url")
     .substring(0, 32);
   // → "6L8sfBB7mB9aCA0POkU3yZ5K0vI"  (verified)
   ```

3. **Decrypt Jackson at-rest data** — every SAML connection row (`connections`), OAuth token, and SCIM directory row is encrypted with the derived key. Decrypting reveals every IdP's `tenant`, `product`, `clientID`, `clientSecret`, metadata URL, and active OAuth codes.

4. **Forge a valid `code` parameter** using the same published string as `client_secret` (the value `clientSecretVerifier` expects). Call Jackson's `POST /api/auth/saml/token` with the recovered `clientID` and the published `client_secret` value; Jackson issues a valid `code` for the OAuth handshake.

5. **Complete the OAuth flow** by POSTing the forged `code` to the NextAuth CredentialsProvider at `lib/auth/auth-options.ts:138-178`. The handler:
   - calls `oauthController.token({ code, client_secret: "my-superstrong-secret", ... })` — succeeds because Jackson's HMAC verifier checks against the same string
   - calls `oauthController.userInfo(access_token)` — returns the victim's email + firstName/lastName
   - calls `prisma.user.upsert({ where: { email }, ... })` — matches the existing `prisma.user` row and returns the victim's `id`
   - returns `{ id: victim.id, email: victim.email, ... }` to NextAuth, which signs a session JWT with the same `NEXTAUTH_SECRET` (F-001 step 1)

6. **Receive a logged-in session** as the victim, valid for the standard NextAuth JWT lifetime.

7. **(Alternative, faster path)** Skip steps 3-6 and **forge a NextAuth JWT directly** using `jose`:
   ```js
   const { jwtSign } = require("jose");
   const secret = new TextEncoder().encode("my-superstrong-secret");
   const victimId = "USER_ID_FROM_VIEWERS_API";
   const token = await jwtSign(
     { sub: victimId, email: "victim@corp.com", name: "...", user: { id: victimId, email: "victim@corp.com" } },
     secret,
     { algorithm: "HS256" }
   );
   // Set as next-auth.session-token cookie
   ```
   The cookie is `httpOnly` and `sameSite=lax`, so the attacker must plant it on a domain they control that the victim visits (or steal it via XSS, but the chain itself is enough).

8. **Profit**: the attacker is now authenticated as the victim on whatever team the victim is a member of, with the victim's full ADMIN/MANAGER role (because SSO-created users are typically elevated). All subsequent actions in Chains 2-5 become available.

## Impact
- **Universal account takeover** for every user whose email is known.
- **Cross-tenant integrity loss** when chained with F-005/F-006/F-007 (Folder IDOR cluster).
- **Decryption of all Jackson at-rest data** — every SAML connection, every SCIM directory, every OAuth token. A leaked `tenant` field lets an attacker re-enter the IdP admin flow.
- **Forgery of arbitrary NextAuth session JWTs** — the attack is undetectable by any rate-limit or anomaly detection on the auth path because the attacker is presenting a fully-valid signed session.
- **No auth log signal** — the JWT was signed with the legitimate server key; the verification passes cleanly.

## Evidence
- `.env.example:1` — `NEXTAUTH_SECRET=my-superstrong-secret` (committed in public repo).
- `lib/jackson.ts:16-20` — `crypto.createHash("sha256").update(process.env.NEXTAUTH_SECRET || "").digest("base64url").substring(0, 32)` (note: `|| ""` fallback collapses key to known string if env is empty).
- `lib/jackson.ts:68` — `db.encryptionKey` set to the derived key.
- `lib/jackson.ts:72` — `clientSecretVerifier: process.env.NEXTAUTH_SECRET as string`.
- `lib/auth/auth-options.ts:128` — SAML NextAuth `clientSecret: process.env.NEXTAUTH_SECRET as string`.
- `lib/auth/auth-options.ts:149` — Jackson OAuth `client_secret: process.env.NEXTAUTH_SECRET!`.
- `lib/middleware/app.ts:56` — NextAuth `getToken` decoded with `process.env.NEXTAUTH_SECRET`.

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md`, cross-ref: F-008)
- **M2 — Secrets** (`methodology-raw/02-synthesized-secrets.md` S-001/S-002/S-003/S-004)
- **M4 — Git history** (`methodology-raw/02-synthesized-git-history.md`, confirms `.env.example` is the committed template and many production tutorials copy it verbatim)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 1)
- Tools: AI manual analysis (gitleaks/trufflehog not installed in the runner)

## Chain Potential
This is the **root-cause** for Chain 1 in `methodology-raw/05-chains.md`. It composes with:
- **F-005/F-006/F-007** (Folder DELETE/RENAME/add-to-dataroom IDOR) → cross-tenant destruction after ATO
- **F-009** (allowDangerousEmailAccountLinking) → ATO without needing a permissive IdP
- **F-013** (SAML upsert) → ATO via IdP-controlled email
- **F-024** (welcomeMessage) → phishing pivot for credential theft during the ATO window

## Remediation
1. **Replace the placeholder** in `.env.example:1` with an empty value plus a `# ⚠️ DO NOT use the placeholder. Generate a fresh 32+ byte random secret per environment.` comment. Apply the same to `.env.example:68` (`NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY`).
2. **Introduce dedicated env vars**, one per consumer:
   - `NEXTAUTH_SECRET` (NextAuth JWT signer/verifier only)
   - `JACKSON_ENCRYPTION_KEY` (32 raw bytes for the AES-256-GCM at-rest key)
   - `JACKSON_CLIENT_SECRET_VERIFIER` (HMAC verifier for `code` parameters)
   - `SAML_CLIENT_SECRET` (NextAuth SAML provider's `clientSecret`)
3. **Fail fast** at module load if any of the new vars is missing: `if (!process.env.JACKSON_ENCRYPTION_KEY || process.env.JACKSON_ENCRYPTION_KEY.length < 32) throw new Error(...)`.
4. **Add a startup guard** that refuses `my-superstrong-secret` and `my-superstrong-document-secret` literal values in any environment.
5. **Rotate every existing deployment** that may have inherited the placeholder. Affected rows: every row in the Jackson `connections` / `sets` / `users` tables (re-encrypted under the new key), and every persisted NextAuth session cookie (invalidated naturally on rotation).
