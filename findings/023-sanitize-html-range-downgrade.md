# Finding: sanitize-html@^2.17.3 Declared Range Allows Downgrade to Vulnerable 2.17.3

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H = 9.3 (if the resolved version is 2.17.3 and any future caller uses a permissive sanitize-html config).

## Affected Code
- File: `package.json`
- Line: 156 (`"sanitize-html": "^2.17.3"`)
- Function: n/a — package manifest
- Endpoint: n/a — affects every caller using `sanitize-html` with a permissive config

## Description
The current lockfile pins `sanitize-html` to `2.17.4`, which is **patched** for both relevant 2026 CVEs (per `methodology-raw/00-early-web-intel.md` §2.6):
- **CVE-2026-44990** (CRITICAL 9.3) — Apostrophe XSS via `xmp` raw-text passthrough. Vulnerable: `= 2.17.3` only.
- **CVE-2026-40186** (MOD 6.1) — `allowedTags` bypass via entity-decoded text in `nonTextTags` elements. Vulnerable: `< 2.17.4`.

However, `package.json:156` declares `"sanitize-html": "^2.17.3"`. The caret allows `2.17.4` and above (`^2.17.3 := >=2.17.3 <3.0.0`), which means:
- A clean `npm install` against the registry **today** resolves to `2.17.4` (because `^2.17.3` will pick the highest 2.x available, and 2.17.4 is the latest patched).
- However, if a future release is published that is **not** semver-pinned for security (e.g. a `2.17.5` that introduces a regression), or if the lockfile is regenerated (`rm package-lock.json && npm install`) on a registry that has older mirrors, the resolved version could be `2.17.3` which is vulnerable to CVE-2026-44990.

The papermark configuration today (`allowedTags: []`, `allowedAttributes: {}`) **incidentally** prevents both CVEs from being exploitable — but the codebase already has **three sibling call sites** of `sanitize-html` (`lib/utils/sanitize-html.ts`, `pages/branding.tsx`, `pages/datarooms/[id]/branding/index.tsx`) and the `pages/branding.tsx` + `pages/datarooms/[id]/branding/index.tsx` callers use the same `{ allowedTags: [], allowedAttributes: {} }` config inline. A future caller using `sanitize-html` with the default config (which allows `<a href="">`, `<img>`, `<table>`, etc.) would be immediately vulnerable.

## Proof of Concept

### Pre-conditions
- A clean `npm install` (without the lockfile) resolves `sanitize-html` to a vulnerable version.
- A future caller uses `sanitize-html` with a permissive config (e.g. allowing `<a href="">`).

### Step-by-step exploitation (latent)
1. Lockfile is regenerated: `rm package-lock.json && npm install`.
2. The new `package-lock.json` resolves `sanitize-html` to `2.17.3` (e.g. on a registry mirror that has not yet published `2.17.4`).
3. A future feature adds a new caller that uses `sanitize-html` with the default config (allowing `<a href="">`).
4. Attacker submits a payload with `<a href="javascript:alert(1)">` — bypasses the default sanitizer.
5. **Or** — the CVE-2026-44990 apostrophe XSS via `xmp` raw-text passthrough: attacker submits `<xmp>'><img src=x onerror=alert(1)></xmp>` and the sanitizer passes the payload through to the sink.

## Impact
- **Latent XSS** — if the lockfile is regenerated and the resolved version is vulnerable, any future caller using a permissive `sanitize-html` config is exposed.
- **Today: no practical exploit** — the lockfile pins `2.17.4`, and every current caller uses the strictest possible config (`allowedTags: []`, `allowedAttributes: {}`).

## Evidence
```json
// package.json:156
"sanitize-html": "^2.17.3"
```

## Methodology
- **M3 — Sanitizer-bypass** (`methodology-raw/01-sanitizer-bypass.md` Finding 3)
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` §2.6 — CVE-2026-44990, CVE-2026-40186)

## Chain Potential
- None directly. The finding is a defense-in-depth concern.

## Remediation
Tighten the declared range to `"sanitize-html": "~2.17.4"`. The tilde constrains to `>=2.17.4 <2.18.0`, which:
- Prevents downgrade to `2.17.3` (CVE-2026-44990) on any `npm install` against a misconfigured registry.
- Allows `2.17.4` patch releases (which by SemVer must be backwards-compatible bug fixes).
- Forces an explicit bump to `2.18.0` (or any future major) via a `package.json` edit, which is the natural review point.

Alternatively, switch to `"sanitize-html": "2.17.4"` (exact pin) if the team accepts the maintenance cost of manually bumping for each security release.
