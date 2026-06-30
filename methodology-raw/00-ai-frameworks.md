# Framework & Version Recon — papermark

## Tech Stack Summary

| Layer | Technology | Version | Notes |
|-------|-----------|---------|-------|
| Language | TypeScript | ^5 | Strict mode enabled in tsconfig |
| Runtime | Node.js | >=24 | Per package.json engines |
| Frontend Framework | Next.js | ^14.2.35 | App Router + Pages Router hybrid; hybrid rendering (SSG, SSR, ISR) |
| UI Library | React | ^18.3.1 | Client + Server Components |
| CSS | Tailwind CSS | ^3.4.19 | With @tailwindcss/typography + @tailwindcss/forms |
| UI Component Lib | Tremor React | ^3.18.7 | Dashboard components |
| Rich Text | TipTap | ^3.22.5 | With Image/Placeholder/YouTube extensions |
| Database | PostgreSQL | (inferred) | Connection pooling via pgBouncer-style URL |
| ORM | Prisma | 6.5.0 | schema folder with relationJoins preview feature |
| Auth Framework | NextAuth.js | ^4.24.13 | JWT strategy, Prisma adapter |
| SAML SSO | BoxyHQ SAML Jackson | ^26.2.0 | Enterprise feature |
| OAuth/OIDC | oidc-provider | ^9.8.3 | Custom OAuth provider server |
| WebAuthn | Hanko Passkeys | ^0.3.1 | Passkey authentication |
| Validation | Zod | ^3.25.76 | Schema validation throughout |
| HTML Sanitization | sanitize-html | ^2.17.3 | Used in XSS-sensitive contexts |
| Password Hashing | bcryptjs | ^3.0.3 | Password hashing |
| JWT | jsonwebtoken | ^9.0.3 | Invitation tokens, unsubscribe tokens |
| File Upload | TUS Protocol | @tus/server ^1.10.2 | Resumable uploads |
| Background Jobs | Trigger.dev | 4.4.6 | Workflow engine |
| Queue | Upstash QStash | ^2.11.0 | HTTP-based job queue |
| Redis | Upstash Redis | ^1.38.0 | Rate limiting, caching |
| Rate Limiting | Upstash Ratelimit | ^2.0.8 | API rate limiting |
| Object Storage | AWS S3 | SDK ^3.1053.0 | Document file storage |
| Analytics | Tinybird | (custom) | View/document analytics |
| Product Analytics | PostHog | ^1.376.0 | Product telemetry |
| Payments | Stripe | ^16.12.0 | Billing/subscriptions |
| Email | Resend | ^6.12.3 | Transactional email |
| MCP | @modelcontextprotocol/sdk | ^1.29.0 | MCP server over Streamable HTTP |
| AI SDK | Vercel AI SDK | ^6.0.191 | AI chat features |
| Internationalization | i18next | ^26.3.0 | Viewer-facing i18n |
| Deployment | Vercel | — | Production hosting |

---

## Per-Framework Vulnerability Patterns

### Next.js 14.2.35

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| SSRF in getServerSideProps | ✅ Pages use getStaticProps/getServerSideProps for viewer pages | `pages/view/[linkId]/index.tsx:84` | LOW | SSRF protection already confirmed in SAST; data fetching uses Prisma not fetch |
| Middleware bypass via excluded paths | ✅ /api/* is excluded from middleware matcher | `middleware.ts:49` | MEDIUM | API routes do their own auth via NextAuth getToken/getServerSession, but the matcher pattern explicitly excludes `/api/`, `/oauth/`, `/.well-known/`, `/mcp/`. The `/api/` exclusion is by design (API routes self-authenticate), `/oauth/` routes are handled by oidc-provider, `/mcp` does its own bearer auth. |
| open redirect via callbackUrl | ✅ AppMiddleware normalizes `next` param | `lib/middleware/app.ts:12-48` | LOW | Properly validates path origin and strips protocol-relative paths; triple-decodes to catch double-encoding bypasses |
| API route auth bypass | ✅ Pages Router API routes use getToken/getServerSession | Multiple files | MEDIUM | Consistent pattern of `getToken({ req })` at top of each handler; no discovered gaps |
| Server-Side Request Forgery (SSRF) | ✅ Not found (already verified clean in SAST) | — | — | SAST confirmed all external fetch paths SSRF-safe |
| Image optimization SSRF | ✅ Remote patterns allowlisted | `next.config.mjs:382-456` | LOW | Only known CDNs and papermark domains allowed |
| CSP headers (Report-Only mode) | ✅ CSP is Report-Only, not enforced | `next.config.mjs:219-230` | MEDIUM | CSP is set to `Content-Security-Policy-Report-Only` — violations are reported but NOT blocked. This provides no actual protection against XSS; it only monitors for potential violations. |
| Embed CSP bypass (frame-ancestors *) | ✅ Embed routes allow * in frame-ancestors | `next.config.mjs:258-278` | MEDIUM | Embed routes explicitly set `frame-ancestors *` for iframe embedding, per design — but means any origin can embed these routes |

### React 18.3.1

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| dangerouslySetInnerHTML | ✅ 5 instances found | Multiple files | HIGH | See SAST finding for stored XSS via `document-header.tsx:604` |
| dangerouslySetInnerHTML (user content) | ✅ `helpText` prop | `components/ui/form.tsx:145`, `components/account/upload-avatar.tsx:96` | MEDIUM | These pass `helpText` which may contain user-controlled content |
| dangerouslySetInnerHTML (Shiki output) | ✅ Syntax-highlighted code | `components/webhooks/webhook-events.tsx:97` | LOW | Shiki output is properly escaped HTML |
| dangerouslySetInnerHTML (doc name) | ✅ Document name rendered raw | `components/documents/document-header.tsx:604` | HIGH | **Stored XSS** — SAST finding #1 |
| dangerouslySetInnerHTML (domain docs) | ✅ Instruction text | `components/domains/domain-configuration.tsx:143` | LOW | Static instruction content |
| Prototype pollution via setState | ✅ Not found | — | — | No discovered patterns of user input → setState key paths |
| Client-side XSS via refs | ✅ Not found | — | — | React patterns used consistently |

### Prisma 6.5.0

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| SQLi via $queryRaw | ✅ Used with tagged templates | Multiple files (year-in-review, analytics, viewers, health check) | LOW | All `$queryRaw` uses `Prisma.sql` tagged templates; `$executeRaw` in permission-groups uses generated SQL from schema-safe `buildBulkUpsertPermissionsSql` |
| SQLi via $executeRawUnsafe | ✅ Used only for savepoint management | `lib/folders/bulk-create.ts:51-60` | LOW | Only hardcoded `SAVEPOINT`/`RELEASE SAVEPOINT`/`ROLLBACK TO SAVEPOINT` statements with programmatic savepoint names (not user input) |
| Data exposure via Prisma schema | ✅ Account model stores OAuth tokens | `prisma/schema/schema.prisma:19-25` | MEDIUM | `Account` table stores `refresh_token`, `access_token`, `id_token` in plaintext columns. These contain OAuth credentials from providers (Google, LinkedIn, SAML). Also `User.stripeId` and `User.subscriptionId` are sensitive. |
| Mass assignment / field exposure | ✅ Standard Prisma queries throughout | — | LOW | No discovered over-fetching patterns; views/selects are proper |

### NextAuth.js 4.24.13

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| CSRF protection | ✅ Built into NextAuth | — | LOW | NextAuth handles CSRF natively |
| JWT secret exposure | ✅ Uses env var | `lib/middleware/app.ts:55` | LOW | Uses `process.env.NEXTAUTH_SECRET` |
| Session token cookie | ✅ Configured | `lib/auth/auth-options.ts:184-193` | LOW | `__Secure-next-auth.session-token` with httpOnly, sameSite: lax, secure on Vercel |
| Callback URL validation (open redirect) | ✅ AppMiddleware validates `next` param | `lib/middleware/app.ts:12-48` | LOW | Proper origin validation with triple-decoding |
| allowDangerousEmailAccountLinking | ✅ Google, LinkedIn, SAML providers | `lib/auth/auth-options.ts:35,55,130` | LOW | Acceptable for SSO; account linking by email is generally safe |
| No rate limiting on auth endpoints | ✅ Confirmed by SAST | `pages/api/auth/[...nextauth].ts` | LOW | Missing rate limiting on login/verify; SAST finding #4 |
| Email provider open relay potential | ✅ SendVerificationRequest uses Resend | `lib/auth/auth-options.ts:57-86` | LOW | SMTP relay abuse potential if unauthenticated users can trigger emails |

### oidc-provider 9.8.3

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| Open redirect via redirect_uris | ✅ Mounted at pages/api/oauth/* | `next.config.mjs:23` | MEDIUM | oidc-provider validates redirect_uris by default but configuration must explicitly list allowed URIs |
| PKCE enforcement | ✅ Unknown without config review | — | MEDIUM | Needs config review to confirm PKCE is required for all flows |
| Koa dependency risk | ✅ Externalized in webpack | `next.config.mjs:340-351` | LOW | oidc-provider uses Koa internally; externalized properly |
| Dynamic registration (DCR) exposure | ✅ Configured | `next.config.mjs:128-133` | MEDIUM | DCR endpoint allows new client registration; mitigations (scope limits, redirect URI validation) must be confirmed |

### Zod 3.25.76

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| Information disclosure via error details | ✅ Multiple endpoints return error.format() | Multiple files | MEDIUM | SAST finding #3 — Zod validation error messages leak schema structure |
| Type coercion bypass | ✅ Used with safeParseAsync | Multiple files | LOW | Zod coerces types by default in some cases; schema should validate types strictly |

### sanitize-html 2.17.3

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| Entity decode order (decode after strip) | ✅ decodeHTML called AFTER sanitizeHtml | `lib/utils/sanitize-html.ts:13-14` | HIGH | Critical ordering bug: HTML entities survive tag stripping, then decode to real tags. This is the root cause of the SAST stored XSS finding. |
| allowedTags: [] bypass | ✅ With entity encoding | `lib/utils/sanitize-html.ts:4-7` | HIGH | `allowedTags: []` doesn't strip encoded entities like `&lt;` |
| DOMPurify not used | ✅ Uses sanitize-html instead | — | MEDIUM | sanitize-html is a string-based parser (not DOM-based) and is more prone to bypasses than DOMPurify |

### jsonwebtoken 9.0.3

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| JWT stored in invitation links | ✅ Used for team invites & unsubscribe | `lib/utils/generate-jwt.ts:16-26` | LOW | 24-hour expiry, no refresh tokens, but tokens are in email links |
| No algorithm verification | ✅ jwt.verify used (auto-detects) | `lib/utils/generate-jwt.ts:35` | LOW | `jwt.verify` by default accepts `'none'` if the secret is empty; relies on empty-string check not being true |
| Weak JWT secret | ✅ Uses env var | `lib/utils/unsubscribe.ts:3` | LOW | Security depends on `NEXT_PRIVATE_UNSUBSCRIBE_JWT_SECRET` strength |
| Token in URL | ✅ Unsubscribe links have JWT in query params | `lib/utils/unsubscribe.ts` | LOW | Token in URL vulnerable to referer leakage |

### TUS Protocol (@tus/server 1.10.2)

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| CORS misconfiguration (origin reflection) | ✅ req.headers.origin echoed back | `pages/api/file/tus-viewer/[[...file]].ts:237` | MEDIUM | SAST finding #2 — arbitrary origin reflection with credentials allowed |
| No origin allowlist | ✅ No validation of Origin header | `pages/api/file/tus-viewer/[[...file]].ts:234-250` | MEDIUM | Any origin can make authenticated cross-origin requests |
| Upload endpoint design | ✅ No auth on CORS path | `pages/api/file/tus-viewer/[[...file]].ts:262` | MEDIUM | Comment confirms "No session check - authentication is handled via viewer metadata" — relies on TUS metadata for auth |

### @boxyhq/saml-jackson 26.2.0

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| SAML SSO misconfiguration | ✅ Jackson configured with Prisma + AES-256-GCM | `lib/jackson.ts` | LOW | Properly configured with Prisma backing store and 32-byte encryption key |
| SAML redirect endpoint exposure | ✅ Authorize endpoint at /api/auth/saml/authorize | `lib/auth/auth-options.ts:102` | MEDIUM | SAML authorize endpoint is exposed; relay state validation must be confirmed in Jackson config |
| Open redirect via SAML ACS | ✅ Inherits Jackson security | — | LOW | Jackson handles ACS URL validation |

### Upstash Rate Limit 2.0.8

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| Rate limiting not applied to auth endpoints | ✅ Only webhooks have rate limiting | `pages/api/webhooks/services/[...path]/index.ts:181` | LOW | SAST finding #4 — auth endpoints lack brute-force protection |
| Rate limiting on API endpoints | ✅ Not checked | — | MEDIUM | Missing rate limiting on most API endpoints could enable abuse |

### Vercel AI SDK 6.0.191

| Known Pattern | Found in Code | File:Line | Risk | Notes |
|---------------|---------------|-----------|------|-------|
| Prompt injection (AI chat) | ✅ Chat features use Vercel AI SDK | `ee/features/conversations/` | MEDIUM | AI chat features may be vulnerable to prompt injection; needs review of system prompts |
| Data leakage via AI context | ✅ Dataroom documents as AI context | — | MEDIUM | AI chat has access to document content; prompt injection could leak dataroom content |
| Provider key exposure | ✅ Uses @ai-sdk/openai + @ai-sdk/google-vertex | `package.json:24-25` | LOW | API keys managed via env vars |

---

## Key Risk Summary

| Risk Level | Count | Key Issues |
|-----------|-------|------------|
| CRITICAL | 0 | — |
| HIGH | 2 | Stored XSS via sanitize-html decode order (SAST); dangerouslySetInnerHTML on raw document name |
| MEDIUM | 8 | CORS origin reflection on TUS; CSP report-only mode; Zod info disclosure; oidc-provider DCR; embed frame-ancestors *; JWT in URLs; missing auth rate limiting; AI prompt injection |
| LOW | 6 | OAuth tokens in plaintext DB; middleware matcher exclusions; SAML config risks; allowDangerousEmailAccountLinking; alg confusion in jwt.verify; no PKCE confirm |

---

## SAST Cross-Reference

The SAST pass found 5 issues (1 HIGH, 2 MEDIUM, 2 LOW). This framework analysis confirms:

1. **Stored XSS (HIGH)** — Root cause confirmed in `sanitize-html` decode ordering: `lib/utils/sanitize-html.ts:13-14`
2. **CORS misconfiguration (MEDIUM)** — TUS viewer endpoint at `pages/api/file/tus-viewer/[[...file]].ts:237`
3. **Information disclosure (MEDIUM)** — Zod error formatting at multiple endpoints
4. **No auth rate limiting (LOW)** — NextAuth endpoints lack rate limiting
5. **Webhook data exposure (LOW)** — Webhook event bodies stored in Tinybird
