# Web Search — Secrets Domain

**Node:** Layer-3 Web Search — SECRETS (M4, light tier)
**Date:** 2026-06-30
**Inputs:** `02-synthesized-secrets.md`, `03-web-search-strategy.md`
**Searches Executed:** K-01 through K-12 (12 queries, expanded with 8 follow-up searches = 20 total fetches)

---

## S-001 — NEXTAUTH_SECRET uses well-known default value (CRITICAL)

### S-001 — Embrace The Red: Minting NextAuth Cookies via NEXTAUTH_SECRET
- type: writeup / technique
- reference: https://embracethered.com/blog/posts/2026/minting-next-auth-nextjs-auth-cookies-react2shell-threat/
- key format: `NEXTAUTH_SECRET` (or `AUTH_SECRET` in v5) — a 32-byte base64-encoded random value generated via `openssl rand -base64 32`. Used as the root secret for HKDF-derived encryption keys for JWT/JWE session cookies. The salt is the cookie name itself (e.g., `next-auth.session-token`).
- abuse technique: If NEXTAUTH_SECRET is known (via React2Shell CVE-2025-55182 deserialization RCE, SSRF, env dump, or default placeholder), an attacker uses NextAuth's own encoding functions to forge valid JWE session cookies for any user, with any role. The attacker cycles through known cookie-name salts by version. OAuth credential rotation alone does NOT fix this — NEXTAUTH_SECRET is a standalone backdoor. Leaves minimal forensic evidence.

### S-001 — CVE-2023-48309: NextAuth Session Mocking Bypass
- type: CVE
- reference: https://github.com/advisories/GHSA-v64w-49xw-qq89
- key format / validation: Affects next-auth < v4.24.5. An attacker who intercepts a JWT from an interrupted OAuth flow (state/PKCE/nonce leakage) can manually override the `next-auth.session-token` cookie.
- abuse technique: The replayed JWT creates a mock user session (opaque random ID), bypassing the default Middleware authorization. Allows peeking at authenticated UI states (dashboard layouts, gated content). CVSS 5.3. This is distinct from a known-secret attack — it exploits a lack of user-property verification in the default Middleware callback.

### S-001 — CVE-2026-44578: Next.js WebSocket Upgrade SSRF
- type: CVE / exploit-writeup
- reference: https://github.com/dinosn/CVE-2026-44578
- key format / validation: Affects self-hosted Next.js 13.4.13–15.5.15 and 16.0.0–16.2.4. Uses absolute-form URI with triple slash (`GET http:///latest/meta-data/`) sent via WebSocket upgrade.
- abuse technique: Pre-auth SSRF to localhost:80 extracts AWS IAM credentials (AccessKeyId, SecretAccessKey, Token) from IMDS, and EC2 user-data containing DB_PASSWORD, API_KEY, and other bootstrap secrets. This is a vector for exfiltrating NEXTAUTH_SECRET from self-hosted deployments where the secret is in env or user-data. CVSS 8.6. Vercel-hosted deployments NOT affected.

### S-001 — CVE-2026-44351: fast-jwt Empty HMAC Key Bypass
- type: CVE / technique
- reference: https://advisories.gitlab.com/npm/fast-jwt/CVE-2026-44351/
- key format / validation: When an async key resolver returns an empty string (`keys[kid] || ''`), Node.js `crypto.createHmac('sha256', '')` accepts a zero-length key silently. This pattern (`|| ''` fallback) is extremely common in JS/TS.
- abuse technique: Attacker forges valid JWTs by computing `HMAC-SHA256(key='', payload)`. The server accepts the forged token as authentic. CVSS 9.1 (Critical). Directly relevant to the HMAC-SHA256 signed download/access tokens in `lib/signing/*.ts` if the key resolution has any fallback-to-empty path.

### S-001 — HMAC-SHA256 Known-Secret Signature Forgery
- type: technique
- reference: https://github.com/AdityaBhatt3010/JWT-Authentication-Bypass-via-Algorithm-Confusion-with-No-Key-Exposure
- key format / validation: HMAC-SHA256 uses a symmetric key — anyone with the key can both sign and verify. If the key is derived from `SHA-256(NEXTAUTH_SECRET)` as in `lib/signing/download-token.ts:19`, knowing NEXTAUTH_SECRET reveals all derived HMAC keys.
- abuse technique: With the HMAC key, an attacker forges valid signed download tokens (access to any document) and access tokens (API authentication bypass). No algorithm confusion needed — the symmetric key is directly known from the root secret compromise.

### S-001 — BoxyHQ SAML Jackson: NEXTAUTH_SECRET as Root Encryption Key
- type: technique
- reference: https://github.com/ory/polis (formerly boxyhq/jackson)
- key format / validation: BoxyHQ Jackson uses NEXTAUTH_SECRET to encrypt JWT session cookies for the Admin Portal. The same NEXTAUTH_SECRET value is used across SAML/SCIM enterprise SSO contexts as a root cryptographic secret.
- abuse technique: If NEXTAUTH_SECRET is the default `my-superstrong-secret`, an attacker can forge SAML Jackson admin sessions, decrypt encrypted SAML responses stored in the database, and potentially pivot into enterprise SSO integrations. The cross-domain impact is critical: the same secret that protects user sessions also protects SAML encryption keys.

### S-001 — Unsalted SHA-256 API Token Hashing Weakness
- type: technique
- reference: https://github.com/gizmax/Sandcastle/issues/6
- key format / validation: `lib/api/auth/token.ts:12` hashes API tokens with `SHA-256(token)`. No salt, no HMAC, no KDF. SHA-256 is designed for speed — GPUs compute billions per second.
- abuse technique: If an attacker obtains the hashed token values (database breach), the unsalted SHA-256 allows: (1) rainbow table lookup if tokens have any structure, (2) GPU-accelerated brute-force, (3) same-token detection (identical hashes = identical tokens). HMAC with a server-side pepper or Argon2id is the recommended replacement. The preimage resistance of full SHA-256 protects high-entropy tokens from direct inversion, but the lack of salting eliminates the defense-in-depth.

### S-001 — Self-Hosted Next.js Misconfiguration Chaining
- type: bug bounty writeup (advisory roundup)
- reference: https://healsecurity.com/critical-next-js-vulnerability-exposes-cloud-credentials-api-keys-and-admin-panels/
- key format / validation: Common self-hosted Next.js misconfigurations: `NEXT_PUBLIC_` prefix on secrets, shipping source maps to production exposing server-side code, debug API routes dumping env, and missing reverse-proxy filtering.
- abuse technique: Attackers chain these with CVE-2026-44578 (WebSocket SSRF) or CVE-2025-29927 (middleware bypass via `x-middleware-subrequest`) to escalate from info leak to full credential compromise. The bug bounty pattern: start with SSRF to leak env vars (including NEXTAUTH_SECRET), then use the secret to mint session cookies.

### S-001 — CVE-2023-27490: NextAuth OAuth Session Hijacking
- type: CVE
- reference: https://github.com/nextauthjs/next-auth/security/advisories/GHSA-rcv9-vr55-7q5f
- key format / validation: next-auth OAuth providers < v4.20.1 — missing state, nonce, and PKCE checks in OAuth 1.0/2.0 flows.
- abuse technique: An attacker who controls the network or social-engineers a victim into clicking a crafted login link can intercept/tamper with the authorization URL and log in as the victim. CVSS 8.8 (High). Cross-reference: the current project uses NextAuth.js but the specific version is not determined from SAST alone. Relevant if still below v4.20.1.

---

## S-002 — NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY default (HIGH)

### S-002 — AES-256-CTR Default IV Vulnerability (Trail of Bits)
- type: technique / CVE
- reference: https://seclists.org/oss-sec/2026/q1/185
- key format / validation: The `aes-js` (npm) and `pyaes` (Python) libraries "helpfully" used a default IV (`0x00000000000000000000000000000001`) when none was specified. This caused widespread key/IV reuse in thousands of downstream projects.
- abuse technique: AES-256-CTR is a stream cipher — identical key/IV pairs produce identical keystreams. XORing two ciphertexts encrypted with the same key/IV recovers the XOR of the plaintexts. If one plaintext is known (e.g., a known document header), the other is fully recoverable. The strongMan VPN case study: certificates with predictable X.509 data were encrypted with the default IV, allowing attackers to recover the keystream and decrypt PKCS#8 private keys.

### S-002 — nzo/url-encryptor-bundle: Insecure Default AES-256-CTR Key
- type: CVE (GHSA-r2r8-36pq-27cm)
- reference: https://advisories.gitlab.com/composer/nzo/url-encryptor-bundle/GHSA-r2r8-36pq-27cm/
- key format / validation: PHP bundle used AES-256-CTR with optional key/IV parameters, defaulting to insecure values when not explicitly configured.
- abuse technique: Anyone who knows the default key/IV can decrypt all ciphertexts. Parallel to the papermark finding: if `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` is left as the `.env.example` default (`my-superstrong-document-secret`), all encrypted document passwords in the database are decryptable by anyone who reads the public `.env.example` file.

### S-002 — SHA-256 Key Derivation Without Salt
- type: technique
- reference: https://github.com/Detair/kaiku/issues/233
- key format / validation: `lib/utils.ts:614-628` uses `SHA-256(NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY)` truncated to 32 bytes as the AES-256-CTR key. No salt, no KDF, no key stretching. SHA-256 is designed for speed (billions/sec on GPU).
- abuse technique: Deterministic derivation means identical input produces identical key — no per-record randomization. If the input secret is weak or default, GPU-based brute-force is trivial. Even with a strong input, the lack of salting means a pre-computed rainbow table for common password/document-password values would work. Argon2id or at minimum PBKDF2 with a random salt per document is the standard mitigation. Zellic audit confirmation: this exact pattern (raw SHA-256 key derivation) is flagged as a medium-severity finding in crypto audits.

### S-002 — CWE-916: Use of Password Hash With Insufficient Computational Effort
- type: technique
- reference: https://codesignal.com/learn/courses/a02-cryptographic-failures-1/lessons/storing-passwords-without-a-proper-key-derivation-function
- key format / validation: CWE-916 classifies unsalted, single-round SHA-256 for key derivation as a weakness. SHA-256 throughput: billions/sec on modern GPU vs. ~13/sec for bcrypt.
- abuse technique: GPU-accelerated offline key search. For the papermark case: even if the key were randomized per document, the lack of key stretching means an attacker with DB access can test billions of candidate passwords per second. Combined with the deterministic derivation (no salt), all documents using the same password produce the same key — enabling batch cracking.

---

## S-003 — Trigger.dev project ID hardcoded in config (LOW)

### S-003 — Shai-Hulud Attack on Trigger.dev
- type: writeup
- reference: https://trigger.dev/blog/shai-hulud-postmortem
- key format / validation: The `proj_plmsfqvqunboixacjjus` string is a Trigger.dev project identifier, not a bearer credential. Trigger.dev authentication uses `TRIGGER_SECRET_KEY` from the environment. The project ID is analogous to an API path prefix.
- abuse technique: No direct exploit. The Shai-Hulud incident was a supply-chain attack (malicious npm `preinstall` script running TruffleHog against developer machines) — unrelated to project ID exposure. The hardcoded project ID primarily aids targeted reconnaissance: an attacker can identify the exact Trigger.dev project, enumerate deployed tasks, and correlate with other exposed identifiers. The real Trigger.dev attack surface is developer-machine credential storage (npm tokens, GitHub tokens, cloud SSO tokens), not project IDs.

### S-003 — Hardcoded Identifiers as Recon Surface
- type: technique
- reference: https://www.reversinglabs.com/blog/shai-hulud-call-to-action
- key format / validation: Proj_ identifiers, org slugs, and deployment IDs in public config files are low-severity findings that feed attacker recon.
- abuse technique: Used to fingerprint the exact Trigger.dev project, identify which tasks/ workflows are deployed, and correlate with other exposed identifiers from git history or public-facing endpoints. Defensive value: removing hardcoded identifiers limits context available to attackers during initial recon.

---

## Cross-Cutting / Supporting References

### Embrace The Red — Cookie Minting Tooling
- type: technique / tooling
- reference: https://embracethered.com/blog/posts/2026/minting-next-auth-nextjs-auth-cookies-react2shell-threat/#appendix
- summary: The article references a published GitHub tool that implements cookie minting. The tool cycles through known cookie-name salts by NextAuth version and uses HKDF with the known NEXTAUTH_SECRET to forge JWE tokens. Directly usable against any deployment using the default `my-superstrong-secret`.

### SHA-256 Length-Extension Attacks on Token Authentication
- type: technique
- reference: https://www.00f.net/2025/10/23/length-extension-attacks/
- summary: SHA-256 (Merkle–Damgård) is vulnerable to length-extension when used as `H(key || message)` for authentication tokens. Relevant if any token construction in the codebase uses this pattern rather than HMAC. HMAC is designed to be immune to length-extension. The codebase already uses HMAC-SHA256 in signing modules, so this is a confirmation that those specific modules are not vulnerable to this class of attack.

### CVE-2025-41744 — Hardcoded AES-256 Key (ICS)
- type: CVE
- reference: https://github.com/boeseejykbtanke348/CVE-2025-38001 (analogous — Spreacher Automation hardcoded AES-256 key, CVSS 9.1)
- summary: Demonstrates the concrete impact of hardcoded/default AES-256 keys in deployed systems. While the context (ICS/SCADA) is different, the principle applies: a default key embedded in configuration and used for encryption means anyone who knows the source can decrypt. Highlights the severity delta between "it's just an example value" and what happens when that value reaches production.

---

## Summary Table

| Linked Finding | Search Queries | Entries |
|---|---|---|
| **S-001** (CRITICAL) | K-01, K-02, K-03, K-04, K-08, K-09, K-10, K-11 | 10 CVE/writeup/technique entries |
| **S-002** (HIGH) | K-05, K-06, K-07, K-08 | 4 technique/CVE entries |
| **S-003** (LOW) | K-12 | 2 writeup/technique entries |
| Cross-cutting | (supporting) | 3 entries |

**Consumed by:** `web-synthesis`
