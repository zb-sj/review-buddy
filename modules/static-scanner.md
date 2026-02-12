# Static Scanner (Phase 2.5 - Parallel)

You are performing a deterministic pre-scan of the PR changes. This module is initiated by the Leader in parallel with Phase 1 and 2 to minimize session latency.

## Objectives
- Identify hardcoded secrets and sensitive information.
- Catch debug artifacts (`console.log`, `debugger`, `FIXME`).
- Detect common security anti-patterns (e.g., `eval()`, `innerHTML`).
- Reduce the "noise" and token cost of LLM-based analysis.

## Steps

### 1. Secret & Pattern Scanning
For each changed file, run `search_file_content` (or equivalent grep) with the following patterns on the added lines:

| Category | Patterns to look for |
|----------|----------------------|
| **Secrets** | `(A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}`, `(?i)password|api_key|secret|token|private_key` |
| **Debug** | `console\.log\(`, `debugger;`, `FIXME`, `TODO: [^ ]+` |
| **Security** | `eval\(`, `dangerouslySetInnerHTML`, `\.(innerHTML|outerHTML)` |
| **Logic** | `http://localhost`, `127\.0\.0\.1` |
| **UI/Visual** | `rgba?\(\d+, \d+, \d+`, `#[0-9a-fA-F]{3,6}`, `!important`, `z-index: \d{4,}` |

### 2. Triage Findings
Findings from this scan are automatically categorized as:
- **🔴 Action Required**: Secrets, `eval`, `debugger`.
- **🟡 Recommended**: `dangerouslySetInnerHTML`, `localhost` URLs, `!important`, hardcoded hex/rgb colors (if design tokens are used).
- **🟢 Minor**: `console.log`, `FIXME`, high `z-index`.

### 3. Integration
- Append these findings to the global `all_findings` list with a `source: "static-scanner"` flag.
- Set `confidence: 100` for these findings since they are deterministic.
- During `chunk-reviewer.md`, if a file has static scan findings, display them before the LLM analysis starts.

## Output
- `static_findings`: A list of findings detected via regex/grep.
