# Format Finding

This module defines the consistent format template for all review findings. Every finding surfaced during a review must use this format.

## Severity Levels

There are exactly three severity levels. Use the appropriate badge for each:

| Level | Badge | When to Use |
|-------|-------|-------------|
| Critical | `🔴 Action Required` | Bugs, security vulnerabilities, data loss risks, correctness issues, race conditions |
| Warning | `🟡 Recommended` | Missing error handling, type safety gaps, API design issues, performance concerns, missing validation |
| Info | `🟢 Minor` | Style nits, naming suggestions, minor readability improvements, documentation gaps |

## Finding Template

Format each finding exactly as follows:

```
### {severity_badge} {short_title}

**File:** `{file_path}:{line_number}` | **Category:** {category} | **Confidence:** {score}/100

{description of the issue and why it matters}

```suggestion
{suggested fix if applicable}
```

### 💡 Mentor's Note (Optional)
{A concise "why" or "pro-tip" that explains the underlying principle, project-specific pattern, or domain context. Keep it to 1-2 sentences. This helps both the reviewer and reviewee learn.}

{optional: reference to existing comment if related, e.g. "Related to comment by @reviewer on line 45"}

<!-- 
### Reflexion Logic (Internal Only)
{Agent's internal critique: Why is this finding valid? Did I check for false positives? Is it relevant to the focus area?}
-->
```

### Field Definitions

- **severity_badge**: One of `🔴 Action Required`, `🟡 Recommended`, or `🟢 Minor`.
- **short_title**: A concise, specific description of the issue (e.g., "Unchecked null dereference in user lookup", "Missing error boundary around API call"). Do not use generic titles like "Potential issue" or "Code smell".
- **file_path**: The path of the file relative to the repository root, as it appears in the PR diff.
- **line_number**: The line number in the new version of the file where the issue occurs. If the issue spans multiple lines, use the starting line number.
- **category**: A short category label. Use one of: `bug`, `security`, `performance`, `error-handling`, `type-safety`, `api-design`, `logic`, `concurrency`, `accessibility`, `testing`, `naming`, `style`, `documentation`, `maintainability`.
- **score**: A confidence score from 0 to 100 indicating how certain you are that this is a real issue and not a false positive.
- **description**: 1-3 sentences explaining what the issue is, why it matters, and what could go wrong if it is not addressed. Be specific -- reference variable names, function calls, and conditions.
- **suggestion block**: Include a code suggestion block when you can provide a concrete fix. Omit it when the fix is too context-dependent or architectural. The suggestion should be valid code that could replace the problematic lines.
- **related comment reference**: If an existing PR comment from another reviewer discusses the same or a related issue, mention it. This avoids duplicate feedback and shows awareness of prior review.
- **Reflexion Logic (HTML Comment)**: An internal reasoning block wrapped in `<!-- -->`. Use this during the Actor-Critic loop to justify the finding. It should NOT be included in the final markdown shown to the user or posted to GitHub.

## Filtering Rules

Apply these rules to decide which findings to include in the output:

1. **Confidence threshold**: Only surface findings with confidence >= 80. Discard anything below 80 -- do not show it to the user or include it in any output.

2. **Self-review mode** (`--self` flag is set): Suppress all `🟢 Minor` findings entirely. Only show `🔴 Action Required` and `🟡 Recommended`. The rationale is that authors reviewing their own code do not need nits.

3. **Focus mode** (`--focus` flag is set to a specific area like `security`, `performance`, `error-handling`):
   - For findings whose category matches the focus area, add +10 to the confidence score (cap at 100). This makes borderline findings in the focus area more likely to pass the threshold.
   - Do not remove findings outside the focus area -- still show them if they pass the confidence threshold.

## Ordering Rules

Within each chunk's output, group findings by severity in this order:
1. `🔴 Action Required` findings first
2. `🟡 Recommended` findings second
3. `🟢 Minor` findings last (unless suppressed by self-review mode)

Within each severity group, order findings by confidence score descending (highest confidence first).

## GitHub Posting Conversion

When findings are posted to GitHub as PR review comments (via the `github-post` module), apply these conversions:

- Convert the suggestion code block from triple-backtick `suggestion` to GitHub's suggestion syntax:
  ````
  ```suggestion
  replacement code here
  ```
  ````
  This allows the PR author to apply the suggestion directly from the GitHub UI.

- Strip the `**Confidence:** {score}/100` field from the posted version -- it is useful for the local review but noisy on GitHub.

- Keep the severity badge, title, file reference, category, and description intact.
