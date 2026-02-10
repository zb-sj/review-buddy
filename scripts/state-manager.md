# State Manager

This module manages review session state for pause/resume capability. It handles persisting the chunk plan, accumulated findings, and review progress to disk so that a review can be interrupted and resumed later.

## File Locations

All state files are relative to the **project root** (the directory containing `.git`):

- **State file**: `.review-buddy/review-buddy-state.md`
- **Findings export file**: `.review-buddy/review-buddy-findings.md`

Ensure the `.review-buddy/` directory exists before writing. Create it if it does not exist.

## State File Structure

The state file uses markdown with YAML frontmatter:

```markdown
---
pr_owner: "owner"
pr_repo: "repo"
pr_number: 123
head_sha: "abc123def456"
started: "2024-01-15T10:30:00Z"
updated: "2024-01-15T10:45:00Z"
mode: "full"
focus: ""
self_review: false
---

## Chunk Plan

| Chunk | Files | LOC | Status |
|-------|-------|-----|--------|
| 1 | types.ts, interfaces.ts | 120 | reviewed |
| 2 | utils/helper.ts, utils/format.ts | 280 | reviewed |
| 3 | core/engine.ts | 450 | in-progress |
| 4 | components/Form.tsx, components/List.tsx | 310 | pending |
| 5 | __tests__/engine.test.ts | 200 | pending |

## Findings

### 🔴 Action Required Unchecked null dereference in user lookup

**File:** `core/engine.ts:42` | **Category:** bug | **Confidence:** 95/100
**Marked for GitHub:** true

The `getUser()` call can return null when the user ID is not found, but the result is used without a null check on line 45.

```suggestion
const user = getUser(id);
if (!user) {
  throw new UserNotFoundError(id);
}
```

---

(additional findings separated by `---`)

## Comments by File

```json
{
  "core/engine.ts": [
    {
      "author": "reviewer1",
      "line": 42,
      "body": "This looks like it could NPE",
      "created_at": "2024-01-15T09:00:00Z"
    }
  ],
  "utils/helper.ts": []
}
```
```

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `pr_owner` | string | GitHub repository owner |
| `pr_repo` | string | GitHub repository name |
| `pr_number` | integer | Pull request number |
| `head_sha` | string | The HEAD commit SHA of the PR at the time the review started |
| `started` | string | ISO 8601 timestamp of when the review session began |
| `updated` | string | ISO 8601 timestamp of the last state update |
| `mode` | string | `"full"` or `"quick"` |
| `focus` | string | Focus area (e.g., `"security"`, `"performance"`) or empty string if none |
| `self_review` | boolean | Whether `--self` mode is active |

### Chunk Status Values

- `pending` -- not yet reviewed
- `in-progress` -- currently being reviewed (at most one chunk at a time)
- `reviewed` -- review complete for this chunk
- `skipped` -- user chose to skip this chunk

### Finding Fields in State

Each finding in the `## Findings` section uses the format from `assets/finding-template.md` with one additional field:

- `**Marked for GitHub:** true/false` -- inserted after the Confidence line, controls whether this finding will be posted as a GitHub review comment.

Findings are separated by `---` (horizontal rule) for easy parsing.

## Operations

### 1. Save State

Write or overwrite the state file at `.review-buddy/review-buddy-state.md`.

Steps:
1. Ensure `.review-buddy/` directory exists in the project root.
2. Set `updated` to the current ISO 8601 timestamp.
3. Write the full state file with YAML frontmatter, chunk plan table, findings section, and comments-by-file section.

When to call: After every chunk review completes, after any finding is added or modified, and after any chunk status change.

### 2. Load State

Read and parse the state file. Return the structured data (frontmatter fields, chunk plan as a list, findings as a list, comments by file as a map).

Steps:
1. Check if `.review-buddy/review-buddy-state.md` exists.
2. If it does not exist, return `null` (indicates no prior session).
3. Read the file contents.
4. Parse the YAML frontmatter to extract metadata fields.
5. Parse the chunk plan table into a list of objects: `{ chunk: number, files: string[], loc: number, status: string }`.
6. Parse the findings section by splitting on `---` and extracting each finding's fields.
7. Parse the comments-by-file JSON block.
8. Return the full structured state.

### 3. Validate State

Check whether the loaded state is still valid for the current PR.

Steps:
1. Load the state using the Load State operation.
2. If state is `null`, return `{ valid: false, reason: "no_state" }`.
3. Fetch the current PR head SHA by running:
   ```
   gh pr view {PR_NUMBER} --repo {PR_OWNER}/{PR_REPO} --json headRefOid -q '.headRefOid'
   ```
4. Compare the fetched SHA with `head_sha` from the state file.
5. If they match, return `{ valid: true }`.
6. If they differ, warn the user:
   ```
   Warning: PR head has changed since the review started.
     State SHA:   {state_head_sha}
     Current SHA: {current_head_sha}

   Options:
     1. Continue with existing state (findings may reference outdated line numbers)
     2. Restart the review from scratch

   Which would you prefer? [1/2]
   ```
7. If the user chooses 1, proceed with existing state. If 2, delete the state file and return `{ valid: false, reason: "user_restart" }`.

Also check that `pr_owner`, `pr_repo`, and `pr_number` match the current PR being reviewed. If they do not match, this state file belongs to a different PR -- inform the user and offer to delete it.

### 4. Mark Chunk

Update a chunk's status in the chunk plan.

Steps:
1. Load the current state.
2. Find the chunk by its number in the chunk plan.
3. Update its status to the new value (`pending`, `in-progress`, `reviewed`, or `skipped`).
4. Save the state.

Valid transitions:
- `pending` -> `in-progress`
- `in-progress` -> `reviewed`
- `in-progress` -> `skipped`
- `pending` -> `skipped`

Do not allow other transitions (e.g., `reviewed` -> `pending`). If an invalid transition is attempted, log a warning but do not block execution.

### 5. Add Finding

Append a new finding to the findings section.

Steps:
1. Load the current state.
2. Format the finding using the `assets/finding-template.md` template.
3. Add `**Marked for GitHub:** true` by default (the user can toggle this later).
4. Append the finding to the `## Findings` section, separated from the previous finding by `---`.
5. Save the state.

### 6. Toggle GitHub Flag

Mark or unmark a specific finding for GitHub posting.

Steps:
1. Load the current state.
2. Identify the finding by its short title or by its index (1-based).
3. Toggle the `Marked for GitHub` value between `true` and `false`.
4. Save the state.

### 7. Export Findings

Write all findings marked for GitHub to `.review-buddy/review-buddy-findings.md` for use by the post-only mode.

Steps:
1. Load the current state.
2. Filter findings to only those with `Marked for GitHub: true`.
3. Write them to `.review-buddy/review-buddy-findings.md` with a header:
   ```markdown
   # Review Findings: {PR_OWNER}/{PR_REPO}#{PR_NUMBER}

   **Exported:** {ISO 8601 timestamp}
   **Head SHA:** {head_sha}
   **Total findings:** {count}

   ---

   {findings in assets/finding-template.md format, separated by ---}
   ```
4. Report the export path and count to the user:
   ```
   Exported {count} findings to .review-buddy/review-buddy-findings.md
   ```

### 8. Clean Up

Remove state files after a successful GitHub post or when the user explicitly requests cleanup.

Steps:
1. Delete `.review-buddy/review-buddy-state.md` if it exists.
2. Delete `.review-buddy/review-buddy-findings.md` if it exists.
3. Delete `.review-buddy/review-todo.md` if it exists.
4. Confirm to the user:
   ```
   Cleaned up review state and todo files.
   ```

Do not delete the `.review-buddy/` directory itself -- it may contain other files.

## Edge Cases

### State file does not exist

This is normal for a first run or after cleanup. Treat it as a fresh session -- the calling module should create a new chunk plan and initialize state from scratch.

### State file is corrupted or unparseable

If the YAML frontmatter cannot be parsed, or the chunk plan table is malformed, or any required field is missing:

1. Warn the user:
   ```
   Warning: Review state file is corrupted or has an unexpected format.
   Starting a fresh review session.
   ```
2. Delete the corrupted state file.
3. Return `null` so the calling module creates a fresh session.

### Head SHA has changed

Handled in the Validate State operation above. The key principle is: never silently continue with stale state. Always inform the user and let them decide.

### Concurrent sessions on different PRs

The state file does not support multiple simultaneous reviews. If a state file exists for PR `owner/repo#100` but the user invokes a review for `owner/repo#200`:

1. Warn the user:
   ```
   An existing review session was found for owner/repo#100.
   Starting a new review for owner/repo#200 will discard the previous session.

   Continue? [y/n]
   ```
2. If yes, delete the old state file and start fresh.
3. If no, abort and suggest the user resume the existing review with `/review continue`.
