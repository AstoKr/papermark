# Finding: $executeRawUnsafe Savepoint Name (Defense-in-Depth Gap)

## Severity
LOW (dead-code today)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:L/A:L = 4.0 (dead-code today; latent SQL-injection sink if a future caller passes request-derived data).

## Affected Code
- File: `lib/folders/bulk-create.ts`
- Line: 51, 54, 57-60 (inside `withUniqueConstraintRetry`: `tx.$executeRawUnsafe(\`SAVEPOINT "${savepointName}"\`)`)
- Function: `withUniqueConstraintRetry`
- Endpoint: n/a — used by folder bulk-create operations

## Description
`await tx.$executeRawUnsafe(\`SAVEPOINT "${savepointName}"\`)` uses string concatenation with double-quoted SQL identifiers. The current callers pass `bulk_main_folder_lvl_${depth}` and `bulk_dr_folder_lvl_${depth}` where `depth` is a numeric loop counter, so practical risk is zero. The function signature, however, accepts `savepointName: string` as free-form input — a future caller that passes request-derived data would activate a SQL injection that the double-quote wrapping does not protect against (no escaping of embedded `"`).

## Proof of Concept

### Pre-conditions (today, no practical exploit)
- The only two callers of `withUniqueConstraintRetry` pass `bulk_main_folder_lvl_${depth}` / `bulk_dr_folder_lvl_${depth}` with `depth` as a numeric loop counter. There is no user input flow.

### Latent exploit (if a future caller passes request-derived data)
1. A future caller passes `savepointName = req.body.name` (e.g. from a folder name input).
2. The caller sets `name = '"; DROP TABLE "Folder"; --'`.
3. The executed SQL becomes:
   ```sql
   SAVEPOINT ""; DROP TABLE "Folder"; --"
   ```
   The double-quote wrapping does not prevent injection — the attacker breaks out with a `"` character.

## Impact
- **Latent SQL injection** — if a future caller passes request-derived data, the attacker can execute arbitrary SQL.
- **Today: no practical exploit** — the only callers use loop integers.

## Evidence
```ts
// lib/folders/bulk-create.ts:45-60
export async function withUniqueConstraintRetry<T>(args: {
  tx: Prisma.TransactionClient;
  savepointName: string;
  attempt: () => Promise<T>;
}): Promise<T> {
  const { tx, savepointName, attempt } = args;
  let lastError: unknown;

  for (let i = 0; i <= MAX_INSERT_RETRIES; i++) {
    await tx.$executeRawUnsafe(`SAVEPOINT "${savepointName}"`);  // ← no escaping
    try {
      const result = await attempt();
      await tx.$executeRawUnsafe(`RELEASE SAVEPOINT "${savepointName}"`);
      return result;
    } catch (error) {
      await tx.$executeRawUnsafe(
        `ROLLBACK TO SAVEPOINT "${savepointName}"`,
      );
      await tx.$executeRawUnsafe(`RELEASE SAVEPOINT "${savepointName}"`);
      ...
    }
  }
}
```

## Methodology
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-005)
- **M1 — Reachability** (`methodology-raw/01-reachability-analysis.md` F5 — DEAD_CODE)

## Chain Potential
- None directly. The finding is a latent defense-in-depth concern.

## Remediation
Replace the four `$executeRawUnsafe` calls with `$executeRaw(Prisma.sql\`SAVEPOINT ${Prisma.raw(savepointName)}\`)` (using `Prisma.raw` only because savepoint identifiers cannot be bound as parameters) and add a regex guard at the top of `withUniqueConstraintRetry`:
```ts
function withUniqueConstraintRetry<T>(args: {
  tx: Prisma.TransactionClient;
  savepointName: string;  // ← keep as string, but validate
  attempt: () => Promise<T>;
}): Promise<T> {
  if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(args.savepointName)) {
    throw new Error("Invalid savepoint name");
  }
  // ... rest of function
}
```

Even simpler: drop the parameter and build the savepoint name internally from a fixed prefix + the retry counter:
```ts
const savepointName = `retry_${i}`;
```
