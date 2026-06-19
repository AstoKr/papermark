# lib — sheet

# lib/sheet Module

Utilities for parsing and normalizing Excel spreadsheet data (XLSX format) into structured JSON.

## Overview

The `lib/sheet` module provides a single export `parseSheet` that fetches an Excel file from a URL and converts it into a normalized, typed data structure. It uses the `xlsx` library under the hood and handles the normalization of uneven rows and Excel-style column ordering (A, B, ... Z, AA, AB, ...).

## Dependencies

```typescript
import * as XLSX from "xlsx";
```

This module depends on the `xlsx` package, which must be installed in the project.

## Type Definitions

```typescript
type RowData = { [key: string]: any };
```

Represents a single row in the spreadsheet as a mapping of column reference to cell value.

```typescript
type SheetData = {
  sheetName: string;
  columnData: string[];
  rowData: RowData[];
};
```

The normalized output structure for each sheet:
- `sheetName` — The original name of the sheet tab in the workbook
- `columnData` — Sorted array of all unique column references (A, B, C, etc.)
- `rowData` — Array of row objects, each containing all columns with empty strings for missing values

## API Reference

### `parseSheet`

```typescript
parseSheet: (options: { fileUrl: string }) => Promise<SheetData[]>
```

Parses an Excel file from a remote URL and returns normalized sheet data.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `fileUrl` | `string` | URL pointing to the XLSX file to parse |

**Returns:** `Promise<SheetData[]>` — An array of `SheetData` objects, one for each sheet in the workbook.

**Throws:** Standard network errors if the fetch fails, or XLSX parsing errors for malformed files.

## How It Works

### Step-by-Step Execution

1. **Fetch** — Downloads the file from `fileUrl` as an `ArrayBuffer`
2. **Parse** — Uses `XLSX.read()` to parse the raw bytes into a workbook object
3. **Iterate Sheets** — Loops through each sheet name in `workbook.SheetNames`
4. **Convert to JSON** — Uses `sheet_to_json()` with `header: "A"` to get row objects keyed by column references (e.g., `{ A: "Name", B: "Age" }`)
5. **Collect Keys** — Flattens all row objects and deduplicates to get every unique column reference
6. **Sort Columns** — Sorts keys using `customSort` for proper Excel column ordering
7. **Normalize Rows** — Iterates through each row and fills in any missing columns with empty strings
8. **Return** — Pushes the structured data to the result array for all sheets

### Custom Column Sorting

Excel columns follow a specific ordering pattern: A, B, ... Z, AA, AB, ... AZ, BA, etc. The `customSort` function handles this:

```typescript
const customSort = (a: string, b: string) => {
  if (a.length === b.length) {
    return a.localeCompare(b); // Same length: alphabetical (A, B, C)
  }
  return a.length - b.length;   // Different length: shorter first (Z before AA)
};
```

This ensures column order matches what users expect in spreadsheet applications.

### Row Normalization

Spreadsheets often have uneven rows—some rows have data in column C while others only have columns A and B. The normalization step ensures every row object has every column key:

```typescript
// Before normalization (mixed columns)
[{ A: "Alice", B: "25" }, { A: "Bob", C: "Developer" }]

// After normalization (all rows have A, B, C)
[
  { A: "Alice", B: "25", C: "" },
  { A: "Bob", B: "", C: "Developer" }
]
```

This simplifies downstream consumption of the data since consumers don't need to check for missing keys.

## Usage Example

```typescript
import { parseSheet } from "@/lib/sheet";

async function loadEmployeeData() {
  const sheets = await parseSheet({ 
    fileUrl: "https://example.com/employees.xlsx" 
  });
  
  for (const sheet of sheets) {
    console.log(`Sheet: ${sheet.sheetName}`);
    console.log(`Columns: ${sheet.columnData.join(", ")}`);
    
    for (const row of sheet.rowData) {
      console.log(row);
    }
  }
}
```

### Accessing Specific Data

```typescript
const sheets = await parseSheet({ fileUrl: "/data/report.xlsx" });

// Get first sheet
const firstSheet = sheets[0];

// Get all column headers
const columns = firstSheet.columnData;

// Get a specific cell value
const cellValue = firstSheet.rowData[0]["A"];

// Find a row by a known column value
const targetRow = firstSheet.rowData.find(row => row["A"] === "Target");
```

## Data Flow

```
Remote XLSX File
       │
       ▼
   fetch() ──► ArrayBuffer
       │
       ▼
  XLSX.read() ──► Workbook
       │
       ▼
  For each Sheet
       │
       ├─► sheet_to_json() ──► RowData[]
       │
       ├─► Collect all keys ──► columnData (sorted)
       │
       ├─► Normalize rows ──► rowData (all keys present)
       │
       ▼
   SheetData object
       │
       ▼
   SheetData[] (final result)
```

## Integration Notes

This module is a self-contained utility. It receives data from external sources (via URL) and returns normalized structures intended for consumption by other modules. Typical consumers would:

- Display spreadsheet data in a UI
- Transform the data for API submission
- Perform calculations or aggregations
- Validate spreadsheet contents against schemas

No specific return format assumptions should be made beyond the exported TypeScript types.