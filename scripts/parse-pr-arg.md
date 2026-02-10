# Parse PR Argument

This module normalizes a PR argument into three variables: `PR_OWNER`, `PR_REPO`, and `PR_NUMBER`.

## Input

The user provides a PR reference as `$PR_ARG` in one of these formats:
- **No argument** (empty/null): auto-detect from current branch
- **Bare number**: `123`
- **Full URL**: `https://github.com/owner/repo/pull/123`
- **Short form**: `owner/repo#123`

## Parsing Steps

### Step 0: Auto-detect from current branch (no argument)

If `$PR_ARG` is empty, null, or not provided:

1. Get the current branch name:
   ```
   git branch --show-current
   ```
2. Look for an open PR associated with this branch:
   ```
   gh pr view --json number,headRefName,state,url -q 'select(.state == "OPEN") | .number'
   ```
   This checks the current branch's open PR. If `gh pr view` succeeds, it returns the PR number.
3. If a PR is found, derive `PR_OWNER` and `PR_REPO` from the repo (same as the bare number case below), and set `PR_NUMBER` to the returned number.
4. If no open PR is found for the current branch, display:
   ```
   No open PR found for branch "{branch_name}".

   Provide a PR explicitly:
     /review-buddy 123
     /review-buddy https://github.com/org/repo/pull/123
     /review-buddy org/repo#123
   ```
   Stop here.

### Step 1: Detect format and extract components

Check the formats in this order:

1. **Full URL** -- if `$PR_ARG` contains `github.com`:
   - Extract owner, repo, and number by matching the pattern: `github\.com/([^/]+)/([^/]+)/pull/(\d+)`
   - The URL may have a trailing slash, query parameters, or fragment -- ignore everything after the number.
   - Set `PR_OWNER` to the first capture group, `PR_REPO` to the second, `PR_NUMBER` to the third.

2. **Short form** -- if `$PR_ARG` contains `#`:
   - Split on `#` to get `left_part` and `PR_NUMBER`.
   - Split `left_part` on `/` to get `PR_OWNER` and `PR_REPO`.
   - If `left_part` does not contain exactly one `/`, this is an invalid short form -- go to the error step.

3. **Bare number** -- otherwise:
   - Set `PR_NUMBER` to `$PR_ARG`.
   - Derive `PR_OWNER` and `PR_REPO` from the current repository by running:
     ```
     gh repo view --json owner,name -q '.owner.login + "/" + .name'
     ```
   - If that command fails (e.g., `gh` not authenticated or not in a repo), fall back to parsing the git remote:
     ```
     git remote get-url origin
     ```
     Then extract owner and repo from the remote URL. Handle both SSH (`git@github.com:owner/repo.git`) and HTTPS (`https://github.com/owner/repo.git`) formats. Strip any trailing `.git` suffix.
   - If both methods fail, go to the error step.

### Step 2: Validate PR_NUMBER

- Confirm that `PR_NUMBER` is a positive integer (matches `^\d+$` and is greater than 0).
- If it is not a valid positive integer, go to the error step.

### Step 3: Store results

After successful parsing, the following variables are available for use by other modules:

- `PR_OWNER` -- the GitHub repository owner (user or organization)
- `PR_REPO` -- the repository name
- `PR_NUMBER` -- the pull request number (as an integer)

Confirm the parsed result to the user in a single line:

```
Targeting PR: {PR_OWNER}/{PR_REPO}#{PR_NUMBER}
```

## Error Handling

If parsing fails at any point, display a clear error message:

```
Error: Could not parse PR reference "{$PR_ARG}".

Usage:
  /review-buddy                                  # auto-detect from current branch
  /review-buddy 123                              # PR number (uses current repo)
  /review-buddy https://github.com/org/repo/pull/123   # full URL
  /review-buddy org/repo#123                     # short form
```

Do not proceed with any further review steps after a parsing error.
