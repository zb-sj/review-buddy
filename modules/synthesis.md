# Synthesis (Phase 5)

You are performing Phase 5 of a chunked PR review — aggregating all findings, determining a verdict, and presenting the final summary.

## Prerequisites

- Phase 4 (Chunk Review) completed or skipped-to-synthesis
- `all_findings` — accumulated findings from all reviewed chunks
- `chunks_reviewed` / `chunks_skipped` — review coverage info
- `positive_notes` — collected positive observations
- `comments_by_file` — existing PR comments with trust levels
- `unresolved_count` / `verify_count` — from Phase 2
- `review_verdicts` — existing review verdicts from other reviewers
- `pr_metadata` — PR title, author, etc.

## Steps

### 1. Aggregate Findings & Impact Analysis

Group all findings across chunks by their potential impact:

- **Functional Risk** (🔴 Action Required): Findings that could cause bugs, crashes, or security breaches in production.
- **Technical Debt** (🟡 Recommended / 🟢 Minor): Findings related to maintainability, naming, and best practices.

For each **Functional Risk** finding, prepare a "Business Impact" statement (e.g., "Could lead to data corruption", "Vulnerable to SQL injection").

Also, load `.review-buddy/review-todo.md` and identify any uncompleted tasks in the `## 📝 Ad-hoc Tasks` section.

### 2. Compute Unaddressed Comment Impact

Count existing comment threads that still need attention:
- `open_threads` = count of `[Open]` trust level comments
- `verify_threads` = count of `[Resolved - verify]` trust level comments
- `attention_needed` = `open_threads` + `verify_threads`

These factor into the verdict.

### 3. Determine Verdict & Reasoning

Apply this decision tree and provide a **"Reasoning for Verdict"** block that explains the synthesis of findings.

```
if action_required.length > 0:
    verdict = "Request Changes"
    reason = "Critical functional risks identified that may impact system stability or security."
elif open_threads > 0:
    verdict = "Request Changes"
    reason = "{N} unresolved review threads from previous reviewers"
elif recommended.length > 0 or verify_threads > 0:
    verdict = "Approve with Suggestions"
    reason = "{N} recommendations + {M} threads to verify"
else:
    verdict = "Approve"
    reason = "No significant issues found"
```

**Verdict badges:**
- ✅ **Approve** — Ship it
- 🟡 **Approve with Suggestions** — Good to go, but consider the recommendations
- 🔴 **Request Changes** — Issues that should be addressed before merging

### 4. Identify Key Themes & Educational Insights

Scan across all findings and group by recurring patterns or learning opportunities. 

- **Key Themes**: Common technical issues (e.g., "Error handling gaps", "Type safety concerns").
- **Educational Insights (Growth Areas - if `mentoring` is true)**: Aggregate the "Mentor's Notes" into 1-2 high-level takeaways for the reviewee. (e.g., "Deepening understanding of the project's new concurrency utility", "Transitioning from class components to functional hooks").

Present up to 3 key themes and 1-2 growth areas (if `mentoring` is true).

### 5. Display Final Summary

```
---
## Review Summary — PR #{PR_NUMBER}: {title}

### Verdict: {verdict_badge} {verdict}
{reason}

{if chunks_skipped > 0:}
> ⚠️ {chunks_skipped} of {total_chunks} chunks were skipped. This review may be incomplete.

### Findings Overview

| Severity | Count | Marked for GitHub |
|----------|-------|-------------------|
| 🔴 Action Required | {count} | {github_count} |
| 🟡 Recommended | {count} | {github_count} |
| 🟢 Minor | {count} | {github_count} |
| **Total** | **{total}** | **{github_total}** |

### Unaddressed Comments from Previous Reviews

| Status | Count |
|--------|-------|
| 🔴 Open threads | {open_threads} |
| 🟡 Resolved - verify | {verify_threads} |
| **Needs attention** | **{attention_needed}** |

{if attention_needed > 0:}
> ⚠️ There are {attention_needed} comment threads from previous reviews that may still need attention. Consider checking these before approving.

### Pending Tasks from Todo List

{List all uncompleted ad-hoc tasks from `.review-buddy/review-todo.md`}

{if pending_tasks_count > 0:}
> ⚠️ You have {pending_tasks_count} uncompleted tasks. Address these before finalizing the review.

### Key Themes

{For each theme:}
1. **{theme_title}** — {description} ({N} related findings)

{if mentoring is true:}
### 💡 Educational Growth Areas

{List 1-2 key takeaways or architectural pro-tips for the reviewee based on the synthesis of findings.}
{endif}

### All Findings

{List all findings grouped by severity, using assets/finding-template.md format}

#### 🔴 Action Required ({count})
{findings}

#### 🟡 Recommended ({count})
{findings}

#### 🟢 Minor ({count})
<details>
<summary>Show {count} minor suggestions</summary>
{findings}
</details>

### Positive Aspects

{Collected positive_notes, or:}
- {general positive observation about code quality, test coverage, etc.}

---
```

### 6. User Gate — Posting Decision

Solicit user intent following the **Agnostic Interaction Protocol** (`references/PROTOCOL.md`).

**Options:**
1. **Post all marked findings to GitHub** — Submit a review with all `marked_for_github` findings ({count} comments)
2. **Post only Action Required** — Submit only the critical findings
3. **Select specific findings** — Choose which findings to post (show numbered list)
4. **Save locally** — Write findings to `.review-buddy/review-buddy-findings.md` for later posting with `--post-only`
5. **Discard** — Don't save or post anything

#### If user chooses a posting option (1, 2, or 3):
- If there are uncompleted tasks in `.review-buddy/review-todo.md`, warn the user:
  "⚠️ You have {N} uncompleted tasks. Are you sure you want to post the review? [y/n]"
- If confirmed (or no tasks), proceed to `github-post.md` module with the selected findings and verdict.

#### If user chooses "Select specific findings":
- Display a numbered list of all findings
- Ask the user to enter the numbers they want to post (comma-separated)
- Mark only those findings for GitHub posting
- Return to the Posting Decision gate

#### If user chooses "Save locally":
- Use `scripts/state-manager.md` to export findings to `.review-buddy/review-buddy-findings.md`
- Display: "Findings saved to `.review-buddy/review-buddy-findings.md`. Post later with `/review-buddy --post-only`."
- Clean up state and todo files.

#### If user chooses "Discard":
- Clean up state and todo files.
- Exit.

### 7. Output

Pass the following to the posting module:
- `findings_to_post` — list of findings selected for GitHub posting
- `verdict` — Approve / Approve with Suggestions / Request Changes
- `verdict_body` — the summary text to use as the review body
- `pr_owner`, `pr_repo`, `pr_number` — PR identity
