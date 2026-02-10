# Context Assembly (Phase 1)

You are performing Phase 1 of a chunked PR review. Your goal is to gather all context about the PR and present a clear overview before diving into the review.

## Prerequisites

- PR has been parsed into `PR_OWNER`, `PR_REPO`, `PR_NUMBER` using `scripts/parse-pr-arg.md`

## Steps

### 1. Load GitHub Tools

Use `ToolSearch` to load the required GitHub MCP tools:
- Search for `+github pull_request` to load PR-related tools
- Search for `+github issue` to load issue tools (for linked issues)

If GitHub MCP tools are not available, fall back to `gh` CLI commands.

### 2. Fetch PR Data (parallel)

Make these calls **in parallel** since they are independent:

1. **PR metadata**: Use `pull_request_read` (or `gh pr view {PR_NUMBER} --repo {PR_OWNER}/{PR_REPO} --json title,body,author,baseRefName,headRefName,headRefOid,labels,state,isDraft,mergeable,additions,deletions,changedFiles,createdAt,updatedAt`)
2. **Changed files list**: Use `get_files` from PR read tools (or `gh api repos/{PR_OWNER}/{PR_REPO}/pulls/{PR_NUMBER}/files --paginate`)
3. **CI/check status**: Use `get_status` (or `gh pr checks {PR_NUMBER} --repo {PR_OWNER}/{PR_REPO}`)
4. **Commits**: Use `list_commits` (or `gh api repos/{PR_OWNER}/{PR_REPO}/pulls/{PR_NUMBER}/commits`)

### 3. Project Environment & Domain Discovery

Before proceeding, understand the project's technical and domain context:
- **Technical Stack**: Read `package.json`, `tsconfig.json`, `Cargo.toml`, or equivalent manifest files. Look for key dependencies that dictate patterns (e.g., `react`, `express`, `prisma`, `jest`).
- **Domain Context**: Analyze `README.md`, documentation folders (`docs/`), and key architectural files to understand the project's purpose (e.g., "E-commerce checkout system", "Infrastructure-as-Code for AWS").
- **Coding Style**: Identify established coding styles (e.g., "Standard ESLint", "Functional-first approach").
- **Global Pattern Search**: If the PR introduces new utility patterns, use `search_file_content` to check if similar patterns already exist in the codebase to avoid duplication.

**Output**: A concise mental model of the project's domain and standards to inform "Mentor's Notes" in later phases.

### 4. Extract Linked Issues

Scan the PR body for issue references:
- Pattern: `#(\d+)`, `fixes #(\d+)`, `closes #(\d+)`, `resolves #(\d+)`
- Pattern: `https://github.com/{owner}/{repo}/issues/(\d+)`
- For each linked issue, fetch its title and status using `issue_read` or `gh issue view`

### 4. Compute Summary Statistics

From the changed files list, compute:
- **Total files changed**: count of files
- **Total lines added**: sum of additions
- **Total lines removed**: sum of deletions
- **Total LOC changed**: additions + deletions
- **Estimated chunks**: ceil(total_LOC / 300)
- **Estimated review time**: chunks × 5 minutes (rough heuristic)

### 5. Display PR Overview

Present the overview in this format:

```
## PR Overview

**{title}** #{PR_NUMBER}
Author: @{author} | Branch: {head} → {base} | Created: {date}
Status: {state} {draft_badge} | Mergeable: {mergeable}
Labels: {labels}
CI: {check_status_summary}

### Change Summary

| Metric | Value |
|--------|-------|
| Files changed | {count} |
| Lines added | +{additions} |
| Lines removed | -{deletions} |
| Estimated chunks | {chunks} |
| Estimated time | ~{time} min |

### Files Changed

| File | Status | +/- | Category |
|------|--------|-----|----------|
| {path} | {modified/added/deleted/renamed} | +{a}/-{d} | {auto-categorize} |
| ... | ... | ... | ... |

### Commits ({count})

- {sha_short} {message} (@{author})
- ...

### Linked Issues

- #{number}: {title} ({status})
- ...
```

### 6. Size-Based Recommendations

Based on the PR size, provide a recommendation:
- **< 100 LOC**: "This is a small PR. Consider using `--quick` for a faster review."
- **100-1000 LOC**: "Good size for chunked review. Estimated {N} chunks."
- **1000-2000 LOC**: "Large PR. Chunked review recommended. Consider focusing on specific areas with `--focus`."
- **2000+ LOC**: "⚠️ Very large PR ({LOC} lines across {files} files). Consider whether this should be split. Proceeding with chunked review."
- **50+ files**: "⚠️ This PR touches {N} files. Review will be chunked but may take a while."

### 7. User Gate

Ask the user to choose how to proceed:

**Options:**
1. **Proceed** — Start the full chunked review
2. **Set focus** — Focus on a specific area (security, performance, correctness, types, error-handling)
3. **Quick mode** — Switch to quick single-pass review
4. **Cancel** — Stop the review

Use `AskUserQuestion` with these options.

### 8. Output

Store the following for use by subsequent phases:
- `pr_metadata` — full PR metadata object
- `changed_files` — list of changed files with paths, status, additions, deletions, patch content
- `ci_status` — CI check results
- `commits` — list of commits
- `linked_issues` — list of linked issue references
- `head_sha` — the HEAD SHA for state management
- `user_choice` — proceed / focus area / quick / cancel
