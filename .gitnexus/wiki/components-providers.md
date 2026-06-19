# components — providers

# PostHog Providers Module

## Overview

The `components/providers/` module provides PostHog analytics integration for the application. It consists of two components that work together to initialize PostHog, identify authenticated users, and sync team-level group analytics data.

**File structure:**

```
components/providers/
├── posthog-provider.tsx     # Main provider: initializes PostHog and identifies users
└── posthog-group-sync.tsx   # Syncs team data as a PostHog group
```

## PostHogCustomProvider

The root provider component that initializes the PostHog client and wraps the application tree.

### Purpose

- Initializes the PostHog JavaScript SDK on the client
- Identifies authenticated users and attaches user properties to events
- Provides PostHog context to all child components via `posthog-js/react`

### How It Works

```tsx
export const PostHogCustomProvider = ({ children }) => {
  // ...
  if (typeof window !== "undefined" && posthogConfig) {
    posthog.init(posthogConfig.key, {
      api_host: posthogConfig.host,
      ui_host: "https://eu.posthog.com",
      disable_session_recording: true,
      autocapture: false,
      loaded: (posthog) => {
        // Identify user or reset on load
      },
    });
  }
  return <PostHogProvider client={posthog}>{children}</PostHogProvider>;
};
```

**Initialization behavior:**

1. Checks for client-side execution (`typeof window !== "undefined"`)
2. Retrieves configuration via `getPostHogConfig()`
3. Initializes PostHog with specific settings:
   - `disable_session_recording: true` — Session recording is disabled
   - `autocapture: false` — Click and form autocapture is disabled
   - `ui_host` points to EU PostHog instance

**User identification:**

When PostHog finishes loading, it calls `getSession()` to check authentication status:

- **Authenticated user:** Calls `posthog.identify()` with email and user ID as user properties
- **No session:** Calls `posthog.reset()` to clear any stored user data

```tsx
getSession().then((session) => {
  if (session) {
    posthog.identify(email ?? userId, {
      email,
      userId,
    });
  } else {
    posthog.reset();
  }
});
```

### Usage

This provider wraps the entire application in `pages/_app.tsx`. It must be placed outside any components that need PostHog access.

```tsx
<PostHogCustomProvider>
  <TeamProvider>
    <Component {...pageProps} />
  </TeamProvider>
</PostHogCustomProvider>
```

## PostHogGroupSync

A side-effect component that syncs the current team context to PostHog as a `team` group for group analytics.

### Purpose

- Associates all analytics events with the user's current team
- Enables team-level aggregation and filtering in PostHog dashboards
- Syncs team properties (name, plan, join date) as group traits

### How It Works

```tsx
export function PostHogGroupSync() {
  const { currentTeam } = useTeam();
  const lastGroupKeyRef = useRef<string | null>(null);

  useEffect(() => {
    // Skip if PostHog not configured
    if (!getPostHogConfig()) return;
    if (!currentTeam?.id) return;

    // Build team properties
    const properties: Record<string, unknown> = {};
    if (currentTeam.name) properties.name = currentTeam.name;
    if (currentTeam.plan) properties.plan = currentTeam.plan;
    if (currentTeam.createdAt) {
      properties.date_joined = new Date(currentTeam.createdAt).toISOString();
    }

    // Set the team group in PostHog
    posthog.group("team", currentTeam.id, properties);
  }, [currentTeam?.id, currentTeam?.name, currentTeam?.plan, currentTeam?.createdAt]);

  return null;
}
```

**Group properties synced:**

| Property | Source | Description |
|----------|--------|-------------|
| `name` | `currentTeam.name` | Team display name |
| `plan` | `currentTeam.plan` | Subscription tier |
| `date_joined` | `currentTeam.createdAt` | When the team was created |

### Usage Requirements

This component must be rendered:

1. **Inside `<PostHogCustomProvider>`** — Requires PostHog to be initialized
2. **Inside `<TeamProvider>`** — Requires the team context to access `currentTeam`

```tsx
<PostHogCustomProvider>
  <TeamProvider>
    <PostHogGroupSync />
    {/* rest of app */}
  </TeamProvider>
</PostHogCustomProvider>
```

The component returns `null` and performs no visible rendering — it exists purely for its side effect.

## Architecture

```mermaid
graph TD
    App[pages/_app.tsx] -->|wraps| PostHogCustomProvider
    App -->|wraps| TeamProvider
    
    PostHogCustomProvider -->|initializes| PostHogSDK[posthog-js]
    PostHogCustomProvider -->|calls| getPostHogConfig["getPostHogConfig()"]
    PostHogCustomProvider -->|calls| getSession["getSession()"]
    
    TeamProvider -->|provides| useTeam
    
    PostHogGroupSync -->|uses| useTeam
    PostHogGroupSync -->|calls| getPostHogConfig
    PostHogGroupSync -->|calls| posthog_group["posthog.group('team', ...node[")"]"]
    
    style PostHogSDK fill:#f9f,stroke:#333,stroke-width:2px
```

## Key Behaviors

### Server-Side Rendering Safety

Both components include `typeof window !== "undefined"` checks to prevent PostHog initialization during SSR:

```tsx
if (typeof window === "undefined") return;
```

### Debug Mode

When `NODE_ENV === "development"`, PostHog debug mode is enabled automatically, logging all events to the browser console.

### Missing Configuration

If `getPostHogConfig()` returns falsy (no PostHog key configured), initialization and group syncing are skipped silently.

### Dependencies

| Dependency | Source | Purpose |
|------------|--------|---------|
| `getPostHogConfig()` | `@/lib/posthog` | Retrieves API key and host from environment |
| `useTeam()` | `@/context/team-context` | Accesses current team state |
| `getSession()` | `next-auth/react` | Checks authentication status |
| `posthog-js` | external | Analytics SDK |
| `posthog-js/react` | external | React context provider |