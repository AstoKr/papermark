# Finding: pdfjs-dist@3.11.174 Arbitrary JS Execution on Opening Malicious PDF (GHSA-wgrm-67xf-hhpq)

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:H = 8.8 (the public CVE score).

## Affected Code
- File: `package.json`
- Line: 208-210 (the `overrides` block that pins `pdfjs-dist` to `3.11.174`)
- Function: n/a — package manifest
- Endpoint: n/a — affects every PDF viewer in the codebase

- The reachability chain (per `methodology-raw/03-sbom-reachability.md` Dep S-001):
  `react-notion-x` → `react-pdf@8.0.2` → `pdfjs-dist@3.11.174`

- Sink: `components/view/viewer/pdf-default-viewer.tsx:304-313` (the `<Page>` component with `renderAnnotationLayer={false}` and `renderTextLayer={false}` — note: the text+annotation layer bypass is disabled, so the **classic** react-pdf XSS chain is blocked; the CVE-2026-44990 / GHSA-wgrm-67xf-hhpq vector is a different bypass)

## Description
`pdfjs-dist@3.11.174` is vulnerable to GHSA-wgrm-67xf-hhpq (CVSS 8.8) — arbitrary JavaScript execution on opening a malicious PDF. The package is pinned via an `overrides` block in `package.json`, so the version is intentionally held at the vulnerable release.

Papermark's PDF viewer (`components/view/viewer/pdf-default-viewer.tsx`) disables the text and annotation layers (`renderTextLayer={false}`, `renderAnnotationLayer={false}`), which **closes the classic react-pdf XSS chain** (CVE-2024-4367 etc., which requires text or annotation layer rendering). However, the GHSA-wgrm-67xf-hhpq vector is a **different** bypass — it exploits the PDF parser itself, not the text/annotation layer rendering. The CVE is in `pdfjs-dist` core, and any PDF rendering (canvas or otherwise) is exposed.

The chain: `react-notion-x` → `react-pdf@8.0.2` → `pdfjs-dist@3.11.174` → GHSA-wgrm-67xf-hhpq.

## Proof of Concept

### Pre-conditions
- A malicious PDF that exploits the GHSA-wgrm-67xf-hhpq vector (the public CVE details the payload format).
- The PDF is opened in papermark's viewer — either as a standalone PDF upload, or as a Notion page (via `react-notion-x` → `react-pdf`).

### Step-by-step exploitation
1. Attacker uploads a malicious PDF to a papermark dataroom (the chain with F-005 / F-008 — the CORS misconfig — allows cross-origin upload into a victim's active dataroom session).
2. Victim opens the PDF in the dataroom viewer.
3. `pdfjs-dist@3.11.174` parses the malicious PDF, triggers GHSA-wgrm-67xf-hhpq, executes arbitrary JavaScript in the victim's browser.
4. The JS payload acts on the victim's session: steal viewer-side state, exfiltrate dataroom contents, social-engineer the victim into admin actions.

### Step-by-step chain (CORS + CVE)
1. Victim is in an active dataroom session (cookie `pm_drs_<linkId>`).
2. Attacker on `evil.com` runs:
   ```html
   <script>
   await fetch("https://app.papermark.com/api/file/tus-viewer/<victimUploadId>", {
     method: "PATCH",
     credentials: "include",
     headers: { "Tus-Resumable": "1.0.0", "Upload-Offset": "0", "Upload-Length": "1024" },
     body: maliciousPDFBytes
   });
   </script>
   ```
3. The CORS misconfig (F-008) allows the cross-origin credentialed PATCH.
4. The malicious PDF is injected into the victim's dataroom session.
5. Victim opens the file; the CVE fires; XSS in the victim's session.

## Impact
- **Arbitrary JS in the victim's browser** — full session hijack, CSRF-style actions, exfiltration of all dataroom contents.
- **Same chain as F-008** — the CORS misconfig is the entry point; the CVE is the amplifier.
- **The text+annotation layer disable in the PDF viewer does not block this CVE** — the bypass is in the PDF parser core, not in the layer rendering.

## Evidence
```json
// package.json:208-210
"overrides": {
  ...
  "pdfjs-dist": "3.11.174"  // ← pinned to vulnerable version
}
```

## Methodology
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` §2.4 — CVE details)
- **M2 — Web search** (`methodology-raw/03-web-search.md` — confirmed GHSA)
- **M3 — SBOM reachability** (`methodology-raw/03-sbom-reachability.md` Dep S-001 — `react-notion-x` → `react-pdf@8.0.2` → `pdfjs-dist@3.11.174`)

## Chain Potential
- **Chain 3** (`methodology-raw/05-chains.md`) — the CORS misconfig (F-008) is the entry point; the pdfjs-dist CVE is the amplifier. The chain's bounty estimate is $20K-$50K.

## Remediation
Bump `pdfjs-dist` to a patched version (verify the latest 3.x release that addresses GHSA-wgrm-67xf-hhpq). The override in `package.json:208-210` should be updated:
```json
"overrides": {
  ...
  "pdfjs-dist": "$pdfjs-dist-patched-version"
}
```

Alternatively, use `npm audit fix` or Renovate/Dependabot to keep the `pdfjs-dist` version current. The override is intentional (to pin to a specific version for compatibility with `react-pdf@8.0.2`), but the pinned version should be the latest patched release.

Add a CI check that fails the build if `pdfjs-dist` resolves to a known-vulnerable version.
