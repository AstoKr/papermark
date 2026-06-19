# Findings Summary — Papermark

**Workflow:** whitebox-bug-finder
**Target:** papermark (worktree `task-whitebox-papermark`)
**Date:** 2026-06-19
**Scope:** All `findings/*.md` (28 VALID) + `methodology-raw/05-chains.md` (8 chains) + `methodology-raw/06-validated.md` (severity audit)

---

## Executive Summary

**Total validated findings:** 28 (HIGH-confidence, source-verified)

### Severity Distribution

| Severity | Count | Notes |
|----------|-------|-------|
| **CRITICAL** | 6 | Includes F-004 architectural root-cause (subsumes F-001, F-002, F-003) |
| **HIGH** | 10 | Including 2 latent (defense-in-depth) and 1 web-confirmed CVE |
| **MEDIUM** | 9 | Includes 3 latent and 1 range-pinning defense-in-depth |
| **LOW** | 3 | Includes 1 dead-code, 1 latent, 1 UX-only |
| **Total** | **28** | |

### Chains Detected: 8 (7 VALID, 1 weakened)

| Chain | Combined Impact | Bounty Est. | Status |
|-------|-----------------|-------------|--------|
| C-1 Published-placeholder 4-purpose secret → universal ATO | Forge NextAuth JWT, decrypt Jackson DB, forge SAML `code`, log in as any user | **$40K–$80K** | VALID |
| C-2 OAuth email linking → ATO + cross-tenant IDOR | ATO any user → cross-team folder destruction | $15K–$40K | VALID (caveat) |
| C-3 CORS misconfig + PDF/Notion CVE → dataroom file-injection XSS | Credentialed cross-origin arbitrary JS in viewer session | $20K–$50K | VALID (conditional) |
| C-4 Recon oracle + missing-auth progress token | Enumerate `documentVersionId`, mint Trigger.dev tokens | $8K–$20K | VALID |
| C-5 Folder IDOR + SAML ATO → total cross-tenant destruction | ATO + delete all folders/docs in target tenant | $20K–$50K | VALID |
| C-6 QStash bypass + nodemailer/systeminformation (self-hosted) | Mass email + conditional RCE/file-read | $10K–$25K | VALID (conditional) |
| C-7 INTERNAL_API_KEY timing attack + 26-route blast | Recover bearer → 26 internal endpoints accessible | $8K–$20K | VALID (weakened link) |
| C-8 CSV injection chain → admin XSS/DDE via exported Q&A | Formula injection in admin Excel export | $3K–$10K | VALID |

### Top 3 Highest-Impact Findings

1. **F-004 — Published-Placeholder 4-Purpose `NEXTAUTH_SECRET` (CRITICAL, CVSS 9.8–10.0)**
   Single public string (`my-superstrong-secret` in `.env.example:1`) unlocks four security boundaries. Architectural defect, trivially exploitable on any unrotated deployment.
2. **F-008 — tus-viewer CORS Reflects Origin with Credentials (CRITICAL, CVSS 8.2)**
   Textbook CORS misconfig on a credentialed endpoint. Independently disclosed (GH #2178). Chain amplifies to 9+ via transitive pdfjs-dist / dompurify CVEs.
3. **F-005 — Folder DELETE IDOR by `folderId` only (CRITICAL)**
   Independently disclosed (GH #2078, open 4+ months). Single-fetch by `folderId` enables cross-tenant destructive IDOR; chains with SAML ATO (F-013) for full cross-tenant destruction.

### Submission Priority at a Glance

1. **Lead with chains** — chains pay more than individual findings and demonstrate deeper analysis.
2. **Recommended submission order:** C-1 → C-3 → C-5 → C-2 → C-4 → C-6 → C-7 → C-8.
3. **Top individual findings:** F-004 (CRITICAL, leads C-1) → F-008 (CRITICAL, leads C-3) → F-005/F-006/F-007 (CRITICAL IDOR cluster).
4. **Estimated total bounty (if all chains + criticals accepted):** **$124K–$275K** (sum of chain bounty estimates, with overlap discount).

---

## Findings Ranked by Priority

Rank: severity → exploitability → chain potential → novelty.

| # | ID | Title | Severity | CVSS | Chain? | File |
|---|-----|-------|----------|------|--------|------|
| 1 | F-004 | Published-Placeholder 4-Purpose NEXTAUTH_SECRET | CRITICAL | 9.8–10.0 | YES (C-1, leads) | [findings/001-published-placeholder-4purpose-secret.md](001-published-placeholder-4purpose-secret.md) |
| 2 | F-001 | NEXTAUTH_SECRET reused as Jackson DB encryption key with empty fallback | CRITICAL | 9.1 | YES (C-1) | (subsumed by F-004 file) |
| 3 | F-008 | tus-viewer CORS Reflects Origin with Credentials | CRITICAL | 8.2 | YES (C-3, leads) | [findings/005-tus-viewer-cors-misconfig.md](005-tus-viewer-cors-misconfig.md) |
| 4 | F-005 | Folder DELETE IDOR by `folderId` only (GH #2078) | CRITICAL | 8.2 | YES (C-2, C-5) | [findings/002-folder-delete-idor.md](002-folder-delete-idor.md) |
| 5 | F-006 | Folder RENAME IDOR by `folderId` only | CRITICAL | 7.5 | YES (C-2, C-5) | [findings/003-folder-rename-idor.md](003-folder-rename-idor.md) |
| 6 | F-007 | Folder Add-to-Dataroom IDOR | CRITICAL | 7.5 | YES (C-5) | [findings/004-folder-add-to-dataroom-idor.md](004-folder-add-to-dataroom-idor.md) |
| 7 | F-002 | NEXTAUTH_SECRET used as SAML/Jackson OAuth clientSecret | HIGH | 8.1 | YES (C-1) | (subsumed by F-004 file) |
| 8 | F-027 | pdfjs-dist@3.11.174 arbitrary JS execution (CVE) | HIGH | 8.8 | YES (C-3) | [findings/024-pdfjs-dist-cve.md](024-pdfjs-dist-cve.md) |
| 9 | F-009 | allowDangerousEmailAccountLinking: true on Google/LinkedIn/SAML | HIGH | 8.1 | YES (C-2, C-5) | [findings/006-allow-dangerous-email-account-linking.md](006-allow-dangerous-email-account-linking.md) |
| 10 | F-010 | QStash signature verification no-op when VERCEL !== "1" | HIGH | 7.5 | YES (C-6, leads) | [findings/007-qstash-bypass.md](007-qstash-bypass.md) |
| 11 | F-011 | INTERNAL_API_KEY jobs use non-constant-time comparison | HIGH | 7.5 | YES (C-7, leads) | [findings/008-internal-api-key-timing-attack.md](008-internal-api-key-timing-attack.md) |
| 12 | F-013 | SAML CredentialsProvider upserts user row from IdP-controlled userinfo | HIGH | 8.1 | YES (C-2, C-5) | [findings/010-saml-upsert-ato.md](010-saml-upsert-ato.md) |
| 13 | F-014 | Missing auth on `/api/progress-token` mints Trigger.dev tokens | HIGH | 7.5 | YES (C-4, leads) | [findings/011-missing-auth-progress-token.md](011-missing-auth-progress-token.md) |
| 14 | F-015 | Missing auth on `/api/document-processing-status` (recon oracle) | HIGH | 6.5 | YES (C-4, C-5) | [findings/012-missing-auth-document-processing-status.md](012-missing-auth-document-processing-status.md) |
| 15 | F-016 | AES-256-CTR without auth tag on document passwords | HIGH | 7.5 | YES (C-7, weakened) | [findings/013-aes-ctr-no-auth.md](013-aes-ctr-no-auth.md) |
| 16 | F-024 | welcomeMessage stored without server-side sanitization | HIGH | 7.5 | YES (C-8) | [findings/021-welcome-message-no-sanitize.md](021-welcome-message-no-sanitize.md) |
| 17 | F-012 | dangerouslySetInnerHTML on prismaDocument.name (latent stored XSS) | HIGH (latent) | 6.1 | NO | [findings/009-latent-xss-prisma-document-name.md](009-latent-xss-prisma-document-name.md) |
| 18 | F-003 | Placeholder values in `.env.example` realistic enough to ship | MEDIUM | 5.3 | YES (C-1) | (subsumed by F-004 file) |
| 19 | F-021 | CSV injection in dataroom index CSV export | MEDIUM | 5.4 | YES (C-8, leads) | [findings/018-csv-injection-dataroom-index.md](018-csv-injection-dataroom-index.md) |
| 20 | F-022 | CSV injection in team-conversations Q&A export | MEDIUM | 5.4 | YES (C-8) | [findings/019-csv-injection-conversations-export.md](019-csv-injection-conversations-export.md) |
| 21 | F-025 | Agreement.name write path skips sanitizePlainText | MEDIUM (latent) | 4.3 | YES (C-8) | [findings/022-agreement-name-no-sanitize.md](022-agreement-name-no-sanitize.md) |
| 22 | F-017 | DOM-XSS surface on domainJson.apexName (DNS-label constrained) | MEDIUM (latent) | 4.3 | NO | [findings/014-dom-xss-apex-name.md](014-dom-xss-apex-name.md) |
| 23 | F-020 | CSV injection (formula + delimiter) in client-side downloadCSV | MEDIUM | 4.3 | NO | [findings/017-csv-injection-download-csv.md](017-csv-injection-download-csv.md) |
| 24 | F-026 | sanitize-html@^2.17.3 allows downgrade (CVE-2026-44990) | MEDIUM | 5.3 | NO | [findings/023-sanitize-html-range-downgrade.md](023-sanitize-html-range-downgrade.md) |
| 25 | F-028 | REVALIDATE_TOKEN passed as URL query parameter | MEDIUM | 4.3 | NO | [findings/025-revalidate-token-secret-in-url.md](025-revalidate-token-secret-in-url.md) |
| 26 | F-018 | XXE-adjacent risk in Python DOCX sanitizer | LOW (latent) | 2.7 | NO | [findings/015-xxe-python-docx-sanitizer.md](015-xxe-python-docx-sanitizer.md) |
| 27 | F-019 | $executeRawUnsafe savepoint name (dead-code) | LOW | 2.7 | NO | [findings/016-execute-raw-unsafe-savepoint.md](016-execute-raw-unsafe-savepoint.md) |
| 28 | F-023 | Non-RFC-5987 Content-Disposition on S3 presigned URL | LOW | 2.7 | NO | [findings/020-content-disposition-non-rfc5987.md](020-content-disposition-non-rfc5987.md) |

### Additional conditional findings (HIGH/MODERATE, dependent on additional verification)

17 conditional findings (F-029 through F-045) are tracked in `methodology-raw/06-validated.md` but excluded from this ranked table because each requires a precondition (grep for `raw:`, host-metrics init, etc.) to confirm reachability. They become reportable as amplifiers once the precondition is verified — see Chains 6 and 7.

---

## Chains Summary

### C-1: Published-placeholder 4-purpose secret → universal ATO

- **Findings involved:** F-004 (leads; subsumes F-001, F-002, F-003)
- **Bounty estimate:** $40,000 – $80,000
- **Combined impact:** Single public string (`my-superstrong-secret` in `.env.example:1`) unlocks four security boundaries: NextAuth JWT signing, Jackson DB encryption at rest, Jackson OAuth `client_secret`, and SAML NextAuth `clientSecret`. Any unrotated deployment is fully compromised.
- **PoC summary:**
  1. Read `.env.example:1` (public repo).
  2. Derive Jackson encryption key from the same string.
  3. Decrypt all Jackson DB rows → all enterprise IdP connections, OAuth tokens, SCIM data.
  4. Forge valid `code` parameter → complete OAuth handshake → log in as any user.
  5. (Alt) Forge NextAuth session JWT directly with the same string.
- **Reporting strategy:** Single root-cause report citing the 4-consumer architectural defect. Lead with the published-placeholder angle.

### C-2: OAuth email account linking → ATO + cross-tenant IDOR

- **Findings involved:** F-009 (allowDangerousEmailAccountLinking), F-005/F-006 (Folder IDOR cluster), F-013 (SAML upsert ATO), F-015 (recon oracle)
- **Bounty estimate:** $15,000 – $40,000
- **Combined impact:** ATO of any registered Papermark user → cross-tenant folder destruction via the folder DELETE IDOR (resource fetch keyed by `folderId` only).
- **PoC summary:**
  1. Attacker creates personal Google account at `victim@corp.com` (or exploits SAML ATO via F-013).
  2. Initiates OAuth flow on `app.papermark.com`; NextAuth's dangerous-linking code path matches the existing `prisma.user` row by email.
  3. Attacker is now logged in as the victim.
  4. Uses victim's team membership + folder IDOR to delete/rename folders in OTHER teams (resource fetch keyed by `folderId` only, no `teamId` scoping).
- **Caveat:** Google Workspace ATO claim is limited to personal Gmail or unverified Workspace domains. The SAML ATO pivot (F-013) is the unconditional path.
- **Reporting strategy:** Frame as NextAuth config footgun + cross-tenant IDOR pivot. The papermark-specific novelty is the lack of a `signIn` callback that would scope the linking.

### C-3: CORS misconfig + PDF/Notion renderer CVE → dataroom file-injection XSS

- **Findings involved:** F-008 (CORS, leads), F-027 (pdfjs-dist CVE)
- **Bounty estimate:** $20,000 – $50,000
- **Combined impact:** Credentialed cross-origin TUS upload → arbitrary JS execution in viewer's session via vulnerable pdfjs-dist (or dompurify via Notion path).
- **PoC summary:**
  1. Victim in active dataroom session (cookie `pm_drs_<linkId>`).
  2. Attacker hosts `evil.com` with payload using `fetch()` with `credentials: 'include'`.
  3. CORS preflight succeeds (F-008 reflects Origin + sets credentials).
  4. Attacker injects malicious PDF into the victim's dataroom session via TUS PATCH.
  5. Victim opens the injected file → `pdfjs-dist@3.11.174` CVE GHSA-wgrm-67xf-hhpq → arbitrary JS executes in victim's browser.
- **Caveat:** The chain requires either a known upload-id or a shared link the attacker also has. Reporting should frame this as "credentialed cross-origin interaction with in-progress uploads" — not "silent file injection" without the precondition.
- **Reporting strategy:** Lead with the chain (the CVE pivot is the novel amplifier). The CORS alone is a known duplicate (GH #2178).

### C-4: Recon oracle + missing-auth progress token

- **Findings involved:** F-015 (recon oracle), F-014 (progress token mint)
- **Bounty estimate:** $8,000 – $20,000
- **Combined impact:** Enumerate valid `documentVersionId` values via the recon oracle → mint Trigger.dev public access tokens for each. The recon oracle converts the "valid ID required" precondition into "no precondition."
- **PoC summary:**
  1. `GET /api/teams/<any>/documents/document-processing-status?documentVersionId=<id>` → 200/404 oracle (no auth).
  2. `GET /api/progress-token?documentVersionId=<id>` → `{ publicAccessToken }` (no auth).
  3. Use the token against Trigger.dev's public-API surface for that version.
- **Reporting strategy:** Single bug report. This is the natural follow-up to the recent auth sweep at commit `9db42913` — exactly the kind of "adjacent endpoint missed by the sweep" that bounty programs reward.

### C-5: Folder IDOR + SAML ATO + missing-auth recon → total cross-tenant destruction

- **Findings involved:** F-005, F-006 (Folder DELETE/RENAME IDOR), F-007 (Add-to-Dataroom IDOR), F-013 (SAML upsert), F-015 (recon oracle)
- **Bounty estimate:** $20,000 – $50,000
- **Combined impact:** SAML ATO → ADMIN role on tenant A → cross-tenant folder IDOR destruction on tenant B → cascade deletes child folders, documents, and S3 file versions. The recon oracle provides the `folderId` enumeration that turns "blind delete one folder" into "enumerate and destroy all folders."
- **PoC summary:**
  1. ATO via SAML `userinfo` upsert (F-013) — no `signIn` callback scopes the upsert.
  2. Become ADMIN on a "burner" tenant.
  3. Discover target tenant's folderIds via recon oracle (F-015) or share-URL scraping.
  4. `DELETE /api/teams/<burnerTenantId>/folders/manage/<targetFolderId>` — resource fetch keyed by `folderId` only.
  5. Cascade delete.
- **Reporting strategy:** Lead with the SAML ATO pivot; the IDOR alone is a known duplicate (GH #2078).

### C-6: QStash bypass + nodemailer `raw:` bypass + systeminformation RCE → self-hosted mass email + SSRF/file-read

- **Findings involved:** F-010 (QStash bypass, leads), F-030 (nodemailer, dependent), F-031 (systeminformation RCE, dependent)
- **Bounty estimate:** $10,000 – $25,000 (self-hosted only)
- **Combined impact:** On self-hosted deployments (per GH Issue #2160), an unauthenticated attacker triggers arbitrary welcome emails (email harvest/bombing) and forces domain-cron iteration. If `nodemailer raw:` is used or `host-metrics` is initialized in the otel SDK, the chain amplifies to file-read (SSRF, AWS credential exfiltration) or RCE.
- **PoC summary:**
  1. Identify self-hosted target (deployment where `process.env.VERCEL !== "1"`).
  2. `POST /api/cron/welcome-user` with `{ userId: "<victimUserId>" }` — no Upstash-Signature required.
  3. Email-bomb via repeat or domain-cron chain.
  4. (Dependent) `nodemailer raw:` → file-read RCE; (dependent) `systeminformation` → RCE.
- **Caveat:** The dependent CVEs require grep verification of `lib/email/` and `instrumentation.ts` / `sentry.*.config.ts` before reporting. If neither is reachable, the chain degrades to mass-email DoS only.
- **Reporting strategy:** Self-hosted-only, conditional report. The QStash bypass (F-010) is independently reportable; the dependent CVEs are amplifiers.

### C-7: INTERNAL_API_KEY timing attack + 26-route blast radius + AES-CTR no-auth

- **Findings involved:** F-011 (timing-unsafe, leads), F-016 (AES-CTR, weakened link)
- **Bounty estimate:** $8,000 – $20,000 (downgraded from original estimate)
- **Combined impact:** Timing-attack on any of the 4 jobs routes recovers the shared `INTERNAL_API_KEY` (~30 min from a single VM). The token unlocks 26 internal endpoints: notification redirect (phishing), presigned S3 URL generation (theft), batch processing (DoS), link creation (spam), billing cancellation (financial).
- **PoC summary:**
  1. Time the 4 jobs routes (~8K requests, ~30 min) → recover bearer token.
  2. Use the token against the 26 internal endpoints (notification redirect, batch DoS, presigned URL mint, link spam).
  3. (Weakened) AES-CTR password-forgery link requires an additional primitive not provided by this chain — report separately if reachable.
- **Weakness:** The original chain's AES-CTR password-forgery step assumed DB write access; the presigned URL flow gives S3 read access, not DB write. The chain should be reported as timing-attack → 26-route blast radius, with AES-CTR as a related defense-in-depth concern.
- **Reporting strategy:** Report the timing-attack + 26-route blast radius as a standalone finding (medium-high severity, novel combination). AES-CTR is a separate defense-in-depth concern.

### C-8: CSV injection chain → admin XSS / DDE via exported Q&A

- **Findings involved:** F-021 (dataroom index CSV, leads), F-022 (Q&A export), F-024 (welcomeMessage), F-025 (Agreement.name)
- **Bounty estimate:** $3,000 – $10,000
- **Combined impact:** Multiple paths to plant a CSV-injection payload that fires when a team admin opens the export in Excel: via `welcomeMessage` (HIGH severity standalone), via `Agreement.name`, or via AI-generated Q&A answers. The first three legs are direct exploits.
- **PoC summary:**
  1. Attacker is a team admin (or ATOs admin via Chain 2/5).
  2. Plant payload via `welcomeMessage = "=WEBSERVICE(...)"`, poisoned Agreement.name, or poisoned Q&A submission.
  3. Team admin opens CSV export in Excel → formula fires → exfiltrate spreadsheet contents.
- **Caveat:** The theoretical 4th leg (CSV cell containing HTML → transitive XSS via pdfjs-dist/dompurify) does not work — Excel does not re-parse HTML in CSV cells.
- **Reporting strategy:** Defense-in-depth finding; nice-to-have for a single engagement but not the highest-impact chain.

---

## Submission Priority

Recommended submission order (lead with chains, highest impact first):

1. **C-1 — Published-placeholder 4-purpose secret** ($40K–$80K). Single public string, single PoC, architectural defect, $50K+ ceiling. Lead with this.
2. **C-3 — CORS + transitive CVE chain** ($20K–$50K). The CORS is web-confirmed (GH #2178); lead with the **chain** (the CVE pivot is the novel amplifier).
3. **C-5 — ATO + cross-tenant destruction** ($20K–$50K). IDOR portion is duplicate; the SAML ATO pivot is the novel contribution.
4. **C-2 — OAuth email linking ATO** ($15K–$40K). Reportable as a NextAuth config footgun; the cross-tenant IDOR pivot is the bonus.
5. **C-4 — Recon oracle + progress token** ($8K–$20K). Both endpoints are novel; the chain framing is the "reportable as one bug" structure.
6. **C-6 — QStash bypass + nodemailer/systeminformation** ($10K–$25K, self-hosted). Conditional on grep results for `raw:` and host-metrics init.
7. **C-7 — INTERNAL_API_KEY timing attack + 26-route blast** ($8K–$20K). Note AES-CTR is a separate defense-in-depth concern.
8. **C-8 — CSV injection chain** ($3K–$10K). Defense-in-depth; nice-to-have.

After chains, fill in with high-severity individual findings not already covered:
- F-024 (welcomeMessage no server sanitize, HIGH) — also reportable standalone as a defense-in-depth gap.
- F-012 (latent stored XSS on prismaDocument.name, HIGH latent) — reportable as defense-in-depth.
- F-017 (DOM-XSS on domainJson.apexName, MEDIUM latent) — bundle with F-012 as a "DOM-XSS sink cluster."

### Notes for the reporter

- **Already-public duplicates:** F-005 (GH #2078), F-008 (GH #2178). Reporting should lead with the **chain** (C-3, C-5) rather than the standalone finding to avoid duplicate-bounty issues.
- **Architectural subsumption:** F-001, F-002, F-003 are subsumed into F-004 (the 4-purpose architectural defect). Report F-004 alone; cite the consumers as evidence.
- **Conditional chains:** C-6 requires grep verification before submission. C-7 should be re-cast as timing-attack + blast radius without the AES-CTR password-forgery link.
- **Defense-in-depth cluster:** F-012 + F-017 are reported together as a "DOM-XSS sink cluster" rather than individually (per M-004 merge).
- **Low-severity standalone:** F-018, F-019, F-023 are not bounty-eligible; skip unless filing a defense-in-depth batch.

---

## PHASE_2_CHECKPOINT

- [x] All 28 finding files read (F-001 through F-028)
- [x] Chains read from `methodology-raw/05-chains.md` (8 chains)
- [x] Validation status read from `methodology-raw/06-validated.md` (28 VALID, 7 VALID chains, 1 weakened)
- [x] Summary ranked by priority (severity → exploitability → chain potential)
- [x] Chains summarized with combined impact and bounty estimates
- [x] Submission priority clear (C-1 → C-3 → C-5 → C-2 → C-4 → C-6 → C-7 → C-8, then fill-in high-severity individuals)
