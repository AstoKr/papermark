# lib — incoming-webhooks

# lib/incoming-webhooks

Utilities for generating and validating incoming webhook identifiers and secrets.

## Overview

This module provides cryptographic utilities for creating and parsing webhook IDs used in the incoming webhooks system. Webhook IDs encode team information while remaining URL-safe, and include cryptographically random segments for uniqueness and security.

## Webhook ID Format

Webhook IDs follow a structured format designed for parseability and collision resistance:

```
T{encoded_team_id}/B{8_chars}/{24_chars}
```

| Component | Description | Example |
|-----------|-------------|---------|
| `T` prefix | Identifies team portion | `T` |
| `encoded_team_id` | Team ID in URL-safe base64 | `QUJDREVGRw` |
| `B` prefix + 8 chars | Bot identifier segment | `BAbCdEfGh` |
| 24 chars | Secret segment | `xK9mPqR2sTuVwXyZaBcDeFgH` |

### Example Full ID

```
TQUJDREVGRw/BAbCdEfGh/xK9mPqR2sTuVwXyZaBcDeFgH
```

## Key Functions

### `generateWebhookId(teamId)`

Creates a complete webhook ID for a given team.

```typescript
function generateWebhookId(teamId: string): string
```

**Parameters:**
- `teamId` — The original team identifier string

**Returns:** A formatted webhook ID combining the encoded team ID with two random segments.

**Usage:**

```typescript
const webhookId = generateWebhookId("team_abc123");
// Returns: "T{encoded}/B{8_chars}/{24_chars}"
```

### `extractTeamId(webhookId)`

Recovers the original team ID from a webhook ID.

```typescript
function extractTeamId(webhookId: string): string | null
```

**Parameters:**
- `webhookId` — A webhook ID string

**Returns:** The original team ID, or `null` if the format is invalid.

**Usage:**

```typescript
const teamId = extractTeamId("TQUJDREVGRw/BAbCdEfGh/xK9mPqR2sTuVwXyZaBcDeFgH");
// Returns: "team_abc123" or null if malformed
```

### `isValidWebhookId(webhookId)`

Validates that a string conforms to the webhook ID format.

```typescript
function isValidWebhookId(webhookId: string): boolean
```

**Validation rules:**
- Exactly 3 segments separated by `/`
- First segment starts with `T`
- Second segment starts with `B` and has exactly 9 characters (prefix + 8 chars)
- Third segment has exactly 24 characters

**Usage:**

```typescript
if (isValidWebhookId(incomingId)) {
  const teamId = extractTeamId(incomingId);
}
```

### `generateWebhookSecret()`

Generates a cryptographically secure secret for webhook authentication.

```typescript
function generateWebhookSecret(): string
```

**Returns:** A secret string prefixed with `whsec_`.

**Usage:**

```typescript
const secret = generateWebhookSecret();
// Returns: "whsec_{random_id}"
```

## Internal Encoding Functions

### Team ID Encoding

Team IDs are converted to URL-safe base64 to ensure webhook IDs remain safe in URLs and database fields:

```typescript
// Encode: "my_team_id" → "bXlfdGVhbV9pZA"
encodeTeamId(teamId: string): string

// Decode: "bXlfdGVhbV9pZA" → "my_team_id"
decodeTeamId(encoded: string): string
```

The encoding replaces standard base64 characters (`+`, `/`, `=`) with URL-safe alternatives (`-`, `_`).

### Random String Generation

```typescript
generateBase62String(length: number): string
```

Generates a cryptographically random string using only alphanumeric characters (0-9, A-Z, a-z). Uses `crypto.randomBytes()` for secure randomness.

## Security Considerations

- **Cryptographic randomness**: All random segments use `crypto.randomBytes()`, suitable for security-sensitive operations
- **Collision resistance**: The 32-character random portion (8 + 24) provides ~192 bits of entropy
- **Team isolation**: Team IDs are encoded, not stored as plaintext, preventing enumeration

## Usage Example

```typescript
import {
  generateWebhookId,
  extractTeamId,
  isValidWebhookId,
  generateWebhookSecret,
} from "./incoming-webhooks";

// Create a webhook for a team
const teamId = "team_production_01";
const webhookId = generateWebhookId(teamId);
const webhookSecret = generateWebhookSecret();

// Later, validate and extract team info
if (isValidWebhookId(webhookId)) {
  const recoveredTeamId = extractTeamId(webhookId);
  // recoveredTeamId === teamId
}
```

## Dependencies

- `crypto` (Node.js built-in) — Cryptographic random byte generation
- `../id-helper` — Provides `newId()` for webhook secret generation