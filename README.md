# Review Buddy

**Chunked interactive PR review** — the only agent skill that walks you through PRs chunk by chunk, like a real pair-review partner.

## Why Another Review Skill?

Most PR review skills do one thing: dump a wall of findings on you. They run a single pass over the diff, produce a long list, and leave you to sort through it. That's not a review — that's a report.

**Review Buddy is different.** It breaks PRs into logical chunks, walks you through each one interactively, and only posts to GitHub what you've actually reviewed and approved. It's the difference between a code scanner and a pair-programming partner.

### How It Compares

| Feature | Typical Review Skill | Review Buddy |
|---------|---------------------|--------------|
| Review style | Single-pass dump | Interactive chunk-by-chunk walkthrough |
| Chunk logic | None (file list) | Semantic grouping by commit, dependency, and naming |
| Reviewer control | Take it or leave it | Deep-dive, skip, pause, resume, deselect per-finding |
| Existing comments | Ignored | Digested and surfaced — won't duplicate feedback |
| False positive reduction | None | Actor-Critic self-reflection on every finding |
| GitHub posting | All-or-nothing | Selective — post all, post critical only, or pick individual findings |
| State persistence | None | Pause mid-review, resume later with `--continue` |
| Focus modes | Generic | Security, performance, correctness, types, error-handling |
| Self-review | No | `--self` suppresses nits for reviewing your own PR |
| Visual review | No | Figma design comparison, Playwright/Cypress regression diffs |
| Platform lock-in | Usually Claude Code only | Agent-agnostic — works on any platform supporting SKILL.md |
| Educational value | Findings only | Mentor notes explain *why*, grounded in project conventions |

## Quick Start

```bash
# Install (Claude Code)
claude skill add --from zb-sj/review-buddy

# Review the current branch's PR
/review-buddy

# Review a specific PR
/review-buddy 123

# Quick single-pass review
/review-buddy --quick

# Focus on security
/review-buddy --focus security

# Self-review before requesting teammates
/review-buddy --quick --self
```

## How It Works

Review Buddy runs a structured 5-phase workflow:

1. **Context Assembly** — Fetches PR metadata, diffs, CI status, linked issues, and discovers the project's tech stack and coding patterns. Detects visual changes automatically.

2. **Parallel Background Analysis** — While you review the overview, two analyses run in parallel:
   - *Comments Digest*: Aggregates existing review threads so nothing gets missed
   - *Static Scanner*: Catches low-hanging fruit (secrets, debug logs, anti-patterns)

3. **Chunk Planning** — Groups files into semantic chunks (~300 LOC each) based on commits, dependencies, and naming. Orders them for reviewability — foundations first, consumers after.

4. **Interactive Chunk Review** — Walks you through each chunk with an Actor-Critic loop that self-reflects on every finding to minimize false positives. At each chunk you can:
   - **Continue** to the next chunk
   - **Deep-dive** into a specific finding
   - **Pause & save** to resume later
   - **Skip to synthesis** when you've seen enough

5. **Synthesis & Posting** — Aggregates findings, computes a verdict (Approve / Suggest / Request Changes), and lets you choose exactly what to post to GitHub.

## All Options

```
/review-buddy [PR] [options]

Arguments:
  [PR]              PR number, URL, or owner/repo#number
                    (optional — defaults to current branch's open PR)

Options:
  --quick           Single-pass review (no chunking)
  --focus <area>    security, performance, correctness, types, error-handling
  --self            Self-review mode (suppresses nits)
  --visual          Force visual regression review
  --no-mentoring    Disable educational mentor notes
  --continue        Resume a paused review
  --post-only       Post saved findings to GitHub
```

## Requirements

- `gh` CLI authenticated with GitHub
- An agent runtime that supports the [Agent Skills](https://agentskills.io) open standard (SKILL.md)

## License

MIT
