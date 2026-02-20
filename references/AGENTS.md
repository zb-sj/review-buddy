# AGENTS.md - Review Buddy Context

This file provides context and instructions for AI agents working on or with the **Review Buddy** skill.

## Project Overview
**Review Buddy** is an interactive, chunked Pull Request review system designed to reduce cognitive overload for reviewers. It breaks down large PRs into logical groups of files (chunks), surfaces existing reviewer feedback, and facilitates a guided review process that culminates in a structured GitHub review submission.

The project is implemented as a modular collection of Markdown-based instruction sets and scripts, designed to be executed by an AI agent with access to GitHub tools.

## Tech Stack
- **Logic:** Markdown-driven workflows (`.md` files) containing structured instructions for AI agents.
- **Integration:** GitHub CLI (`gh`) and GitHub MCP tools (e.g., `pull_request_read`, `pull_request_review_write`).
- **State Management:** Markdown-as-DB pattern, storing session state in `.review-buddy/review-buddy-state.md`.
- **Interaction:** Agnostic Interaction Protocol (defined in `references/PROTOCOL.md`) for handling user gates.

## Design Philosophy: Agnostic yet Innovative

Review Buddy follows a strict **Agnostic Approach** while maintaining an **Innovation-First** mindset:

1.  **Platform Agnostic**: Core logic must reside in instructional Markdown and utilize standard tools (GitHub CLI, File System). Avoid proprietary vendor APIs that lock the skill into a single agent or IDE.
2.  **Education-First**: Act as a mentor, not just a scanner. Every finding should ideally include a "Why" or a "Pro-tip" that grounds the suggestion in project standards or engineering principles. This educates both the interactive reviewer and the PR author.
3.  **Protocol-First**: Always provide a standard Markdown fallback for interaction gates (see `references/PROTOCOL.md`).
3.  **Future-Ready Exploration**: While maintaining agnosticism, agents are encouraged to explore and pilot upcoming technologies (e.g., experimental MCP servers, new GitHub CLI features, or advanced reasoning models) to improve review depth. 
4.  **Fallback Guarantee**: Any "cutting-edge" feature must have a functional fallback to ensure the skill remains operational in basic environments.

## Setup & Requirements
- **GitHub CLI:** The agent must have `gh` authenticated and available in the environment.
- **MCP Tools:** Access to GitHub MCP tools is preferred for richer interactions, though `gh` fallback exists.
- **State Path:** Ensure `.review-buddy/` directory exists or can be created in the project root for state persistence.

## Development Conventions

### 1. State Persistence
Review state is stored in `.review-buddy/review-buddy-state.md`. This file uses YAML frontmatter for metadata and Markdown tables/blocks for findings. Use `scripts/state-manager.md` logic for all state operations.

### 2. Interaction Protocol
User gates must follow `references/PROTOCOL.md`. The protocol auto-detects the best available interaction method (structured selection tools, IDE prompts, or plain I/O fallback).

### 3. Chunking Logic
Files are grouped into chunks of ~300 LOC. See `modules/chunk-planner.md` for the priority-based sorting and grouping algorithm.

### 4. Findings Format
All findings must adhere to the template in `assets/finding-template.md`, including severity triage (🔴 Action Required, 🟡 Suggestion, 🔵 Comment).

## How to Extend
- **Adding a Phase:** Create a new file in `modules/` and integrate it into the `review.md` workflow.
- **Modifying Logic:** Update the corresponding `.md` file. Since the "code" is instructions, ensure they remain unambiguous and tool-centric.
- **Adding a Script:** Place utility logic in `scripts/` using the `_name.md` convention.

## Project Structure

```

/

├── SKILL.md            # Entry point & Router

├── review.md           # Full review workflow

├── quick.md            # Single-pass workflow

├── continue.md         # Resume workflow

├── post.md             # Post-only workflow

├── modules/            # Phase-specific logic (Planning, Synthesis, etc.)

├── scripts/            # Utility logic (State, Parsing, etc.)

├── assets/             # Static templates (Findings format)

└── references/         # Documentation & Protocols (README, AGENTS, PROTOCOL)

```
