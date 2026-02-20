# 🤖 Review Buddy

**Chunked interactive PR review** — the only agent skill that walks you through PRs chunk by chunk, like a real pair-review partner.

---

## 🤔 Why Another Review Skill?

Most PR review skills do one thing: dump a wall of findings on you. They run a single pass over the diff, produce a long list, and leave you to sort through it. That's not a review — that's a report.

**Review Buddy is different.** It breaks PRs into logical chunks, walks you through each one interactively, and only posts to GitHub what you've actually reviewed and approved. It's the difference between a code scanner and a pair-programming partner.

### How It Compares

| Feature | Typical Review Skill | Review Buddy |
| ------- | -------------------- | ------------ |
| Review style | Single-pass dump | Interactive chunk-by-chunk walkthrough |
| Chunk logic | None (file list) | Semantic grouping by commit, dependency, and naming |
| Reviewer control | Take it or leave it | Deep-dive, skip, pause, resume, deselect per-finding |
| Existing comments | Ignored | Digested, surfaced, and replied in-thread — won't duplicate feedback |
| False positive reduction | None | Actor-Critic self-reflection on every finding |
| GitHub posting | All-or-nothing | Selective — post all, post critical only, or pick individual findings |
| State persistence | None | Pause mid-review, resume later with `--continue` |
| Focus modes | Generic | Security, performance, correctness, types, error-handling |
| Self-review | No | `--self` suppresses nits for reviewing your own PR |
| Visual review | No | Figma design comparison, Playwright/Cypress regression diffs |
| Platform lock-in | Usually Claude Code only | Agent-agnostic — works on any platform supporting SKILL.md |
| Educational value | Findings only | Mentor notes explain *why*, grounded in project conventions |

---

## 🚀 Key Features

- **🧠 Cognitive Load Management**: Breaks large PRs into reviewable chunks (~300 LOC) based on semantic grouping.
- **🎯 Smart Prioritization**: Groups files by commit, dependency, and naming to ensure you review foundational changes first.
- **💬 Context Awareness**: Surfaces existing comments and unresolved threads so you don't miss previous feedback.
- **🔗 Thread Continuation**: Replies in existing comment threads instead of creating duplicates.
- **🧩 Agnostic & Future-Ready**: Built to run on any AI platform using standard protocols, while proactively integrating the latest AI and developer tooling innovations.
- **🎓 Educational & Mentoring**: Provides concise "Mentor's Notes" (why, pro-tips, domain context) for each finding to help both the reviewer and reviewee learn and grow.
- **⏯️ Pause & Resume**: Saves your progress in a local state file, allowing you to stop and continue your review anytime.
- **⚡ Multiple Modes**:
  - **Interactive**: The default guided experience.
  - **Quick**: A single-pass review for small PRs.
  - **Visual**: Automated and interactive visual regression review (via Figma, Playwright, or MobileMCP).
  - **Self-Review**: Optimized for reviewing your own work (suppresses minor nits).
  - **Focused**: Concentrates analysis on specific areas like `security`, `performance`, or `types`.

---

## 🎨 Visual Review & Regression

Review Buddy automatically detects visual changes and triggers an enhanced review phase when:
- **UI Components/Styles** are changed (`.tsx`, `.jsx`, `.css`, `.scss`, etc.).
- **Figma Links** are found in the PR description.
- **Playwright/Cypress Snapshots** are updated in the PR.

### How it helps:
1. **Design Fidelity**: Compares your implementation against Figma designs using multimodal AI.
2. **Regression Testing**: Identifies and surfaces visual diffs from automated snapshot tests.
3. **Mobile Inspection**: Leverages MobileMCP for mobile-specific visual verification.

To force a visual review even if not automatically detected, use:
```bash
/review-buddy --visual
```

---

## 🛠 How It Works

Review Buddy follows a structured 5-phase workflow:

1.  **Phase 1: Context Assembly**: Fetches PR metadata, diffs, and project context.
2.  **Phase 2: Comments Digest**: Aggregates existing review comments to highlight areas needing attention or verification.
3.  **Phase 3: Chunk Planning**: Groups files into semantic chunks by commit, dependency, and naming for an optimized review order.
4.  **Phase 4: Interactive Review**: Walks you through each chunk one by one, allowing you to record findings and mark them as "Action Required" or "Suggestions."
5.  **Phase 5: Synthesis & Post**: Summarizes all findings, provides a verdict (Approve/Request Changes), and submits the review to GitHub.

---

## 💻 Usage

Invoke Review Buddy using the `/review-buddy` command followed by optional arguments.

### Basic Commands

```bash
/review-buddy                # Review the open PR for the current branch
/review-buddy 123            # Review PR #123
/review-buddy --quick        # Quick, non-interactive review of the current PR
/review-buddy --continue     # Resume a previously paused review session
/review-buddy --post-only    # Submit saved findings to GitHub without re-reviewing
```

### Advanced Options

- `--focus <area>`: Focus on `security`, `performance`, `correctness`, `types`, or `error-handling`.
- `--self`: Self-review mode (suppresses minor nits).
- `--visual`: Force visual regression review even if not auto-detected.
- `--no-mentoring`: Disable educational mentor notes and pro-tips.
- `[PR]`: Supports PR numbers (`123`), URLs (`https://github.com/org/repo/pull/123`), or shorthand (`org/repo#123`).

### Examples

```bash
# Focus on security for a specific PR
/review-buddy 456 --focus security

# Perform a quick self-review before asking for teammates
/review-buddy --quick --self
```

---

## 📂 Project Structure

- `SKILL.md`: The main entry point and argument router.
- `review.md`: The full interactive 5-phase review workflow.
- `quick.md`: Single-pass review workflow.
- `continue.md`: Logic for resuming a paused session.
- `post.md`: Post-only workflow for submitting saved findings.
- `modules/`: Individual phase implementations (Planning, Reviewing, Synthesis, etc.).
- `scripts/`: Utility scripts for state management, PR parsing, and ad-hoc tasks.
- `assets/`: Static templates for findings and other reports.
- `references/`: Interaction protocols (PROTOCOL.md).
- `.github/`: GitHub-rendered docs (this README, AGENTS.md) — kept outside the skill path.

---

## ⚙️ Requirements

- **GitHub CLI (`gh`)**: Must be authenticated and available in your shell.
- **AI Agent Environment**: Any agent runtime supporting the [Agent Skills](https://agentskills.io) open standard (SKILL.md).
- **Permissions**: Needs read access to PRs/Issues and write access to PR Reviews.

---

## 📝 State Management

Review Buddy maintains a local state file at `.review-buddy/review-buddy-state.md`. This file tracks:
- The current phase and chunk.
- The planned review order.
- All recorded findings and their severity.
- PR metadata.

*Note: You can safely delete this file if you want to clear your local review history.*

---

## 🤝 Contributing

This project is built using a "Markdown-as-Code" architecture. Workflows are defined in instructional Markdown files that the AI agent interprets and executes. To contribute, you can modify the logic in the `modules/` or `scripts/` directories.
