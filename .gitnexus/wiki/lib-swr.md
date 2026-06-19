# lib — swr

# lib/swr — SWR Data Fetching Hooks

This module contains React hooks that wrap [SWR](https://swr.vercel.app/) for data fetching from the application's API routes. They provide a type-safe, consistent interface for consuming server data across the Next.js frontend.

## Architecture Overview

All hooks in this module follow a consistent pattern: they retrieve the current team ID from the `TeamContext`, construct an API endpoint, and delegate fetching to SWR. This creates a uniform data-fetching layer where every API call is automatically deduplicated, cached, and revalidated according to sensible defaults.

```mermaid
flowchart TD
    subgraph "React Components"
        C1[UI Components]
        C2[Pages]
    end

    subgraph "lib/swr"
        H1[useDocument]
        H2[useDataroom]
        H3[usePlan]
        H4[useViewers]
        H5[...other hooks]
    end

    subgraph "Dependencies"
        TC[TeamContext]
        F[fetcher utility]
        R[Next Router]
    end

    subgraph "API Routes"
        A1[/api/teams/:teamId/documents]
        A2[/api/teams/:teamId/datarooms]
        A3[/api/teams/:teamId/billing/plan]
        A4[...other endpoints]
    end

    C1 & C2 --> H1 & H2 & H3 & H4 & H5
    H1 & H2 & H3 & H4 & H5 --> TC
    H1 & H2 & H3 & H4 & H5 --> F
    H1 & H2 & H3 & H4 & H5 --> R
    H1 --> A1
    H2 --> A2
    H3 --> A3
    H5 --> A4
```

## Core Concepts

### Team-Scoped Fetching

Every hook that fetches team-level data requires a `teamId`. The hooks obtain this from `useTeam` in `context/team-context.tsx`. If no team is selected, requests are skipped entirely by passing `null` as the URL:

```typescript
const { data } = useSWR<Document>(
  teamId && `/api/teams/${teamId}/documents/${id}`,
  fetcher
);
```

### Return Shape Convention

Hooks return a consistent object shape with `loading`, `error`, and often `mutate` for programmatic revalidation:

```typescript
return {
  document: data,
  loading: !error && !data,
  error,
  mutate,
};
```

### SWR Configuration Patterns

Different data types require different caching strategies. The module defines three main patterns:

| Pattern | `dedupingInterval` | `revalidateOnFocus` | Use Case |
|---------|-------------------|---------------------|----------|
| Standard | 10–30s | `false` | Most data |
| Static/Rarely changing | 60s | `false` | Teams, billing plans |
| Fast polling | 3s (refreshInterval) | `false` | Processing status |
| Keep previous | 20–30s | `false` + `keepPreviousData` | Paginated lists |

## Document Hooks

### `useDocument`

Fetches a single document by ID from the URL query parameter `id`. Uses aggressive caching because document data changes infrequently after initial load.

```typescript
const { document, primaryVersion, loading, error, mutate } = useDocument();
```

### `useDocuments`

Fetches the document list with support for search, sorting, and pagination. Passes query parameters to the API:

```typescript
const { documents, searchFolders, pagination, isValidating, loading } = useDocuments();
// Returns: documents, optional folders when searching, pagination metadata
```

### `useDocumentOverview`

Returns an aggregated response containing the document, user limits, feature flags, and view/link counts in a single request:

```typescript
const { document, limits, featureFlags, team, counts } = useDocumentOverview();
```

### `useDocumentVisits`

Fetches paginated view data for a document with optional dataroom scoping:

```typescript
const { views, loading, error, mutate } = useDocumentVisits(
  page, 
  limit,
  documentId,  // optional override
  { dataroomId, scope }  // optional: 'dataroom' | 'other'
);
```

### `useDocumentStats`

Fetches aggregated analytics for a document, including views, average completion rate, and duration data:

```typescript
const { stats, loading, error } = useDocumentStats(documentId);
```

## Dataroom Hooks

### `useDataroom`

Core hook for fetching dataroom metadata. Includes 404 handling that redirects to the datarooms list:

```typescript
const { dataroom, loading, error } = useDataroom(dataroomId?);
```

### `useDatarooms`

Fetches the datarooms list with support for filtering by search query, status, and tags:

```typescript
const { datarooms, totalCount, loading, mutate } = useDatarooms();
```

### `useDataroomItems`

Combines folders and documents into a single sorted list for displaying a dataroom folder's contents:

```typescript
const { items, folderCount, documentCount, isLoading, error } = useDataroomItems({
  root: boolean,
  name: string[]  // folder path segments
});
```

Items are sorted using `sortByIndexThenName` from `lib/utils/sort-items-by-index-name.ts`.

### `useDataroomLinks`

Fetches all links associated with a dataroom:

```typescript
const { links, loading, error } = useDataroomLinks();
```

### `useDataroomStats`

Fetches analytics data for an entire dataroom, including visitor views and document-specific metrics:

```typescript
const { stats, loading, error } = useDataroomStats({ excludeTeamMembers });
```

### `useDataroomGroups`

Fetches viewer groups for a dataroom with optional document or folder scoping:

```typescript
const { viewerGroups, loading, error, mutate } = useDataroomGroups({
  documentId,
  folderId,
  dataroomId
});
```

### `useDataroomGroup`

Fetches a single group including its members and access controls:

```typescript
const { viewerGroup, viewerGroupMembers, viewerGroupPermissions } = useDataroomGroup();
```

### `useDataroomViewerGroups`

Fetches permission groups with their associated links:

```typescript
const { permissionGroups, loading, error, mutate } = useDataroomPermissionGroups();
```

### `useDataroomSearch`

Searches documents and folders within a dataroom by name:

```typescript
const { documents, folders, isLoading, error } = useDataroomSearch({ query });
```

### `useDataroomDocumentOverview`

Fetches document data scoped to a dataroom, allowing dataroom-scoped members to access document details:

```typescript
const { document, limits, featureFlags, counts } = useDataroomDocumentOverview();
```

### `useDataroomDocumentStats`

Fetches per-document statistics when viewing a document within a dataroom context:

```typescript
const { stats, loading, error } = useDataroomDocumentStats(documentId);
```

### `useDataroomViewDocumentStats`

Fetches document-level stats for a specific view within a dataroom visit:

```typescript
const { documentStats, loading, error } = useDataroomViewDocumentStats({
  dataroomId, dataroomViewId, enabled
});
```

### `useDataroomDocumentPageStats`

Fetches page-level duration data for a document viewed during a dataroom visit:

```typescript
const { duration, loading, error } = useDataroomDocumentPageStats({
  dataroomId, dataroomViewId, documentViewId, documentId, enabled
});
```

## Viewer/Visitor Hooks

### `useViewers`

Fetches paginated visitor list with sorting and optional search:

```typescript
const { viewers, pagination, sorting, isValidating, loading } = useViewers(
  page, pageSize, sortBy, sortOrder
);
```

### `useViewer`

Fetches detailed information for a single viewer including all their document views:

```typescript
const { viewer, durations, loading, loadingDurations } = useViewer(
  page, pageSize, sortBy, sortOrder
);
```

Fetches durations as a separate request when the viewer has views.

### `useLinkVisits`

Fetches visits for a specific link:

```typescript
const { views, hiddenFromPause, loading, error } = useLinkVisits(linkId);
```

## Billing Hooks

### `usePlan`

The most frequently used hook for plan-based feature gating. Returns parsed plan details plus convenience boolean flags:

```typescript
const { 
  plan,              // 'free' | 'starter' | 'pro' | etc.
  planName,          // Human-readable name from PLAN_NAME_MAP
  isTrial, 
  isCustomer,
  isPro,
  isDatarooms,
  isBusiness,
  isFree,
  discount,
  loading 
} = usePlan({ withDiscount });
```

The hook parses compound plan strings like `"pro+drtrial+old"` into structured data.

## Agreement/Signing Hooks

### `useAgreements`

Fetches all agreements with link and response counts:

```typescript
const { agreements, loading, error } = useAgreements();
```

### `useAgreementResponses`

Fetches all responses to a specific agreement, including signing status and envelope data:

```typescript
const { agreement, responses, loading, error, mutate } = useAgreementResponses(agreementId);
```

## Annotation Hooks

### `useAnnotations`

Fetches annotations for a document (admin side):

```typescript
const { annotations, loading, error, mutate } = useAnnotations(documentId, teamId);
```

### `useViewerAnnotations`

Fetches viewer-visible annotations for a document link. Only fetches when `viewId` is available to avoid 404s on unviewed documents:

```typescript
const { annotations, loading, error, mutate } = useViewerAnnotations(linkId, documentId, viewId);
```

## Integration Hooks

### `useSlackIntegration`

Fetches the team's Slack integration status (immutable cache):

```typescript
const { integration, loading, error, mutate } = useSlackIntegration({ enabled });
```

### `useSlackChannels`

Fetches available Slack channels with support for forced refresh:

```typescript
const { channels, loading, error, refresh } = useSlackChannels({ enabled });
```

## Settings Hooks

### `useBrand`

Fetches team branding settings:

```typescript
const { brand, loading, error } = useBrand();
```

### `useDataroomBrand`

Fetches dataroom-specific branding:

```typescript
const { brand, loading, error } = useDataroomBrand({ dataroomId });
```

### `useNotificationPreferences`

Fetches and updates team notification preferences:

```typescript
const { preferences, role, isLoading, updatePreferences } = useNotificationPreferences();
```

## Team Hooks

### `useTeams`

Fetches all teams the current user belongs to. Waits for router readiness and session:

```typescript
const { teams, loading, isValidating } = useTeams();
```

### `useGetTeam`

Fetches the current team details:

```typescript
const { team, loading, error } = useGetTeam();
```

## Tag Hooks

### `useTags`

Fetches tags with optional filtering, sorting, and pagination. Supports query validation via Zod schema:

```typescript
const { tags, tagCount, loading, isValidating } = useTags({
  query: { search, sortBy, sortOrder, page, pageSize },
  enabled,
  includeLinksCount
});
```

## Authentication Hooks

### `usePasskeys`

Fetches the user's passkey credentials:

```typescript
const { passkeys, loading, error, mutate, isValidating } = usePasskeys();
```

### `useSAML`

Fetches SAML SSO configuration:

```typescript
const { saml, connections, issuer, acs, configured, loading } = useSAML();
```

### `useSCIM`

Fetches SCIM directory sync configuration:

```typescript
const { scim, directories, provider, configured, loading } = useSCIM();
```

### `useDomains`

Fetches team domains (disabled by default, must be explicitly enabled):

```typescript
const { domains, loading, error } = useDomains({ enabled });
```

## Link Hooks

### `useLink`

Fetches a single link by ID from URL:

```typescript
const { link, loading, error } = useLink();
```

### `useDomainLink`

Fetches a link by domain and slug:

```typescript
const { link, loading, error } = useDomainLink();
```

## Other Hooks

### `useInvoices`

Fetches billing invoices:

```typescript
const { invoices, loading, error } = useInvoices();
```

### `useInvitations`

Fetches team invitations:

```typescript
const { invitations, loading, error } = useInvitations();
```

### `useVisitorGroups`

Fetches visitor groups:

```typescript
const { visitorGroups, loading, error, mutate } = useVisitorGroups();
```

### `useTeamAI`

Checks AI feature availability for the team:

```typescript
const { 
  isAIFeatureEnabled,
  isAIEnabled,
  isAdmin,
  canManageAI,
  canUseAI,
  isLoading 
} = useTeamAI();
```

### `useStats` / `useVisitorStats`

Legacy hooks for document-level analytics. Prefer `useDocumentStats` and `useDataroomStats` for new code.

### `useLimits`

Re-exports from `@/ee/limits/swr-handler` for platform limits.

## Adding New Hooks

Follow this template for new hooks in this module:

```typescript
import { useTeam } from "@/context/team-context";
import useSWR from "swr";

import { fetcher } from "@/lib/utils";

export type NewDataType = { /* ... */ };

export function useNewData(param?: string) {
  const teamInfo = useTeam();
  const teamId = teamInfo?.currentTeam?.id;

  const { data, error, mutate } = useSWR<NewDataType>(
    teamId && param
      ? `/api/teams/${teamId}/new-endpoint/${param}`
      : null,
    fetcher,
    {
      // Choose based on data volatility:
      // - 10000: frequently changing (stats, visits)
      // - 30000: moderately stable (documents, datarooms)
      // - 60000+: rarely changing (teams, billing)
      dedupingInterval: 30000,
      revalidateOnFocus: false,
    },
  );

  return {
    newData: data,
    loading: !error && !data,
    error,
    mutate,
  };
}
```