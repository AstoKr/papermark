# lib — middleware

# Middleware Module (`lib/middleware`)

This module contains Next.js middleware functions that intercept and process HTTP requests before they reach route handlers. Each middleware targets a specific domain of request processing: authentication, custom domains, webhooks, and analytics.

## Architecture Overview

The middleware layer sits at the edge of the application, processing requests before they reach API routes or page components. Each middleware handles a distinct concern:

```mermaid
flowchart TD
    Request["Incoming Request"] --> Route{Route Pattern}
    
    Route -->|"/services/*"| WebhookMW["IncomingWebhookMiddleware"]
    Route -->|PostHog paths| PostHogMW["PostHogMiddleware"]
    Route -->|Custom domain| DomainMW["DomainMiddleware"]
    Route -->|Default app paths"| AppMW["AppMiddleware"]
    
    WebhookMW --> WebhookAPI["/api/webhooks/*"]
    PostHogMW --> PostHog["PostHog (EU)"]
    DomainMW --> DomainView["/view/domains/[domain]/[slug]"]
    AppMW --> AuthCheck{Authenticated?}
    
    AuthCheck -->|No| LoginPage["/login"]
    AuthCheck -->|Yes + new user| WelcomePage["/welcome"]
    AuthCheck -->|Yes + existing| NextHandler["Next.js Handler"]
```

## AppMiddleware (`app.ts`)

Handles authentication flow and session management for the main application.

### Authentication Guard

When a request arrives without a valid session token and the path isn't `/login`, the middleware redirects to the login page:

```typescript
if (!token?.email && path !== LOGIN_PATH) {
  const loginUrl = new URL(LOGIN_PATH, req.url);
  loginUrl.searchParams.set("next", nextPath);
  return NextResponse.redirect(loginUrl);
}
```

The `next` query parameter preserves the intended destination so users return to their original target after authentication.

### Query Parameter Preservation

Certain paths carry meaningful query parameters that must survive the login redirect:

| Path Pattern | Preserved Parameters |
|--------------|---------------------|
| `/auth/confirm-email-change` | Full search string |
| `/access/*` | Full search string |

Other paths only preserve the pathname portion in the `next` parameter.

### Next Parameter Normalization

The `normalizeNextPath` function (and its helper `isProtocolRelativePath`) validates and sanitizes the `next` parameter to prevent open redirect vulnerabilities:

```typescript
function normalizeNextPath(nextPath: string | null, requestUrl: string): string {
  // Handles single and double URL encoding
  for (let i = 0; i < 3; i += 1) {
    try {
      const decoded = decodeURIComponent(normalized);
      if (decoded === normalized) break;
      normalized = decoded;
    } catch { break; }
  }
  
  // Rejects non-absolute paths and cross-origin targets
  if (!normalized.startsWith("/") || isProtocolRelativePath(normalized)) {
    return DEFAULT_AUTH_REDIRECT_PATH;
  }
  
  const targetUrl = new URL(normalized, requestUrl);
  if (targetUrl.origin !== requestOrigin) {
    return DEFAULT_AUTH_REDIRECT_PATH;
  }
}
```

### New User Welcome Flow

Authenticated users whose account was created within the last 10 seconds are redirected to `/welcome` unless they arrived via an invitation link:

```typescript
if (
  token?.email &&
  token?.user?.createdAt &&
  new Date(token?.user?.createdAt).getTime() > Date.now() - 10000 &&
  path !== "/welcome" &&
  !isInvited
) {
  return NextResponse.redirect(new URL("/welcome", req.url));
}
```

### Authenticated Login Redirect

When an authenticated user accesses `/login`, they redirect to the `next` parameter (normalized) or `/dashboard`:

```typescript
if (token?.email && path === LOGIN_PATH) {
  const nextPath = normalizeNextPath(url.searchParams.get("next"), req.url);
  return NextResponse.redirect(new URL(nextPath, req.url));
}
```

## DomainMiddleware (`domain.ts`)

Manages routing for custom domains used as data room aliases.

### Root Path Handling

Requests to the domain root (`/`) check Redis for a configured redirect URL:

```typescript
if (path === "/") {
  const redirectUrl = await getDomainRedirectUrl(host);
  if (redirectUrl) {
    return NextResponse.redirect(new URL(redirectUrl, req.url), { status: 302 });
  }
  return NextResponse.redirect(new URL("https://www.papermark.com", req.url));
}
```

If no redirect is configured, requests fall through to the main Papermark domain.

### Path Protection

Blocked pathnames and file extensions (indicating direct file access attempts) return a 404 rewrite:

```typescript
if (BLOCKED_PATHNAMES.includes(path) || path.includes(".")) {
  url.pathname = "/404";
  return NextResponse.rewrite(url, { status: 404 });
}
```

### Custom Domain Rewrite

Valid requests for custom domains rewrite to the domain-specific view handler:

```typescript
url.pathname = `/view/domains/${host}${path}`;
return NextResponse.rewrite(url, {
  headers: {
    "X-Robots-Tag": "noindex",
    "X-Powered-By": "Papermark - Secure Data Room Infrastructure for the modern web",
  },
});
```

The `X-Robots-Tag: noindex` header prevents search engines from indexing content accessed through custom domains.

## IncomingWebhookMiddleware (`incoming-webhooks.ts`)

Routes incoming webhook requests from third-party services.

### Path Rewriting

Requests matching `/services/*` rewrite to `/api/webhooks/services/*`:

```typescript
if (path.startsWith("/services/")) {
  url.pathname = `/api/webhooks${path}`;
  return NextResponse.rewrite(url);
}
```

### Host Detection

The exported `isWebhookPath` helper determines whether a request originates from the configured webhook base host:

```typescript
export function isWebhookPath(host: string | null) {
  if (!process.env.NEXT_PUBLIC_WEBHOOK_BASE_HOST) {
    return false;
  }
  return host === process.env.NEXT_PUBLIC_WEBHOOK_BASE_HOST;
}
```

This function is used by the main `middleware.ts` to determine which middleware stack should process the request.

## PostHogMiddleware (`posthog.ts`)

Proxies PostHog analytics requests through the application infrastructure.

### Route Selection

Selects the appropriate PostHog endpoint based on the request path:

```typescript
const hostname = url.pathname.startsWith("/ingest/static/")
  ? "eu-assets.i.posthog.com"
  : "eu.i.posthog.com";
```

### Header Forwarding

Only specific headers pass through to PostHog, preventing sensitive application headers from leaking:

```typescript
const allowedHeaders = [
  "accept", "accept-encoding", "accept-language", "authorization",
  "content-type", "content-length", "user-agent", "referer", "origin",
  "forwarded", "x-forwarded-for", "x-forwarded-host", "x-forwarded-proto",
  "x-real-ip", "x-posthog-*",
];
```

### CORS Preflight

OPTIONS requests receive a 200 response without modification, allowing the PostHog SDK to complete CORS handshakes:

```typescript
if (req.method === "OPTIONS") {
  return new NextResponse("", { status: 200 });
}
```

## Security Considerations

### Open Redirect Prevention

`AppMiddleware` validates the `next` parameter by:
1. Decoding up to three times to handle double-encoding attempts
2. Requiring paths to start with `/`
3. Rejecting protocol-relative URLs (`//evil.com`)
4. Ensuring the final origin matches the request origin

### Cross-Site Request Forgery

The middleware preserves query parameters for specific sensitive paths (`/access/*`, `/auth/confirm-email-change`) to maintain the integrity of invitation links and email change confirmations during authentication.

### Domain Isolation

`DomainMiddleware` prevents directory traversal attacks through file extension blocking and ensures custom domain content isn't indexed by search engines.

## Dependencies

| Function | External Dependency |
|----------|---------------------|
| `getToken` | `next-auth/jwt` |
| `getDomainRedirectUrl` | `@/lib/api/domains/redis` |
| `BLOCKED_PATHNAMES` | `@/lib/constants` |
| `NEXTAUTH_SECRET` | Environment variable |
| `NEXT_PUBLIC_WEBHOOK_BASE_HOST` | Environment variable |