# Finding: QStash Signature Verification is a No-Op on Non-Vercel Deployments

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:L/I:L/A:L = 8.2 (unauthenticated attacker can invoke any cron route on self-hosted deployments; affects 6 routes including domain deletion + email floods).

## Affected Code
- File: `lib/cron/verify-qstash.ts`
- Line: 11-14 (`if (process.env.VERCEL !== "1") return;` — silently accepts any caller)
- Function: `verifyQstashSignature`
- Endpoint: All routes under `app/api/cron/*` (6 routes confirmed)

- File: `app/api/cron/domains/route.ts`
- Line: 24-34 (the route-level `if (process.env.VERCEL === "1") verify` check; no `else` branch)
- Function: `POST` handler
- Endpoint: `POST /api/cron/domains`

- Other affected cron routes (per `methodology-raw/01-structural-analysis.md` Finding 5):
  - `app/api/cron/year-in-review/route.ts`
  - `app/api/cron/welcome-user/route.ts`
  - `app/api/cron/dataroom-digest/daily/route.ts`
  - `app/api/cron/dataroom-digest/weekly/route.ts`

## Description
All cron routes call `verifyQstashSignature` to authenticate that the request came from QStash (Upstash's job scheduler). The function checks `process.env.VERCEL === "1"` and **returns early without verification** when the env flag is anything other than `"1"`. This is a dev escape hatch that ships to any non-Vercel deployment (self-hosted enterprise installs, staging, CI). The receiver is also instantiated with `currentSigningKey: process.env.QSTASH_CURRENT_SIGNING_KEY || ""` (`lib/cron/index.ts:13`) — an empty key passes trivially. The handler call sites in the routes also gate the `receiver.verify(...)` call behind the same `VERCEL === "1"` check, so the bypass is consistent across the stack.

On a self-hosted deployment, an unauthenticated attacker can invoke any of the 6 cron routes, triggering:
- Arbitrary welcome emails to any `userId` (email harvest, email bombing)
- Year-in-review queue flush
- Dataroom digest job drain
- Domain-cron iteration (sends escalating reminder emails; on day 30, calls `deleteDomain`)

## Proof of Concept

### Pre-conditions
- A self-hosted papermark deployment where `process.env.VERCEL !== "1"` (i.e. anything other than Vercel production).
- The deployment is reachable from the internet (typical for self-hosted enterprise installs per GH Issue #2160).

### Step-by-step exploitation

1. **Identify the target** — a self-hosted papermark deployment. The attacker's recon: look for the default welcome-user endpoint by hitting `POST /api/cron/welcome-user` and observing the response. If it returns 200 (instead of 401 with an `Upstash-Signature` check), the deployment is non-Vercel and the bypass is active.

2. **Trigger welcome emails** (email harvest + bombing):
   ```bash
   curl -X POST 'https://papermark.example.com/api/cron/welcome-user' \
     -H 'Content-Type: application/json' \
     -d '{"userId": "<victimUserId>"}'
   ```
   No `Upstash-Signature` header is required. `verify-qstash.ts:12` returns early when `VERCEL !== "1"`. The handler reads `userId` from the body and dispatches a welcome email to the victim's registered address. The 200 response confirms the email is valid (the email was accepted by the email provider).

3. **Email-bomb the victim** (repeat step 2 in a loop) or **trigger the domain-cron** for cascading damage:
   ```bash
   curl -X POST 'https://papermark.example.com/api/cron/domains' \
     -H 'Content-Type: application/json' \
     -d '{}'
   ```
   The handler iterates all domains in the database. For each domain, it sends escalating reminder emails based on the domain's `expiryWarningSentAt` history. On day 30, it calls `deleteDomain` (line 24-34 of `app/api/cron/domains/route.ts`), permanently removing the domain and breaking every custom-domain link in the tenant.

4. **(Conditional — Chain 6 amplifier)** If any papermark email send path uses `nodemailer`'s `raw:` option (CVE GHSA-p6gq-j5cr-w38f), the attacker can craft a request that hits a `raw:`-based send and reads arbitrary files (`/proc/self/environ`, AWS credentials) or fetches arbitrary URLs (cloud metadata endpoints). Verify by `grep -r 'raw:' lib/email/ pages/api/`. If found, the chain is full file-read + SSRF.

5. **(Conditional — Chain 6 amplifier)** If `host-metrics` is initialized in the otel SDK (`grep -r 'host-metrics' instrumentation.ts sentry.*.config.ts`), the `systeminformation@5.23.8` CVE family enables RCE on the papermark server. The attacker controls the NM connection name via the networkInterface call site, which execSync's into `nmcli connection show "${connectionName}"`.

6. **Cascade** — once AWS credentials or `NEXTAUTH_SECRET` is exfiltrated, chain into F-001/F-004 (4-purpose NEXTAUTH_SECRET) for full ATO.

## Impact
- **Email bombing / phishing pivot** (any deployment): arbitrary welcome emails to any `userId`.
- **Domain deletion** (self-hosted): the domain-cron's `deleteDomain` call on day 30 breaks every custom-domain link in the tenant. An attacker can accelerate this by repeatedly triggering the cron — the `expiryWarningSentAt` timestamp gets updated on each call, but if the call is rapid-fire, the attacker can compress the 30-day warning cycle (verify the timestamp logic; if it's "days since `createdAt`", the attacker cannot accelerate; if it's "calls since `lastSentAt`", they can).
- **Resource exhaustion** (self-hosted): the year-in-review and dataroom-digest jobs each iterate large result sets and dispatch emails; an attacker can amplify email queue and DB load.
- **Amplifier for F-001 (4-purpose NEXTAUTH_SECRET)**: if the deployment uses the published placeholder, the attacker can chain into universal ATO.

## Evidence
```ts
// lib/cron/verify-qstash.ts:11-14
export const verifyQstashSignature = async ({
  req,
  rawBody,
}: {
  req: Request;
  rawBody: string;
}) => {
  // skip verification in local development
  if (process.env.VERCEL !== "1") {
    return;            // ← silently accepts any caller
  }
  ...
};
```
```ts
// app/api/cron/domains/route.ts:24-34
if (process.env.VERCEL === "1") {
  const isValid = await receiver.verify({ signature, body });
  if (!isValid) return new Response("Unauthorized", { status: 401 });
}
// ← no else branch: outside Vercel, anyone can POST, cron iterates domains
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 5)
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` §2.2)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 6)

## Chain Potential
- **Chain 6** (`methodology-raw/05-chains.md`) — the QStash bypass is the entry point; chains with `nodemailer@7.0.13` `raw:` CVE (F-030) and `systeminformation@5.23.8` RCE CVE (F-031) for full file-read / RCE on self-hosted deployments.
- **F-001/F-004** (4-purpose NEXTAUTH_SECRET) — chain into universal ATO once `NEXTAUTH_SECRET` is exfiltrated from `/proc/self/environ`.

## Remediation
1. **Replace the dev escape hatch** with an explicit `NODE_ENV` check:
   ```ts
   if (process.env.NODE_ENV !== "production") {
     return; // dev escape hatch — explicit
   }
   ```
   The current `VERCEL !== "1"` check is environment-specific and ships to self-hosted production deployments.

2. **Add a fail-fast guard** at module load:
   ```ts
   if (!process.env.QSTASH_CURRENT_SIGNING_KEY) {
     throw new Error("QStash signing key not configured");
   }
   ```
   Today the empty-key fallback (`|| ""`) silently passes signature verification with an empty key.

3. **Move verification to a real middleware** (`middleware.ts` with a matcher on `/api/cron/*`) so the dev escape hatch can't accidentally be bypassed by a missing call site.

4. **For each cron route, change the route-level check** from `if (process.env.VERCEL === "1") verify` to `if (process.env.NODE_ENV === "production") verify`. The current code is **inverted** — it says "verify only on Vercel" instead of "verify in production."
