# Comments Digest (Phase 2)

You are performing Phase 2 of a chunked PR review. Your goal is to fetch all existing PR comments, categorize them with trust-aware resolution analysis, and present a summary before the actual review begins.

## Prerequisites

- Phase 1 (Context Assembly) completed
- `PR_OWNER`, `PR_REPO`, `PR_NUMBER` are set
- `changed_files` list is available from Phase 1

## Steps

### 1. Fetch All Comments (parallel)

Make these calls **in parallel**:

1. **Review comments** (inline/thread comments):
   Use `pull_request_read` to get review comments, or:
   ```
   gh api repos/{PR_OWNER}/{PR_REPO}/pulls/{PR_NUMBER}/comments --paginate
   ```

2. **Review summaries** (top-level reviews with verdicts):
   Use `pull_request_read` to get reviews, or:
   ```
   gh api repos/{PR_OWNER}/{PR_REPO}/pulls/{PR_NUMBER}/reviews --paginate
   ```

3. **General issue comments** (non-review conversation):
   ```
   gh api repos/{PR_OWNER}/{PR_REPO}/issues/{PR_NUMBER}/comments --paginate
   ```

### 2. Categorize Comments

Sort every comment into one of four buckets:

#### Bucket 1: Unresolved Threads
- Review comments where the thread is **not resolved** (no `resolved` marker, or explicitly unresolved)
- These are the highest priority — they represent open feedback

#### Bucket 2: Resolved Threads
- Review comments marked as resolved
- **Do NOT blindly trust resolution status** — proceed to Step 3 for verification

#### Bucket 3: Bot Comments
- Identify by author username patterns: `*[bot]`, `codecov*`, `sentry*`, `github-actions*`, `dependabot*`, `renovate*`, `sonarcloud*`, `coveralls*`
- Also check for `author.type === "Bot"` in the API response
- Extract key info: coverage changes, security alerts, CI results, deployment URLs

#### Bucket 4: Review Verdicts
- Top-level review submissions with state: `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`
- Record: reviewer name, verdict, body summary, date

### 3. Resolution Trust Analysis

For each **resolved thread** (Bucket 2), determine if the concern was actually addressed:

#### Analysis Process:

1. **Read the comment**: Extract what change was requested
2. **Find the file in diff**: Look up the commented file in `changed_files`
3. **Check the line**: Compare the original commented line/region with the current diff

#### Trust Level Assignment:

- **`[Addressed]`** — The resolved thread's concern is reflected in the code:
  - The diff contains changes at or near the commented line
  - The changes align with what was requested (e.g., comment said "add null check" and a null check was added)
  - Assign this level when you can see clear evidence the feedback was acted upon

- **`[Resolved - verify]`** — Resolved but concern may persist:
  - The commented code region is **unchanged** in the diff
  - OR the changes near the comment don't clearly address the feedback
  - OR the comment was about a broader concern (architecture, approach) that's hard to verify from diff alone
  - These should be surfaced to the reviewer for human judgment

- **`[Outdated]`** — The code context no longer exists:
  - The file was deleted
  - The commented lines were removed entirely
  - A rename moved the code and the old path no longer exists

- **`[Open]`** — Not resolved (from Bucket 1, included here for completeness)

#### Important Notes:
- Don't spend excessive time on resolution analysis. A quick check of the diff near the commented lines is sufficient.
- When in doubt between `[Addressed]` and `[Resolved - verify]`, choose `[Resolved - verify]` — it's better to surface a false positive than miss an unaddressed concern.
- For comments about style/naming that were resolved: check if the specific name/style was changed.
- For comments about logic/behavior: check if the code structure changed in the relevant area.

### 4. Build `comments_by_file` Map

Create a structured map for inline display during Phase 4:

```
comments_by_file = {
  "src/utils/helper.ts": [
    {
      trust_level: "[Open]",
      line: 42,
      author: "reviewer1",
      body: "This could throw if input is null",
      thread_id: "...",
      date: "2024-01-15"
    },
    {
      trust_level: "[Resolved - verify]",
      line: 78,
      author: "reviewer2",
      body: "Consider using a Map instead of Object for better perf",
      thread_id: "...",
      date: "2024-01-14"
    }
  ],
  "src/core/engine.ts": [...]
}
```

### 5. Display Comments Summary

Present the digest in this format:

```
## Existing Review Comments

### Summary

| Trust Level | Count |
|-------------|-------|
| 🔴 Open (unresolved) | {count} |
| 🟡 Resolved - verify | {count} |
| ✅ Addressed | {count} |
| ⚪ Outdated | {count} |
| 🤖 Bot comments | {count} |

### Review Verdicts

| Reviewer | Verdict | Date | Summary |
|----------|---------|------|---------|
| @{name} | {APPROVED/CHANGES_REQUESTED} | {date} | {one-line summary} |

### Unresolved Threads ({count})

{For each open thread:}
- **{file}:{line}** — @{author}: "{truncated_body}" ({date})

### Needs Verification ({count})

{For each [Resolved - verify] thread:}
- **{file}:{line}** — @{author}: "{truncated_body}" — ⚠️ Code appears unchanged

### Bot Reports

{For each bot comment, one-line summary:}
- 🤖 **{bot_name}**: {key finding, e.g., "Coverage: 78% (-2.1%)" or "2 security warnings"}
```

If there are **no comments at all**, display:
```
## Existing Review Comments

No existing comments on this PR. You'll be the first reviewer! 🎯
```

### 6. Output

Store the following for use by subsequent phases:
- `comments_by_file` — the structured map for inline display in chunk review
- `unresolved_count` — number of [Open] threads
- `verify_count` — number of [Resolved - verify] threads
- `review_verdicts` — list of existing review verdicts
- `bot_summaries` — extracted bot report summaries
