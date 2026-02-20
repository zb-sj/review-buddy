# Continue Review Mode

This workflow resumes a previously paused full review session. It loads saved state and picks up from the first pending chunk.

Invoked via: `/review continue` or `/review continue <PR_ARG>`

## Context Management

**Lazy-load modules**: Only read a module file when you are about to execute that step. Re-read `modules/chunk-reviewer.md` before each chunk if deep into the review. Do not carry forward raw instructions from completed phases.

## Workflow

### Step 1: Parse PR argument (if provided)

If `<PR_ARG>` is provided, follow `scripts/parse-pr-arg.md` to parse it into `PR_OWNER`, `PR_REPO`, `PR_NUMBER`.

If no `<PR_ARG>` is provided, the PR identity will be read from the saved state file. Proceed to Step 2.

### Step 2: Load state

Follow the **Load State** operation from `scripts/state-manager.md` to read `.review-buddy/review-buddy-state.md`.

**If no state file exists:**

```
No review session found to resume.

To start a new review:
  /review <PR_NUMBER>        # full review
  /review quick <PR_NUMBER>  # quick single-pass review
```

Stop here.

**If state file exists but is corrupted:**

Follow the corruption handling in `scripts/state-manager.md` -- warn the user, delete the file, and display the same "no session found" message above. Stop here.

### Step 3: Validate state

If `<PR_ARG>` was provided in Step 1, verify that the loaded state matches the requested PR:
- `PR_OWNER`, `PR_REPO`, and `PR_NUMBER` from the argument must match the state's `pr_owner`, `pr_repo`, and `pr_number`.
- If they do not match, present a gate following the **Agnostic Interaction Protocol** (`references/PROTOCOL.md`):

  Context: "The saved session is for {pr_owner}/{pr_repo}#{pr_number}, but you requested {PR_OWNER}/{PR_REPO}#{PR_NUMBER}."

  **Options:**
  1. **Resume saved session** — Continue reviewing {pr_owner}/{pr_repo}#{pr_number}
  2. **Start fresh** — Discard saved session and begin a new review of {PR_OWNER}/{PR_REPO}#{PR_NUMBER}

  If the user chooses 1, continue with the saved state. If 2, delete the state file and stop -- the user should invoke `/review-buddy <PR_ARG>` to start fresh.

Follow the **Validate State** operation from `scripts/state-manager.md` to check if the PR head SHA has changed since the session was saved. If it has diverged, present the user with the options described in `scripts/state-manager.md` (continue with stale data or restart).

### Step 4: Display session summary

Show the user a summary of the saved session:

```
Resuming review: {pr_owner}/{pr_repo}#{pr_number}
Mode: {mode} | Focus: {focus or "none"} | Self-review: {self_review}
Started: {started} | Last updated: {updated}

Chunk Progress:
  {chunk_table with status indicators}

Findings so far: {count} ({action_required_count} critical, {recommended_count} recommended, {minor_count} minor)
```

For the chunk table, use visual indicators:
- `[x]` for `reviewed` chunks
- `[-]` for `skipped` chunks
- `[ ]` for `pending` chunks
- `[>]` for `in-progress` chunks (should be at most one; treat as pending on resume)

### Step 5: Resume chunk review (Phase 4)

Identify the first chunk with status `pending` or `in-progress` (treat `in-progress` as `pending` on resume since the previous session was interrupted mid-chunk).

For each remaining pending chunk, follow `modules/chunk-reviewer.md`:
1. Mark the chunk as `in-progress` using the **Mark Chunk** operation from `scripts/state-manager.md`.
2. Run the chunk review as described in `chunk-reviewer.md`, displaying findings and the user gate.
3. After the user confirms or skips the chunk:
   - Mark the chunk as `reviewed` or `skipped`.
   - Add any findings using the **Add Finding** operation from `scripts/state-manager.md`.
   - Save state using the **Save State** operation.
4. Proceed to the next pending chunk.

The user can interrupt at any point. State is saved after each chunk, so the session can be resumed again later with `/review continue`.

### Step 6: Synthesis (Phase 5)

Once all chunks are `reviewed` or `skipped`, follow `modules/synthesis.md` to aggregate all findings (both from the current session and from previously reviewed chunks loaded from state), determine an overall verdict, and display the summary.

The synthesis module will present a user gate asking whether to post findings to GitHub.

### Step 7: GitHub posting (conditional)

If the user chooses to post:
1. Follow `modules/github-post.md` to post findings marked for GitHub.
2. After successful posting, follow the **Clean Up** operation from `scripts/state-manager.md` to remove state files.

If the user declines:
1. Follow the **Export Findings** operation from `scripts/state-manager.md` to save findings to `.review-buddy/review-buddy-findings.md`.
2. Display:
   ```
   Findings saved to .review-buddy/review-buddy-findings.md
   You can post them later with: /review post
   ```
