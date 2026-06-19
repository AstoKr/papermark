# components — analytics

# Analytics Components Module

The `components/analytics` module provides a suite of UI components for displaying analytics data on the dashboard. It handles document views, link analytics, visitor tracking, and time range selection with built-in sorting, pagination, and export functionality.

## Architecture Overview

The module consists of two categories: a reusable card wrapper and data visualization components. All data-fetching components integrate with the team context and billing system to enforce feature restrictions.

```mermaid
graph TB
    subgraph "Analytics Module"
        AC[AnalyticsCard]
        DVC[DashboardViewsChart]
        TR[TimeRangeSelect]
        DT[DocumentsTable]
        LT[LinksTable]
        VT[ViewsTable]
        VST[VisitorsTable]
    end
    
    subgraph "External Dependencies"
        TeamCtx[useTeam]
        BillingHook[usePlan]
        API[/api/analytics]
    end
    
    DT --> TeamCtx
    LT --> TeamCtx
    VT --> TeamCtx
    VST --> TeamCtx
    TR --> TeamCtx
    
    DT --> BillingHook
    LT --> BillingHook
    VT --> BillingHook
    VST --> BillingHook
    
    DT --> API
    LT --> API
    VT --> API
    VST --> API
```

## AnalyticsCard

**File:** `analytics-card.tsx`

A reusable card container that provides consistent styling for analytics sections. It includes a header with a title, optional column headers for tabular data, and an optional icon.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `title` | `string` | Card heading text |
| `icon` | `ReactNode` | Optional icon displayed in the header |
| `columnHeaders` | `{ label: string; width?: string }[]` | Optional column headers for tables |
| `children` | `ReactNode` | Card content |
| `className` | `string` | Additional CSS classes for the card wrapper |
| `contentClassName` | `string` | Additional CSS classes for the content area |

### Usage Example

```tsx
<AnalyticsCard
  title="Top Documents"
  columnHeaders={[
    { label: "Views", width: "w-20" },
    { label: "Last Viewed", width: "w-32" }
  ]}
>
  <DocumentsTable startDate={startDate} endDate={endDate} />
</AnalyticsCard>
```

## DashboardViewsChart

**File:** `dashboard-views-chart.tsx`

A responsive bar chart component that displays document view counts over time. It adapts its display granularity based on the selected time range.

### Props

| Prop | Type | Description |
|------|------|-------------|
| `timeRange` | `TimeRange` | One of `"24h"`, `"7d"`, `"30d"`, or `"custom"` |
| `data` | `{ date: string; views: number }[]` | View data points |
| `startDate` | `Date` | Custom range start date |
| `endDate` | `Date` | Custom range end date |

### Adaptive Time Grouping

The chart automatically adjusts aggregation based on the time range:

| Time Range | Aggregation | Display Format |
|------------|-------------|----------------|
| `24h` | Hourly slots | `h:mm aa` (e.g., "3:00 PM") |
| `7d` | Daily slots | `EEE, MMM d` (e.g., "Mon, Jan 15") |
| `30d` | Daily slots | `MMM d` (e.g., "Jan 15") |
| `custom` (≤30 days) | Daily slots | `MMM d` |
| `custom` (31-365 days) | Weekly slots | `MMM d` |
| `custom` (>365 days) | Monthly slots | `MMM yyyy` |

### Bar Size Calculation

Bar width adjusts to maintain readability across different data densities:

```typescript
const barSize = useMemo(() => {
  if (timeRange === "24h") return 14;
  if (timeRange === "7d") return 32;
  if (timeRange === "30d") return 16;
  if (totalDays > 365) return 32;
  if (totalDays > 30) return 22;
  return 16;
}, [...]);
```

## TimeRangeSelect

**File:** `time-range-select.tsx`

A dropdown component combining preset time ranges with a calendar picker for custom date ranges.

### Time Range Options

| Value | Label | Description |
|-------|-------|-------------|
| `"24h"` | Last 24 hours | Shows hourly data |
| `"7d"` | Last 7 days | Shows daily data |
| `"30d"` | Last 30 days | Shows daily data |
| `"custom"` | Custom Date | Opens calendar for arbitrary range |

### Props

```typescript
interface TimeRangeSelectProps {
  value: TimeRange;
  onChange: (value: TimeRange) => void;
  customRange: CustomRange;
  setCustomRange: (range: CustomRange) => void;
  onCustomRangeComplete?: (range: CustomRange) => void;
  slug: React.MutableRefObject<boolean>;
  isPremium?: boolean;
}
```

### Premium Gating

Non-premium users are restricted to viewing data from the last 30 days. The `isPremium` prop controls:

- Whether the custom date picker is available
- The `disabled` state for dates beyond 30 days ago in the calendar
- Display of the upgrade prompt in the dropdown

```typescript
// Minimum date for non-premium users
const minDate = isPremium ? undefined : subDays(new Date(), 30);
```

## Table Components

All four table components (`DocumentsTable`, `LinksTable`, `ViewsTable`, `VisitorsTable`) share a common architecture built on TanStack Table.

### Common Features

- **Sorting**: Click column headers to toggle ascending/descending order
- **Pagination**: Built-in pagination via `DataTablePagination`
- **Export**: CSV export with paywall for free users
- **Data Fetching**: SWR-based fetching with `keepPreviousData: true`
- **Empty States**: Informative messages when no data exists

### DocumentsTable

**File:** `documents-table.tsx`

Displays analytics for individual documents.

| Column | Description |
|--------|-------------|
| Documents | Link to the document with name |
| Views | Total view count |
| Avg Duration | Average time spent viewing |
| Last Viewed | Relative time with UTC/local tooltip |

```typescript
interface Document {
  id: string;
  name: string;
  views: number;
  avgDuration: string;
  lastViewed: Date | null;
}
```

### LinksTable

**File:** `links-table.tsx`

Displays analytics for shared links.

| Column | Description |
|--------|-------------|
| Links | Link name with URL and copy button |
| Document | Associated document name (linkable) |
| Views | Total view count |
| Avg Duration | Average time spent |
| Last Viewed | Relative time |

The component includes a `CopyButton` sub-component that copies the URL to clipboard with toast feedback.

### ViewsTable

**File:** `views-table.tsx`

Displays individual view records with rich metadata.

| Column | Description |
|--------|-------------|
| Recent Views | Viewer email with status badges |
| Document | Document name (linkable) |
| Link | Link name |
| Time Spent | Formatted duration |
| Completion | Gauge showing completion percentage |
| Last Viewed | Relative timestamp |

**Status Badges** (displayed next to viewer email):

| Badge | Icon | Meaning |
|-------|------|---------|
| Verified | `BadgeCheckIcon` | Identity verified |
| Internal | `BadgeInfoIcon` | Team member |
| Agreement | `FileBadgeIcon` | Signed agreement |
| Downloaded | `DownloadCloudIcon` | Downloaded document |
| Dataroom | `ServerIcon` | Accessed via dataroom |
| Feedback | `ThumbsUp/DownIcon` | Feedback response |

```typescript
interface View {
  id: string;
  viewerEmail: string | null;
  documentName: string;
  linkName: string;
  viewedAt: Date;
  totalDuration: number;
  completionRate: number;
  verified?: boolean;
  internal?: boolean;
  agreementResponse?: any;
  downloadedAt?: Date;
  dataroomId?: string;
  feedbackResponse?: any;
}
```

### VisitorsTable

**File:** `visitors-table.tsx`

Displays aggregated visitor statistics.

| Column | Description |
|--------|-------------|
| Visitor | Avatar, name, verified badge, document count |
| Total Views | Number of views by this visitor |
| Total Time Spent | Cumulative time across all views |
| Last Active | Relative last activity time |

```typescript
interface Visitor {
  email: string;
  viewerId: string | null;
  totalViews: number;
  lastActive: Date;
  uniqueDocuments: number;
  verified: boolean;
  totalDuration: number;
  viewerName?: string | null;
}
```

## Common Patterns

### API Data Fetching

All table components follow the same SWR pattern:

```typescript
const interval = router.query.interval || "7d";
const { data } = useSWR<DataType>(
  teamInfo?.currentTeam?.id
    ? `/api/analytics?type=${type}&interval=${interval}&teamId=${teamInfo.currentTeam.id}${interval === "custom" ? `&startDate=${format(startDate, "MM-dd-yyyy")}&endDate=${format(endDate, "MM-dd-yyyy")}` : ""}`
    : null,
  fetcher,
  {
    keepPreviousData: true,
    revalidateOnFocus: false,
  }
);
```

### Export Flow

Each table includes an `UpgradeOrExportButton` that conditionally renders either an export button or an upgrade prompt:

```typescript
const UpgradeOrExportButton = () => {
  if (isFree && !isTrial) {
    return (
      <UpgradeButton
        text="Export"
        clickedPlan={PlanEnum.Pro}
        trigger="dashboard_documents_export"
        variant="outline"
        size="sm"
      />
    );
  }
  return (
    <Button variant="outline" size="sm" onClick={handleExport}>
      <Download className="!size-4" />
      Export
    </Button>
  );
};
```

### Subscription Paused Warning

When a team's subscription is paused, `ViewsTable` and `VisitorsTable` display a warning banner for hidden views:

```tsx
{isPaused && hiddenFromPause > 0 && (
  <div className="flex ... border-orange-200 bg-orange-50 ...">
    <AlertTriangleIcon className="inline-block h-4 w-4 text-orange-500" />
    {hiddenFromPause} view{hiddenFromPause !== 1 ? "s" : ""} occurred
    after your team was paused...
  </div>
)}
```

## Integration Points

### Team Context (`useTeam`)

All table components access the current team ID via `useTeam()` to filter analytics data:

```typescript
const teamInfo = useTeam();
const teamId = teamInfo?.currentTeam?.id;
```

### Billing Context (`usePlan`)

The `usePlan()` hook provides plan status for gating features:

```typescript
const { isTrial, isFree, isPaused } = usePlan();
```

### Upstream Components

The module is consumed by the dashboard page, which wraps tables in `AnalyticsCard` components:

```tsx
// In pages/dashboard.tsx
<AnalyticsCard title="Page Views" icon={<BarChart3 className="h-4 w-4" />}>
  <DashboardViewsChart timeRange={timeRange} data={viewsData} />
</AnalyticsCard>

<AnalyticsCard
  title="Top Documents"
  columnHeaders={[{ label: "Views" }, { label: "Last Viewed" }]}
>
  <DocumentsTable startDate={startDate} endDate={endDate} />
</AnalyticsCard>
```

### API Endpoint

All components fetch from `/api/analytics` with query parameters:

| Parameter | Values | Description |
|-----------|--------|-------------|
| `type` | `documents`, `links`, `views`, `visitors` | Data type |
| `interval` | `24h`, `7d`, `30d`, `custom` | Time range |
| `teamId` | Team UUID | Filter by team |
| `startDate` | `MM-dd-yyyy` | Custom range start |
| `endDate` | `MM-dd-yyyy` | Custom range end |