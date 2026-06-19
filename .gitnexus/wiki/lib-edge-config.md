# lib — edge-config

# Edge Config Module (`lib/edge-config`)

The edge-config module provides a typed interface to Vercel Edge Config, a globally distributed key-value store optimized for low-latency reads. This module centralizes all Edge Config access in the codebase, making it easy to manage runtime configuration without deploying code changes.

## Purpose

This module enables several runtime behaviors that would otherwise require code deployments to change:

- **Email blacklist enforcement** — Block sign-ins from specific email addresses
- **Custom team emails** — Override default sender addresses per team
- **Embeddable domain allowlisting** — Control which domains can embed content via iframes
- **Trusted team verification** — Mark specific teams as trusted for elevated permissions

## How Edge Config Works

Edge Config is backed by Vercel's global edge network. When `process.env.EDGE_CONFIG` is set, the `@vercel/edge-config` client reads from this store. All functions in this module follow the same pattern:

```typescript
if (!process.env.EDGE_CONFIG) {
  return safeDefaultValue;
}
```

This ensures the application functions normally in environments without Edge Config configured.

## Module Structure

```
lib/edge-config/
├── blacklist.ts          # Email blacklist checking
├── custom-email.ts       # Team-specific custom email retrieval
├── embeddable-domains.ts # Domain allowlist for embeds
└── trusted-teams.ts      # Team trust verification
```

## Components

### Email Blacklist (`blacklist.ts`)

```typescript
isBlacklistedEmail(email: string): Promise<boolean>
```

Checks whether an email address matches any entry in the blacklist stored under the `emails` key.

**Matching behavior:**
- Uses a case-insensitive regex constructed from all blacklist entries joined with `|`
- Supports glob-style patterns since regex is applied (e.g., `.*@temp\.com` blocks all emails from that domain)

**Returns:** `false` if Edge Config is unavailable, the key is missing, or no match is found.

**Used by:** `signIn` handler in `api/auth/[...nextauth].ts`

---

### Custom Email per Team (`custom-email.ts`)

```typescript
getCustomEmail(teamId?: string): Promise<string | null>
```

Retrieves a custom sender email for a specific team from the `customEmail` key, which stores a JSON object mapping team IDs to email addresses.

**Returns:** The custom email string if found, or `null` if:
- `EDGE_CONFIG` is not set
- `teamId` is not provided
- The key doesn't exist or is malformed
- No custom email is defined for the given team

**Used by:** `sendOtpVerificationEmail` in `lib/emails/send-email-otp-verification.ts`

---

### Embeddable Domains (`embeddable-domains.ts`)

This file contains the most complex logic, implementing hostname matching for iframe embedding restrictions.

#### `getEmbeddableDomains(): Promise<string[]>`

Reads the `embeddableDomains` key and returns an array of allowed hostnames. Returns an empty array if Edge Config is unavailable or the value is not an array.

#### `isEmbeddableUrl(url: string | null | undefined): Promise<boolean>`

Determines whether a URL can be used as an embed source.

**Validation steps:**
1. Returns `false` for null/undefined/empty URLs
2. Parses the URL and returns `false` if parsing fails
3. Returns `false` if the protocol is not `https:`
4. Returns `false` if the allowlist is empty
5. Returns `true` if the hostname matches any allowlist entry

**Hostname matching rules:**

| Allowlist Entry | Matches | Does Not Match |
|-----------------|---------|----------------|
| `example.com` | `example.com` | `app.example.com` |
| `.example.com` | `app.example.com`, `nested.app.example.com` | `example.com` |

Matching is case-insensitive.

**Used by:**
- `POST` handler in `api/views/route.ts`
- `POST` handler in `api/views-dataroom/route.ts`

---

### Trusted Teams (`trusted-teams.ts`)

```typescript
isTrustedTeam(teamId: string): Promise<boolean>
```

Checks whether a team ID exists in the `trustedTeams` array stored in Edge Config.

**Returns:** `false` if Edge Config is unavailable, the key is missing, or the team ID is not in the list.

**Used by:**
- `validateRedirectUrl` in `api/domains/validate-redirect-url.ts`
- `handle` in `documents/[id]/update-link-url.ts`
- `processDocument` in `api/documents/process-document.ts`
- `run` in `lib/trigger/pdf-to-image-route.ts`

## Error Handling Pattern

All functions in this module implement defensive error handling:

```typescript
try {
  const result = await get("key");
  // Validate and transform result
} catch (e) {
  // Return safe default (empty array, null, or false)
}
```

This ensures that transient Edge Config failures don't crash the application — callers receive a safe default value and the request continues.

## Usage Example

```typescript
import { isBlacklistedEmail, isEmbeddableUrl, isTrustedTeam } from "~/lib/edge-config";

// Block a sign-in from a blacklisted email
const isBlocked = await isBlacklistedEmail(user.email);
if (isBlocked) {
  throw new Error("Sign-in not allowed");
}

// Check if a URL can be embedded
const canEmbed = await isEmbeddableUrl(sourceUrl);
if (!canEmbed) {
  return { error: "Domain not allowed for embedding" };
}

// Verify team trust level
const isTrusted = await isTrustedTeam(teamId);
if (isTrusted) {
  // Grant elevated permissions
}
```

## Configuration Keys

The module expects the following keys to be configured in your Vercel Edge Config store:

| Key | Type | Description |
|-----|------|-------------|
| `emails` | `string[]` | Array of email patterns to block |
| `customEmail` | `Record<string, string>` | Team ID to custom sender email mapping |
| `embeddableDomains` | `string[]` | Allowed hostnames for iframe embeds |
| `trustedTeams` | `string[]` | Array of trusted team IDs |