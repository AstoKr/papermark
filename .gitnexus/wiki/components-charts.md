# components — charts

# Charts Module

The `components/charts` module provides bar chart visualizations for displaying document reading analytics — specifically, time spent per page across document versions.

## Overview

This module renders interactive bar charts using the [Tremor](https://tremor.so/) library, supporting two visualization modes:

1. **Per-version comparison** — Shows average time spent per page for each document version
2. **Aggregated sum** — Shows total time spent per page across all versions

The charts display time in a `MM:SS` format and include a custom tooltip that fetches and displays document page thumbnails on hover.

## Module Structure

```
components/charts/
├── bar-chart.tsx          # Main chart component with data transformation
├── bar-chart-tooltip.tsx  # Custom tooltip with thumbnail support
└── utils.ts               # Types, formatters, and color utilities
```

## Data Flow

```mermaid
flowchart TD
    A[External Caller] -->|"Data[] or SumData[]"| B[BarChartComponent]
    B -->|transformData / renameSumDurationKey| C[TransformedData[]]
    C --> D[Tremor BarChart]
    D -->|onValueChange| E[State Update]
    D -->|hover| F[CustomTooltip]
    F -->|pageNumber, documentId| G[useDocumentThumbnail]
    G -->|imageUrl| F
```

## Types

Defined in `utils.ts`:

```typescript
// Per-version data structure
type Data = {
  pageNumber: string;
  data: {
    versionNumber: number;
    avg_duration: number;
  }[];
};

// Aggregated data structure
type SumData = {
  pageNumber: string;
  sum_duration: number;
};

// Tremor-compatible format
type TransformedData = {
  pageNumber: string;
  [key: string]: number | string;  // Dynamic version keys
};
```

## BarChartComponent (`bar-chart.tsx`)

The main exported component that handles data transformation and renders the Tremor `BarChart`.

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `data` | `any` | — | Chart data in `Data[]` or `SumData[]` format |
| `isSum` | `boolean` | `false` | When `true`, renders aggregated view instead of per-version |
| `isDummy` | `boolean` | `false` | When `true`, renders placeholder data with gray styling |
| `versionNumber` | `number` | — | Passed to tooltip for thumbnail fetching |
| `documentId` | `string` | — | Passed to tooltip for thumbnail fetching |

### Data Transformation

The component transforms raw data into Tremor's expected format:

**Per-version mode** (`isSum=false`):
```typescript
// Input: Data[]
{ pageNumber: "1", data: [{ versionNumber: 1, avg_duration: 5000 }] }

// Output: TransformedData[]
{ pageNumber: "1", "Version 1": 5000 }
```

**Aggregated mode** (`isSum=true`):
```typescript
// Input: SumData[]
{ pageNumber: "1", sum_duration: 15000 }

// Output (renamed key)
{ pageNumber: "1", "Time spent per page": 15000 }
```

### Color Assignment

Version numbers are mapped to Tremor color values using a cycle of 22 colors. The `getColors()` utility distributes colors across version categories to ensure visual distinction:

```typescript
const colors = ["emerald", "teal", "gray", "orange", ...];
// Version 1 → emerald, Version 2 → teal, Version 23 → emerald (wraps)
```

## CustomTooltip (`bar-chart-tooltip.tsx`)

A custom tooltip component that displays when hovering over chart bars. It shows page metadata and fetches document thumbnails on demand.

### Behavior

1. **Extracts payload data** — Gets `pageNumber`, `documentId`, and `versionNumber` from the chart payload
2. **Fetches thumbnail** — Calls `useDocumentThumbnail` with the page number and document ID
3. **Renders tooltip** — Displays page number, thumbnail image (if available), and a list of version times with color indicators

### Thumbnail Fetching

The tooltip always calls `useDocumentThumbnail()` regardless of hover state — this is an intentional pattern to pre-fetch data so thumbnails are available immediately when hovering. The component returns `null` if `active` is false or no payload exists.

### Dependencies

- `useRouter` from `next/router` — Falls back to `router.query.id` if `documentId` isn't in the payload
- `useDocumentThumbnail` — Fetches the page thumbnail image URL

## Utility Functions (`utils.ts`)

### timeFormatter

Converts milliseconds to a `MM:SS` string:

```typescript
timeFormatter(125000)  // "02:05"
timeFormatter(5000)    // "00:05"
```

### getColorForVersion

Maps a version string (e.g., `"Version 1"`) to a Tremor color value:

```typescript
getColorForVersion("Version 1")  // "emerald"
getColorForVersion("Version 2")  // "teal"
```

### getColors

Returns an array of colors for a list of version numbers, useful for the Tremor `colors` prop:

```typescript
getColors(["Version 1", "Version 2", "Version 3"])
// ["emerald", "teal", "gray"]
```

## Usage Example

```tsx
import BarChartComponent from "@/components/charts/bar-chart";

// Per-version comparison
<BarChartComponent
  data={[
    { pageNumber: "1", data: [{ versionNumber: 1, avg_duration: 5000 }] },
    { pageNumber: "2", data: [{ versionNumber: 1, avg_duration: 8000 }] },
  ]}
  versionNumber={1}
  documentId="doc-123"
/>

// Aggregated view
<BarChartComponent
  data={[
    { pageNumber: "1", sum_duration: 15000 },
    { pageNumber: "2", sum_duration: 20000 },
  ]}
  isSum
  versionNumber={1}
  documentId="doc-123"
/>
```

## External Dependencies

| Dependency | Source | Purpose |
|------------|--------|---------|
| `BarChart` | `@tremor/react` | Base chart rendering |
| `useDocumentThumbnail` | `lib/swr/use-document.ts` | Page thumbnail fetching |
| `useRouter` | `next/router` | Document ID from URL |
| Tremor utilities | `tailwind-merge` | Color and styling classes |