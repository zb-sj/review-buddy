# Post-Only Mode

This workflow posts previously saved review findings to GitHub without performing a new review. It reads findings from the exported findings file and lets the user select which ones to post.

Invoked via: `/review post` or `/review post <PR_ARG>`

## Workflow

### Step 1: Parse PR argument (if provided)

If `<PR_ARG>` is provided, follow `scripts/parse-pr-arg.md` to parse it into `PR_OWNER`, `PR_REPO`, `PR_NUMBER`.

If no `<PR_ARG>` is provided, the PR identity will be read from the findings file header. Proceed to Step 2.

### Step 2: Load findings

Read the findings export file at `.review-buddy/review-buddy-findings.md`.

**If the file does not exist:**

```
No saved findings found.

Run a review first to generate findings:
  /review <PR_NUMBER>        # full review
  /review quick <PR_NUMBER>  # quick single-pass review
```

Stop here.

**If the file exists**, parse it:
1. Extract the header fields: PR identity (`PR_OWNER/PR_REPO#PR_NUMBER`), export timestamp, head SHA, total findings count.
2. Parse each finding separated by `---`, extracting: severity badge, short title, file path, line number, category, confidence, description, suggestion block (if any), and the marked-for-GitHub flag.

### Step 3: Validate PR identity

If `<PR_ARG>` was provided, verify it matches the findings file:
- If they do not match, warn:
  ```
  The saved findings are for {file_owner}/{file_repo}#{file_number}, but you specified {PR_OWNER}/{PR_REPO}#{PR_NUMBER}.

  The saved findings can only be posted to the PR they were generated for.
  ```
  Stop here.

If no `<PR_ARG>` was provided, use the PR identity from the findings file as `PR_OWNER`, `PR_REPO`, `PR_NUMBER`.

### Step 4: Check head SHA

Fetch the current PR head SHA:
```
gh pr view {PR_NUMBER} --repo {PR_OWNER}/{PR_REPO} --json headRefOid -q '.headRefOid'
```

Compare with the head SHA in the findings file. If they differ, warn:

```
Warning: PR head has changed since findings were generated.
  Findings SHA: {findings_sha}
  Current SHA:  {current_sha}

Line numbers in findings may no longer be accurate.
Continue posting anyway? [y/n]
```

If the user declines, stop here.

### Step 5: Display findings summary

Show a numbered list of all findings with their current posting status:

```
Saved findings for {PR_OWNER}/{PR_REPO}#{PR_NUMBER}:

  #  Severity              Title                           File              Post?
  1  🔴 Action Required    Unchecked null dereference      core/engine.ts:42   [x]
  2  🟡 Recommended        Missing error boundary          api/handler.ts:15   [x]
  3  🟡 Recommended        Unbounded retry loop            utils/fetch.ts:88   [x]
  4  🟢 Minor              Inconsistent naming             types.ts:12         [ ]

Total: 4 findings (3 marked for posting)
```

### Step 6: User selection

Let the user adjust which findings to post:

```
Options:
  all     - Mark all findings for posting
  none    - Unmark all findings
  toggle N - Toggle finding #N
  post    - Proceed with posting marked findings
  cancel  - Cancel without posting

Your choice:
```

Repeat the prompt until the user chooses `post` or `cancel`.

If the user types `cancel`:
```
Posting cancelled. Findings are still saved in .review-buddy/review-buddy-findings.md
```
Stop here.

### Step 7: Post to GitHub

Follow `modules/github-post.md` to post only the findings marked for GitHub:

1. Create a pending review on the PR.
2. For each marked finding, add an inline review comment at the appropriate file and line number.
3. Submit the review with the appropriate verdict based on the findings being posted:
   - If any `🔴 Action Required` findings are included: request changes.
   - If only `🟡 Recommended` or `🟢 Minor` findings: comment (no change request).

### Step 8: Clean up

After successful posting:

1. Follow the **Clean Up** operation from `scripts/state-manager.md` to remove both `.review-buddy/review-buddy-state.md` and `.review-buddy/review-buddy-findings.md`.
2. Display:
   ```
   Posted {count} findings to {PR_OWNER}/{PR_REPO}#{PR_NUMBER}.
   Review state cleaned up.
   ```
