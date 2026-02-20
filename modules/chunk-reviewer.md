# Chunk Reviewer (Phase 4)

You are performing Phase 4 of a chunked PR review — the interactive per-chunk analysis loop. For each chunk, you display existing comments, analyze the diff, present findings, and let the user control the flow.

## Prerequisites

- Phase 1–3 completed
- `chunks` — ordered list of file groups with LOC and metadata
- `comments_by_file` — existing PR comments with trust levels
- `changed_files` — full file data including patch/diff content
- `user_start_chunk` — which chunk to begin from
- `focus` — optional focus area (security, performance, correctness, types, error-handling)
- `self_review` — boolean, if true suppress Minor findings

## Per-Chunk Loop

For each chunk starting from `user_start_chunk`:

### Step 1: Display Chunk Header

```
---
## Chunk {N} of {total} — {semantic_label}

**Files:** {file1}, {file2}, ...
**LOC:** {chunk_loc} lines changed | **Est. time:** ~{minutes} min
---
```

### Step 2: Show Existing Comments on This Chunk's Files

For each file in this chunk, check `comments_by_file` and display existing comments with trust-aware ordering:

**Display order** (most important first):

1. **`[Open]` threads** — Unresolved, need attention:
   ```
   🔴 [Open] {file}:{line} — @{author} ({date}):
   > {comment body, truncated to 200 chars}
   ```

2. **`[Resolved - verify]` threads** — Resolved but concern may persist:
   ```
   🟡 [Resolved - verify] {file}:{line} — @{author} ({date}):
   > {comment body, truncated to 200 chars}
   > ⚠️ Code in this area appears unchanged — verify this was addressed
   ```

3. **`[Addressed]` threads** — Collapsed, resolved and confirmed:
   ```
   <details>
   <summary>✅ {count} addressed comments on this chunk's files</summary>
   - {file}:{line} — @{author}: "{brief}" ✅
   - ...
   </details>
   ```

4. **`[Outdated]` threads** — Omitted by default. Only show if user asks.

If there are **no existing comments** on this chunk's files, display:
```
No existing review comments on these files.
```

### Step 3: Agnostic A2A Team Meeting

The Leader orchestrates the review using the **Agnostic A2A Protocol** (`modules/teams/protocol.md`) to ensure professional, multi-perspective feedback.

#### 3.1: Look-ahead (Background)
While the user is interacting with Chunk {N}, the Leader triggers the A2A analysis for Chunk {N+1} in the background.

#### 3.2: Independent Discovery (Parallel)
The Leader assigns the `[A2A-TASK-ASSIGNMENT]` to the following Teammates:
- **`security-specialist`**: High-confidence security & secret scanning.
- **`performance-architect`**: Algorithmic and resource efficiency.
- **`clean-code-mentor`**: Readability and pattern adherence.

Each teammate returns an `[A2A-TECHNICAL-REPORT]`.

#### 3.3: The Agnostic Challenge Turn (Share & Challenge)
The Leader shares reports among teammates for critique:
- **Example**: Security report is sent to `performance-architect` via `[A2A-CHALLENGE-REQUEST]`.
- **Goal**: Identify trade-offs (e.g., "The security fix adds O(n) overhead").

#### 3.4: Consolidation & Synthesis
The Leader (Review Buddy) reviews the A2A reports and critiques:
1. **Resolve Conflicts**: Weights findings based on `focus` area and impact.
2. **Translate**: Converts technical A2A data into "Buddy" language.
3. **Format**: Groups into `Action Required`, `Recommended`, and `Minor` using the findings template.

Only findings with consolidated **confidence >= 80** are moved to Step 4.

### Step 4: Present Findings

Group findings by severity using the format from `assets/finding-template.md`.

**Finding numbers are globally sequential across the entire review** — they do NOT reset per chunk. Maintain running counters (`next_H`, `next_M`, `next_L`) that persist across all chunks. For example, if Chunk 1 produces H1, H2, and M1, then Chunk 2's first critical finding is H3 (not H1).

```
### Findings for Chunk {N}

{If any Action Required findings:}
#### 🔴 Action Required

{findings numbered H{next_H}, H{next_H+1}, ... formatted per assets/finding-template.md}

{If any Recommended findings:}
#### 🟡 Recommended

{findings numbered M{next_M}, M{next_M+1}, ... formatted per assets/finding-template.md}

{If any Minor findings AND not in --self mode:}
#### 🟢 Minor
<details>
<summary>{count} minor suggestions</summary>
{findings numbered L{next_L}, L{next_L+1}, ... formatted per assets/finding-template.md}
</details>

{If no findings at all:}
✅ **No issues found in this chunk.** The changes look good.
```

**Rules**:
- Only include findings with **confidence >= 80/100**
- In `--self` mode, **suppress Minor findings entirely** (don't even show collapsed)
- Finding IDs (H1, M3, L5, etc.) must be unique across the whole review — never reuse a number
- If a finding relates to an existing comment from `comments_by_file`, cross-reference it **with the thread_id** so `github-post` can reply in-thread:
  ```
  > 💬 Related to @{author}'s comment at {file}:{line} (thread_id: {thread_id})
  ```
- Each finding should be actionable — don't flag things that are clearly intentional or follow established patterns in the codebase

### Step 5: Positive Observations

After findings (or instead of, if there are none), briefly note 1-2 positive aspects of the chunk if any stand out:

```
**👍 Well done:**
- Clean separation of concerns in the new helper functions
- Good test coverage for edge cases
```

Keep this brief. Skip if nothing particularly noteworthy.

### Step 6: User Gate

Display the current progress from `.review-buddy/review-todo.md` before the gate.

Present this gate following the **Agnostic Interaction Protocol** (`references/PROTOCOL.md`).

**Primary options:**
1. **Continue** — Mark findings for GitHub and move to chunk {N+1}
2. **Deep-dive** — Explore a specific finding in more detail
3. **Pause & save** — Save progress and exit (resume with `--continue`)
4. **Skip to synthesis** — Jump to final summary

**Free-form commands** (user can type these directly):
- **"discard"** — Remove this chunk's findings (won't be posted)
- **"todo: {text}"** — Add a custom task to `.review-buddy/review-todo.md`
- **"deselect H3, M1"** — Unmark specific findings from GitHub posting (by default, Continue marks all)

If user types **"todo: {text}"** (free-form):
- Use `scripts/todo-manager.md` to append the task.
- Re-display the User Gate for the current chunk.

If user types **"discard"** (free-form):
- Remove all findings from this chunk (won't be posted to GitHub).
- Display confirmation and re-display the User Gate.

If user types **"deselect {IDs}"** (free-form):
- Unmark the specified finding IDs from GitHub posting (e.g., "deselect H3, M1").
- Display which findings remain marked and re-display the User Gate.

If user chooses **"Pause & save"**:
- Use `scripts/state-manager.md` and `scripts/todo-manager.md` to save current state and progress.
- Mark current chunk as "reviewed" in the todo list.
- Display: "Progress saved. Resume with `/review-buddy --continue`."
- Stop the review loop

If user chooses **"Deep-dive"**:
- Ask which finding to explore
- Provide more detailed analysis: broader context, potential fixes, impact assessment
- After deep-dive, return to the user gate for this same chunk

### Step 7: Save State After Each Chunk

After the user chooses to continue (or skip), update the state:
- Mark the current chunk as "reviewed" (or "skipped")
- Append any new findings to the accumulated findings list
- Update the `updated` timestamp
- Persist the current finding counters (`next_H`, `next_M`, `next_L`) so the next chunk continues the sequence

## Loop Termination

The loop ends when:
1. All chunks have been reviewed → proceed to Phase 5 (Synthesis)
2. User chooses "Skip to synthesis" → proceed to Phase 5 with partial data
3. User chooses "Pause & save" → exit and save state

## Output

Pass the following to Phase 5:
- `all_findings` — accumulated findings from all reviewed chunks, each with severity, file, line, description, `marked_for_github` flag, and `related_thread_id` (if the finding references an existing comment thread)
- `chunks_reviewed` — count of chunks actually reviewed
- `chunks_skipped` — count of chunks skipped
- `positive_notes` — collected positive observations
