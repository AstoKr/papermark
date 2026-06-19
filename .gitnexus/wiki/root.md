# Root

# Root Module

The root module contains the foundational configuration files that bootstrap the Papermark application. These files control how Next.js handles requests, configures the build pipeline, manages dependencies, and sets up the development environment.

## Overview

Papermark is a Next.js application that uses a layered configuration approach:

```
┌─────────────────────────────────────────────────────────────┐
│                     Vercel Edge Layer                        │
│                   (middleware.ts)                            │
├─────────────────────────────────────────────────────────────┤
│                   Next.js Application                        │
│            (next.config.mjs, pages, app/)                    │
├─────────────────────────────────────────────────────────────┤
│                   Build Configuration                        │
│     (package.json, tsconfig.json, tailwind.config.js)        │
└─────────────────────────────────────────────────────────────┘
```

## Request Routing Flow

The `middleware.ts` file intercepts all incoming requests and routes them based on the request path and host header:

```mermaid
flowchart TD
    A[Request] --> B{API Host?}
    B -->|Yes| C{Allowed v1 path?}
    C -->|Yes| D[Pass through to Next.js]
    C -->|No| E[/] --> F[Redirect to docs]
    E -->|Other| G[Return 404]
    B -->|No| H{Analytics path?}
    H -->|/ingest/*| I[PostHog Middleware]
    H -->|No| J{Webhook host?}
    J -->|Yes| K[Webhook Middleware]
    J -->|No| L{Custom domain?}
    L -->|Yes| M[Domain Middleware]
    L -->|No| N{Standard app path?}
    N -->|Yes| O[App Middleware]
    N -->|No| P{View route with valid path?}
    P -->|Yes| D
    P -->|No| Q[Rewrite to /404]
```

### Host-Based Routing

The application uses three distinct hosts:

| Host | Purpose |
|------|---------|
| `app.papermark.com` | Main application (dashboard, settings) |
| `api.papermark.com` | Public v1 API surface only |
| `mcp.papermark.com` | MCP (Model Context Protocol) endpoint |

### Path Matching Logic

The middleware matcher excludes internal paths:

```typescript
export const config = {
  matcher: [
    "/((?!api/|oauth/|mcp/?$|\\.well-known/|_next/|_static|vendor|_icons|_vercel|favicon.ico|sitemap.xml|robots.txt).*)",
  ],
};
```

- `api/` - API routes (handled separately)
- `oauth/` - OIDC provider endpoints (bypassed for consent UI)
- `mcp/?$` - MCP endpoint only (not subpaths like `/mcp-oauth/*`)
- `_next/`, `_static`, `_vercel` - Next.js internals

## Middleware Chain

### API Host Restriction

When requests arrive at `api.papermark.com`, the middleware enforces that only the public v1 API surface is accessible:

```typescript
const apiHost = process.env.NEXT_PUBLIC_API_BASE_HOST;
if (apiHost && requestHostname === apiHost) {
  if (path === "/v1" || path.startsWith("/v1/") || path === "/openapi.json") {
    return NextResponse.next();
  }
  if (path === "/") {
    return NextResponse.redirect("https://www.papermark.com/docs/api", 302);
  }
  return new NextResponse(null, { status: 404 });
}
```

### Analytics Paths

PostHog proxy requests (`/ingest/*`) are routed to the PostHog middleware for event collection.

### Incoming Webhooks

Webhooks identified by host are handled by `IncomingWebhookMiddleware` from `lib/middleware/incoming-webhooks.ts`.

### Custom Domains

User-configured custom domains trigger `DomainMiddleware` which serves their documents under their own domain.

### Blocked Pathnames

View routes (`/view/*`) are checked against a blocklist to prevent path traversal attacks:

```typescript
if (
  path.startsWith("/view/") &&
  (BLOCKED_PATHNAMES.some((blockedPath) => path.includes(blockedPath)) ||
    path.includes("."))
) {
  url.pathname = "/404";
  return NextResponse.rewrite(url, { status: 404 });
}
```

## Next.js Configuration

### Rewrites

The `next.config.mjs` rewrites configuration handles subdomain routing and OIDC discovery:

```typescript
async rewrites() {
  const afterFiles = [
    { source: "/oauth/:path*", destination: "/api/oauth/:path*" },
    { source: "/.well-known/openid-configuration", destination: "/api/oauth/.well-known/openid-configuration" },
    { source: "/.well-known/oauth-protected-resource", destination: "/api/well-known/oauth-protected-resource" },
  ];
  // ...
}
```

The `beforeFiles` rules handle subdomain-specific routing with `has` conditions:

```typescript
if (apiHost) {
  beforeFiles.push(
    { source: "/v1/:path*", destination: "/api/v1/:path*", has: [{ type: "host", value: apiHost }] },
    { source: "/oauth/:path*", destination: "/404", has: [{ type: "host", value: apiHost }] },
  );
}
```

### Security Headers

The headers configuration applies security headers to all routes and specific directives for embed routes:

```typescript
{
  source: "/view/:path*/embed",
  headers: [{ key: "Content-Security-Policy", value: "frame-ancestors *;" }],
}
```

Embed routes relax the `frame-ancestors` CSP directive to allow iframe embedding.

### Image Domains

The `prepareRemotePatterns()` function generates the allowed remote image domains from environment variables, supporting multiple CloudFront distributions and blob storage providers:

```typescript
function prepareRemotePatterns() {
  let patterns = [
    { protocol: "https", hostname: "assets.papermark.io" },
    { protocol: "https", hostname: "d2kgph70pw5d9n.cloudfront.net" },
    // ... more domains
  ];

  if (process.env.NEXT_PRIVATE_UPLOAD_DISTRIBUTION_HOST) {
    patterns.push({ protocol: "https", hostname: process.env.NEXT_PRIVATE_UPLOAD_DISTRIBUTION_HOST });
  }
  // ...
}
```

### External Packages

Several packages are marked as server-side externals or stubs:

```typescript
webpack: (config, { isServer }) => {
  if (isServer) {
    externals.push("oidc-provider", "koa");
  }
  config.resolve.alias = {
    ...config.resolve.alias,
    "@google-cloud/kms": false,
    "mongodb": false,
    "mysql": false,
  };
}
```

## Configuration Files

### package.json

The project requires Node.js >= 24 and includes:

| Category | Notable Dependencies |
|----------|---------------------|
| **Framework** | Next.js, React, TypeScript |
| **Auth** | NextAuth.js, oidc-provider, @teamhanko/passkeys |
| **Database** | Prisma, PostgreSQL |
| **Storage** | AWS S3 SDK, @tus/server, @vercel/blob |
| **UI** | Tailwind, shadcn/ui, Radix UI, Tremor |
| **Analytics** | PostHog, Tinybird |
| **Payments** | Stripe |
| **Email** | Resend, React Email |
| **PDF** | mupdf, pdf-lib, @libpdf/core |
| **AI** | OpenAI SDK, AI SDK |
| **Background Jobs** | Trigger.dev |
| **MCP** | @modelcontextprotocol/sdk |

### tsconfig.json

The TypeScript configuration uses bundler module resolution and includes the `ee/` directory for enterprise features:

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", "trigger.config.ts"]
}
```

### tailwind.config.js

Custom color palette includes the standard shadcn colors plus a `sidebar` color scheme for the application layout.

### trigger.config.ts

Background jobs are configured in the `lib/trigger` and `ee/**/lib/trigger` directories using Trigger.dev with Prisma, FFmpeg, and Python support.

## Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `NEXT_PUBLIC_API_BASE_HOST` | API subdomain (e.g., `api.papermark.com`) |
| `NEXT_PUBLIC_MCP_BASE_HOST` | MCP subdomain (e.g., `mcp.papermark.com`) |
| `NEXT_PUBLIC_APP_BASE_HOST` | App subdomain (e.g., `app.papermark.com`) |
| `NEXT_PUBLIC_BASE_URL` | Asset prefix for production CDN |
| `NEXT_PRIVATE_UPLOAD_DISTRIBUTION_HOST` | CloudFront distribution for uploads |
| `TINYBIRD_TOKEN` | Analytics data pipeline token |

## Development Workflow

```bash
# Install dependencies
npm install

# Generate Prisma client and run migrations
npm run dev:prisma

# Start development server
npm run dev

# Run with email preview (React Email)
npm run email

# Start Stripe webhook listener
npm run stripe:webhook

# Start Trigger.dev local development
npm run trigger:v4:dev
```

## Integration Points

The root module connects to several internal modules:

- **`lib/middleware/app`** - App middleware for authenticated routes
- **`lib/middleware/domain`** - Custom domain handling
- **`lib/middleware/incoming-webhooks`** - Webhook processing
- **`lib/middleware/posthog`** - Analytics proxy
- **`lib/constants`** - Blocked pathnames list
- **`prisma/schema`** - Database schema (via postinstall)