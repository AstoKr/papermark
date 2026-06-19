# Finding: REVALIDATE_TOKEN Passed as URL Query Parameter (Secret-in-URL)

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N = 5.0 (low-impact target today, but the anti-pattern is repeated across 9 files; the secret can leak via server logs, Referer headers, CDN logs, browser history, screenshots, or stack traces).

## Affected Code
- File: `lib/api/links/revalidate.ts`
- Line: 19-21 (`await fetch(\`${revalidateUrl}/api/revalidate?secret=${revalidateToken}&linkId=${link.id}&hasDomain=${link.domainId ? "true" : "false"}\`)`)
- Function: `revalidateLinkById`, `revalidateLinksForPermissionGroup`
- Endpoint: `GET /api/revalidate?secret=<REVALIDATE_TOKEN>&linkId=<id>&hasDomain=<true|false>`

- Other call sites (8 additional files):
  - `pages/api/revalidate.ts:14` (the consumer)
  - `lib/trigger/pdf-to-image-route.ts:343`
  - `lib/trigger/convert-pdf-direct.ts:333`
  - `lib/api/documents/process-document.ts:296`
  - `pages/api/links/index.ts:491`
  - `pages/api/links/[id]/index.ts:743`
  - `pages/api/links/[id]/archive.ts:102`
  - `pages/api/teams/[teamId]/enable-advanced-mode.ts:126`

## Description
Every internal caller serializes `REVALIDATE_TOKEN` into the URL query string. Well-known leak vectors apply:
- **Server logs** — the URL with the secret is logged at the application level (Next.js default logging) and at the reverse-proxy level (Vercel logs, CloudFront logs, etc.).
- **Referer headers** — if the revalidated page links to an external resource (e.g. an analytics endpoint), the Referer header carries the full URL including the secret.
- **CDN logs** — Vercel's edge logs include the full URL.
- **Browser history** — if the revalidation is triggered from a browser-side script (not the case here, but possible in a future refactor).
- **Screenshots / screen shares** — a debug screenshot of a server log or browser dev tools would expose the secret.
- **Stack traces** — the URL is included in the stack trace of any error during the fetch.

The `/api/revalidate` endpoint itself is a low-impact target (DoS / cache stampede amplifier), but the anti-pattern matters because the same code shape is used elsewhere and could trivially be extended to higher-impact tokens (e.g. `INTERNAL_API_KEY`).

## Proof of Concept

### Pre-conditions
- The attacker has read access to server logs (e.g. via a compromised logging endpoint, a misconfigured log shipper, or a Vercel dashboard access).
- The revalidation flow has fired at least once.

### Step-by-step exploitation
1. Attacker gains read access to server logs (e.g. via a Vercel dashboard compromise, a CloudFront log leak, or a misconfigured Datadog/Sentry endpoint).
2. The attacker greps for `?secret=` and finds the `REVALIDATE_TOKEN` value.
3. The attacker can now call `GET /api/revalidate?secret=<recovered_token>&linkId=<any>&hasDomain=<any>` to force revalidation of any link.
4. **Impact**: the attacker can trigger a cache stampede on the link's page (forcing a re-render on every visitor), amplify load on the database and the page-rendering pipeline, and cause a DoS on the link's page.

## Impact
- **REVALIDATE_TOKEN leak** → cache stampede, DoS on the link's page.
- **Anti-pattern** — the same code shape could be extended to higher-impact tokens (`INTERNAL_API_KEY`, `QSTASH_CURRENT_SIGNING_KEY`, `NEXT_PRIVATE_VERIFICATION_SECRET`). A future maintainer who copy-pastes the pattern for a higher-impact token would ship a much more serious bug.
- **No active exploit today** — the `REVALIDATE_TOKEN` is not currently observed in source, and the leak vectors require a separate compromise (log access, dashboard access).

## Evidence
```ts
// lib/api/links/revalidate.ts:19-21
await fetch(
  `${revalidateUrl}/api/revalidate?secret=${revalidateToken}&linkId=${link.id}&hasDomain=${link.domainId ? "true" : "false"}`,
);
```
```ts
// pages/api/revalidate.ts:14
if (req.query.secret !== process.env.REVALIDATE_TOKEN) {
  return res.status(401).json({ message: "Invalid token" });
}
```

## Methodology
- **M2 — Secrets** (`methodology-raw/02-synthesized-secrets.md` S-005)

## Chain Potential
- None directly. The finding is a defense-in-depth / anti-pattern concern.

## Remediation
Move the token to an `x-revalidate-secret` HTTP header on both producer and consumer; deprecate the `?secret=` query path:
```ts
// Producer (lib/api/links/revalidate.ts)
await fetch(`${revalidateUrl}/api/revalidate?linkId=${link.id}&hasDomain=${link.domainId ? "true" : "false"}`, {
  headers: { "x-revalidate-secret": revalidateToken },
});

// Consumer (pages/api/revalidate.ts)
if (req.headers["x-revalidate-secret"] !== process.env.REVALIDATE_TOKEN) {
  return res.status(401).json({ message: "Invalid token" });
}
```
Apply the same fix to all 8 additional call sites.

Additionally, add a lint rule that rejects `?secret=`, `?token=`, `?key=`, `?api_key=` patterns in `fetch()` calls.
