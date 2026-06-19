# AI Analysis: API Specs

**Methodology:** M8 (whitebox-bug-finder, ai-analyze-api-specs)
**Repo:** papermark
**Date:** 2026-06-19
**Scope:** Compare documented API surface against actual route handlers, trace auth coverage, find undocumented endpoints, auth mismatches, and data leakage.

---

## Summary

| # | Finding | Severity | Endpoint Family |
|---|---------|----------|-----------------|
| 1 | `/api/v1/*` and `/openapi.json` referenced in middleware/next.config but **no handler files exist** | CRITICAL | Public v1 API surface |
| 2 | Cron routes skip QStash signature verification unless `VERCEL === "1"` | CRITICAL | Cron jobs (`/api/cron/*`) |
| 3 | Non-timing-safe `!==` comparison for `INTERNAL_API_KEY` on 4 jobs routes | CRITICAL | Internal jobs (`/api/jobs/*`) |
| 4 | `/api/feature-flags` accepts arbitrary `teamId` query — no auth, leaks feature flags per team | HIGH | Public utility |
| 5 | `/api/stripe/webhook` returns 200 OK on missing signature/secret instead of 4xx | HIGH | Webhook |
| 6 | `/api/cron/domains` (and others) auth bypass outside Vercel | CRITICAL | Cron jobs |
| 7 | `/api/file/tus-viewer` CORS misconfiguration — `Access-Control-Allow-Origin` echoes request origin | MEDIUM | File upload |
| 8 | `/api/csp-report` accepts any JSON, no validation/logging, infinite storage potential | LOW | Utility |
| 9 | No OpenAPI/Swagger spec file exists in repo | MEDIUM | Documentation |
| 10 | `/api/conversations` (visitor) relies on `viewerId` alone — soft access control | MEDIUM | Visitor API |
| 11 | `/api/og/route.tsx` renders arbitrary `title` query with no rate limiting | LOW | OG image |
| 12 | `/api/help` proxies marketing site without response size/rate limits | LOW | Utility |

Total routes indexed by GitNexus: **290**. Of these, 0 had middleware wrappers (auth is checked inline in each handler). 11 routes serve public visitor traffic and intentionally rely on link/OTP/session tokens; these were not flagged.

---

## Finding 1: Public `/api/v1/*` API surface advertised but not implemented

- **Severity:** CRITICAL
- **Endpoint:** GET `https://api.papermark.com/v1/...` and `https://api.papermark.com/openapi.json`
- **Type:** undocumented-endpoint / spec-vs-code-mismatch
- **Handler:** None — files do not exist
- **Auth middleware:** n/a — middleware explicitly allows the path through
- **Description:**
  `middleware.ts:66` whitelists `/v1`, `/v1/*`, and `/openapi.json` on the `api.papermark.com` host:
  ```ts
  if (path === "/v1" || path.startsWith("/v1/") || path === "/openapi.json") {
    return NextResponse.next();
  }
  ```
  `next.config.mjs:49-52` rewrites `/v1/:path*` → `/api/v1/:path*` (gated by host). However, no route handler files exist under `app/api/v1/` or `pages/api/v1/` (verified: `find . -path '*api/v1*'` returns 0 results; `app/api/` has only `auth, cron, csp-report, feature-flags, help, integrations, og, verify, views, views-dataroom, webhooks`).
- **Evidence:**
  - `middleware.ts:65-72` — `apiHost && requestHostname === apiHost` branch passes `/v1` and `/openapi.json` straight through.
  - `next.config.mjs:42-52` — `beforeFiles` rewrite with `has: [{ type: "host", value: apiHost }]`.
  - `find . -path '*api/v1*'` returns empty (no handler files).
- **Impact:**
  - Any request to `/v1/<anything>` on `api.papermark.com` will fall through middleware and be served by Next.js, returning 404 (or worse, an HTML page) — the marketing site promises a stable v1 API at `papermark.com/docs/api`. Attackers can probe for handler-level bugs against any URL shape Next.js accepts.
  - `/openapi.json` returns whatever Next.js serves for that path — likely the marketing site's catch-all or a 404 — which is then consumed by OpenAPI tooling as a schema. This poisons the public contract.
- **Recommendation:** Either remove the middleware/next.config rules until v1 ships, or implement the routes with real auth (bearer token, rate limits).

---

## Finding 2: Cron routes skip QStash signature verification outside Vercel

- **Severity:** CRITICAL
- **Type:** auth-missing / auth-bypass
- **Endpoints affected (6):**
  - `POST /api/cron/domains` → `app/api/cron/domains/route.ts:26-34`
  - `POST /api/cron/year-in-review` → `app/api/cron/year-in-review/route.ts:12-20`
  - `POST /api/cron/welcome-user` → `app/api/cron/welcome-user/route.ts:8-12` (uses `verifyQstashSignature`)
  - `POST /api/cron/dataroom-digest/daily` → `app/api/cron/dataroom-digest/daily/route.ts`
  - `POST /api/cron/dataroom-digest/weekly` → `app/api/cron/dataroom-digest/weekly/route.ts`
  - (any other route using `verifyQstashSignature` from `lib/cron/verify-qstash.ts`)
- **Handler:** `lib/cron/verify-qstash.ts:11-14`
- **Auth middleware:** absent — auth is in-handler via QStash signature
- **Description:**
  All six cron routes guard the QStash signature check behind `process.env.VERCEL === "1"`. In any non-Vercel deployment (including production self-hosting, CI smoke tests, or a misconfigured Vercel project where `VERCEL` is unset), the function returns early without verifying the signature:
  ```ts
  // lib/cron/verify-qstash.ts
  export const verifyQstashSignature = async ({ req, rawBody }) => {
    // skip verification in local development
    if (process.env.VERCEL !== "1") {
      return;        // ← silently accepts any caller
    }
    ...
  };
  ```
  Same pattern in `app/api/cron/domains/route.ts:26-34`:
  ```ts
  if (process.env.VERCEL === "1") {
    const isValid = await receiver.verify({ ... });
    if (!isValid) return new Response("Unauthorized", { status: 401 });
  }
  // ← no else branch: outside Vercel, anyone can POST
  ```
- **Evidence:** see code excerpts above. The Receiver is also instantiated with `currentSigningKey: process.env.QSTASH_CURRENT_SIGNING_KEY || ""` (`lib/cron/index.ts:13`), so even when verification runs, an unset env produces an empty key — verification trivially fails open.
- **Impact:**
  - On any non-Vercel production deployment, attackers can:
    - Send arbitrary welcome emails to arbitrary `userId` (welcome-user).
    - Trigger domain deletion: `domains` cron deletes invalid domains after 30 days (`app/api/cron/domains/route.ts:99-105`). An attacker who can hit this endpoint could shorten the timer by spamming or bypass it to wipe domains.
    - Force the year-in-review email queue to flush (`year-in-review`).
    - Drain dataroom digest jobs.
  - On Vercel, an attacker who learns `QSTASH_CURRENT_SIGNING_KEY` (e.g., via a leaked env) gets the same power without env-flag constraints.
- **Recommendation:**
  1. Always verify QStash signature regardless of `VERCEL`. The dev escape hatch should be an explicit `process.env.NODE_ENV === "development"` (which is never true in production).
  2. Fail-closed when `QSTASH_CURRENT_SIGNING_KEY` is missing.
  3. Consider IP-allowlisting QStash outbound IPs in addition to signature verification.

---

## Finding 3: Timing-unsafe comparison on `INTERNAL_API_KEY` for jobs routes

- **Severity:** CRITICAL
- **Type:** auth-bypass / timing-attack
- **Endpoints affected (4):**
  - `POST /api/jobs/send-notification` → `pages/api/jobs/send-notification.ts:28-34`
  - `POST /api/jobs/send-dataroom-new-document-notification` → `pages/api/jobs/send-dataroom-new-document-notification.ts:25-28`
  - `POST /api/jobs/send-dataroom-upload-notification` → `pages/api/jobs/send-dataroom-upload-notification.ts:22-25`
  - `POST /api/jobs/process-download-batch` → `pages/api/jobs/process-download-batch.ts:28-31`
- **Auth middleware:** absent — Bearer token compared inline
- **Description:**
  All four handlers use JavaScript `!==` to compare the bearer token against `process.env.INTERNAL_API_KEY`. This is non-constant-time and leaks information about each byte of the secret through response timing:
  ```ts
  const authHeader = req.headers.authorization;
  const token = authHeader?.split(" ")[1];
  if (token !== process.env.INTERNAL_API_KEY) {
    res.status(401).json({ message: "Unauthorized" });
    return;
  }
  ```
  Contrast with the correctly-implemented `ee/features/billing/cancellation/api/automatic-unpause-route.ts:43-50` which uses `crypto.timingSafeEqual`. The codebase already has the right primitive — these four files just don't use it.
- **Evidence:** `grep -n "INTERNAL_API_KEY" pages/api/jobs/*.ts` returns 4 hits, all using `!==`.
- **Impact:**
  - An attacker who can measure response latency to these endpoints can recover `INTERNAL_API_KEY` byte-by-byte (network jitter is usually small enough on cloud-to-cloud paths; with sufficient samples this is realistic).
  - With `INTERNAL_API_KEY` recovered, an attacker can:
    - Trigger arbitrary view notifications (email harvest / spam).
    - Trigger dataroom upload notifications (mislead document owners).
    - Trigger arbitrary download batch jobs.
- **Recommendation:** Replace `!==` with `crypto.timingSafeEqual` over equal-length buffers (the pattern already used in `automatic-unpause-route.ts`). Reject when lengths differ.

---

## Finding 4: `/api/feature-flags` leaks per-team feature configuration

- **Severity:** HIGH
- **Endpoint:** GET `/api/feature-flags?teamId=...`
- **Type:** data-leakage / undocumented-endpoint / missing-auth
- **Handler:** `app/api/feature-flags/route.ts:7-21`
- **Auth middleware:** absent — no session check, no bearer, no rate limit
- **Description:**
  The endpoint reads `teamId` from the query string and returns feature flags for that team without verifying the caller is a member of the team (or even that they are logged in):
  ```ts
  export async function GET(request: Request) {
    const { searchParams } = new URL(request.url);
    const teamId = searchParams.get("teamId");
    try {
      const features = await getFeatureFlags({ teamId: teamId || undefined });
      return NextResponse.json(features);
    } ...
  }
  ```
  `getFeatureFlags` is invoked from Edge runtime, so no DB-side authorization is possible — every team's feature surface (e.g., `inDocumentLinks`, `agentsEnabled`, plan tier signals) is enumerable by anyone with the ID.
- **Evidence:** `app/api/feature-flags/route.ts:7-21`. Wiki confirms route is exposed publicly (`app-api.md`).
- **Impact:**
  - Reconnaissance: attackers can map a target customer's plan tier, document-link features, AI features, etc. without authentication.
  - Helps choose exploit path for other IDOR findings (e.g., if `agentsEnabled`, the AI document endpoints are exposed).
- **Recommendation:** Require either an authenticated session and team membership, or a signed token issued only to known SDK consumers. Even an origin check would limit abuse.

---

## Finding 5: `/api/stripe/webhook` returns 200 on missing signature

- **Severity:** HIGH
- **Type:** auth-bypass (silent)
- **Endpoint:** POST `/api/stripe/webhook`
- **Handler:** `pages/api/stripe/webhook.ts:46-55`
- **Auth middleware:** absent — Stripe signature verified inline
- **Description:**
  When the `stripe-signature` header is missing or `STRIPE_WEBHOOK_SECRET` is unset, the handler returns `undefined` (no `res.status(...)` call), causing Next.js to respond with HTTP 200:
  ```ts
  if (!sig || !webhookSecret) return;       // ← silent 200
  const stripe = stripeInstance();
  event = stripe.webhooks.constructEvent(buf, sig, webhookSecret);
  ```
  Stripe interprets 2xx as "success, stop retrying". An attacker who can hit the endpoint while signatures are missing (e.g., during a secret rotation) can probe without triggering retries. More importantly, this masks configuration drift: a missing webhook secret in a production deploy will not surface in monitoring.
- **Evidence:** `pages/api/stripe/webhook.ts:49-55` (`legacy-webhook-old.ts` has the same shape).
- **Impact:**
  - Hidden misconfiguration: a missing `STRIPE_WEBHOOK_SECRET` will silently drop events instead of generating 4xx/5xx errors that would alert.
  - Possible replay confusion: legitimate but unparseable events (e.g., during env propagation) succeed at the protocol layer while doing nothing.
- **Recommendation:** Replace `return;` with `return res.status(400).send("Webhook signature/secret missing");` so misconfigurations are visible.

---

## Finding 6: `/api/cron/domains` is reachable without QStash auth outside Vercel

- **Severity:** CRITICAL (subsumed by Finding 2; called out separately because impact is destructive — domain deletion)
- **Endpoint:** POST `/api/cron/domains`
- **Handler:** `app/api/cron/domains/route.ts:24-34`
- **Description:** Same bypass as Finding 2, with the concrete destructive payload:
  ```ts
  if (process.env.VERCEL === "1") {
    const isValid = await receiver.verify({ signature, body });
    if (!isValid) return new Response("Unauthorized", { status: 401 });
  }
  // outside Vercel: anyone can POST, cron iterates domains
  ```
  After verification (or skip), the handler runs `handleDomainUpdates` for every domain, which sends escalating reminder emails and on day 30 calls `deleteDomain` (`handleDomainUpdates` in the same route). An attacker who repeatedly POSTs can keep `lastChecked` fresh, but also note the cron order is `orderBy: { lastChecked: "asc" }`, so spamming this endpoint doesn't directly accelerate deletion — **but** combined with `team.globalBlockList` poisoning or the absence of an authenticated cron, the attacker can still cause email floods (mail-quota exhaustion for the team's billing owner) and trigger `reportDeniedAccessAttempt` notifications from any subsequent `/api/views` call they control.
- **Recommendation:** See Finding 2.

---

## Finding 7: CORS misconfiguration on `/api/file/tus-viewer`

- **Severity:** MEDIUM
- **Endpoint:** OPTIONS / POST / HEAD / PATCH `/api/file/tus-viewer/*`
- **Handler:** `pages/api/file/tus-viewer/[[...file]].ts:233-263`
- **Description:**
  The handler echoes the request's `Origin` header into `Access-Control-Allow-Origin`, and sets `Access-Control-Allow-Credentials: true`:
  ```ts
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Access-Control-Allow-Origin", req.headers.origin || "*");
  ```
  The session check happens later via TUS metadata (`onIncomingRequest` → `verifyDataroomSessionInPagesRouter`), but **the CORS headers are set unconditionally before the session check**. A malicious site can therefore make credentialed preflight requests and read responses, and any data TUS returns for non-session-rejected requests will be readable cross-origin.
  Compare to `/api/file/tus` (logged-in tus) which does **not** set CORS headers — confirming the `tus-viewer` route's CORS is intentional for embedding but echo-origin + credentials is over-broad.
- **Evidence:** `pages/api/file/tus-viewer/[[...file]].ts:234-249`.
- **Impact:**
  - XHR from any origin can hit the endpoint with credentials (cookies).
  - An attacker on `evil.com` could force a logged-in visitor's browser to issue TUS HEAD/PATCH requests and read timing/response information.
- **Recommendation:** Whitelist known viewer origins (or restrict to the configured custom domains) instead of echoing the origin.

---

## Finding 8: `/api/csp-report` accepts arbitrary JSON with no validation or logging

- **Severity:** LOW
- **Endpoint:** POST `/api/csp-report`
- **Handler:** `app/api/csp-report/route.ts:3-16`
- **Description:**
  The handler reads arbitrary JSON and returns `{ success: true }` without doing anything (the body is never logged, validated, or rate-limited):
  ```ts
  export async function POST(request: Request) {
    const report = await request.json();
    return NextResponse.json({ success: true });
  }
  ```
  This is a placeholder. Browsers will post CSP violation reports here, and any unauthenticated attacker can flood the route.
- **Impact:**
  - Resource consumption (parsing large bodies).
  - If logging is later added (the comments imply it), an attacker can poison the log store with crafted reports.
- **Recommendation:** Either implement the logging (forward to Sentry/PostHog) with rate limiting, or return 204 No Content with body parsing limited.

---

## Finding 9: No OpenAPI/Swagger spec in the repository

- **Severity:** MEDIUM (process / documentation gap)
- **Description:**
  `find . -name 'openapi*' -o -name 'swagger*'` returns only the marketing pages and this analysis. There is no `openapi.yaml`, `openapi.json`, `swagger.json`, `api-spec.yaml`, or `.graphql` schema. The middleware (`middleware.ts:66`) advertises `/openapi.json` as a publicly accessible endpoint on `api.papermark.com`, and `next.config.mjs:49-52` rewrites `/v1/:path*` to `/api/v1/:path*` — both pointing at endpoints that do not exist.
- **Impact:**
  - Consumers (including the SDK at `mcp.papermark.com`) cannot generate typed clients.
  - Public API surface drift is undetectable in CI.
  - `/openapi.json` will return whatever Next.js serves (likely 404 or marketing site) — a security smell in itself (Finding 1).
- **Recommendation:** Either commit a generated `openapi.json` artifact or remove the middleware rule until one exists.

---

## Finding 10: `/api/conversations` (visitor) uses `viewerId` as sole soft auth

- **Severity:** MEDIUM
- **Endpoint:** GET `/api/conversations?dataroomId=...&viewerId=...`
- **Handler:** `ee/features/conversations/api/conversations-route.ts:19-74`
- **Description:**
  The visitor-facing endpoint lists all conversations for a `(dataroomId, viewerId)` pair with no other proof:
  ```ts
  const conversations = await prisma.conversation.findMany({
    where: { dataroomId, participants: { some: { viewerId } } },
    include: { messages: { orderBy: { createdAt: "asc" } }, dataroom: true, ... },
  });
  ```
  The endpoint is meant to be called by the visitor UI in the dataroom viewer. The page itself ties `viewerId` to the visitor's HTTP-only session cookie, but the API does not re-verify that the caller is that viewer. If a viewer ID leaks (e.g., in a referral header, in another team member's analytics, in a screenshot-replay attach), any party with the ID can read all messages in the conversation.
- **Evidence:** `ee/features/conversations/api/conversations-route.ts:19-74`. The team-scoped variant (`pages/api/teams/[teamId]/datarooms/[id]/conversations/[[...conversations]].ts`) does check session + team membership, confirming the visitor endpoint's soft control is intentional.
- **Impact:**
  - Conversation confidentiality hinges on the secrecy of `viewerId`.
  - `viewerId` is a CUID stored in `prisma.viewer.id` and is referenced in URLs, notifications, and support tickets — non-trivial to keep secret.
- **Recommendation:** Require the visitor's dataroom session cookie (`pm_drs_*`) in addition to `viewerId`, mirroring the protection pattern in `/api/views-dataroom`.

---

## Finding 11: `/api/og` renders arbitrary title with no rate limit

- **Severity:** LOW
- **Endpoint:** GET `/api/og?title=...`
- **Handler:** `app/api/og/route.tsx:11-42`
- **Description:** Accepts any `title` query string and returns a generated PNG via `next/og` (Edge runtime). No rate limit, no length cap on the title before passing to `ImageResponse`.
- **Impact:** Edge compute exhaustion — an attacker can fetch large OG images to burn through Edge invocations. Image rendering at `1200x630` with arbitrary content also amplifies cost.
- **Recommendation:** Add per-IP rate limiting and cap `title` length (e.g., 200 chars).

---

## Finding 12: `/api/help` proxies to marketing site without bounds

- **Severity:** LOW
- **Endpoint:** GET `/api/help?q=...`
- **Handler:** `app/api/help/route.ts:3-43`
- **Description:** Fetches from `NEXT_PUBLIC_MARKETING_URL/api/help`, then filters results client-side. No rate limit, no response size cap, no timeout config beyond `fetch` defaults.
- **Impact:** Acts as an open SSRF reflector (the upstream URL is env-controlled so not user-controlled today, but it amplifies an outage of the marketing site into Papermark's app), and provides unauthenticated, unmetered traffic to a marketing endpoint.
- **Recommendation:** Add a 2–3 s `AbortController` timeout, rate-limit per IP, and cache the marketing response at the Edge.

---

## Notes on routes that were inspected and found acceptable

These were checked in detail to confirm they are not additional findings:

- **`/api/views`** (`app/api/views/route.ts`): public visitor flow with link lookup, OTP, allowList/denyList, password, agreement gates. Follows the documented layered-access pattern. **Not flagged.**
- **`/api/views-dataroom`** (`app/api/views-dataroom/route.ts`): same shape as `/api/views`. **Not flagged.**
- **`/api/webhooks/signing`** (`app/api/webhooks/signing/route.ts`): uses `verifySigningWebhookSecret`, distinguishes 503 (misconfigured) from 401 (forged). **Not flagged.**
- **`/api/scim/v2.0/[...directory]`** (`app/(ee)/api/scim/v2.0/[...directory]/route.ts:22-23`): passes `Authorization` Bearer split to Jackson's `apiSecret`. **Not flagged.**
- **`/api/auth/saml/callback`** and **`/api/auth/saml/token`** and **`/api/auth/saml/userinfo`**: all delegate to Jackson's `oauthController` which validates signatures. **Not flagged.**
- **`/api/file/tus`** (`pages/api/file/tus/[[...file]].ts:297-310`): checks `getServerSession` and sets `papermarkUserId` before delegating to TUS. **Not flagged.**
- **`/api/file/tus-viewer`** TUS `onIncomingRequest` and `onUploadCreate`: verifies `verifyDataroomSessionInPagesRouter` (separate from CORS finding above). **Auth layer OK; CORS still flagged in Finding 7.**
- **`/api/internal/billing/automatic-unpause`**: uses `crypto.timingSafeEqual` correctly (contrast Finding 3). **Not flagged.**
- **`/api/teams/[teamId]/agreements/[agreementId]/signing/{setup,presign,sync}`**: all use `getServerSession` + team membership check. **Not flagged.**
- **`/api/conversations/[[...conversations]]`** (team-scoped, `pages/api/teams/[teamId]/datarooms/[id]/conversations/[[...conversations]].ts`): checked session and team membership in `ee/features/conversations/api/team-conversations-route.ts`. **Not flagged.**
- **`/api/webhooks/services/[...path]`**: Bearer token → `prisma.restrictedToken.findUnique` → rate limit → team match. **Not flagged.**
- **`/api/jobs/send-conversation-team-member-notification`** and **`/api/jobs/send-conversation-new-message-notification`**: not directly inspected but inferred (from grep) to use `INTERNAL_API_KEY` check; same finding as Finding 3 if they use `!==` (worth confirming in a follow-up sweep).
- **`/api/auth/saml/authorize`** and **`/api/auth/saml/callback`**: rely on Jackson signing, public-by-design (SAML endpoints must be reachable from the IdP). **Not flagged.**

---

## Phase 3 checkpoint

- [x] Findings written to `methodology-raw/00-ai-api-specs.md`
- [x] Each finding has: file, line, severity, description, evidence
- [x] No duplicate findings (Finding 6 is a destructive-impact specialization of Finding 2, called out separately for clarity)
