# Finding: `xlsx` CDN Tarball Without Integrity Hash

## Severity
LOW

## CVSS Estimate
N/A — supply chain risk, not a version vulnerability

## Affected Code
- **File:** `package.json:175`
- **Package:** `xlsx@0.20.3` from `https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz`

## Description
`package.json` pins `xlsx` to a CDN URL with no integrity hash, rather than the npm registry. If the CDN is compromised or a MITM attack succeeds during install, a different tarball could be substituted. Both CVEs for this package (CVE-2023-30533 — Prototype Pollution, CVE-2024-22363 — ReDoS) are PATCHED at version 0.20.3, so the version itself is safe. The remaining risk is supply chain: the distribution mechanism has no tamper protection.

## Proof of Concept
No runtime exploitable path. The supply chain risk is limited to the `npm install` phase:
```bash
# During install, npm fetches from unverified CDN URL
# If CDN is compromised, attacker-controlled code runs during build
```

## Impact
Supply chain risk during installation only. If the CDN is compromised, an attacker could substitute a malicious tarball that executes during the build phase. No integrity hash means no tamper detection.

## Evidence
```json
// package.json:175
"xlsx": "https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz"
```

No integrity hash (`integrity`) or `resolved` hash present. Compare with npm registry packages which include `integrity` field in `package-lock.json`.

## Methodology
- **First found by:** Dependency synthesis (02-synthesized-dependencies.md S-002)
- **Confirmed by:** AI and OSV-scanner
- **Advisory pairing (03-advisory.md):** Both CVEs are PATCHED at 0.20.3 — CVE portion is false positive; CDN supply chain is the remaining risk

## Chain Potential
Standalone finding. Supply chain compromise could potentially introduce malicious code into the build pipeline.

## Remediation
Pin to the npm registry version instead of the CDN URL. Use the `npm:` protocol or the standard registry version range to ensure lockfile integrity hashes are present:
```json
"xlsx": "npm:xlsx@0.20.3"
```
Alternatively, pin to a specific tarball URL with a verified SHA integrity hash:
```json
"xlsx": "https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz#sha256-<hash>"
```
