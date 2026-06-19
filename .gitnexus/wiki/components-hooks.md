# components — hooks

# Hooks Module

This module provides React hooks for managing optimistic data updates and tracking user login preferences.

## Overview

The hooks are located in `components/hooks/` and include:

- **`useOptimisticUpdate`** — Manages optimistic UI updates with SWR and toast notifications
- **`useLastUsed`** — Persists and displays the user's last-used login method

## useOptimisticUpdate

Provides optimistic data mutation with automatic toast feedback. The hook wraps SWR's `mutate` function to enable optimistic updates where the UI reflects changes immediately while the request is in-flight.

```typescript
const { data, isLoading, update } = useOptimisticUpdate<T>(url, toastCopy?)
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | `string` | SWR endpoint to fetch and mutate |
| `toastCopy` | `{ loading, success, error }` | Optional custom toast messages |

### Return Value

| Property | Type | Description |
|----------|------|-------------|
| `data` | `T` | The current fetched data |
| `isLoading` | `boolean` | Whether initial fetch is in progress |
| `update` | `(fn, optimisticData) => Promise` | Mutation function |

### Usage

```typescript
const { data, update } = useOptimisticUpdate<UserPreferences>(
  "/api/user/preferences"
);

await update(
  (currentData) => api.updatePreferences({ theme: "dark" }),
  { ...currentData, theme: "dark" }  // optimisticData
);
```

### How It Works

The `update` function:

1. Accepts a transformation function `fn` that receives current data and returns a `Promise<T>`
2. Accepts `optimisticData` — the expected result applied immediately to the cache
3. Calls `mutate` with `rollbackOnError: true` — reverts to the original data if the mutation fails
4. Shows a toast notification with the provided copy (or defaults)
5. Returns a `toast.promise` for caller awaiting

### Integration

Used by `UpdateMailSubscribe` in `components/account/update-subscription.tsx` for subscription modifications.

---

## useLastUsed

Tracks and persists the user's most recent authentication method, with a companion display component.

```typescript
const [lastUsed, setLastUsed] = useLastUsed();
```

### Supported Login Types

```typescript
type LoginType = "passkey" | "google" | "credentials" | "linkedin";
```

### Return Value

| Value | Type | Description |
|-------|------|-------------|
| `lastUsed` | `LoginType \| undefined` | The stored login method |
| `setLastUsed` | `(type: LoginType) => void` | Updates and persists the value |

### Storage Behavior

- **Read:** On mount, reads from `localStorage` key `last_papermark_login`
- **Write:** Updates localStorage whenever `lastUsed` changes
- **Clear:** Removes the key if `lastUsed` is set to `undefined`

### LastUsed Component

A visual indicator that displays "Last used" next to the associated login option.

```tsx
<LastUsed className="custom-styles" />
```

**Positioning:** Absolute positioning with CSS transforms — displays to the right on mobile, offset to the side on larger screens using a CSS triangle pointer.

### Integration

Used in `app/(auth)/login/page-client.tsx` to:

1. Show which login method was previously used
2. Persist the user's choice when they authenticate