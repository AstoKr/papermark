# AI Test Coverage Analysis

**Workflow:** whitebox-bug-finder (M10)
**Date:** 2026-06-19
**Branch:** archon/task-whitebox-papermark

---

## Executive Summary

**The papermark codebase has ZERO test files and ZERO test infrastructure.**

This is not a coverage gap that can be remedied with a few more tests. There is no test runner configured (no Jest, Vitest, Playwright, Cypress, or Mocha), no test scripts in `package.json`, no test configuration files, and no test files of any kind (`.test.*`, `.spec.*`, `__tests__/`, `test_*.py`, etc.) anywhere in the source tree outside `node_modules`.

Because of this, **every** security-critical function in the codebase is effectively untested. The "High-Priority Untested Code" section below does not single out a few weak spots — it catalogs the full surface area where bugs can hide without any automated safety net.

This document is a **prioritization signal** for the rest of the whitebox bug-finder workflow. Every bug-hunting node should treat papermark security-critical code as unverified and weight manual review accordingly.

---

## Methodology

1. Read `.gitnexus/wiki/overview.md` for architectural context.
2. Read `.gitnexus/wiki/lib-auth.md` for the security-critical auth module context.
3. Searched for test files using glob patterns: `**/*.test.{ts,tsx}`, `**/*.spec.{ts,tsx}`, `**/test_*`, `**/__tests__/**`, `**/cypress/**`.
4. Searched with bash `find` for any test file extensions under the project root (excluding `node_modules` and `.next`).
5. Searched `package.json` for test framework references (jest, vitest, mocha, playwright, cypress, test scripts).
6. Searched for top-level test config files (`jest.config.*`, `vitest.config.*`, `playwright.config.*`, `.mocharc*`).
7. Catalogued security-critical directories: `lib/auth/`, `lib/signing/`, `lib/webhook/`, `lib/middleware/`, `lib/billing/`, `lib/cron/`, `lib/incoming-webhooks/`, `lib/integrations/slack/`, `lib/utils/ip.ts`, `lib/utils/geo.ts`, `ee/features/security/`, `ee/stripe/`, plus security-sensitive API routes under `pages/api/auth`, `pages/api/file/`, `pages/api/stripe/`, `pages/api/agreements/signing/`, `pages/api/passkeys/`, `pages/api/teams/`, `app/api/auth/`, `app/api/verify/`, `app/api/webhooks/`, `app/api/views*`.

---

## Findings

### F1 — Zero test files in entire repository

**Evidence:**

- `find . -type f \( -name "*.test.*" -o -name "*.spec.*" -o -name "*_test.*" -o -name "test_*" \) ! -path "*/node_modules/*" ! -path "*/.next/*"` returns 0 results.
- No `__tests__/`, `tests/`, `test/`, `spec/`, or `cypress/` directories exist anywhere in the source tree.
- `package.json` has no `test` script and no test-framework dependency (no jest, vitest, @testing-library, playwright, cypress).
- No `jest.config.*`, `vitest.config.*`, `playwright.config.*`, or `.mocharc*` at the repo root.
- Glob searches for `**/*.test.ts`, `**/*.test.tsx`, `**/*.spec.ts`, `**/*.spec.tsx`, `**/test_*`, `**/__tests__/**`, `**/cypress/**` all return "No files found".

**Implication:** The "Test Coverage Map" below maps 100% of the security-critical surface to "(none)" with a coverage of "ZERO". This is not a per-file issue — it is a project-wide posture.

**Risk:** Any regression in session, token, signature, webhook, or auth code reaches production with no automated guard. Bugs discovered in security code must rely on code review and runtime incidents, not tests.

---

### Test Coverage Map

The "Test File" column is "(none)" for every entry because no test files exist in the repo.

#### `lib/auth/` — Authentication & Session Management

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/auth/auth-options.ts` | (none) | ZERO | YES — NextAuth providers, OAuth account linking, SAML, passkeys |
| `lib/auth/link-session.ts` | (none) | ZERO | YES — link sharing session lifecycle, fingerprint validation, rate limit, access cap |
| `lib/auth/dataroom-auth.ts` | (none) | ZERO | YES — dataroom session creation/verification, fingerprint hashing, IP normalization |
| `lib/auth/preview-auth.ts` | (none) | ZERO | YES — preview session tokens, ownership binding |

#### `lib/signing/` — Agreement Signing & Download Tokens

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/signing/access-token.ts` | (none) | ZERO | YES — HMAC-signed 90-day access token, `timingSafeEqual` signature comparison |
| `lib/signing/download-token.ts` | (none) | ZERO | YES — download authorization token |
| `lib/signing/download.ts` | (none) | ZERO | YES — gated document download |
| `lib/signing/agreements.ts` | (none) | ZERO | YES — agreement lifecycle |
| `lib/signing/envelopes.ts` | (none) | ZERO | YES — signing envelope state |
| `lib/signing/mirror.ts` | (none) | ZERO | YES — external signing mirror |
| `lib/signing/agreement-storage.ts` | (none) | ZERO | YES — signed-agreement persistence |
| `lib/signing/template-upload.ts` | (none) | ZERO | YES — template upload path |
| `lib/signing/client.ts` | (none) | ZERO | YES — external signing client (Documenso) |

#### `lib/webhook/` — Outbound Webhooks

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/webhook/signature.ts` | (none) | ZERO | YES — HMAC-SHA256 webhook signature generation |
| `lib/webhook/send-webhooks.ts` | (none) | ZERO | YES — outbound webhook delivery (signature consumer) |
| `lib/webhook/transform.ts` | (none) | ZERO | YES — payload shaping |
| `lib/webhook/triggers/link-created.ts` | (none) | ZERO | PARTIAL — payload content |
| `lib/webhook/triggers/document-created.ts` | (none) | ZERO | PARTIAL — payload content |
| `lib/webhook/types.ts`, `constants.ts` | (none) | ZERO | NO — type/constant files |

#### `lib/middleware/` — Request Middleware

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/middleware/app.ts` | (none) | ZERO | YES — app-level request gating |
| `lib/middleware/domain.ts` | (none) | ZERO | YES — custom-domain routing |
| `lib/middleware/incoming-webhooks.ts` | (none) | ZERO | YES — incoming webhook auth |
| `lib/middleware/posthog.ts` | (none) | ZERO | NO — analytics |

#### `lib/incoming-webhooks/` — Incoming Webhook Identity

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/incoming-webhooks/index.ts` | (none) | ZERO | YES — webhook ID generation/extraction, team-id encoding (enumeration risk) |

#### `lib/cron/` — Cron Authentication

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/cron/verify-qstash.ts` | (none) | ZERO | YES — QStash signature verification for all cron jobs |
| `lib/cron/index.ts` | (none) | ZERO | PARTIAL — QStash client setup |

#### `lib/billing/` — Billing/Plan Display

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/billing/team-plan-custom-messaging.ts` | (none) | ZERO | PARTIAL — plan messaging only |

#### `ee/features/security/` — Security Features

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `ee/features/security/index.ts` | (none) | ZERO | YES — security feature entry |
| `ee/features/security/lib/fraud-prevention.ts` | (none) | ZERO | YES — fraud prevention logic |
| `ee/features/security/lib/ratelimit.ts` | (none) | ZERO | YES — rate limit enforcement |
| `ee/features/security/sso/index.ts` | (none) | ZERO | YES — SSO flow |
| `ee/features/security/sso/product.ts` | (none) | ZERO | YES — SSO product mapping |
| `ee/features/security/sso/constants.ts` | (none) | ZERO | PARTIAL — SSO constants |

#### `ee/stripe/` — Stripe Billing & Webhooks

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `ee/stripe/index.ts` | (none) | ZERO | YES — Stripe instance factory |
| `ee/stripe/client.ts` | (none) | ZERO | YES — Stripe client setup |
| `ee/stripe/utils.ts` | (none) | ZERO | YES — billing utilities |
| `ee/stripe/webhooks/checkout-session-completed.ts` | (none) | ZERO | YES — post-payment state mutation |
| `ee/stripe/webhooks/customer-subscription-updated.ts` | (none) | ZERO | YES — plan-change state mutation |
| `ee/stripe/webhooks/customer-subscription-deleted.ts` | (none) | ZERO | YES — cancellation state mutation |
| `ee/stripe/webhooks/invoice-upcoming.ts` | (none) | ZERO | YES — billing notification |
| `ee/stripe/functions/*` | (none) | ZERO | PARTIAL — plan lookup helpers |
| `ee/stripe/constants.ts` | (none) | ZERO | NO — constants |

#### `lib/utils/` — Network Identity Helpers

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/utils/ip.ts` | (none) | ZERO | YES — IP extraction (dataroom session validation depends on this) |
| `lib/utils/geo.ts` | (none) | ZERO | PARTIAL — local constant |

#### `lib/integrations/slack/` — Slack OAuth & Webhook Auth

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `lib/integrations/slack/install.ts` | (none) | ZERO | YES — Slack OAuth install |
| `lib/integrations/slack/events.ts` | (none) | ZERO | YES — Slack event verification |
| `lib/integrations/slack/uninstall.ts` | (none) | ZERO | YES — Slack uninstall cleanup |
| `lib/integrations/slack/client.ts` | (none) | ZERO | PARTIAL — Slack API client |
| `lib/integrations/slack/env.ts` | (none) | ZERO | PARTIAL — Slack env config |
| `lib/integrations/slack/utils.ts` | (none) | ZERO | PARTIAL |
| `lib/integrations/slack/templates.ts` | (none) | ZERO | PARTIAL |
| `lib/integrations/slack/types.ts` | (none) | ZERO | NO — types |
| `lib/integrations/install.ts` | (none) | ZERO | PARTIAL |

#### Security-Sensitive API Routes (`pages/api` & `app/api`)

| Source File | Test File | Coverage | Security-Critical? |
|-------------|-----------|----------|-------------------|
| `pages/api/auth/[...nextauth].ts` | (none) | ZERO | YES — NextAuth entrypoint |
| `pages/api/file/s3/get-presigned-post-url.ts` | (none) | ZERO | YES — S3 upload URL minting (auth gate, slugify, content-disposition) |
| `pages/api/file/s3/get-presigned-get-url.ts` | (none) | ZERO | YES — S3 download URL minting |
| `pages/api/file/s3/get-presigned-get-url-proxy.ts` | (none) | ZERO | YES — proxied S3 download |
| `pages/api/file/s3/multipart.ts` | (none) | ZERO | YES — multipart upload auth |
| `pages/api/file/tus/[[...file]].ts` | (none) | ZERO | YES — tus upload server |
| `pages/api/file/tus-viewer/[[...file]].ts` | (none) | ZERO | YES — viewer tus endpoint |
| `pages/api/file/browser-upload.ts` | (none) | ZERO | YES — browser-side upload proxy |
| `pages/api/file/image-upload.ts` | (none) | ZERO | YES — image upload auth |
| `pages/api/file/notion/index.ts` | (none) | ZERO | PARTIAL — Notion import |
| `pages/api/stripe/webhook.ts` | (none) | ZERO | YES — Stripe webhook entry (raw-body buffering, signature verify, event dispatch) |
| `pages/api/stripe/webhook-old.ts` | (none) | ZERO | YES — legacy Stripe webhook |
| `pages/api/agreements/signing/download.ts` | (none) | ZERO | YES — signed-agreement download |
| `pages/api/agreements/signing/session.ts` | (none) | ZERO | YES — signing session |
| `pages/api/agreements/signing/status.ts` | (none) | ZERO | YES — signing status |
| `pages/api/agreements/signing/complete.ts` | (none) | ZERO | YES — signing completion (state mutation) |
| `pages/api/passkeys/register.ts` | (none) | ZERO | YES — WebAuthn passkey registration |
| `pages/api/teams/*` | (none) | ZERO | YES — team management (membership, invitations, plan changes) |
| `pages/api/links/index.ts` | (none) | ZERO | YES — link CRUD |
| `pages/api/links/[id]/*` | (none) | ZERO | YES — link lifecycle, annotations, archive, preview, duplicate |
| `pages/api/links/download/*` | (none) | ZERO | YES — download job endpoints |
| `pages/api/links/domains/[...domainSlug].ts` | (none) | ZERO | YES — domain-based link access |
| `pages/api/account/index.ts` | (none) | ZERO | YES — account settings |
| `pages/api/account/passkeys.ts` | (none) | ZERO | YES — account passkey list |
| `app/api/auth/verify-code/route.ts` | (none) | ZERO | YES — auth code verification |
| `app/api/verify/login-link/route.ts` | (none) | ZERO | YES — login link verification |
| `app/api/views/route.ts` | (none) | ZERO | YES — view tracking (link/preview session) |
| `app/api/views-dataroom/route.ts` | (none) | ZERO | YES — dataroom view tracking |
| `app/api/views/pages/route.ts` | (none) | ZERO | PARTIAL — page view tracking |
| `app/api/webhooks/callback/route.ts` | (none) | ZERO | YES — webhook callback |
| `app/api/webhooks/signing/route.ts` | (none) | ZERO | YES — signing webhook |
| `app/api/cron/*` | (none) | ZERO | YES — cron endpoints (rely on QStash signature verify) |
| `app/api/integrations/slack/oauth/{authorize,callback}/route.ts` | (none) | ZERO | YES — Slack OAuth |

---

### High-Priority Untested Code (Security-Critical)

All entries below have **NONE** test coverage by definition (no tests exist anywhere). The "Risk" column describes what bugs can hide in each function.

#### A. Session creation & verification

- **File:** `lib/auth/link-session.ts`
  - **Functions:**
    - `createLinkSession` — mints 48-byte URL-safe session token, stores session in Redis with TTL.
    - `verifyLinkSessionToken` — checks expiration, fingerprint OR legacy UA, linkId match, increments access count, enforces `maxAccesses`, applies 100-rpm rate limit.
    - `verifyLinkSession` / `verifyLinkSessionInPagesRouter` — public entry points (App Router vs Pages Router).
    - `revokeLinkSession` — deletes Redis session + viewer set membership.
  - **Why critical:** This is the entire access-control boundary for shared document/dataroom links. Bugs here expose every shared link to hijacking, allow limit-bypass, or break revocation.
  - **Test coverage:** NONE
  - **Risk:** Fingerprint/UA mismatch (e.g. off-by-one in case-normalization) could lock out legitimate users OR allow cookie-reuse from another browser. The access-count math (`accessCount += 1` then `> maxAccesses` check) is subtle — does the creating request count? Do burst reads between rate-limit resets let an attacker exceed `maxAccesses`?

- **File:** `lib/auth/dataroom-auth.ts`
  - **Functions:**
    - `generateSessionFingerprint` — SHA-256 over `userAgent | acceptLanguage | secChUa | secChUaPlatform | secChUaMobile`. Pipe-joiner; ordering matters.
    - `normalizeIp` — maps `::1`, `::ffff:127.0.0.1`, `::ffff:<ipv4>` to normalized form.
    - `createDataroomSession` — 32-byte hex token, normalizes IP at creation only.
    - `verifyDataroomSession` — fingerprint validation for new sessions, IP fallback for legacy.
    - `verifyDataroomSessionInPagesRouter` / `getDataroomSessionByLinkIdInPagesRouter` — Pages Router variants.
    - `updateDataroomSessionVerified` — flips `verified` flag after OTP (raw input typed as `string | typeof raw`).
  - **Why critical:** Dataroom sessions gate access to sensitive document rooms and OTP-protected downloads.
  - **Test coverage:** NONE
  - **Risk:** Fingerprint joiner choice (`|`) is fine only if no field can contain `|` — Accept-Language can. `updateDataroomSessionVerified` accepts an arbitrary session token and writes back without verifying the caller owns it (line 263-293) — a caller-supplied token can be flipped to `verified: true` if the token exists, because there is no ownership check.

- **File:** `lib/auth/preview-auth.ts`
  - **Functions:** `createPreviewSession`, `verifyPreviewSession`.
  - **Why critical:** Preview sessions allow document owners to view unpublished links.
  - **Test coverage:** NONE
  - **Risk:** Does not validate `userId` is the link's owner — only that the session's userId matches the caller's userId. If an attacker can mint a preview session for any link/user pair (e.g. via the minting endpoint), they can preview. No fingerprint binding (by design) means a stolen preview token is enough.

- **File:** `lib/auth/auth-options.ts`
  - **Functions:** NextAuth `authOptions` configuration including Google/LinkedIn/Email/Passkey/SAML providers, account-linking, `createUser` event.
  - **Why critical:** Every authenticated user identity flows through this. `allowDangerousEmailAccountLinking: true` on Google is intentional but is a foot-gun if misconfigured.
  - **Test coverage:** NONE
  - **Risk:** Account-linking collision logic, JWT callback shape, SAML IdP mapping, passkey flow all run untested. The `createUser` hook fires analytics, signed-up event, and QStash welcome — any thrown error in there could break signup silently.

#### B. Token signing & verification

- **File:** `lib/signing/access-token.ts`
  - **Functions:**
    - `mintSignedAgreementAccessToken` — HMAC-SHA256 over `base64url(JSON)` payload, joined with `.`, 90-day TTL.
    - `parseSignedAgreementAccessToken` — splits, validates length=2, recomputes HMAC, `crypto.timingSafeEqual` comparison.
    - `verifySignedAgreementAccessToken` — additional checks for `linkId`, `agreementId`, optional `agreementResponseId`.
    - `buildSignedAgreementAccessCookie` — assembles cookie with `Path=/`, `HttpOnly`, `SameSite=Lax`, optional `Secure`.
  - **Why critical:** These tokens bind a browser to a specific `(linkId, agreementId, agreementResponseId)` tuple for 90 days. They survive page refreshes and replace email lookups (which would enable enumeration, per the inline comment).
  - **Test coverage:** NONE
  - **Risk:** The `getTokenSecret()` throws if `NEXTAUTH_SECRET` is missing — but at request time this is fatal, not configurable. Any change to payload shape breaks existing tokens with no test safety net. `timingSafeEqual` requires equal-length buffers; the length check (line 93-95) is correct but the comparison itself is the linchpin. Bypass risk: cookie is scoped to `Path=/` per the comment, meaning it reaches views/status/download — that scope is wide.

- **File:** `lib/signing/download-token.ts` (exists, contents not read here)
  - **Why critical:** Download authorization.
  - **Test coverage:** NONE

- **File:** `lib/webhook/signature.ts`
  - **Function:** `createWebhookSignature` — HMAC-SHA256 over `JSON.stringify(body)` (Web Crypto API).
  - **Why critical:** Outbound webhook authenticity is verified by receivers using this signature.
  - **Test coverage:** NONE
  - **Risk:** `JSON.stringify(body)` is non-canonical — key order depends on object shape. If a payload object ever has keys in a different order across calls, the signature changes. Receivers that re-serialize the body will compute a different signature. No `verifyWebhookSignature` companion exists in this file — verification logic lives in receivers.

#### C. Webhook signature verification

- **File:** `pages/api/stripe/webhook.ts`
  - **Function:** `webhookHandler` — buffers raw body, calls `stripe.webhooks.constructEvent`, switches on event type, dispatches to checkout/subscription/invoice handlers.
  - **Why critical:** Stripe billing state mutations depend on this. A signature bypass lets an attacker mutate subscription state.
  - **Test coverage:** NONE
  - **Risk:** The relevant-events allowlist is correctly enforced (line 58). But the body-buffering + raw-body requirement (`bodyParser: false`, `supportsResponseStreaming: true`) is fragile — any future change to Next.js config can silently break it. Event handler errors are caught but the response code semantics are unclear (line 80+).

- **File:** `lib/cron/verify-qstash.ts`
  - **Function:** `verifyQstashSignature` — verifies `Upstash-Signature` header using QStash receiver.
  - **Why critical:** All cron endpoints (`app/api/cron/*`) and QStash-triggered jobs rely on this. **Development bypass:** returns early if `process.env.VERCEL !== "1"` (line 12) — meaning signature verification is disabled in non-Vercel environments. If a non-Vercel production deploy exists, this is a hole.
  - **Test coverage:** NONE
  - **Risk:** The VERCEL-bypass must never reach production. Any non-Vercel prod (self-hosted, Docker) would accept unsigned cron requests.

#### D. Middleware

- **File:** `middleware.ts` + `lib/middleware/incoming-webhooks.ts`
  - **Functions:** Edge middleware — host-based routing, path allowlists, incoming webhook auth.
  - **Why critical:** This is the first line of defense for every non-API request. Bypass means unauthenticated access to authenticated pages.
  - **Test coverage:** NONE
  - **Risk:** The matcher regex (line 49) excludes `mcp/?$` — the end-anchor is intentional per the comment, but the `/mcp` substring check is subtle. Custom-domain detection (`isCustomDomain`) has multiple host-suffix checks that could be gamed by registering a subdomain like `papermark.com.attacker.com` (does not currently match — host must include `papermark.io` or `papermark.com` — but any future whitelist addition needs care).

#### E. Upload & download authorization

- **File:** `pages/api/file/s3/get-presigned-post-url.ts`
  - **Function:** `handler` — `getServerSession`, team-membership check via Prisma, slugifies filename, mints S3 PUT presigned URL.
  - **Why critical:** Upload URL minting. The team membership check uses `users: { some: { userId: ... } }` — fine.
  - **Test coverage:** NONE
  - **Risk:** `path.parse(fileName)` + `safeSlugify(name)` — does `safeSlugify` strip path separators? If not, an attacker with team access can write to `otherTeam/docId/...` keys. The key prefix uses `${team.id}/${docId}/...` — safe IF `docId` and `slugifiedName` are safe, but `docId` comes from request body unvalidated. A crafted `docId` of `../otherteam/foo` could traverse (depends on S3 key interpretation; S3 treats keys as strings, but downstream code that joins paths from the key could be vulnerable).

- **File:** `pages/api/file/s3/get-presigned-get-url.ts` and `get-presigned-get-url-proxy.ts`
  - **Function:** Mint GET presigned URLs.
  - **Why critical:** Download authorization.
  - **Test coverage:** NONE
  - **Risk:** Without reading these files, the proxy variant is a candidate for IDOR (does it check that the requesting user/team owns the key being signed?).

- **File:** `pages/api/file/tus/[[...file]].ts`, `pages/api/file/tus-viewer/[[...file]].ts`
  - **Why critical:** Resumable upload endpoints. Tus protocol has its own auth hooks.
  - **Test coverage:** NONE
  - **Risk:** Tus config (auth callbacks, allowed file sizes/types) is set in code; without tests, a regression in size limits or mime-type allowlist could allow arbitrary upload.

#### F. Slack OAuth & event verification

- **File:** `lib/integrations/slack/install.ts`, `events.ts`, `uninstall.ts`
  - **Why critical:** Slack signing-secret verification, OAuth state, install/uninstall lifecycle.
  - **Test coverage:** NONE

#### G. Stripe webhook handlers (state mutation)

- **Files:** `ee/stripe/webhooks/{checkout-session-completed,customer-subscription-updated,customer-subscription-deleted,invoice-upcoming}.ts`
  - **Why critical:** These mutate billing state. Signature verification happens upstream in `pages/api/stripe/webhook.ts`, but the handlers themselves run unsanitized DB writes.
  - **Test coverage:** NONE
  - **Risk:** Replay of an old event could re-apply state changes if the handlers don't check event timestamps or idempotency keys. Each handler is its own untested surface.

#### H. Passkey (WebAuthn) registration

- **File:** `pages/api/passkeys/register.ts`, `pages/api/account/passkeys.ts`
- **Why critical:** WebAuthn registration ties a credential to a user account. Mis-binding allows a user to register a passkey under another user's account.
- **Test coverage:** NONE

#### I. Link access flow

- **File:** `pages/api/links/[id]/index.ts`, `pages/api/links/[id]/archive.ts`, `pages/api/links/[id]/preview.ts`, `pages/api/links/[id]/duplicate.ts`, `pages/api/links/[id]/annotations.ts`, `pages/api/links/domains/[...domainSlug].ts`
- **Why critical:** All link CRUD + domain-routed access. IDOR risk on every endpoint.
- **Test coverage:** NONE

#### J. Download job endpoints

- **File:** `pages/api/links/download/{index,jobs,bulk,verify}.ts`, `pages/api/links/download/[jobId].ts`, `pages/api/links/download/file/[jobId]/[partIndex].ts`, `pages/api/links/download/dataroom-document.ts`, `pages/api/links/download/dataroom-folder.ts`
- **Why critical:** These stream document bytes to downloaders. Authorization must be airtight; otherwise any authenticated user can fetch any document by guessing job IDs.
- **Test coverage:** NONE

---

### Weak-Coverage Functions

Because there is no test infrastructure, there is no concept of "weak" coverage vs. "zero" coverage. **Every** security-critical function has ZERO coverage.

---

## Prioritization Signal for Other Nodes

For the rest of the whitebox-bug-finder workflow, treat the entire papermark security-critical surface as **unverified**. Specifically:

1. **Manual code review is the only safety net.** When another node (e.g., `ai-analyze-bugs`, `ai-audit-auth`) flags an issue, there is no test to confirm or refute it — every finding must be triaged by reading the code.

2. **High-leverage manual targets** (untested, high-blast-radius):
   - `lib/auth/dataroom-auth.ts` — `updateDataroomSessionVerified` accepts an arbitrary token and flips `verified` without caller-binding. Worth checking whether the route handler that calls it gates the token.
   - `lib/cron/verify-qstash.ts` — `VERCEL !== "1"` bypass. If papermark ships a self-hosted/Docker story, this is a hole.
   - `pages/api/file/s3/get-presigned-post-url.ts` — `safeSlugify` + `docId` from body → key path injection.
   - `lib/webhook/signature.ts` — `JSON.stringify` non-canonicalization for HMAC.
   - `lib/signing/access-token.ts` — 90-day token, `Path=/` scope.
   - All `ee/stripe/webhooks/*` handlers — replay/idempotency.

3. **Where the absence of tests hurts most:** auth + signing + webhook verification. These are precisely the functions where a single typo can grant unauthorized access. Without tests, the next regression is one commit away.

4. **Coverage cannot be fixed by adding one or two test files.** A meaningful test suite for this codebase would require: a test runner (Jest or Vitest), a Redis mock layer (Upstash Redis is used everywhere), a Prisma test DB, Next.js API route testing (supertest or similar), integration tests for OAuth providers. This is a project-level investment, not a per-PR concern.

---

## PHASE_3_CHECKPOINT
- [x] Findings written to `methodology-raw/00-ai-test-coverage.md`
- [x] Each entry has: file, function, why critical, test coverage, risk
- [x] No duplicate entries (one F1 finding for the "zero test files" headline, plus the catalog)