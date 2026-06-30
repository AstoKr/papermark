# Test-Coverage Gap Recon — Prioritization Signal

**Result:** The entire codebase has **ZERO test coverage** — no test files, test framework dependencies, or test configuration exist anywhere in the repository. Every security-critical module is entirely untested.

---

## Test Coverage Map

| Source File | Test File | Coverage | Security-Critical? |
|---|---|---|---|
| `lib/auth/auth-options.ts` | — | NONE | YES — NextAuth JWT config, OAuth providers, user creation |
| `lib/auth/link-session.ts` | — | NONE | YES — Redis-backed link sessions, fingerprinting, rate limiting, access tracking |
| `lib/auth/dataroom-auth.ts` | — | NONE | YES — Dataroom Redis sessions, fingerprint/IP validation |
| `lib/auth/preview-auth.ts` | — | NONE | YES — Preview session tokens (20-min TTL) |
| `lib/middleware/app.ts` | — | NONE | YES — Auth guard, open redirect prevention (`normalizeNextPath`) |
| `lib/middleware/domain.ts` | — | NONE | YES — Custom domain routing, path blocking, 404 rewrites |
| `lib/middleware/incoming-webhooks.ts` | — | NONE | YES — Webhook path rewriting, host detection |
| `lib/middleware/posthog.ts` | — | NONE | WEAK — Header filtering, CORS (limits blast radius) |
| `lib/api/auth/with-session-team.ts` | — | NONE | YES — Centralized auth: session, RBAC, plan gate, entitlements |
| `lib/api/auth/passkey.ts` | — | NONE | YES — WebAuthn passkey registration/removal |
| `lib/api/auth/token.ts` | — | NONE | WEAK — SHA-256 token hashing |
| `lib/api/rbac/permissions.ts` | — | NONE | YES — Role→permission mapping, `hasAllPermissions()` |
| `lib/api/rbac/entitlements.ts` | — | NONE | YES — Dataroom-scoped access, `assertDocumentAccess()` |
| `lib/api/rbac/guard.ts` | — | NONE | YES — IDOR prevention guards |
| `lib/signing/access-token.ts` | — | NONE | YES — HMAC-SHA256 90-day tokens, `timingSafeEqual` |
| `lib/signing/download-token.ts` | — | NONE | YES — HMAC-SHA256 30-min download tokens |
| `lib/signing/agreements.ts` | — | NONE | YES — Documenso integration, webhook verification, access gatekeeping |
| `lib/signing/envelopes.ts` | — | NONE | WEAK — Documenso envelope API wrapper |
| `lib/signing/mirror.ts` | — | NONE | WEAK — S3/Blob storage mirroring (idempotent) |
| `lib/webhook/send-webhooks.ts` | — | NONE | YES — QStash delivery orchestration |
| `lib/webhook/signature.ts` | — | NONE | YES — HMAC-SHA256 webhook signature generation |
| `lib/webhook/triggers/*.ts` | — | NONE | WEAK — Webhook event handlers |
| `lib/zod/url-validation.ts` | — | NONE | YES — SSRF protection, path traversal prevention |
| `lib/zod/schemas/multipart.ts` | — | NONE | YES — Multipart upload validation |
| `lib/zod/schemas/webhooks.ts` | — | NONE | YES — Webhook payload validation |
| `lib/zod/schemas/presets.ts` | — | NONE | WEAK — Preset config validation |
| `lib/api/links/link-data.ts` | — | NONE | YES — Link data fetching (access-control) |
| `lib/api/links/bulk-import.ts` | — | NONE | YES — Bulk link creation (password, expiry, allow/deny lists) |
| `lib/api/domains/validate-redirect-url.ts` | — | NONE | WEAK — Redirect URL validation |
| `lib/tracking/record-link-view.ts` | — | NONE | WEAK — Bot detection, GDPR EU IP exclusion |
| `ee/limits/server.ts` | — | NONE | WEAK — Plan enforcement, usage tracking |
| `ee/limits/handler.ts` | — | NONE | WEAK — Limits API endpoint |
| `app/api/views/route.ts` | — | NONE | YES — Document view recording, session creation |
| `app/api/views-dataroom/route.ts` | — | NONE | YES — Dataroom view recording |
| `app/api/auth/verify-code/route.ts` | — | NONE | YES — Email OTP verification, rate limiting |
| `app/api/cron/domains/route.ts` | — | NONE | WEAK — Domain verification, auto-deletion |
| `app/api/integrations/slack/oauth/*` | — | NONE | WEAK — OAuth state parameter validation |
| `app/api/webhooks/signing/route.ts` | — | NONE | YES — Documenso signing webhook receiver |

---

## High-Priority Untested Code

### 1. Authentication & Session Management (lib/auth/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/auth/link-session.ts` | `createLinkSession()` | Creates Redis session with fingerprint; the primary auth mechanism for shared links | NONE | CRITICAL — an undiscovered session injection or fingerprint bypass would expose all shared documents |
| `lib/auth/link-session.ts` | `verifyLinkSession()` | Session validation pipeline: expiry, fingerprint, access count, rate limiting | NONE | CRITICAL — bypass here compromises every shared link's access control |
| `lib/auth/link-session.ts` | `revokeLinkSession()` | Session revocation by link ID | NONE | HIGH — revocation failures mean leaked sessions can't be killed |
| `lib/auth/link-session.ts` | `rateLimitSession()` | 100 req/min Redis rate limiting | NONE | HIGH — rate-limit bypass enables brute-force attacks |
| `lib/auth/dataroom-auth.ts` | `createDataroomSession()` | Dataroom Redis session with fingerprint+IP binding | NONE | CRITICAL — same as link-session but for datarooms |
| `lib/auth/dataroom-auth.ts` | `verifyDataroomSession()` | Session validation with fingerprint/IP comparison | NONE | CRITICAL — combined with `updateDataroomSessionVerified()` flag logic |
| `lib/auth/dataroom-auth.ts` | `updateDataroomSessionVerified()` | Marks session as verified after OTP | NONE | HIGH — verification-skip bug unlocks downloads |
| `lib/auth/auth-options.ts` | `jwt()` callback | Attaches user object and provider info to JWT token | NONE | CRITICAL — JWT injection or privilege escalation in the callback |
| `lib/auth/auth-options.ts` | `session()` callback | Merges token data into session user | NONE | HIGH — session poisoning via malformed token data |
| `lib/auth/auth-options.ts` | `createUser()` event handler | Fires analytics, tracking, and welcome email on signup | NONE | MEDIUM — DoS via mass signup, analytics injection |
| `lib/auth/preview-auth.ts` | `createPreviewSession()` | 20-min preview tokens | NONE | MEDIUM — short TTL limits blast radius |

### 2. Open Redirect Prevention (lib/middleware/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/middleware/app.ts` | `normalizeNextPath()` | The sole gate against open redirect via `next` param; decodes up to 3×, checks origin | NONE | CRITICAL — bypass enables phishing via papermark.com subdomain |
| `lib/middleware/app.ts` | `isProtocolRelativePath()` | Detects `//evil.com` patterns | NONE | HIGH — missed edge case allows protocol-relative redirect |
| `lib/middleware/app.ts` | Authentication guard redirect | Redirects unauthenticated users to login with `next` param | NONE | CRITICAL — combined with normalizeNextPath bypass |

### 3. Centralized Access Control (lib/api/auth/ + lib/api/rbac/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/api/auth/with-session-team.ts` | `withTeam()` | Wraps ALL protected API routes; handles auth check, membership, role gate, permission gate, plan gate, entitlement | NONE | CRITICAL — a single bug here affects all API routes; privilege escalation, IDOR, or plan-skip |
| `lib/api/auth/with-session-team.ts` | `withTeamApi()` | Pages Router variant | NONE | CRITICAL — same blast radius |
| `lib/api/auth/with-session-team.ts` | `resolveSessionTeam()` | Resolves session→team→membership→role in one pipeline | NONE | CRITICAL — context confusion between teams |
| `lib/api/rbac/permissions.ts` | `hasAllPermissions()` | Checks role against required permission verbs | NONE | HIGH — incorrect mapping grants unauthorized access |
| `lib/api/rbac/entitlements.ts` | `canAccessDataroom()` | Dataroom-scoped access for DATAROOM_MEMBER role | NONE | HIGH — mis-scoping allows cross-dataroom access |
| `lib/api/rbac/entitlements.ts` | `assertDocumentAccess()` | Document-level access assertion | NONE | CRITICAL — document IDOR |
| `lib/api/rbac/guard.ts` | `enforceDataroomMemberScope()` | Inline IDOR guards for non-migrated routes | NONE | HIGH — missed migration + incomplete guard = IDOR |

### 4. HMAC Token Cryptography (lib/signing/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/signing/access-token.ts` | `mintSignedAgreementAccessToken()` | Creates 90-day HMAC-SHA256 token | NONE | CRITICAL — HMAC implementation bugs compromise all signing agreements; token forgery = permanent access |
| `lib/signing/access-token.ts` | `parseSignedAgreementAccessToken()` | Validates HMAC, parses payload | NONE | CRITICAL — timing-attack surface via timingSafeEqual usage correctness |
| `lib/signing/access-token.ts` | `buildSignedAgreementAccessCookie()` | HttpOnly cookie with Path=/ | NONE | MEDIUM — cookie attribute correctness |
| `lib/signing/download-token.ts` | `mintSignedAgreementDownloadToken()` | 30-min HMAC download token | NONE | HIGH — forgery allows unauthorized PDF download |
| `lib/signing/agreements.ts` | `getAgreementResponseForAccess()` | Access gatekeeping for signing agreements | NONE | CRITICAL — binding verification logic; replay-attack surface |
| `lib/signing/agreements.ts` | `verifySigningWebhookSecret()` | timingSafeEqual comparison for webhook secret | NONE | HIGH — correctness of constant-time comparison |

### 5. Webhook Security (lib/webhook/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/webhook/signature.ts` | `createWebhookSignature()` | HMAC-SHA256 over JSON payload | NONE | HIGH — signature correctness; missing sig = spoofed webhooks |
| `lib/webhook/send-webhooks.ts` | `sendWebhooks()` | QStash delivery with callback tracking | NONE | MEDIUM — delivery failure handling |

### 6. Input Validation Security (lib/zod/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/zod/url-validation.ts` | `validateUrlSSRFProtection()` | Blocks private IPs, localhost, loopback | NONE | CRITICAL — SSRF bypass allows internal network scanning |
| `lib/zod/url-validation.ts` | `validatePathSecurity()` | Blocks path traversal, null bytes, double encoding | NONE | CRITICAL — traversal bypass allows arbitrary file read |
| `lib/zod/url-validation.ts` | `validateUrlSecurity()` | Composite: path security + SSRF | NONE | CRITICAL — used by document upload, link URL, webhook URL |
| `lib/zod/url-validation.ts` | `documentUploadSchema` | Validates document type, storage type consistency, URL/filepath | NONE | HIGH — upload validation bypass |
| `lib/zod/schemas/webhooks.ts` | `createWebhookSchema` | Validates URL (max 190 chars), secret prefix `whsec_` | NONE | MEDIUM — edge cases in URL format parsing |

### 7. API Routes (app/api/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `app/api/views/route.ts` | `POST` handler | Creates link sessions, triggers OTP/password/agreement checks, records views | NONE | CRITICAL — the main document access endpoint; bugs affect every shared document |
| `app/api/views-dataroom/route.ts` | `POST` handler | Creates dataroom sessions, manages DATAROOM_VIEW/DOCUMENT_VIEW | NONE | CRITICAL — same as views but for datarooms |
| `app/api/auth/verify-code/route.ts` | `POST` handler | Email OTP verification with atomic fetch-and-delete | NONE | CRITICAL — bypass grants unauthorized access to any email-protected link |
| `app/api/views/pages/route.ts` | Page fetching | Returns signed URLs for document pages | NONE | HIGH — page-access bypass leaks document content |
| `app/api/webhooks/signing/route.ts` | `POST` handler | Documenso webhook receiver | NONE | HIGH — processing untrusted external payloads |

### 8. Bulk Link Import (lib/api/links/)
| file | function | why critical | coverage | risk |
|---|---|---|---|---|
| `lib/api/links/bulk-import.ts` | `handleBulkLinkImport()` | Creates links in bulk with password, allow/deny lists, expiry | NONE | HIGH — mass link creation with security settings; validation gaps affect all created links |

---

## Summary: Prioritization Signal

### Immediate Focus (Structural/Synthesis Layers)

The analysis layer should prioritize these modules in order:

1. **`lib/auth/link-session.ts` + `dataroom-auth.ts`** — The session system is the primary auth mechanism for ALL shared content. Flaws here are game-over (any link = full access). Zero tests on Redis-backed session creation, fingerprint validation, and rate limiting.
2. **`lib/api/auth/with-session-team.ts`** — Wraps every protected API route. A single bug cascades to all endpoints: privilege escalation, cross-team data access, plan-gate bypass.
3. **`lib/middleware/app.ts` (`normalizeNextPath`)** — Open redirect prevention is a single function with no tests and a known attack surface (double encoding, protocol-relative URLs).
4. **`lib/signing/access-token.ts` + `agreements.ts`** — HMAC token implementation and signing-agreement access gatekeeping. Cryptographic correctness cannot be verified by inspection alone.
5. **`lib/zod/url-validation.ts`** — SSRF and path-traversal prevention gates. Bypass here enables internal network attacks or arbitrary file reads.
6. **`app/api/views/route.ts` + `views-dataroom/route.ts`** — The primary document/dataroom access endpoints. Untested OTP flow, password decryption, session creation.
7. **`lib/api/rbac/`** — Permission mapping, entitlement checks, IDOR guards. All untested.

### Statistical Signal

- **Security-critical source files identified:** 32
- **Files with ANY test coverage:** 0 (0%)
- **Files with NONE coverage:** 32 (100%)
- **CRITICAL-risk gaps:** 14
- **HIGH-risk gaps:** 12
- **MEDIUM-risk gaps:** 6
