# 🤖 Review Buddy

**Review Buddy** is an interactive, chunked Pull Request review partner designed to help developers review PRs without cognitive overload. It intelligently breaks down large PRs into logical groups of files (chunks), surfaces existing reviewer feedback, and guides you through a focused review process.

---

## 🚀 Key Features

- **🧠 Cognitive Load Management**: Breaks large PRs into reviewable chunks (~300 LOC) based on file type and dependency order.
- **🎯 Smart Prioritization**: Groups files by category (Types → Config → Core Logic → Components → Tests) to ensure you review foundational changes first.
- **💬 Context Awareness**: Surfaces existing comments and unresolved threads so you don't miss previous feedback.
- **🧩 Agnostic & Future-Ready**: Built to run on any AI platform (Gemini, Claude, etc.) using standard protocols, while proactively integrating the latest AI and developer tooling innovations.
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
3.  **Phase 3: Chunk Planning**: Categorizes files and builds a dependency graph to create an optimized review order.
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
- `--self`: Self-review mode (less critical of nits).
- `--PR`: Supports URLs (`https://github.com/org/repo/pull/123`) or shorthand (`org/repo#123`).

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
- `modules/`: Individual phase implementations (Planning, Reviewing, Synthesis, etc.).
- `scripts/`: Utility scripts for state management, PR parsing, and ad-hoc tasks.
- `assets/`: Static templates for findings and other reports.
- `references/`: Documentation and interaction protocols.

---

## ⚙️ Requirements

- **GitHub CLI (`gh`)**: Must be authenticated and available in your shell.
- **AI Agent Environment**: Designed to run within compatible agent skill enabled agents.
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
