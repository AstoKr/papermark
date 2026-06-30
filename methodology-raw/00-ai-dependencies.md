# AI Dependency / CVE Recon — papermark

## Finding Summary

| # | Package | Severity | CVE(s) | Used? | Reachable? |
|---|---------|----------|--------|-------|------------|
| 1 | `cuid` 3.0.0 | **HIGH** | — (design flaw) | YES (routes) | YES |
| 2 | `cookie` 0.4.2 (transitive via engine.io) | **MEDIUM** | CVE-2024-47764 | transitive | UNLIKELY |
| 3 | `xlsx` CDN-pinned | **MEDIUM** | — (supply chain) | YES (sheet parsing) | INDIRECT |
| 4 | `querystring` (deprecated Node built-in) | **LOW** | — (deprecated) | YES | NO |
| 5 | `eslint` 8.57.0 | **LOW** | — (EOL) | dev | NO |
| 6 | `oidc-provider` 9.8.3 | **LOW** | — (unused dep) | NO | NO |
| 7 | `@modelcontextprotocol/sdk` 1.29.0 | **LOW** | — (unused dep) | NO | NO |
| 8 | `tokenlens` 1.3.1 | **LOW** | — (unused dep) | NO | NO |
| 9 | `sanitize-html` 2.17.4 | **LOW** | — | YES (plaintext mode) | NO |
| 10 | `ms` 2.1.3 | **LOW** | — (patched) | YES (lib/utils) | NO |

---

## Detailed Findings

### 1. `cuid` 3.0.0 — Predictable ID Generation

- **severity**: HIGH
- **package**: cuid
- **version**: 3.0.0
- **vulnerable range**: all versions (design limitation)
- **CVE**: None assigned; documented design weakness
- **description**: CUIDs encode timestamps and use a sequential counter, making them
  predictable/guessable. Unlike `nanoid` (which uses `crypto.randomBytes`), an attacker
  who observes a few CUIDs can predict future values with high confidence.
- **used-in-code**: YES (3 import sites)
  - `pages/api/teams/[teamId]/datarooms/[id]/groups/[groupId]/permissions.ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/[permissionGroupId].ts:5`
  - `pages/api/teams/[teamId]/datarooms/[id]/permission-groups/index.ts:4`
- **reachable-from-user-input**: YES — used to generate IDs for permission groups
  and permission entries. These routes accept authenticated team-member requests,
  meaning a team member could predict permission group IDs and potentially enumerate
  or tamper with permission-group resources.
- **evidence**: `package.json` line 106 (`"cuid": "^3.0.0"` → resolved 3.0.0).
  The project also depends on `nanoid` 5.1.11 (secure), so migration is trivial.
- **remediation**: Replace `cuid` with `nanoid` (already a dependency) for all permission
  group ID generation.

---

### 2. `cookie` 0.4.2 — Cookie Prefix Injection (transitive)

- **severity**: MEDIUM
- **package**: cookie (transitive via engine.io → socket.io → @trigger.dev/core)
- **version**: 0.4.2
- **vulnerable range**: < 0.7.0
- **CVE**: CVE-2024-47764
- **description**: Cookie prefix injection vulnerability. The `cookie` package does not
  reject cookies with `__Host-` / `__Secure-` prefix when the cookie does not have the
  `domain` and `path` attributes set correctly. This can lead to cookie integrity bypass
  in certain configurations.
- **used-in-code**: Transitive dependency only
- **reachable-from-user-input**: UNLIKELY — Used by `engine.io` inside
  `@trigger.dev/core` for server-to-server WebSocket communication with the Trigger.dev
  cloud infrastructure. Not exposed to user HTTP requests directly.
- **evidence**: `package-lock.json` → `node_modules/engine.io/node_modules/cookie`
  version 0.4.2. Dependency chain: `@trigger.dev/sdk@4.4.6` → `@trigger.dev/core@4.4.6`
  → `socket.io@4.7.4` → `engine.io@~6.5.2` → `cookie@~0.4.1`.
- **remediation**: Monitor Trigger.dev SDK for updates that bump `engine.io` past v6.6.0
  (which already uses `cookie@~0.7.2`).

---

### 3. `xlsx` via SheetJS CDN — Supply Chain Risk

- **severity**: MEDIUM
- **package**: xlsx
- **version**: 0.20.3
- **vulnerable range**: N/A (supply chain)
- **CVE**: N/A
- **description**: The `xlsx` dependency is pinned to a full CDN URL rather than an npm
  registry version:
  ```
  "xlsx": "https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz"
  ```
  This means:
  - If the CDN is compromised, the build artifact is compromised — no integrity hash
    verification is enforced at install time.
  - If the CDN URL becomes stale or the domain is abandoned, installs will fail.
  - npm audit / SCA scanners cannot version-check packages installed from remote URLs.
  - The npm registry version (`xlsx@0.20.3`) is the same tarball mirrored to the
    registry, so switching to the registry version fixes this without changing behavior.
- **used-in-code**: YES (3 import sites)
  - `lib/utils/get-page-number-count.ts`
  - `lib/sheet/index.ts`
  - `ee/features/ai/lib/trigger/process-excel-for-ai.ts`
- **reachable-from-user-input**: INDIRECT — These functions are called during document
  upload processing (XLSX page-count extraction) and AI spreadsheet ingestion. An
  attacker who uploads a crafted spreadsheet triggers this code path, but the dependency
  itself is consumed at build time, not runtime.
- **evidence**: `package.json` line 175.
- **remediation**: Switch to `"xlsx": "^0.20.3"` (npm registry) or add a `resolutions`/
  `overrides` entry with an integrity check.

---

### 4. `querystring` — Deprecated Node.js Built-in

- **severity**: LOW
- **package**: querystring (Node.js built-in)
- **version**: N/A (Node built-in, deprecated since Node v21)
- **vulnerable range**: N/A
- **CVE**: N/A
- **description**: The `querystring` module is deprecated in Node.js and may produce
  unexpected behavior with special characters. It does not handle nested objects or
  arrays like the modern `URLSearchParams` or `qs`.
- **used-in-code**: YES — imported via `ParsedUrlQuery from "querystring"` in
  `lib/utils.ts:13`
- **reachable-from-user-input**: NO — The `ParsedUrlQuery` type import is only used
  for Next.js router type compatibility, not for runtime parsing of untrusted input.
- **evidence**: `lib/utils.ts:13`.
- **remediation**: Replace type import with `URLSearchParams` or inline the minimal type.

---

### 5. `eslint` 8.57.0 — End of Life

- **severity**: LOW
- **package**: eslint
- **version**: 8.57.0
- **vulnerable range**: all 8.x (EOL since Oct 2024)
- **CVE**: N/A (dev dependency)
- **description**: ESLint 8.x reached end of life. No more security patches will be
  released. However, since this is a dev dependency used only during development/lint,
  it does not affect production attack surface.
- **used-in-code**: Dev dependency only
- **reachable-from-user-input**: NO
- **evidence**: `package.json` line 110 (`"eslint": "8.57.0"`).
- **remediation**: Upgrade to ESLint 9.x.

---

### 6. Unused Dependencies — Increased Attack Surface

The following direct dependencies are declared in `package.json` but never imported
anywhere in the source tree:

#### `oidc-provider` 9.8.3
- **used-in-code**: NO — zero import/require sites found
- **evidence**: Confirmed by Grep for `oidc-provider` across all `.ts/.tsx/.js` files
- **risk**: Adds ~50 transitive packages including `koa`, `koa-router`, and their
  sub-dependencies. Increases the SBOM surface and the window for future vulns.
- **remediation**: Remove from `package.json` unless it is loaded dynamically (e.g.,
  via a feature flag that was not enabled in this scan).

#### `@modelcontextprotocol/sdk` 1.29.0
- **used-in-code**: NO — zero import/require sites found
- **evidence**: Confirmed by Grep for `@modelcontextprotocol` across all source files
- **risk**: Adds `express@^5.2.1`, `zod-to-json-schema`, and many other transitive deps.
  Possibly intended for a future MCP server integration but currently dead weight.
- **remediation**: Remove until the MCP integration is implemented.

#### `tokenlens` 1.3.1
- **used-in-code**: NO — zero import/require sites found
- **evidence**: Confirmed by Grep for `tokenlens` across all source files
- **remediation**: Remove.

---

## Override Analysis

The `overrides` block in `package.json` shows active awareness of transitive vulns:

| Override | Target Version | Notes |
|----------|---------------|-------|
| `node-forge` → 1.4.0 | Inside `@boxyhq/saml-jackson` | Fixes known CVEs; correct |
| `axios` → 1.16.0 | Inside `@boxyhq/saml-jackson` | Latest; correct |
| `@xmldom/xmldom` → 0.9.10 | Inside `@boxyhq/saml20@1.13.2` | Fixes multiple CVEs; correct |
| `protobufjs` → 8.0.3 | For `@opentelemetry/otlp-transformer@0.212.0` | Override did NOT take effect — resolved to 7.5.8 (narrowly scoped; target version not present) |
| `tar` → 7.5.10 | Global | Latest; correct |
| `lodash` → 4.18.1 | Global | Latest on npm; correct |

The overrides appear well-maintained and address the most common vulnerable transitive
deps. The `protobufjs` override is a no-op (too narrowly scoped) but 7.5.8 is secure.

---

## Dependency Profile Summary

- **Total direct dependencies**: ~150 production + ~20 dev
- **Total resolved packages**: 2,122 (including all transitive)
- **Language**: Node.js / TypeScript only (no Go, Python, Ruby)
- **Package manager**: npm with `package-lock.json` v3
- **Runtime**: Next.js 14.2.35 on Node.js ≥24

### Patched / Secure (notable)

| Package | Version | Notes |
|---------|---------|-------|
| next | 14.2.35 | All known CVEs patched (incl. CVE-2025-29927 middleware bypass) |
| next-auth | 4.24.14 | Latest on npm; all known CVEs patched |
| jsonwebtoken | 9.0.3 | All CVEs fixed in ≥9.0.1 |
| ms | 2.1.3 | CVE-2017-20162 fixed in ≥2.0.1 |
| ua-parser-js | 1.0.41 | All CVEs fixed in ≥1.0.40 |
| sanitize-html | 2.17.4 | Latest; usage is restricted to plaintext-strip mode |
| zod | 3.25.76 | Latest on npm; used pervasively for request validation |
| @upstash/ratelimit | 2.0.8 | Used for API rate limiting in Redis and fraud prevention |
| archiver | 7.0.1 | Latest; no reported CVEs |
| tar | 7.5.10 | Fixed via override; CVE-2023-44205 patched |
| marked (3 versions) | 15.0.12, 16.4.2, 17.0.6 | All up-to-date; no actionable vulns |
| axios | 1.16.0 | Latest; fixed via override |

---

## Flags for Downstream Stages

- `cuid` replacement → hand off to `synthesize-dependencies` for code-change analysis
- `xlsx` CDN → hand off to `synthesize-dependencies` for registry switch
- Unused deps (`oidc-provider`, `@modelcontextprotocol/sdk`, `tokenlens`) → hand off
  to `sbom-reachability` for confirmed dead-code elimination
- `protobufjs` stale override → note for `advisory-pairing`
- `eslint` EOL → low-priority tech debt (dev only)
