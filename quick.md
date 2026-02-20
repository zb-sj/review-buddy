# Quick Review Mode

This workflow performs a single-pass review of a PR without chunking. It is faster than a full review but less thorough -- best for small PRs or when a quick sanity check is needed.

Invoked via: `/review quick <PR_ARG>` or `/review quick <PR_ARG> --focus <area>` or `/review quick <PR_ARG> --self` or `/review quick <PR_ARG> --no-mentoring`

## Flags

| Flag | Effect |
|------|--------|
| `--self` | Self-review mode: suppress Minor findings (see `assets/finding-template.md`) |
| `--focus <area>` | Boost confidence for findings matching the focus area (e.g., `security`, `performance`) |
| `--no-mentoring` | Disable educational mentor notes and pro-tips |

## Context Management

**Lazy-load modules**: Only read a module file when you are about to execute that step. Do not pre-read all referenced modules — PR data will consume significant context.

## Workflow

### Step 1: Parse PR argument

Follow `scripts/parse-pr-arg.md` to parse `<PR_ARG>` into `PR_OWNER`, `PR_REPO`, `PR_NUMBER`.

If parsing fails, stop.

### Step 2: Context assembly (Phase 1)

Follow `modules/context-assembly.md` to fetch PR metadata, changed files, CI status, commits, and linked issues.

Display the PR overview to the user. Wait for the user gate -- the user must confirm to proceed.

### Step 3: Comments digest (Phase 2)

Follow `modules/comments-digest.md` to fetch all existing PR comments, categorize them with trust levels, and build the `comments_by_file` map.

Display the comment summary to the user.

### Step 4: Single-pass analysis

Instead of chunking the diff, analyze the **entire diff at once** in a single pass.

Instructions for the analysis:

1. Review all changed files together, considering the full context from Phase 1 and Phase 2.
2. For each issue found, create a finding using the format from `assets/finding-template.md`. If `mentoring` is true, also generate a "Mentor's Note" for the finding.
3. Apply all filtering rules from `assets/finding-template.md`:
   - Only include findings with confidence >= 80.
   - If `--self` is set, suppress all `🟢 Minor` findings.
   - If `--focus` is set, boost confidence by +10 for findings matching the focus area.
4. Only surface `🔴 Action Required` and `🟡 Recommended` findings. Even if `--self` is not set, quick mode suppresses `🟢 Minor` findings to keep the output concise.
5. Cross-reference findings against existing comments from the `comments_by_file` map. If an existing comment already covers the same issue, note it in the finding's related comment reference rather than duplicating the feedback.

Display all findings grouped by severity:
- `🔴 Action Required` first
- `🟡 Recommended` second

If no findings meet the threshold, report:

```
No significant issues found in quick review.
```

### Step 5: Synthesis (Phase 5)

Follow `modules/synthesis.md` to aggregate findings, determine an overall verdict, and display the summary. Pass the `mentoring` flag to control the generation of educational summaries.

The synthesis module will present a user gate asking whether to post findings to GitHub.

### Step 6: GitHub posting (conditional)

If the user chooses to post:
1. All findings are marked for GitHub posting by default in quick mode.
2. Follow `modules/github-post.md` to create a pending review, add inline comments for each finding, and submit the review with the verdict.

If the user declines, display:

```
Review complete. Findings were not posted to GitHub.
```

No state files are saved in quick mode -- the entire review happens in a single session with no pause/resume capability.
