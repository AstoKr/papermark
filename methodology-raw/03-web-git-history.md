# Web Search: Git-History Findings

**Node:** Layer-3 Git-History Web Research (M2 light tier)
**Date:** 2026-06-30
**Inputs:** `03-web-search-strategy.md` (queries G-01 through G-12), `02-synthesized-git-history.md`

---

## G-01 — Pattern: Auth Gap by Omission (S-001, S-004, S-006, S-007)

### CVE-2026-46339 — 9router RCE via Middleware Allowlist Gap
- **type:** CVE / technique
- **reference:** https://github.com/decolua/9router/security/advisories/GHSA-fhh6-4qxv-rpqj
- **upstream fix vs target fix:** N/A — upstream advisory; reflects the exact same pattern found in papermark (middleware `matcher` used an explicit allowlist protecting only 8 routes, leaving 40+ API routes completely unauthenticated). Papermark has a similar middleware at `middleware.ts` that only protects certain routes, while `conversations-route.ts` has zero auth on all handlers.
- **takeaway:** The pattern of "auth gap by omission" — routes added without middleware coverage — is a high-value, recurring bug class. Papermark's CRITICAL S-001 (conversations zero auth) is a text-book example.

### CVE-2026-44575 — Next.js Core: Middleware Bypass via .rsc / Segment-Prefetch
- **type:** CVE
- **reference:** https://github.com/vercel/next.js/security/advisories/GHSA-267c-6grr-h53f
- **upstream fix vs target fix:** N/A — framework-level CVE in Next.js itself. Papermark runs 14.2.35 (EOL, no fix available). This vulnerability means middleware-based auth in papermark can be bypassed entirely by appending `.rsc` to any protected URL.
- **takeaway:** Papermark's middleware-based protections are inherently bypassable because Next.js 14.2.35 is EOL and won't receive the fix. Every route that relies exclusively on middleware (versus inline `getServerSession`) is potentially exposed.

### Clerk createRouteMatcher Bypass (GHSA-vqx2-fgx2-5wq9)
- **type:** CVE / technique
- **reference:** https://github.com/clerk/javascript/security/advisories/GHSA-vqx2-fgx2-5wq9
- **upstream fix vs target fix:** N/A — papermark does not use Clerk. But the pattern is relevant: `createRouteMatcher` that implicitly whitelists unlisted routes is the same anti-pattern as the explicit allowlist matcher in middleware.ts.
- **takeaway:** Confirms that allowlist-style middleware matchers are a recurring security smell across the Next.js ecosystem.

---

## G-02 — Pattern: Sanitizer Decode Ordering (S-002)

### CVE-2026-40186 — sanitize-html allowedTags Bypass via Entity-Decoded Text
- **type:** CVE / technique
- **reference:** https://github.com/advisories/GHSA-9mrh-v2v3-xpfm
- **upstream fix vs target fix:** This is a vulnerability in the `sanitize-html` library itself (regression in 2.17.1). Papermark's `sanitizePlainText` bug is different — it's an application-level ordering bug (`sanitizeHtml` then `decodeHTML`), not a library flaw. However, the technique is identical: entity-encoded payloads survive tag stripping and become real HTML after entity decoding.
- **takeaway:** The `sanitizeHtml → decodeHTML` ordering bug in papermark is a known anti-pattern with multiple CVEs confirming the technique works. CVE-2026-40186 provides independent validation of the attack vector.

### CVE-2026-44990 — sanitize-html `<xmp>` Bypass (CVSS 9.5)
- **type:** CVE
- **reference:** Already documented in synthesized findings (S-002 mentions it). The `<xmp>` element bypass works in sanitize-html's default configuration.
- **upstream fix vs target fix:** This is a library-level CVE in sanitize-html. Papermark's `sanitizePlainText` has `allowedTags: []` (strips all tags), but the entity-decoding reordering means the `<xmp>` bypass compounds with the application-level bug.
- **takeaway:** Two independent sanitizer bypass techniques (entity decode ordering + `<xmp>` tag) converge on the same sink at `document-header.tsx`, amplifying the stored XSS risk.

---

## G-03 — Pattern: Tracking Endpoints → DB Vectors (S-006, S-007)

### claustra — Next.js Security Analyzer (C03 / C02 patterns)
- **type:** technique
- **reference:** https://socket.dev/npm/package/claustra
- **upstream fix vs target fix:** N/A — static analysis tool. The C03 rule ("webhook route handlers that read request body or perform DB writes without calling a recognized signature verifier") and C02 rule ("Server Actions that perform mutations without an authorization check") directly describe papermark's `record_reaction.ts` (S-006) and `feedback/index.ts` (S-007).
- **takeaway:** These patterns are known enough to warrant dedicated SAST rules. The fact that both endpoints write directly to PostgreSQL (unlike `record_view.ts`/`record_click.ts` which go to Tinybird) makes them particularly dangerous.

### Fluid Attacks: FLAT-H5LEF — LobeHub Unauthenticated SSRF
- **type:** writeup
- **reference:** https://db.fluidattacks.com/vul/FLAT-H5LEF/
- **upstream fix vs target fix:** LobeHub's `/webapi/proxy` endpoint accepted URLs via POST body without authentication — similar to papermark's tracking endpoints accepting data without auth. Fixed by adding `checkAuth()` middleware.
- **takeaway:** Reinforces that public POST endpoints that accept arbitrary data are a recurring bug bounty finding.

---

## G-04 — Pattern: EOL Framework Risk (S-003)

### Next.js 14 EOL — No Patches for 13+ CVEs
- **type:** CVE roundup
- **reference:** https://github.com/vercel/next.js/discussions/89551
- **upstream fix vs target fix:** Next.js 14.2.35 is EOL. The Vercel maintainer explicitly stated: *"Migrating to v15 is the way. v14 is end-of-life, that's why a patch was not shipped."* The following CVEs have NO FIX for v14:
  - CVE-2026-23869 (DoS in RSC, CVSS 7.5)
  - CVE-2026-23864 (DoS via RSC Server Functions)
  - CVE-2026-23870 (DoS in RSC)
  - CVE-2026-44578 (WebSocket-upgrade SSRF, CVSS 8.6)
  - CVE-2026-44576 (Cache poisoning of RSC responses)
  - CVE-2026-44575 (Middleware bypass via .rsc, CVSS 7.5)
  - CVE-2026-44574 (Dynamic route parameter injection, CVSS 8.1)
- **takeaway:** Papermark running 14.2.35 has zero protection against 7+ high/critical CVEs with no upgrade path short of migrating to v15. The EOL risk is not theoretical — multiple CVEs are publicly disclosed with exploit PoCs.

### CVE-2026-44578 — Next.js WebSocket Upgrade SSRF (CVSS 8.6)
- **type:** CVE / exploit PoC
- **reference:** https://github.com/dinosn/CVE-2026-44578
- **upstream fix vs target fix:** Public exploit PoC available. Requires raw TCP (not curl) to send absolute-form URIs. Self-hosted Next.js only. Papermark is self-hosted and on v14 → no fix available.
- **takeaway:** Pre-auth SSRF to localhost:80 with a public PoC. Papermark's SSRF module doesn't protect against this because it's a framework-level bypass, not an application-level fetch.

---

## G-05 — Pattern: CORS + TUS Upload (S-005)

### CVE-2026-57957 — Papermark TUS-based CORS Misconfiguration (!!!)
- **type:** CVE
- **reference:** https://cvetodo.com/cve/CVE-2026-57957
- **upstream fix vs target fix:** This is a CVE **against papermark itself** for the exact finding S-005. Describes papermark ≤ v0.22.0: "TUS-based viewer upload endpoint reflects arbitrary `Origin` headers with `Access-Control-Allow-Credentials: true`." Remediation: upgrade to v0.23.0+ with proper origin allowlisting.
- **takeaway:** **CONFIRMED CVE** — this is an independently validated vulnerability that matches the structural analysis finding S-005. The fix (origin allowlisting) is what papermark should apply.

### CVE-2026-42091 — goshs: PUT + Wildcard CORS + Missing CSRF
- **type:** CVE / technique
- **reference:** https://github.com/advisories/GHSA-RHF7-WVW3-VJVM
- **upstream fix vs target fix:** Different product but identical pattern: missing CSRF on upload methods, wildcard CORS. This CVE adds credibility to the TUS + CORS attack chain.
- **takeaway:** The combination of CORS origin reflection + credentialed requests + file upload is a known dangerous pattern with exploited CVEs.

### James Kettle — "Exploiting CORS Misconfigurations" (2017)
- **type:** technique
- **reference:** https://2017.appsec.eu/presos/Hacker/Exploiting%20CORS%20Misconfigurations%20for%20Bitcoins%20and%20Bounties%20-%20James%20Kettle%20-%20OWASP_AppSec-Eu_2017.pdf
- **upstream fix vs target fix:** Canonical research on origin reflection + credentialed requests. The `null` Origin technique via sandboxed iframes is directly applicable to papermark's TUS endpoint.
- **takeaway:** The CORS misconfiguration at `tus-viewer/[[...file]].ts:237` is a textbook vulnerability class with well-documented exploitation techniques.

---

## G-06 — Historical VFC: SSRF Pattern (Context)

### CVE-2026-44578 — Next.js WebSocket Upgrade SSRF (see G-04 above)
- **type:** CVE / technique
- **reference:** https://github.com/dinosn/CVE-2026-44578
- **upstream fix vs target fix:** Framework-level SSRF. Papermark's historical VFC (fixed) addressed application-level SSRF in webhook fetching. This CVE shows that the framework itself has a pre-auth SSRF that the application-level fix cannot protect against.
- **takeaway:** The historical SSRF fix was necessary but insufficient — Next.js 14 EOL means new SSRF vectors at the framework level remain unpatched.

---

## G-07 — Historical VFC: Open Redirect / callbackUrl (Context)

### CVE-2022-24858 — NextAuth Default Redirect Callback Open Redirect
- **type:** CVE / technique
- **reference:** https://github.com/nextauthjs/next-auth/security/advisories/GHSA-f9wg-5f46-cjmw
- **upstream fix vs target fix:** Fixed in next-auth v4.3.2. Papermark uses NextAuth.js 4.24.13 (much newer, unaffected). This CVE validates that the `callbackUrl` open redirect pattern is real and was previously exploited in the library.
- **takeaway:** Papermark's historical VFC for open redirect was in its own code (protocol-relative URL validation via `normalizeNextPath`). The library-level CVEs were fixed upstream already.

### CVE-2022-29214 — NextAuth OAuth 1 Open Redirect
- **type:** CVE / technique
- **reference:** https://github.com/nextauthjs/next-auth/security/advisories/GHSA-q2mx-j4x2-2h74
- **upstream fix vs target fix:** Fixed in v4.3.3. Not applicable to current papermark but provides historical context.
- **takeaway:** Multiple open redirect CVEs in NextAuth confirm this was a recurring attack surface.

---

## G-08 — Pattern: Middleware Matcher Exclusions (S-001)

### CVE-2026-44575 — Next.js Middleware Bypass (see G-01 above)
- **type:** CVE
- **reference:** https://securityboulevard.com/2026/05/cve-2026-44575-middleware-authorization-bypass-in-next-js-app-router/
- **upstream fix vs target fix:** Framework-level bypass affecting all self-hosted Next.js deployments. Papermark's v14 is EOL, no fix.
- **takeaway:** All middleware-based protections in papermark are suspect. Routes that lack inline auth (like S-001 conversations) are exposed, but even routes that *do* have middleware protection can be bypassed via the `.rsc` technique.

### CVE-2026-41248 — Clerk URL Decoding Matcher Bypass
- **type:** CVE / technique
- **reference:** https://github.com/clerk/javascript/security/advisories/GHSA-vqx2-fgx2-5wq9
- **upstream fix vs target fix:** URL encoding mismatch between middleware matcher and handler. Papermark doesn't use Clerk, but the technique (URL encoding to bypass path matching) is applicable to any middleware or route-based auth.
- **takeaway:** URL encoding bypass of route matchers is a known technique that papermark's middleware could be vulnerable to.

---

## G-09 — Pattern: XXE in Conversion Pipeline (S-011)

### Microsoft Markitdown XXE (Issue #1565)
- **type:** CVE / writeup
- **reference:** https://github.com/microsoft/markitdown/issues/1565
- **upstream fix vs target fix:** Microsoft's markitdown DOCX pre-processor uses `xml.etree.ElementTree.fromstring()` on untrusted XML — identical to papermark's `docx-sanitizer.py:146`. The recommendation is to replace with `defusedxml`.
- **takeaway:** The same unsafe pattern (`ET.parse` on untrusted DOCX XML) exists in a major Microsoft project. Confirms the finding is a real concern even if Python's ET is not vulnerable by default.

### CVE-2025-6984 — langchain-community EverNoteLoader XXE
- **type:** CVE
- **reference:** https://huntr.com/bounties/a6b521cf-258c-41c0-9edb-d8ef976abb2a
- **upstream fix vs target fix:** `lxml.etree.iterparse()` without disabling external entity references → local file disclosure. Papermark uses `xml.etree.ElementTree` not `lxml`, but the same principle applies: document conversion pipelines that parse user-uploaded XML are a known XXE surface.
- **takeaway:** Document conversion XXE is a recurring bug class across multiple projects (markitdown, langchain, changedetection.io, docling).

### CVE-2026-41895 — changedetection.io XXE
- **type:** CVE
- **reference:** https://github.com/advisories/GHSA-v7cp-2cx9-x793
- **upstream fix vs target fix:** `lxml.etree.XMLParser(strip_cdata=False)` without disabling external entities. Another example of the same pattern in a different tool.
- **takeaway:** Multiple recent CVEs confirm this is an actively exploited vulnerability class.

---

## G-10 — Pattern: TUS + CORS Cross-Domain (S-005)

### CVE-2026-57957 — Papermark TUS CORS (see G-05 above)
- **type:** CVE
- **reference:** https://cvetodo.com/cve/CVE-2026-57957
- **upstream fix vs target fix:** Already a published CVE against papermark. Confirms S-005 is independently validated.
- **takeaway:** This is the strongest signal in the entire git-history search — a confirmed CVE for the exact finding.

### TUS Protocol Security Considerations
- **type:** technique
- **reference:** https://tus.io/protocols/resumable-upload/0-1-x
- **upstream fix vs target fix:** The TUS spec explicitly acknowledges it does not define authentication or authorization — it relies entirely on the transport layer. This makes CORS configuration critical for browser-based TUS uploads.
- **takeaway:** Papermark's TUS endpoint reflects the origin with credentials=true. The protocol design makes this particularly dangerous because TUS provides no auth of its own.

### CVE-2026-32759 — TUS Negative Upload-Length
- **type:** CVE / technique
- **reference:** https://nvd.nist.gov/vuln/detail/cve-2026-32759
- **upstream fix vs target fix:** Different TUS server (File Browser), but validates that TUS implementation bugs are a real attack surface. Parsing `Upload-Length` as signed integer → negative value bypass.
- **takeaway:** TUS protocol implementations are security-sensitive and often have implementation vulnerabilities.

---

## G-11 — Stack-Level: Prisma ORM Patterns

### Mass Assignment via Prisma `.update()` / `.create()`
- **type:** technique / writeup
- **reference:** https://vibe-eval.com/patterns/mass-assignment/
- **upstream fix vs target fix:** N/A — general technique. Directly passing `req.body` to Prisma `.update({ data: req.body })` allows privilege escalation via extra fields like `"role": "admin"`.
- **takeaway:** Papermark's conversation API handlers that accept arbitrary body fields and pass them to Prisma should be audited for mass assignment. The unauthenticated `record_reaction.ts` (S-006) and `feedback/index.ts` (S-007) are the most likely candidates.

### ORM Leaks — Prisma Relational Filtering (elttam Research)
- **type:** technique
- **reference:** https://www.elttam.com/blog/plorming-your-primsa-orm
- **upstream fix vs target fix:** N/A — research technique. If user-controlled input reaches Prisma `where` clauses, attackers can use operators like `startsWith`, `contains`, `not` to extract data via timing side-channels.
- **takeaway:** Papermark's unauthenticated endpoints that accept filter/search parameters and pass them to Prisma queries could be vulnerable to ORM leak attacks. Time-based extraction of `viewerEmail` from `record_reaction.ts` responses is a concrete scenario.

---

## G-12 — Stack-Level: Multi-Finding Chaining

### CVE-2025-29927 — Next.js Middleware Bypass (x-middleware-subrequest)
- **type:** CVE / writeup
- **reference:** https://www.assetnote.io/resources/research/doing-the-due-diligence-analyzing-the-next-js-middleware-bypass-cve-2025-29927
- **upstream fix vs target fix:** Assetnote's deep-dive on CVE-2025-29927. The `x-middleware-subrequest` header forgery bypasses ALL middleware, including auth. Papermark's v14.2.35 is patched for this specific CVE (fix in 14.2.25+). However, the Assetnote research reveals that the fix is incomplete — the polyglot header technique can still bypass in some configurations.
- **takeaway:** Even patched CVEs have residual risk. The middleware bypass technique is a key enabler for chaining: bypass middleware → access unauthenticated routes → achieve data access or injection.

### CVE-2025-55182 — React2Shell: RCE via Prototype Pollution (CVSS 10)
- **type:** CVE / writeup
- **reference:** https://dev.to/devianntsec/cve-2025-55182-react2shell-rce-in-react-server-components-via-prototype-pollution-29pd
- **upstream fix vs target fix:** RCE in React Server Components (React 19.0.0–19.2.0) via `Object.prototype.then` poisoning. Papermark uses React 18.x so this specific CVE does not apply. However, it validates that RSC Flight protocol exploitation is a critical chain component.
- **takeaway:** While React2Shell doesn't affect React 18, the Next.js 14 RSC DoS CVEs (CVE-2026-23864/23869) do apply. The same RSC attack surface that enables React2Shell in React 19 enables DoS in Next.js 14.

### Eclipse on Next.js — Cache Poisoning + Race Condition Chain
- **type:** writeup
- **reference:** https://zhero-web-sec.github.io/research-and-things/eclipse-on-nextjs-conditioned-exploitation-of-an-intended-race-condition
- **upstream fix vs target fix:** Chains a race condition in Next.js's batcher mechanism with CDN misconfiguration to achieve cache poisoning → stored XSS. Papermark's EOL status means no framework-level fix is available for the underlying race condition.
- **takeaway:** Cache-related chaining is a realistic avenue: CDN misconfig + Next.js bugs → stored XSS. Papermark's self-hosted deployment likely has CDN in front.

### CVE-2025-29927 + Cache Poisoning Chain (Assetnote)
- **type:** writeup
- **reference:** https://slcyber.io/research-center/doing-the-due-diligence-analysing-the-next-js-middleware-bypass-cve-2025-29927/
- **upstream fix vs target fix:** The Assetnote research demonstrates chaining middleware bypass with cache poisoning. The `X-Nextjs-Data: 1` header leaks redirect information, enabling cache poisoning attacks against middleware-based auth.
- **takeaway:** Framework-level bypass (middleware) + application-level weakness (no inline auth) + cache/CDN = realistic multi-hop chain for papermark.

---

## Summary of Signal Strength

| Finding | Matches | Strength |
|---------|---------|----------|
| S-001 (conversations zero auth) | CVE-2026-46339 (identical pattern), CVE-2026-44575 (framework bypass), claustra C03 rule | **HIGH** — well-documented recurring pattern |
| S-002 (stored XSS) | CVE-2026-40186 (identical technique), CVE-2026-44990 (amplifying bypass) | **HIGH** — two independent CVEs confirm the technique |
| S-003 (Next.js DoS) | 7+ CVEs, all unpatched on v14 | **CRITICAL** — confirmed EOL, no fix path |
| S-004 (progress-token) | claustra C02 rule (unauthed token generation) | **MEDIUM** — pattern recognized by SAST tools |
| S-005 (CORS TUS) | **CVE-2026-57957 (papermark itself)** | **CONFIRMED** — independently published CVE |
| S-006 (record_reaction) | claustra C03 rule, mass assignment technique | **MEDIUM-HIGH** — technique well-documented |
| S-007 (feedback) | claustra C03 rule | **MEDIUM** — same pattern as S-006 |
| S-008 (revalidate secret) | No direct matches | **LOW** — technique is known but no specific CVEs |
| S-009 (helpText XSS) | General XSS via dangerouslySetInnerHTML | **LOW** — secondary sink, not independently validated |
| S-010 (webhook-events) | No direct matches | **LOW** — shiki escaping makes exploitation unlikely |
| S-011 (XXE docx-sanitizer) | Microsoft Markitdown #1565, CVE-2025-6984, CVE-2026-41895 | **MEDIUM** — same pattern in multiple projects |
| S-012 (ReDoS workflow) | No direct matches | **LOW** — requires admin access; bounded impact |
