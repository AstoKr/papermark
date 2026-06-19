# Framework Analysis — Papermark

**Workflow:** whitebox-bug-finder
**Methodology:** M9 (Framework-specific vulnerability pattern recognition)
**Scope:** Identify all frameworks, languages, versions; map them to known vulnerability patterns; locate WHERE those patterns are used in the codebase.

---

## Tech Stack Summary

| Layer | Technology | Version | Notes |
|-------|------------|---------|-------|
| Runtime | Node.js | `>=24` | pinned via `engines.node` in `package.json:5` |
| Language | TypeScript | `^5` | strict mode on, `target: es5`, `moduleResolution: bundler` (`tsconfig.json`) |
| Framework | Next.js | `^14.2.35` | App Router (`app/`) + Pages Router (`pages/`) hybrid; React 18.3.1 |
| UI | React | `^18.3.1` | Server Components + Client Components; `dangerouslySetInnerHTML` used in 5 component files |
| API / HTTP | Next.js route handlers | n/a | `app/api/*` and `pages/api/*` |
| Auth | NextAuth.js | `^4.24.13` | JWT session strategy, Credentials + Email + Google + LinkedIn + Passkey (Hanko) + SAML (BoxyHQ Jackson) providers |
| OIDC server | `oidc-provider` | `^9.8.3` | Koa-based; exposed at `/oauth/*` (rewritten from `pages/api/oauth/*`) |
| SAML | `@boxyhq/saml-jackson` | `^26.2.0` | pinned `@xmldom/xmldom@0.9.10` via overrides |
| Database | PostgreSQL | n/a | Prisma `6.5.0` ORM; Vercel Postgres |
| ORM | Prisma | `6.5.0` | Client + raw SQL (`Prisma.sql`, `Prisma.raw`, `$executeRawUnsafe`) in 30+ files |
| Cache / Sessions | Upstash Redis | `@upstash/redis ^1.38.0` | Session store + link-session tokens + rate-limit counters |
| Rate limiting | `@upstash/ratelimit` | `^2.0.8` | |
| Storage | AWS S3 SDK | `^3.1053.0` | `@tus/server` + `@tus/s3-store` for resumable uploads |
| Blob | `@vercel/blob` | `^2.4.0` | Vercel Blob |
| MCP | `@modelcontextprotocol/sdk` | `^1.29.0` | Exposed at `mcp.papermark.com/mcp` |
| AI | Vercel AI SDK / OpenAI | `ai ^6.0.191`, `openai ^6.39.0` | |
| Email | Resend + React Email | `^6.12.3` / `^6.3.2` | `react-email` preview server |
| Payments | Stripe | `^16.12.0` | webhook handler at `/api/stripe/webhook` |
| Background jobs | Trigger.dev v4 | `4.4.6` | `lib/trigger/**` |
| Cron / Queues | `@upstash/qstash` | `^2.11.0` | |
| PDF | `mupdf`, `pdf-lib`, `@libpdf/core` | various | |
| Forms | React Hook Form + Zod | `^7.75.0` / `^3.25.76` | server-side validation via `zod-openapi` |
| HTML sanitizer | `sanitize-html` | `^2.17.3` | declared as dep but used in only 3 files (Grep) — coverage gap |
| Markdown | `react-markdown` | `^10.1.0` | |
| Crypto | `jsonwebtoken` `^9.0.3`, `bcryptjs` `^3.0.3`, `crypto.randomBytes` (node) | n/a | HMAC-signed link/view sessions |

> **Multi-router concern.** Papermark ships **both** App Router (`app/`) **and** Pages Router (`pages/`) in the same project. Each route handler must independently implement auth — there is no shared global middleware for the `api/` namespace (the root `middleware.ts` matcher excludes `api/`). This means a missed `getServerSession()` call in any new `app/api/*` route is immediately exploitable.

---

## Framework-Specific Vulnerability Patterns

### 1. Next.js 14.2.35 — HIGH

**Framework:** Next.js 14 (App Router + Pages Router hybrid)

**Known patterns for this version (public CVE database, knowledge cutoff Jan 2026):**
- **CVE-2025-29927** — Middleware authorization bypass via crafted `x-middleware-subrequest` header (CVSS 9.1, fixed in 14.2.25). Papermark is on `^14.2.35`, which includes the fix; **no remediation needed**, but worth verifying middleware doesn't rely on Vercel-specific middleware-subrequest internals.
- **CVE-2024-46982** — Cache poisoning via `Server Components` and Pages Router cache key collisions (fixed in 14.2.10). Patched.
- **CVE-2024-34351** — SSRF in Server Actions (fixed in 14.1.1). Patched.
- **CVE-2023-46227 / CVE-2023-44487** — older middleware bypass / HTTP/2 DoS; all patched at this version.

**Used in code:**
- Root middleware: `middleware.ts:53` — handles API-host restriction, custom domains, webhooks, PostHog proxy. Excludes `api/`, `oauth/`, `mcp/?$`, `.well-known/`, `_next/`, `_static`, `vendor`, `_icons`, `_vercel`, `favicon.ico`, `sitemap.xml`, `robots.txt`. **All non-API routes go through this**.
- Per-route auth: every `app/api/*` route must call `getServerSession(authOptions)` or `getToken()`. The presence of `app/api/` route handlers means **a missed session check on any route is exploitable** — there is no defense-in-depth layer for the `api/` namespace. (Verified via Grep: 30+ files use `getServerSession`, but coverage is opt-in.)

**Key finding — `app/api/` has NO global auth middleware.** Unlike `/app/(ee)/api/...` which goes through AppMiddleware, the `app/api/...` namespace is opted out of root middleware (matcher excludes `api/`). Each individual route is responsible for its own `getServerSession()` call. A grep for `getServerSession` in `app/api/**` shows it is **only used in `views`, `views-dataroom`, `integrations/slack/oauth/authorize`, and `cron/welcome-user`**. Routes like `app/api/help/**`, `app/api/og/**`, `app/api/feature-flags/**` rely entirely on their internal handlers.

**Risk level:** **MEDIUM** (framework patched for CVEs; risk is operational — per-route opt-in auth discipline).

**Vulnerable code paths:**

| File | Pattern | Line |
|------|---------|------|
| `middleware.ts` | host header check uses `host?.split(":")[0]` — safe (strips port) | 64 |
| `middleware.ts` | path-traversal blocklist on `/view/*` uses `.includes()` not strict prefix | 102-108 |
| `next.config.mjs` | CSP is **Report-Only** on default route, only enforced on `/view/:path*/embed` | 219, 262 |
| `next.config.mjs` | CSP allows `'unsafe-inline'` and `'unsafe-eval'` for `script-src` | 222, 265 |

---

### 2. React 18 — HIGH

**Framework:** React 18.3.1

**Known patterns:** `dangerouslySetInnerHTML` XSS, server/client component boundary leakage, hydration mismatch.

**Used in code — `dangerouslySetInnerHTML` is used in 5 component files:**

| File | Line | Source | Sanitized? | Risk |
|------|------|--------|-----------|------|
| `components/documents/document-header.tsx` | 604 | `prismaDocument.name` | **NO** | **HIGH** — Document names are user-controlled (any team member can rename). This is rendered inside a `contentEditable` `<h2>`. A team member uploads a doc named `<img src=x onerror=alert(1)>` and any subsequent page render executes JS. Same XSS surface for any viewer visiting the dashboard, since the document name is shown on the listing. |
| `components/account/upload-avatar.tsx` | 96 | `helpText` (translations / static content) | not verified at code level — appears to come from i18n files | **LOW** — likely static text from `locales/`. Needs review. |
| `components/webhooks/webhook-events.tsx` | 97 | `highlightedCode` from Shiki | Shiki HTML-escapes by default | **LOW** (informational) |
| `components/domains/domain-configuration.tsx` | 143 | `text` prop to `MarkdownText` component | The component trusts caller; in this file only called with hardcoded strings containing `<b><i>` tags (line 131) | **LOW** in current call sites, but `MarkdownText` is a reusable sink — any future caller passing user input becomes XSS. |
| `components/ui/form.tsx` | 145 | `helpText` (form description) | n/a — depends on caller | **LOW–MEDIUM** — needs per-caller audit; if any caller passes user input as `helpText`, it is XSS. |

**HIGHEST RISK:** `components/documents/document-header.tsx:604` — confirmed unsanitized user-controlled input rendered via `dangerouslySetInnerHTML`. This is the **single most exploitable XSS vector** in the codebase. The function escapes nothing, no DOMPurify, no `sanitize-html` call, and the field is editable by any user who can write documents.

**Risk level:** **HIGH** for `document-header.tsx:604`. **MEDIUM** for `form.tsx:145` and `domain-configuration.tsx:139-145` (reusable sink).

---

### 3. Prisma 6.5.0 — MEDIUM

**Framework:** Prisma ORM with PostgreSQL

**Known patterns:** SQL injection via raw queries, unsafe `$executeRawUnsafe`, prototype pollution through untyped query inputs.

**Used in code:** 30+ files use `prisma.$queryRaw`, `$executeRaw`, `$executeRawUnsafe`, `Prisma.sql`, `Prisma.raw`.

**Analysis of unsafe raw query sites:**

| File:Line | Pattern | Safe? |
|-----------|---------|-------|
| `lib/folders/bulk-create.ts:51, 54, 57, 60` | `tx.$executeRawUnsafe(\`SAVEPOINT "${savepointName}"\`)` — `savepointName` is built from a closed set of literal prefixes + integer `depth` | **Safe** — depth is an internal counter, not user-controlled |
| `lib/dataroom/permissions-sql.ts:47` | `Prisma.raw(\`"${table}"\`)` where `table: AccessControlTable` is a closed union | **Safe** (TypeScript compile-time enforced) |
| `pages/api/teams/[teamId]/viewers/index.ts:84,133`, `pages/api/teams/[teamId]/viewers/[id]/index.ts:154,202,233` | `prisma.$queryRaw\` ... \`` using tagged templates | **Safe** — tagged template interpolation is parameterized |
| `pages/api/analytics/index.ts:190,204,218` | `prisma.$queryRaw\` ... \`` | **Safe** — tagged templates |
| `pages/api/health.ts:10` | `prisma.$queryRaw\`SELECT 1\`` | **Safe** — static |
| `lib/year-in-review/get-stats.ts:5,274,302,335,366`, `lib/year-in-review/calculate-percentile.ts` | tagged `prisma.$queryRaw<...>(Prisma.sql\`...\`)` | **Safe** |
| `ee/features/dataroom-invitations/api/link-invite.ts:184` | `tx.$executeRaw\`SELECT pg_advisory_xact_lock(hashtext(${user.id}))\`` | **Safe** — tagged template |
| `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:231`, `[permissionGroupId].ts:307,316,333,354` | `tx.$queryRaw<...>` / `tx.$executeRaw(bulkUpsertSql)` | Needs review of how `bulkUpsertSql` is constructed — composed via `permissions-sql.ts` which uses `Prisma.raw` only on closed union `table` |

**Risk level:** **LOW** — no user-controlled SQL string concatenation found. The two `$executeRawUnsafe` sites use hardcoded SQL with internal-counter interpolation. The `Prisma.raw` site uses a TypeScript-closed union. All other raw queries use the tagged template form which Prisma parameterizes.

**Caveat:** Prisma's `$executeRawUnsafe` is fundamentally an injection-prone primitive. Any future change that passes user input into it becomes immediately exploitable. Recommend adding a lint rule to ban `$executeRawUnsafe` in this codebase.

---

### 4. NextAuth.js 4.24.13 — HIGH

**Framework:** NextAuth.js (next-auth)

**Known patterns for this version:** session token leakage, OAuth account-linking takeover, JWT misconfiguration, CSRF on callback URLs.

**Used in code:** `lib/auth/auth-options.ts` (single source of truth).

**Patterns observed:**

| Pattern | Line | Risk |
|---------|------|------|
| `allowDangerousEmailAccountLinking: true` on Google, LinkedIn, and SAML providers | `lib/auth/auth-options.ts:35, 55, 130` | **HIGH** — NextAuth docs explicitly call this "dangerous." It allows an attacker who controls a Google/SAML account at the same email as a victim to log in as the victim. This is intentional for SAML (no email verification otherwise) but is set on Google and LinkedIn too without justification. Combined with `email` provider below, this enables account takeover for any registered email. |
| Email provider is enabled (`EmailProvider`) | `lib/auth/auth-options.ts:57` | **MEDIUM** — email magic links. Default behavior is to require the user to exist; verify no auto-create. |
| `clientSecret: process.env.NEXTAUTH_SECRET as string` reused as SAML client secret | `lib/auth/auth-options.ts:128, 149` | **LOW** — using the same secret for two distinct crypto purposes; not ideal but acceptable for an internal OIDC dance. |
| `cookies.sessionToken` is `httpOnly: true, sameSite: "lax", secure: VERCEL_DEPLOYMENT` | `lib/auth/auth-options.ts:184-193` | **LOW** — proper cookie flags. `sameSite: "lax"` is documented as a deliberate trade-off in `methodology-raw/00-ai-configs.md:337`. |
| `session: { strategy: "jwt" }` | `lib/auth/auth-options.ts:182` | **Informational** — JWT sessions can't be revoked server-side; combined with `email`-provider sign-in means session invalidation requires user-side (e.g., cookie expiry only). |
| `account.deleteMany({ where: { userId: user.id } })` in jwt callback on email change | `lib/auth/auth-options.ts:223-226` | **Informational** — logs user out of all linked OAuth providers on email change. Intentional but means SAML users lose SSO access on email update. |

**Risk level:** **HIGH** — `allowDangerousEmailAccountLinking: true` is set on three providers including Google and LinkedIn where the documented "dangerous" justification doesn't apply. Any user can be impersonated by an attacker who registers a Google account with the same email.

---

### 5. node-oidc-provider 9.8.3 — MEDIUM

**Framework:** node-oidc-provider (Koa-based)

**Known patterns:** OIDC discovery endpoint spoofing, PKCE downgrade, JWKS rotation timing.

**Used in code:** Mounted at `/oauth/*` (rewritten from `pages/api/oauth/*`). Provider config in `pages/api/auth/[...nextauth].ts`. Schema models in `prisma/schema/oauth.prisma`.

**Patterns observed:**
- PKCE enabled: `checks: ["pkce", "state"]` on SAML provider (`auth-options.ts:100`)
- DCR (Dynamic Client Registration) — inferred from `oidc-provider` default config; the `mcp.papermark.com` docs (`next.config.mjs:128-141`) explicitly mentions DCR
- DCR approval interstitial at `/oauth/authorize` for MCP host (`next.config.mjs:131-134`)
- OAuth discovery served at `/.well-known/openid-configuration` (rewritten from `/api/oauth/.well-known/openid-configuration`)

**Risk level:** **MEDIUM** — `oidc-provider` 9.x is current as of knowledge cutoff; specific CVEs (if any) not verified at the version level. Risk concentrated in the OAuth provider's PKCE/state enforcement rather than in this codebase's wrapper.

---

### 6. tus / @tus/server 1.10.2 — CRITICAL (CORS)

**Framework:** `@tus/server` resumable upload protocol

**Known patterns:** The tus spec allows the client to set `Upload-Metadata` headers containing base64-encoded metadata keys. This server reads `linkId`, `viewerId`, `dataroomId` from those headers and trusts them for auth.

**Used in code:** `pages/api/file/tus-viewer/[[...file]].ts`

**CRITICAL finding — Wildcard CORS with credentials reflection:**

`pages/api/file/tus-viewer/[[...file]].ts:234-250`:
```typescript
const setCorsHeaders = (req: NextApiRequest, res: NextApiResponse) => {
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
  ...
};
```

The server **reflects the request `Origin` header into `Access-Control-Allow-Origin`** AND sets `Access-Control-Allow-Credentials: true`. This is the textbook CORS-misconfiguration enabling cross-origin authenticated reads/writes from any attacker-controlled origin.

Even though `tus-server` itself does not rely on browser cookies (auth is via viewer metadata in the upload-metadata header), this configuration allows any malicious origin to:
1. Initiate resumable uploads against `*.papermark.com` (or any custom domain serving this route).
2. Successfully read responses cross-origin (CORS preflight passes).
3. Use a victim's already-uploaded `Upload-Offset` to inject bytes into an in-progress upload the victim started — though this depends on whether the victim's auth metadata was used.

The comment on line 233 says "to allow custom domains," which is the legit motivation, but the correct implementation is a **closed allowlist** of permitted origins, not reflection.

**Risk level:** **CRITICAL** — this is a textbook CORS misconfiguration. Exploitable by any attacker-controlled origin.

**Remediation:** Replace line 237 with an allowlist check (e.g., `ALLOWED_ORIGINS.includes(req.headers.origin) ? req.headers.origin : 'null'`), or read origin from a closed set of custom-domain values from the database.

---

### 7. sanitize-html 2.17.3 — LOW

**Framework:** `sanitize-html` (declared dependency).

**Known patterns:** Server-side HTML sanitization at the wrong layer (after render, not before).

**Used in code:** Only 3 files use `sanitize-html` (Grep). The much more common React `dangerouslySetInnerHTML` pattern is **NOT preceded by `sanitize-html`** in `components/documents/document-header.tsx:604`, which is the most likely XSS site (see §2).

**Risk level:** **LOW** (sanitization tool exists), but **HIGH** (it's not applied at the highest-risk sink).

---

### 8. lodash 4.18.1 (via override) — INFO

**Framework:** lodash (pinned via `overrides` in `package.json:222`).

The override pins `lodash: 4.18.1`, which fixes the prototype pollution CVEs CVE-2019-10744 / CVE-2020-8203. This is correct and good.

**Risk level:** **INFO** (defensive override applied).

---

### 9. bcryptjs 3.0.3 — LOW

**Framework:** `bcryptjs` (pure-JS bcrypt).

**Known patterns:** Weak password hashing when used at default cost factor.

**Risk level:** **LOW** — usage is for non-critical token hashing (likely OAuth state), not primary user passwords (which are delegated to Google/LinkedIn/SAML/passkey).

---

### 10. `oidc-provider` Koa externalization — INFO

`next.config.mjs:340, 350` marks `oidc-provider` and `koa` as `serverComponentsExternalPackages` and webpack externals. This is correct given Koa's dynamic require usage. **No security impact**, but it means **webpack cannot statically analyze the OAuth route for code-level review** — any CVE inside oidc-provider/Koa is in scope for runtime patching.

---

## Cross-Cutting Findings

### CSP Report-Only mode — MEDIUM

`next.config.mjs:219` sets the default CSP as `Content-Security-Policy-Report-Only`, not enforcing. Only the embed route at line 262 enforces CSP via `Content-Security-Policy`. This means an XSS payload (e.g., from the `document-header.tsx` dangerouslySetInnerHTML) would not be blocked by the browser — only reported. Combined with the `'unsafe-inline' / 'unsafe-eval'` permissions for `script-src` (lines 222, 265), the CSP provides essentially zero runtime mitigation.

**Risk level:** **MEDIUM** — operational posture, not a code defect.

### Open redirect in `normalizeNextPath` — LOW (mitigated)

`lib/middleware/app.ts:12-48` correctly:
- Rejects non-`/`-starting paths.
- Rejects `//` and `/\\` (protocol-relative).
- Parses against `requestUrl` and compares `targetUrl.origin` to `requestOrigin`.
- Handles double URL decoding (3 iterations).

**Risk level:** **LOW** — defense-in-depth correctly applied.

### `customDomain` heuristic in middleware — INFO

`middleware.ts:22-34`: a host is treated as a custom domain if it doesn't include `localhost`, `papermark.io`, `papermark.com`, or end with `.vercel.app` (in non-development). This is a **deny-list** of known-good hosts — any new host (e.g., a new Vercel preview URL pattern, or a typo'd subdomain) is treated as a custom domain and routed to `DomainMiddleware`. **Risk level: INFO** — correct behavior, but relies on the assumption that all legitimate hosts are listed.

---

## Severity Summary

| # | Framework | Severity | Issue |
|---|-----------|----------|-------|
| 1 | Next.js 14.2.35 | MEDIUM | Per-route auth discipline in `app/api/*` (no global middleware for that namespace) |
| 2 | React 18 | **HIGH** | `dangerouslySetInnerHTML` on user-controlled `prismaDocument.name` at `components/documents/document-header.tsx:604` |
| 3 | Prisma 6.5.0 | LOW | `$executeRawUnsafe` and `Prisma.raw` are used safely today (closed unions, internal counters) but should be lint-banned |
| 4 | NextAuth.js 4.24.13 | **HIGH** | `allowDangerousEmailAccountLinking: true` on Google and LinkedIn providers |
| 5 | node-oidc-provider 9.8.3 | MEDIUM | Standard OIDC provider risk surface; PKCE/state enforced |
| 6 | @tus/server 1.10.2 | **CRITICAL** | Wildcard origin reflection + `Access-Control-Allow-Credentials: true` at `pages/api/file/tus-viewer/[[...file]].ts:237` |
| 7 | sanitize-html 2.17.3 | LOW | Present but not applied at highest-risk sink |
| 8 | lodash 4.18.1 | INFO | Defensive override applied |
| 9 | bcryptjs 3.0.3 | LOW | Used for non-critical hashes |
| 10 | CSP enforcement | MEDIUM | Report-Only mode + `unsafe-inline`/`unsafe-eval` |

### PHASE_3_CHECKPOINT
- [x] Findings written to `methodology-raw/00-ai-frameworks.md`
- [x] Each finding has: file, line, severity, description, evidence
- [x] No duplicate findings (cross-cutting concerns called out separately)