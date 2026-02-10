# Static Scanner (Pre-LLM Analysis)

You are performing a deterministic pre-scan of the PR changes. Your goal is to catch low-hanging fruit and obvious anti-patterns before passing the diff to the LLM-based chunk reviewer.

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

### 2. Triage Findings
Findings from this scan are automatically categorized as:
- **🔴 Action Required**: Secrets, `eval`, `debugger`.
- **🟡 Recommended**: `dangerouslySetInnerHTML`, `localhost` URLs.
- **🟢 Minor**: `console.log`, `FIXME`.

### 3. Integration
- Append these findings to the global `all_findings` list with a `source: "static-scanner"` flag.
- Set `confidence: 100` for these findings since they are deterministic.
- During `chunk-reviewer.md`, if a file has static scan findings, display them before the LLM analysis starts.

## Output
- `static_findings`: A list of findings detected via regex/grep.
