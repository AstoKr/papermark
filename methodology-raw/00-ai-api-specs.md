# API Spec & Auth-Coverage Recon — papermark

## Summary

| Finding | Severity |
|---------|----------|
| Conversations API — no auth on any handler | **CRITICAL** |
| `progress-token.ts` generates access tokens without auth | **HIGH** |
| `record_reaction.ts` creates DB records without auth | **MEDIUM** |
| `feedback/index.ts` records responses without auth | **MEDIUM** |
| `revalidate.ts` uses shared secret in query param | **MEDIUM** |
| No API spec exists (OpenAPI/Swagger = absent) | **LOW** |
| GraphQL not used | — |
| IDOR: viewId-based endpoints accept unvalidated IDs | **MEDIUM** |

**API Spec Status:** No `swagger.json`, `openapi.yaml`, `*.graphql`, or any formal API specification file was found anywhere in the repository. The only documentation is the wiki in `.gitnexus/wiki/`. The wiki covers general route categories but is not a complete endpoint inventory. Total actual route files: ~240 across `app/api/` and `pages/api/`.

**Wiki Coverage Gaps (undocumented route groups in wiki):**
- `/api/conversations/**` — multiple handlers
- `/api/teams/[teamId]/incoming-webhooks/**` — incoming webhook management
- `/api/teams/[teamId]/tokens/**` — API token management
- `/api/teams/[teamId]/presets/**` — link presets
- `/api/teams/[teamId]/visitor-groups/**` — visitor groups
- `/api/teams/[teamId]/export-jobs/**` — export jobs
- `/api/teams/[teamId]/viewers/**` — viewer management
- `/api/teams/[teamId]/integrations/slack/**` — Slack integration settings
- `/api/teams/[teamId]/survey` — survey endpoint
- `/api/teams/[teamId]/yearly-recap` — yearly recap settings
- `/api/teams/[teamId]/notifications/**` — notification preferences
- `/api/teams/[teamId]/ignored-domains` — ignored domain settings
- `/api/teams/[teamId]/global-block-list` — block list management
- `/api/teams/[teamId]/folder-documents/**` — folder documents
- `/api/teams/[teamId]/tags/**` — tag management
- `/api/mupdf/**` — PDF annotation endpoints
- `/api/jobs/**` — background job endpoints
- `/api/unsubscribe/**` — unsubscribe endpoints
- `/api/notification-preferences/**` — notification preferences

---

### Conversations API — missing auth on all handlers

- severity: CRITICAL
- endpoint: `GET /api/conversations`, `POST /api/conversations`, `POST /api/conversations/messages`, `POST /api/conversations/notifications`
- type: auth-missing
- handler: `handleRoute` → `conversations-route.ts` @ `ee/features/conversations/api/conversations-route.ts`
- auth middleware: absent
- description: The conversations route mounted at `/api/conversations/[[...conversations]].ts` delegates to `ee/features/conversations/api/conversations-route.ts` which has **four handlers (GET list, POST create, POST messages, POST notifications) — none of which perform any authentication**. There is no `getServerSession()` call, no token check, no API key validation. The `viewerId` parameter is taken on trust from query parameters or request body with zero server-side verification that the caller owns that viewer identity. Adjacent files in the same directory (`team-conversations-route.ts`, `toggle-conversations-route.ts`) demonstrate the correct pattern (`getServerSession(req, res, authOptions)` on every handler), but this route file was missed.
- evidence:
  - File: `ee/features/conversations/api/conversations-route.ts` — no imports from `next-auth`, no `getServerSession` call
  - Entry: `pages/api/conversations/[[...conversations]].ts` — thin passthrough, no auth
  - The handler accepts `viewerId`, `viewId`, `dataroomId` from request body/query params without validation
  - Contrast: `ee/features/conversations/api/team-conversations-route.ts` — every handler starts with:
    ```typescript
    const session = await getServerSession(req, res, authOptions);
    if (!session) { return res.status(401).json({ error: "Unauthorized" }); }
    ```

---

### progress-token.ts — access token generation without auth

- severity: HIGH
- endpoint: `GET /api/progress-token?documentVersionId=xxx`
- type: auth-missing
- handler: `handle` @ `pages/api/progress-token.ts:5`
- auth middleware: absent
- description: This endpoint generates a Trigger.dev public access token for any `documentVersionId` passed as a query parameter. No authentication or authorization check is performed. Anyone who can discover or enumerate a valid `documentVersionId` (a CUID) can obtain a public access token for that document version. While CUIDs are not trivially guessable, they can be leaked through client-side code, referrer headers, or logs.
- evidence:
  ```typescript
  // pages/api/progress-token.ts:6-27
  export default async function handle(req, res) {
    if (req.method !== "GET") { return res.status(405)... }
    const { documentVersionId } = req.query;
    if (!documentVersionId || typeof documentVersionId !== "string") { ... }
    const publicAccessToken = await generateTriggerPublicAccessToken(
      `version:${documentVersionId}`
    );
    return res.status(200).json({ publicAccessToken });
  }
  ```

---

### record_reaction.ts — creates DB records without auth

- severity: MEDIUM
- endpoint: `POST /api/record_reaction`
- type: auth-missing
- handler: `handle` @ `pages/api/record_reaction.ts:6`
- auth middleware: absent
- description: This endpoint accepts `viewId`, `pageNumber`, and `type` from the request body and creates a `Reaction` record in the database. It only validates that `viewId` references an existing view (line 24-33). There is no authentication or session validation — any unauthenticated client can inject arbitrary reaction data tied to any known `viewId`. The endpoint also returns the `view` relation including `documentId`, `dataroomId`, `linkId`, `viewerEmail`, `viewerId`, and `teamId` in the creation response (the `include` on line 30-41), leaking protected viewer metadata.
- evidence:
  ```typescript
  // pages/api/record_reaction.ts:23-42
  const reaction = await prisma.reaction.create({
    data: { viewId, pageNumber, type },
    include: {
      view: { select: { documentId, dataroomId, linkId, viewerEmail, viewerId, teamId } }
    },
  });
  ```

---

### feedback/index.ts — records feedback responses without auth

- severity: MEDIUM
- endpoint: `POST /api/feedback`
- type: auth-missing
- handler: `handle` @ `pages/api/feedback/index.ts:5`
- auth middleware: absent
- description: This endpoint records feedback responses in the database. It validates that the `feedbackId` and `viewId` exist, but performs no authentication or session check. Any unauthenticated client with knowledge of valid IDs can submit arbitrary feedback responses.
- evidence:
  ```typescript
  // pages/api/feedback/index.ts:9-56
  // No getServerSession, no token check, no auth of any kind
  const { answer, feedbackId, viewId } = req.body;
  ```

---

### revalidate.ts — shared secret in query parameter

- severity: MEDIUM
- endpoint: `GET /api/revalidate?secret=xxx&linkId=yyy`
- type: auth-bypass
- handler: `handler` @ `pages/api/revalidate.ts:8`
- auth middleware: shared secret via query param (bypassable)
- description: The revalidation endpoint authenticates via `req.query.secret !== process.env.REVALIDATE_TOKEN`. Passing the shared secret as a URL query parameter exposes it to leakage through: (1) server access logs, (2) `Referer` header when the page loads external resources, (3) browser history if triggered from client-side navigation. The endpoint can revalidate arbitrary view pages and enumerate all links for a team or document. Additionally, if the `REVALIDATE_TOKEN` is ever exposed, an attacker can force mass revalidation of all team links, potentially causing denial-of-service via cache stampede.
- evidence:
  ```typescript
  // pages/api/revalidate.ts:13-14
  if (req.query.secret !== process.env.REVALIDATE_TOKEN) {
    return res.status(401).json({ message: "Invalid token" });
  }
  ```

---

### IDOR: viewId-based public endpoints accept unvalidated IDs

- severity: MEDIUM
- endpoint: `POST /api/record_view`, `POST /api/record_click`, `POST /api/record_video_view`, `POST /api/record_reaction`, `POST /api/feedback`
- type: idor
- handler: multiple files
- auth middleware: absent (by design for public tracking)
- description: Multiple public tracking endpoints accept `viewId`, `linkId`, and `documentId` from the request body without verifying that the caller is authorized to submit data for those IDs. While these endpoints are intentionally public (they track visitor behavior on shared document links), the `record_reaction` endpoint goes further by **creating database records** rather than just publishing analytics events. The `record_view` and `record_click` endpoints publish to Tinybird (analytics pipeline), but `record_reaction` writes directly to the PostgreSQL `Reaction` table, making it a data-injection vector. Additionally, `record_view` accepts `dataroomId`, `versionNumber`, `pageNumber` from the request body without verifying they correspond to the `linkId`/`documentId` pair — enabling analytics pollution by submitting arbitrary metadata.
- evidence:
  - `record_view.ts:59-66` — destructures body fields without server-side consistency verification against link/dataroom
  - `record_click.ts:30-49` — all IDs come from request body, no verification
  - `record_reaction.ts:17-21` — all IDs from body, creates Prisma records

---

### CORS misconfiguration on TUS viewer upload (duplicated from SAST)

- severity: MEDIUM
- endpoint: `OPTIONS, POST, GET, DELETE, PATCH, HEAD /api/file/tus-viewer/[[...file]]`
- type: auth-bypass
- handler: `setCorsHeaders` @ `pages/api/file/tus-viewer/[[...file]].ts:234`
- auth middleware: present (dataroom session), but CORS allows arbitrary origins
- description: The TUS viewer upload endpoint reflects the `Origin` request header verbatim into `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`. This enables CSRF-style attacks against the dataroom session cookie. Already documented in SAST output; included here for completeness of the API-surface view.

---

### No API specification / documentation gap

- severity: LOW
- type: undocumented-endpoint
- description: No formal API specification (OpenAPI/Swagger/GraphQL schema) exists. The wiki provides only partial documentation covering roughly 60% of active route files. This creates an undocumentation tax where security review must rely entirely on code audit.

---

### GraphQL

Not used. No `.graphql` files, no Apollo/GraphQL dependencies, no GraphQL endpoint found. No introspection, query-depth, or field-suggestion concerns apply.

---

## Auth Coverage Heatmap

| Route area | Auth pattern | Coverage |
|-----------|-------------|----------|
| `app/api/` (views, auth, cron, webhooks) | Link/dataroom sessions + QStash sigs + NextAuth | Complete |
| `app/(ee)/api/` (AI, SAML, SCIM, freeze, workflows, uploads) | NextAuth + dataroom sessions + API keys | Complete |
| `pages/api/teams/[teamId]/**` | NextAuth `getServerSession` | Full |
| `pages/api/account/**` | NextAuth `getServerSession` | Full |
| `pages/api/links/**` | Link/dataroom sessions | Full |
| `pages/api/agreements/**` | Signed access tokens | Full |
| `pages/api/file/tus/**` | NextAuth + team membership | Full |
| `pages/api/file/tus-viewer/**` | Dataroom sessions | Present |
| `pages/api/file/s3/**` | NextAuth / INTERNAL_API_KEY | Full |
| `pages/api/file/browser-upload.ts` | NextAuth | Full |
| `pages/api/file/image-upload.ts` | NextAuth | Full |
| `pages/api/file/notion/**` | TBD | Unknown |
| `pages/api/stripe/**` | Stripe webhook sigs | Full |
| `pages/api/analytics/index.ts` | NextAuth | Full |
| `pages/api/mupdf/**` | INTERNAL_API_KEY (Bearer) | Full |
| `pages/api/jobs/**` | Trigger.dev context (internal) | N/A |
| `pages/api/internal/**` | INTERNAL_API_KEY (Bearer) | Full |
| `pages/api/health.ts` | None (intentional) | N/A |
| **`pages/api/conversations/**`** | **NONE** | **MISSING** |
| **`pages/api/progress-token.ts`** | **NONE** | **MISSING** |
| **`pages/api/record_reaction.ts`** | **NONE** | **MISSING** |
| **`pages/api/feedback/index.ts`** | **NONE** | **MISSING** |
| `pages/api/record_view.ts` | None (by design, public tracking) | N/A |
| `pages/api/record_click.ts` | None (by design, public tracking) | N/A |
| `pages/api/record_video_view.ts` | None (by design, public tracking) | N/A |
| `pages/api/report.ts` | None (by design, public reporting) | N/A |
