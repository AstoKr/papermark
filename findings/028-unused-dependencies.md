# Finding: Unused Direct Dependencies Increasing Attack Surface

## Severity
LOW

## CVSS Estimate
N/A — attack surface management finding

## Affected Code
- **File:** `package.json`
- **Packages:** `oidc-provider@9.8.3`, `@modelcontextprotocol/sdk@1.29.0`, `tokenlens@1.3.1`

## Description
Three direct dependencies declared in `package.json` have zero imports in source code. They add ~100+ transitive packages to the SBOM, increasing attack surface with no benefit.

| Package | Version | Transitive Risk |
|---------|---------|----------------|
| `oidc-provider` | 9.8.3 | Pulls in `koa`, `koa-router`, and their sub-dependencies |
| `@modelcontextprotocol/sdk` | 1.29.0 | Pulls in `express@^5.2.1`, `zod-to-json-schema`, and many others |
| `tokenlens` | 1.3.1 | Minimal transitive tree but still unnecessary |

## Proof of Concept
Grep for each package name across all `.ts/.tsx/.js` source files returns zero results. GitNexus graph search confirms no imports or code references to any of the three packages.

```bash
# Verify: no imports found for any of the three packages
grep -r "from 'oidc-provider'" apps/ components/ lib/ pages/ ee/
# No results
```

## Impact
None today. These packages are not called at runtime, so no exploit path exists. However, if a CVE is published for any of their transitive dependencies, the SBOM scanner will flag papermark, requiring manual triage. Every unnecessary transitive dependency is a potential supply chain risk.

## Evidence
Grep for each package name across all `.ts/.tsx/.js` source files returned zero results. GitNexus graph search confirms no imports or code references to any of the three packages.

## Methodology
- **First found by:** Dependency synthesis (02-synthesized-dependencies.md S-005)
- **Confirmed by:** GitNexus graph — zero imports for all 3 packages

## Chain Potential
Standalone finding.

## Remediation
Remove from `package.json`:
```bash
npm uninstall oidc-provider @modelcontextprotocol/sdk tokenlens
```
Re-add only if/when the feature they support is implemented.
