# AGENTS.md - Review Buddy Context

This file provides context and instructions for AI agents working on or with the **Review Buddy** skill.

> **Note:** This file lives in `.github/` so agents **using** Review Buddy as a skill won't load it into context. GitHub still renders `.github/README.md` as the repo landing page. If you're doing agentic development **on** this project, symlink this file to the repo root (`ln -s .github/AGENTS.md AGENTS.md`) so your coding agent auto-discovers it. Don't commit the symlink.

## Project Overview
**Review Buddy** is an interactive, chunked Pull Request review system designed to reduce cognitive overload for reviewers. It breaks down large PRs into semantically coherent chunks (grouped by logical change, not file type), surfaces existing reviewer feedback, and facilitates a guided review process that culminates in a structured GitHub review submission.

The project is implemented as a modular collection of Markdown-based instruction sets and scripts, designed to be executed by an AI agent with access to GitHub tools.

## Tech Stack
- **Logic:** Markdown-driven workflows (`.md` files) containing structured instructions for AI agents.
- **Integration:** GitHub CLI (`gh`) and GitHub MCP tools (e.g., `pull_request_read`, `pull_request_review_write`, `add_reply_to_pull_request_comment`).
- **State Management:** Markdown-as-DB pattern, storing session state in `.review-buddy/review-buddy-state.md`.
- **Interaction:** Agnostic Interaction Protocol (defined in `references/PROTOCOL.md`) for handling user gates.

## Design Philosophy: Agnostic yet Innovative

Review Buddy follows a strict **Agnostic Approach** while maintaining an **Innovation-First** mindset:

1.  **Platform Agnostic**: Core logic must reside in instructional Markdown and utilize standard tools (GitHub CLI, File System). Avoid proprietary vendor APIs that lock the skill into a single agent or IDE.
2.  **Education-First**: Act as a mentor, not just a scanner. Every finding should ideally include a "Why" or a "Pro-tip" that grounds the suggestion in project standards or engineering principles. This educates both the interactive reviewer and the PR author.
3.  **Protocol-First**: Always provide a standard Markdown fallback for interaction gates (see `references/PROTOCOL.md`).
4.  **Future-Ready Exploration**: While maintaining agnosticism, agents are encouraged to explore and pilot upcoming technologies (e.g., experimental MCP servers, new GitHub CLI features, or advanced reasoning models) to improve review depth.
5.  **Fallback Guarantee**: Any "cutting-edge" feature must have a functional fallback to ensure the skill remains operational in basic environments.

## Setup & Requirements
- **GitHub CLI:** The agent must have `gh` authenticated and available in the environment.
- **MCP Tools:** Access to GitHub MCP tools is preferred for richer interactions, though `gh` fallback exists.
- **State Path:** Ensure `.review-buddy/` directory exists or can be created in the project root for state persistence.

## Development Conventions

### 1. State Persistence

Review state is stored in `.review-buddy/review-buddy-state.md`. This file uses YAML frontmatter for metadata and Markdown tables/blocks for findings. Use `scripts/state-manager.md` logic for all state operations.

### 2. Interaction Protocol

User gates must follow `references/PROTOCOL.md`. The protocol auto-detects the best available interaction method (structured selection tools, IDE prompts, or plain I/O fallback). The agent MUST stop after presenting a gate and never auto-proceed. Destructive actions (e.g., discard) require explicit confirmation.

### 3. Context Management (Lazy-Loading)

**Only read a module file when about to execute that phase.** PR diffs, metadata, and review context consume significant space — reading all instructions upfront will cause later phases to lose their detailed gate instructions. After completing a phase, retain only its output data; discard raw instructions.

### 4. Chunking Logic

Files are grouped into semantically coherent chunks of ~300 LOC. Each chunk represents one logical change, feature, or concern — files are grouped by the change they participate in, not by file type or directory. See `modules/chunk-planner.md` for the full algorithm.

### 5. Actor-Critic (Reflexion)

The chunk reviewer uses an Actor-Critic pattern: after generating findings, the agent critiques its own output to minimize false positives and ensure high-quality feedback. See `modules/chunk-reviewer.md` for details.

### 6. Thread Continuation

When posting findings to GitHub, if a finding relates to an existing comment thread, the agent replies in that thread (via `add_reply_to_pull_request_comment`) instead of creating a duplicate inline comment. See `modules/github-post.md` for routing logic.

### 7. Findings Format

All findings must adhere to the template in `assets/finding-template.md`, including severity triage (🔴 Action Required, 🟡 Recommended, 🟢 Minor).

## How to Extend
- **Adding a Phase:** Create a new file in `modules/` and integrate it into the `review.md` workflow.
- **Modifying Logic:** Update the corresponding `.md` file. Since the "code" is instructions, ensure they remain unambiguous and tool-centric.
- **Adding a Script:** Place utility logic in `scripts/` using the `_name.md` convention.

## Project Structure

```text
/
├── SKILL.md                    # Entry point & Router
├── .github/
│   ├── README.md               # GitHub landing page (not in skill path)
│   └── AGENTS.md               # This file (contributor context)
├── review.md                   # Full review workflow
├── quick.md                    # Single-pass workflow
├── continue.md                 # Resume workflow
├── post.md                     # Post-only workflow
├── modules/
│   ├── context-assembly.md     # Phase 1: PR metadata & project discovery
│   ├── comments-digest.md      # Phase 2: Existing comment analysis
│   ├── static-scanner.md       # Phase 2.5: Deterministic anti-pattern scan
│   ├── chunk-planner.md        # Phase 3: Semantic file grouping
│   ├── chunk-reviewer.md       # Phase 4: Per-chunk analysis (Actor-Critic)
│   ├── synthesis.md            # Phase 5: Findings aggregation & verdict
│   ├── github-post.md          # GitHub review submission (with thread replies)
│   ├── visual-discovery.md     # Visual change detection
│   ├── visual-reviewer.md      # Visual regression analysis
│   └── teams/                  # Specialized agent personas
│       ├── protocol.md
│       ├── security-specialist.md
│       ├── performance-architect.md
│       ├── clean-code-mentor.md
│       └── visual-designer.md
├── scripts/
│   ├── parse-pr-arg.md         # PR argument normalization
│   ├── state-manager.md        # Session state persistence
│   └── todo-manager.md         # Agnostic task management
├── assets/
│   └── finding-template.md     # Finding format (with Reflexion support)
└── references/
    └── PROTOCOL.md             # Agnostic interaction protocol
```
