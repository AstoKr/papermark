# Finding: ReDoS in Workflow Engine — Non-Literal RegExp from User Input

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:N/A:L = 3.5

## Affected Code
- **File:** `ee/features/workflows/lib/engine.ts:307`
- **Pattern:** `new RegExp(condition.value as string).test(normalizedEmail)`

## Description
The workflow engine creates a `RegExp` from a user-configurable `condition.value` when evaluating "matches" conditions in workflow routing rules. If a team member with workflow admin access configures a malicious regex pattern (e.g., `(a|aa)+b` with a long repeating string), it could cause ReDoS — blocking the main thread during evaluation.

The code is wrapped in a try/catch block that catches regex syntax errors, but ReDoS occurs during `.test()` evaluation (not during construction), so the try/catch does NOT prevent the CPU exhaustion.

## Proof of Concept
1. Configure a workflow with a "matches" condition using a ReDoS-vulnerable pattern like `(a|aa)+b`
2. Trigger the workflow with a crafted input string of repeating `a` characters
3. The `new RegExp(condition.value).test(normalizedEmail)` call at line 307 enters catastrophic backtracking
4. The main thread is blocked until the regex completes or the per-request timeout fires

## Impact
Limited to denial of service on the specific server process handling the workflow execution. Requires authenticated team member access to configure workflows. Bounded by per-request timeout. Admin-only workflow configuration further limits exploitability.

## Evidence
```typescript
// ee/features/workflows/lib/engine.ts:305-310
case "matches":
  try {
    return new RegExp(condition.value as string).test(normalizedEmail);
  } catch { return false; }
```

## Methodology
- **First found by:** Semgrep custom rule `papermark-regexp-from-user-input` (04-custom-rules.md)
- **Confirmed by:** Structural analysis

## Chain Potential
Standalone finding. Limited chain potential due to admin-only access and try/catch wrapping.

## Remediation
Validate the regex pattern before use — reject patterns with known ReDoS characteristics (nested quantifiers, alternations with overlapping matches). Consider using a regex analysis library (e.g., `safe-regex` or `regexp-tree`) to detect vulnerable patterns at configuration time. Alternatively, set a timeout on regex evaluation using a worker thread or `Promise.race`.
