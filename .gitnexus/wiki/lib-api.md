# lib — api

# lib/api Module

Server-side API utilities, data fetching, and access control for Papermark. This module centralizes shared logic used by Next.js route handlers, including authentication helpers, RBAC enforcement, link data fetching, domain management, and webhook dispatch.

---

## Architecture Overview

The module is organized into functional domains:

```
lib/api/
├── auth/                     # Authentication utilities
│   ├── passkey.ts            # Passkey lifecycle (register, list, remove)
│   ├── token.ts              # Token hashing
│   └── with-session-team.ts  # Unified session + team context wrapper
├── domains/                  # Custom domain management
│   ├── index.ts              # Domain CRUD helpers
│   ├── clear-team-redirects.ts
│   ├── redis.ts              # Redirect URL caching
│   └── validate-redirect-url.ts
├── documents/
│   └── process-document.ts   # Document upload pipeline
├── links/
│   ├── bulk-import.ts        # CSV-based bulk link creation
│   ├── link-data.ts          # Link data fetching for SSR/ISR
│   └── revalidate.ts         # ISR revalidation triggers
├── rbac/                      # Role-based access control
│   ├── permissions.ts        # Permission verb definitions
│   ├── entitlements.ts        # Dataroom-scoped access
│   └── guard.ts               # Inline access guards
├── views/
│   └── send-webhook-event.ts  # View event webhook dispatch
├── teams/
│   └── is-saml-enforced-for-email-domain.ts
└── notification-helper.ts     # Notification job dispatch
```

---

## Role-Based Access Control (RBAC)

The RBAC layer enforces what users can do (`permissions`) and which resources they can access (`entitlements`).

### Permission Verbs (`permissions.ts`)

`PermissionAction` defines granular verbs that routes declare as requirements:

```typescript
export type PermissionAction =
  | "datarooms.read" | "datarooms.write"
  | "documents.read" | "documents.write"
  | "links.read" | "links.write"
  | "analytics.read" | "analytics.team"
  | "team.read" | "team.write"
  | "members.write" | "tokens.write"
  | "webhooks.write" | "domains.write"
  | "branding.write" | "sso.write";
```

Role → permission set mapping:

| Role | Permissions |
|------|-------------|
| `ADMIN` | All verbs |
| `MANAGER` | All except `members.write`, `sso.write` |
| `MEMBER` | Broad access except admin-only structural ops |
| `DATAROOM_MEMBER` | Minimal set scoped to assigned datarooms only |

```typescript
// Check if a role has required permissions
const hasAccess = hasAllPermissions(role, ["links.read", "links.write"]);

// Identify the scoped role (requires per-room entitlement check)
const isScoped = isDataroomScopedRole(role); // true only for DATAROOM_MEMBER
```

### Dataroom Entitlements (`entitlements.ts`)

Complements the verb layer with the "which rooms" check for `DATAROOM_MEMBER`:

```typescript
// Get rooms a user is assigned to within a team
const roomIds = await getAllowedDataroomIds(userId, teamId);

// Does this role/room combination allow access?
canAccessDataroom(role, allowedIds, dataroomId);     // general access
canManageDataroom(role, allowedIds, dataroomId);     // uploads, links, folders

// Assert access for a specific document or link
assertDocumentAccess({ role, userId, teamId, documentId, allowedIds });
assertLinkAccess({ role, userId, teamId, linkId, allowedIds });
```

### Inline Guards (`guard.ts`)

For routes not yet migrated to `withTeam`, these lightweight guards close IDOR gaps for `DATAROOM_MEMBER`:

```typescript
// Team-level route — scoped members are denied outright
const denied = await enforceDataroomMemberScope({ userId, teamId, res });
if (denied) return;

// Room-level route — verify assignment
const denied = await enforceDataroomMemberScope({ userId, teamId, dataroomId, res });
if (denied) return;

// Link-scoped check
const denied = await enforceLinkMemberScope({ userId, teamId, linkId, res });

// Document-scoped check
const denied = await enforceDocumentMemberScope({ userId, teamId, documentId, res });
```

---

## Session + Team Context Wrapper (`with-session-team.ts`)

Centralizes membership checks, role verification, permission gating, plan checks, and dataroom entitlement enforcement for both App Router and Pages Router handlers.

### Usage (App Router)

```typescript
import { withTeam } from "@/lib/api/auth/with-session-team";

export const POST = withTeam(
  async (ctx) => {
    // ctx includes: userId, teamId, role, permissions, membership, team, dataroomId
    return NextResponse.json({ success: true });
  },
  {
    requiredPermissions: ["documents.write"],
    requiredRoles: ["ADMIN", "MANAGER"],
    requiredPlan: (plan) => plan !== "free",
    dataroomParam: "dataroomId",
  },
);
```

### Usage (Pages Router)

```typescript
import { withTeamApi } from "@/lib/api/auth/with-session-team";

export default withTeamApi(
  async (req, res, ctx) => {
    // ctx same as App Router, plus req/res
    res.json(ctx.team);
  },
  {
    requiredPermissions: ["team.read"],
  },
);
```

### Resolution Flow

```
getServerSession()
        │
        ▼
resolveSessionTeam()
        │
        ├── Auth check ──────────────────────────── 401
        │
        ├── TeamId check ────────────────────────── 400
        │
        ├── Membership lookup ───────────────────── 401
        │
        ├── Active status check ────────────────── 403
        │
        ├── Role gate (optional) ───────────────── 403
        │
        ├── Permission gate ─────────────────────── 403
        │
        ├── Default-deny for scoped role ───────── 403 (no requiredPermissions)
        │
        ├── Plan gate (optional) ───────────────── 403
        │
        └── Per-room entitlement (scoped only)
                │
                ├── Resolve room id from params
                │
                └── canAccessDataroom() ────────── 403
```

The `SessionTeamContext` returned on success:

```typescript
interface SessionTeamContext {
  userId: string;
  teamId: string;
  membership: { role, status, blockedAt };
  role: Role;
  permissions: Set<PermissionAction>;
  allowedDataroomIds: string[];   // scoped members only
  dataroomId?: string;             // resolved from params when configured
  team: { id: string; plan: string };
}
```

---

## Link Data Fetching (`links/link-data.ts`)

Fetches and processes link data for Next.js `getStaticProps` and route handlers. Avoids internal HTTP fetches that can fail at the Vercel edge.

### Entry Points

```typescript
// By link ID (for /view/[linkId] routes)
const result = await fetchLinkDataById({ linkId, dataroomDocumentId? });

// By domain + slug (for custom domain links)
const result = await fetchLinkDataByDomainSlug({ domain, slug, dataroomDocumentId? });
```

### Return Type

```typescript
type LinkFetchResult =
  | { status: "ok"; linkType: LinkType; link: LinkRecord; brand: Brand | null; ... }
  | { status: "not_found" | "archived" | "deleted" | "free" | "frozen" };
```

### Link Types Handled

| Link Type | Data Fetched |
|-----------|-------------|
| `DOCUMENT_LINK` | Document, versions, brand, team plan |
| `DATAROOM_LINK` | Dataroom, documents, folders, access controls, brand |
| `DATAROOM_LINK` + `dataroomDocumentId` | Specific document within dataroom |
| `WORKFLOW_LINK` | Minimal — team brand only |

### Brand Cascading

Dataroom-level branding overrides team-level defaults:

```typescript
const brand = {
  logo: dataroomBrand?.logo || teamBrand?.logo,
  brandColor: dataroomBrand?.brandColor || teamBrand?.brandColor,
  // ... all fields cascade in the same pattern
  defaultLanguage: dataroomBrand?.defaultLanguage ?? teamBrand?.defaultLanguage ?? "en",
};
```

---

## Document Processing Pipeline (`documents/process-document.ts`)

Handles document upload and triggers downstream conversion jobs.

```typescript
const document = await processDocument({
  documentData,
  teamId,
  teamPlan,
  userId,
  folderPathName,
  folderId,
  createLink: true,
  isExternalUpload: false,
});
```

### Processing Steps

1. **Type detection** — Derives type from file extension or explicit `supportedFileType`
2. **Notion validation** — For `notion` type: verifies page is publicly accessible
3. **Link URL validation** — For `link` type: validates URL format, checks against keyword blocklist for non-trusted teams
4. **Folder resolution** — Resolves `folderId` from path or explicit parameter
5. **Download-only determination** — Marks CAD, CAD-adjacent, and special formats as non-viewable
6. **Database write** — Creates `Document` and `DocumentVersion` records
7. **Conversion triggers** — Queues background jobs based on type:

| Type | Trigger |
|------|---------|
| Keynote (`.key`, `application/vnd.apple.keynote`) | `convertKeynoteToPdfTask` |
| Docs/Slides (non-Keynote) | `convertFilesToPdfTask` |
| Non-MP4 video | `processVideo` |
| PDF | `convertPdfToImageRoute` |
| Excel with advanced mode | File copy + ISR revalidation |

8. **Webhook dispatch** — `document.created` and optionally `link.created`

---

## Bulk Link Import (`links/bulk-import.ts`)

Handles batch link creation via API (CSV importer or incoming webhook integration).

```typescript
await handleBulkLinkImport(req, res, {
  teamId,
  targetId,        // document or dataroom ID
  linkType: "DOCUMENT_LINK" | "DATAROOM_LINK",
});
```

### Request Schema

```typescript
{
  links: [
    {
      name?: string;
      domain?: string;       // custom domain
      slug?: string;         // link path
      password?: string;
      expiresAt?: string;     // ISO-8601
      emailProtected?: boolean;
      allowDownload?: boolean;
      enableNotification?: boolean;
      enableScreenshotProtection?: boolean;
      allowList?: string[];
      denyList?: string[];
      presetId?: string;
      // ... more fields
    }
  ]
}
```

### Per-Row Processing

- Domain + slug must be provided together
- Presets resolved once per request and cached
- Domain lookups cached per team
- Plan link limit tracked per-request (rows beyond cap return per-row errors)
- Frozen datarooms reject all row processing
- `waitUntil` used for webhook dispatch to avoid blocking

### Response

```typescript
{
  summary: { total, success, failed },
  results: [
    { row: 1, status: "success", linkId, linkUrl },
    { row: 2, status: "error", error: "Link limit reached" },
  ]
}
```

---

## Domain Management (`domains/`)

### Redis-Backed Redirects (`domains/redis.ts`)

Caches redirect URLs in Redis for fast middleware lookups:

```typescript
// Check if plan supports custom redirects
planSupportsRedirects("business"); // true

// Get cached redirect
const url = await getDomainRedirectUrl("docs.example.com");

// Set or clear redirect
await setDomainRedirectUrl("docs.example.com", "https://target.com");
```

### Clearing Redirects (`domains/clear-team-redirects.ts`)

Called when a team is deleted or downgrades from a redirect-eligible plan:

```typescript
await clearTeamDomainRedirects(teamId);
```

Removes redirect data from both Prisma and Redis.

### Redirect URL Validation (`domains/validate-redirect-url.ts`)

Validates URLs before saving:

- Must use HTTPS
- URL security validation (no `javascript:`, etc.)
- Keyword blocklist check (non-trusted teams only)

```typescript
const result = await validateRedirectUrl(url, teamId);
if (!result.valid) {
  return res.status(400).json({ error: result.message });
}
```

### Domain Deletion (`domains.ts`)

Handles domain cleanup including Vercel DNS removal:

```typescript
await deleteDomain("example.com");
await deleteDomain("sub.example.com", { skipPrismaDelete: true }); // Vercel only
```

---

## Passkey Authentication (`auth/passkey.ts`)

Integrates with Hanko for WebAuthn passkey management.

### Registration Flow

```typescript
// Step 1: Client requests options to begin registration
const options = await startServerPasskeyRegistration({ session });

// Step 2: After client completes registration, finalize on server
await finishServerPasskeyRegistration({ credential, session });
```

### Passkey Management

```typescript
// List user's registered passkeys
const passkeys = await listUserPasskeys({ session });

// Remove a specific passkey
await removeUserPasskey({ credentialId, session });
```

`removeUserPasskey` verifies ownership by checking the credential belongs to the user before deletion.

---

## View Webhook Dispatch (`views/send-webhook-event.ts`)

Sends `link.viewed` webhooks when a link is accessed:

```typescript
await sendLinkViewWebhook({ teamId, clickData });
```

### Checks Before Dispatch

- Team must be on a webhook-eligible plan (not `free` or `pro`)
- Team must not be paused
- At least one webhook must be configured with `link.viewed` trigger

### Payload Structure

```typescript
{
  view: { viewedAt, viewId, email, emailVerified, country, city, device, ... },
  link: { id, url, domain, key, name, expiresAt, permissions, ... },
  document?: { id, name, contentType, teamId, createdAt },
  dataroom?: { id, name, teamId, createdAt },
}
```

---

## SAML Enforcement (`teams/is-saml-enforced-for-email-domain.ts`)

Checks whether a user must authenticate via SSO based on their email domain:

```typescript
const enforced = await isSamlEnforcedForEmailDomain(userEmail);
if (enforced) {
  // Block email/password, Google OAuth, magic links
}
```

Skips generic email providers (gmail.com, yahoo.com, etc.).

---

## ISR Revalidation (`links/revalidate.ts`)

Triggers on-demand revalidation for links when content changes:

```typescript
// Single link
await revalidateLinkById(linkId);

// All links using a permission group (after group changes)
await revalidateLinksForPermissionGroup(permissionGroupId);
```

Used after creating, updating, or deleting permission groups to ensure link views reflect current access controls.

---

## Notifications (`notification-helper.ts`)

Dispatches background jobs for view notifications and viewer invitations:

```typescript
// View notification with location data
await sendNotification({ viewId, locationData });

// Dataroom viewer invitation
await sendViewerInvitation({ dataroomId, linkId, viewerIds, senderUserId });
```

Uses `waitUntil` internally to avoid blocking the request handler.

---

## Token Hashing (`auth/token.ts`)

SHA-256 hashing for opaque tokens (e.g., email change tokens):

```typescript
const hash = hashToken(rawToken);
const hashNoSecret = hashToken(rawToken, { noSecret: true });
```

---

## Key Dependencies

| Dependency | Used By |
|------------|---------|
| `prisma` | All modules — database access |
| `next-auth` (`getServerSession`) | `with-session-team.ts` |
| `@vercel/redis` | `domains/redis.ts` |
| `@vercel/blob` | `bulk-import.ts` (preset image upload) |
| `@vercel/edge-config` | `process-document.ts`, `validate-redirect-url.ts` |
| `@prisma/client` (`Role`, `ItemType`) | RBAC, link types |
| `lib/trigger/*` | Document conversion queueing |
| `lib/webhook/send-webhooks` | View webhook dispatch |