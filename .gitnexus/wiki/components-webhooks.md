# components — webhooks

# Webhook Events Component Module

`components/webhooks/webhook-events.tsx`

## Overview

The webhook events module provides a visual interface for displaying webhook delivery history. It renders a scrollable list of webhook events with expandable detail views showing request and response payloads, syntax-highlighted for readability.

## Component Hierarchy

```mermaid
componentDiagram
    direction LR
    WebhookEventList --> WebhookEvent
    WebhookEvent --> CodeHighlighter
    WebhookEvent --> Sheet
    WebhookEvent --> ButtonTooltip
```

## Components

### `WebhookEventList`

The main export. Renders a bordered container with a vertically-stacked list of webhook events.

**Props:**
```typescript
type EventListProps = PropsWithChildren<{
  events: any[];
}>;
```

**Usage Pattern:**

```tsx
<WebhookEventList events={webhook.events} />
```

The `events` array expects objects with this shape:

```typescript
interface WebhookEvent {
  event: string;           // Event type name (e.g., "payment.created")
  event_id: string;        // Unique identifier for the event
  http_status: number;     // Delivery HTTP status code
  timestamp: string;       // ISO 8601 timestamp
  request_body: object;    // JSON payload sent to the endpoint
  response_body: object;   // JSON response received from the endpoint
}
```

---

### `WebhookEvent`

An individual row within the event list. Displays a summary row by default and manages an expandable detail sheet.

**Visual Structure:**

| Icon | HTTP Status | Event Name | Timestamp |
|------|-------------|------------|-----------|
| ✓/✗  | 200         | payment.created | Jan 15, 2025, 10:30:00 AM |

**Status Indicators:**
- `CircleCheck` (green) — HTTP status in 2xx range
- `CircleXIcon` (red/destructive) — HTTP status outside 2xx range or error

**Detail Sheet Content:**

When clicked, a `Sheet` slides in from the right containing:

1. **Header** — Event name and event ID (with copy-to-clipboard button)
2. **Response Section** — HTTP status code and formatted response body
3. **Request Section** — Formatted request body payload

---

### `CodeHighlighter`

An internal component that uses [Shiki](https://shiki.matsu.io/) to apply syntax highlighting to JSON payloads. It supports light and dark theme variants.

**Initialization:**
```typescript
const shiki = await createHighlighter({
  themes: ['material-theme-lighter', 'material-theme-darker'],
  langs: ['json']
});
```

**Theme Selection:**
- Light mode: `material-theme-lighter`
- Dark mode: `material-theme-darker`

The component applies custom inline styles to the Shiki output for consistent padding, font sizing, and scroll behavior.

**Fallback Rendering:**

While the highlighter initializes (which requires async loading of language grammars), a plain `<pre>` block displays the raw JSON.

---

## State Management

| State | Purpose |
|-------|---------|
| `isOpen` (in `WebhookEvent`) | Controls the detail sheet visibility |
| `highlightedCode` (in `CodeHighlighter`) | Caches the rendered HTML output |

The `isDark` flag is derived from `next-themes`:

```typescript
const { theme, systemTheme } = useTheme();
const isDark = theme === "dark" || (theme === "system" && systemTheme === "dark");
```

This drives both the Shiki theme selection and could be extended to other styling decisions.

---

## External Dependencies

| Dependency | Source | Usage |
|------------|--------|-------|
| `useCopyToClipboard` | `@/lib/utils/use-copy-to-clipboard` | Copy event ID to clipboard |
| `useMediaQuery` | `@/lib/utils/use-media-query` | Detect mobile for timestamp formatting |
| `Sheet` components | `@/components/ui/sheet` | Slide-out detail panel |
| `ScrollArea` | `@/components/ui/scroll-area` | Scrollable content in detail panel |
| `Button` | `@/components/ui/button` | Icon buttons (copy action) |
| `ButtonTooltip` | `@/components/ui/tooltip` | Hover tooltips for status icons |

---

## Usage in Parent Components

The `WebhookEventList` is consumed by `WebhookDetail` at `webhooks/[id]/index.tsx`. The parent component is responsible for fetching webhook event data and passing it as the `events` prop.

```tsx
// Simplified example
export default function WebhookDetail({ params }: { params: { id: string } }) {
  const { data } = useWebhookEvents(params.id);
  
  return (
    <div>
      <h1>Webhook Events</h1>
      <WebhookEventList events={data?.events ?? []} />
    </div>
  );
}
```

---

## Timestamp Handling

Timestamps are converted to local time before display:

```typescript
const date = new Date(event.timestamp);
const localDate = new Date(date.getTime() - date.getTimezoneOffset() * 60000);
```

**Display behavior:**
- Mobile (`isMobile`): Shows only time (`10:30:00 AM`)
- Desktop: Shows full date and time (`Jan 15, 2025, 10:30:00 AM`)