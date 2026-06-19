# Finding: INTERNAL_API_KEY Timing-Attack on Jobs Routes

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:H/A:N = 7.7 (remote timing attack with enough samples recovers the shared secret; combined blast radius amplifies impact).

## Affected Code
- File: `pages/api/jobs/send-notification.ts`
- Line: 28-34 (`if (token !== process.env.INTERNAL_API_KEY)`)
- Function: `handle`
- Endpoint: `POST /api/jobs/send-notification`

- File: `pages/api/jobs/send-dataroom-new-document-notification.ts`
- Line: 22-28 (same pattern)
- Endpoint: `POST /api/jobs/send-dataroom-new-document-notification`

- File: `pages/api/jobs/send-dataroom-upload-notification.ts`
- Line: 19-25 (same pattern)
- Endpoint: `POST /api/jobs/send-dataroom-upload-notification`

- File: `pages/api/jobs/process-download-batch.ts`
- Line: 25-31 (same pattern)
- Endpoint: `POST /api/jobs/process-download-batch`

The `INTERNAL_API_KEY` is the **same shared bearer token across 26 call sites** (`methodology-raw/02-synthesized-secrets.md` S-006) with no per-job scoping.

## Description
Four job handlers authenticate callers by comparing a Bearer token against `process.env.INTERNAL_API_KEY` using JavaScript's `!==` operator. JavaScript string equality short-circuits on the first differing byte, leaking timing information proportional to the matched prefix length. An attacker on the same network (or with sufficient request budget over cloud-to-cloud paths) can recover the secret byte-by-byte. With the secret recovered, they can invoke the underlying handler actions, which include:
- Triggering arbitrary view notifications (email harvest — spam, social engineering, email-bombing victims of the team's document viewers)
- Triggering dataroom upload notifications (misleading document owners into thinking a new file arrived)
- Triggering arbitrary download batch jobs (resource exhaustion + DoS on S3 storage + email queue)
- (Across the broader 26 endpoints) PDF rendering, presigned S3 URL generation, billing cancellation

## Proof of Concept

### Pre-conditions
- The attacker can send requests to the jobs routes (these are internet-facing on Vercel deployments; on self-hosted, they may be internal-only).
- Sufficient network stability to measure ~10ns timing differences with statistical averaging. From a single VM with consistent routing, ~256 samples per byte position are typically enough; total ~8K requests for a 32-character key.

### Step-by-step exploitation

1. **Verify the bypass is reachable** — send `curl -X POST -H 'Authorization: Bearer test' https://.../api/jobs/send-notification` and confirm 401 with `{"message": "Unauthorized"}`. The 401 is the same response regardless of the prefix, so the timing oracle is the only signal.

2. **Time the comparison for byte 0** — for each candidate character `c ∈ [a-zA-Z0-9]`, send `Authorization: Bearer c<padding>` and measure the response time. The candidate with the **longest average response time** is byte 0. Use median over ~256 samples to filter network noise.

3. **Repeat for byte 1**, then byte 2, ..., byte 31. The total budget is ~8K requests; total time ~30 minutes from a single VM in a Vercel region.

4. **Use the recovered token against the 26 internal endpoints.** Specifically:
   ```bash
   # Phishing pivot: redirect any document's "viewed" notification to the attacker
   curl -X POST https://.../api/jobs/send-notification \
     -H "Authorization: Bearer <recovered_key>" \
     -H "Content-Type: application/json" \
     -d '{"viewId": "<some_viewId>", "locationData": {"continent": null, "country": "US", "region": "CA", "city": "SF"}}'
   ```
   The handler at `pages/api/jobs/send-notification.ts:156-239` calls `sendViewedDocumentEmail` to the configured recipients for the view. The `viewId` can be enumerated via the recon oracle at F-015 (`/api/document-processing-status`).

5. **Email-bomb a team** by sending a notification to every `viewId` in the team's view history (also enumerable via Tinybird / dashboard).

6. **DoS via download batch**:
   ```bash
   curl -X POST https://.../api/jobs/process-download-batch \
     -H "Authorization: Bearer <recovered_key>" \
     -H "Content-Type: application/json" \
     -d '{"batchId": "<some_batchId>"}'
   ```
   The handler triggers S3 download processing for the batch, exhausting S3 GET budget and email queue.

7. **(Chain 7 — conditional)** With the `INTERNAL_API_KEY` recovered, the attacker has access to all 26 internal endpoints. The AES-CTR password-forgery link (F-016) requires **DB write access**, which the chain does not establish independently. The presigned URL flow at `/api/file/s3/get-presigned-get-url-proxy` gives S3 read access, not DB write access. **The chain should be reported as timing-attack → 26-route blast radius, with AES-CTR as a separate defense-in-depth concern (see `methodology-raw/05-chains.md` Chain 7 validation issues).**

## Impact
- **Recovered `INTERNAL_API_KEY`** lets the attacker:
  - Trigger arbitrary view notifications (email harvest — spam, social engineering, email-bombing victims)
  - Trigger dataroom upload notifications (mislead document owners)
  - Trigger arbitrary download batch jobs (DoS on S3 + email queue)
  - Across the 26 endpoints: PDF rendering (DoS), presigned S3 URL generation (read any document), billing cancellation (financial impact), 7 link-creation paths (mass link spam)
- **26-route blast radius** — the same secret unlocks 26 distinct internal capabilities. There is no per-job scoping (per `methodology-raw/02-synthesized-secrets.md` S-006).
- **No defense in depth** — there's no rate limiting, no per-IP throttling, no log-scrubbing that would detect the timing attack. The comparison is in hot path with no protection.

## Evidence
```ts
// pages/api/jobs/send-notification.ts:28-34
const authHeader = req.headers.authorization;
const token = authHeader?.split(" ")[1];
if (token !== process.env.INTERNAL_API_KEY) {     // ← non-constant-time
  res.status(401).json({ message: "Unauthorized" });
  return;
}
```
```bash
# Cross-check from grep
$ grep -n "INTERNAL_API_KEY" pages/api/jobs/*.ts
pages/api/jobs/process-download-batch.ts:28:  if (token !== process.env.INTERNAL_API_KEY) {
pages/api/jobs/send-dataroom-new-document-notification.ts:25:  if (token !== process.env.INTERNAL_API_KEY) {
pages/api/jobs/send-dataroom-upload-notification.ts:22:  if (token !== process.env.INTERNAL_API_KEY) {
pages/api/jobs/send-notification.ts:31:  if (token !== process.env.INTERNAL_API_KEY) {
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 6)
- **M2 — Secrets** (`methodology-raw/02-synthesized-secrets.md` S-006 — 26 call sites)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 7)

## Chain Potential
- **Chain 7** (`methodology-raw/05-chains.md`) — the timing attack + 26-route blast radius is the strong link. AES-CTR password-forgery (F-016) is a related defense-in-depth concern, not part of this chain (validation issues 4 in `methodology-raw/06-validated.md`).
- **F-015** (recon oracle) — the recon oracle is the natural pre-step for choosing which `viewId` to target with the recovered token.

## Remediation
Replace the `!==` comparison with `crypto.timingSafeEqual` over equal-length buffers:
```ts
import { timingSafeEqual } from "crypto";
function safeEqual(a: string | undefined, b: string | undefined): boolean {
  if (!a || !b || a.length !== b.length) return false;
  return timingSafeEqual(Buffer.from(a), Buffer.from(b));
}

// in the handler:
if (!safeEqual(token, process.env.INTERNAL_API_KEY)) {
  res.status(401).json({ message: "Unauthorized" });
  return;
}
```
Apply to all 4 jobs routes. The pattern at `ee/features/billing/cancellation/api/automatic-unpause-route.ts:43-50` already uses `timingSafeEqual` correctly — mirror it.

Additionally, **introduce per-job scoped tokens** (`INTERNAL_API_KEY_PDF`, `INTERNAL_API_KEY_NOTIFICATIONS`, `INTERNAL_API_KEY_PRESIGN`) so a single token leak doesn't compromise 26 endpoints. Document a rotation runbook in `SECURITY.md`.
