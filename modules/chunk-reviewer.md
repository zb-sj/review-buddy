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
## Chunk {N} of {total} — {category_summary}

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

### Step 3: Analyze the Diff (Actor-Critic Workflow)

Analyze the diff using a two-pass agentic approach to ensure high accuracy and depth.

#### 3.1: Actor Pass (Initial Findings)
Read each file's diff and apply the appropriate **Specialized Persona** based on the `focus` area:

| Focus | Persona / Mindset |
|-------|-------------------|
| **Security** | OWASP-first. Look for injection, improper auth, PII exposure, and unsafe defaults. |
| **Performance** | Efficiency-first. Look for N+1, memory leaks, blocking I/O, and heavy computation. |
| **Correctness** | Reliability-first. Look for logic flaws, edge cases, and state management bugs. |
| **API Design** | DX-first. Look for breaking changes, naming clarity, and consistency. |
| **None** | Generalist. Balance all dimensions. |

Generate a draft list of findings including description and confidence. 

**Educational Value (if `mentoring` is true)**: 
For each finding, brainstorm a concise **"Mentor's Note"**. 
- Explain **why** the issue matters in this project's domain.
- Provide a **pro-tip** (e.g., a more idiomatic way to achieve the same result using existing project utilities).
- Ensure the tone is helpful for the **Reviewer** (learning during the session) and professional for the **Reviewee** (feedback on GitHub).

If `mentoring` is false, skip generating "Mentor's Note".

#### 3.2: Critic Pass (Reflection & Self-Correction)
Review the draft findings against these criteria:
- **Existence check**: Is the problematic code actually present in the added/modified lines?
- **Business Logic check**: Does this finding conflict with the apparent intent of the change?
- **False Positive check**: Is this a known pattern or a false positive for the category?
- **Reflexion Logic**: Document your reasoning in the hidden `<!-- Reflexion Logic -->` field of `assets/finding-template.md`.

Only findings that survive the Critic Pass and have a **confidence >= 80** are moved to Step 4.

### Step 4: Present Findings

Group findings by severity using the format from `assets/finding-template.md`:

```
### Findings for Chunk {N}

{If any Action Required findings:}
#### 🔴 Action Required

{findings formatted per assets/finding-template.md}

{If any Recommended findings:}
#### 🟡 Recommended

{findings formatted per assets/finding-template.md}

{If any Minor findings AND not in --self mode:}
#### 🟢 Minor
<details>
<summary>{count} minor suggestions</summary>
{findings formatted per assets/finding-template.md}
</details>

{If no findings at all:}
✅ **No issues found in this chunk.** The changes look good.
```

**Rules**:
- Only include findings with **confidence >= 80/100**
- In `--self` mode, **suppress Minor findings entirely** (don't even show collapsed)
- If a finding relates to an existing comment, cross-reference it:
  ```
  > 💬 Related to @{author}'s comment at {file}:{line}
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

Solicit user intent following the **Agnostic Interaction Protocol** (`references/PROTOCOL.md`). Display the current progress from `.review-buddy/review-todo.md` before the gate.

**Options:**
1. **Continue to chunk {N+1}** — Move to the next chunk
2. **Deep-dive** — Explore a specific finding in more detail (ask which one)
3. **Mark findings for GitHub** — Mark all or selected findings from this chunk for posting to GitHub
4. **Add Todo** — Add a custom task to `.review-buddy/review-todo.md` for later
5. **Discard findings** — Remove findings from this chunk (won't be posted)
6. **Pause & save** — Save progress and exit (can resume with `--continue`)
7. **Skip to synthesis** — Jump to final summary without reviewing remaining chunks

If user chooses **"Add Todo"**:
- Prompt for the task description.
- Use `scripts/todo-manager.md` to append the task.
- Re-display the User Gate for the current chunk.

If user chooses **"Pause & save"**:
- Use `scripts/state-manager.md` and `scripts/todo-manager.md` to save current state and progress.
- Mark current chunk as "reviewed" in the todo list.
- Display: "Progress saved. Resume with `/review-buddy --continue`."
- Stop the review loop

If user chooses **"Deep-dive"**:
- Ask which finding to explore
- Provide more detailed analysis: broader context, potential fixes, impact assessment
- After deep-dive, return to the user gate for this same chunk

If user chooses **"Mark findings for GitHub"**:
- Show a numbered list of findings from this chunk
- Ask which to mark (all / specific numbers)
- Set `marked_for_github: true` on selected findings
- Continue to the user gate (don't advance chunks yet)

### Step 7: Save State After Each Chunk

After the user chooses to continue (or skip), update the state:
- Mark the current chunk as "reviewed" (or "skipped")
- Append any new findings to the accumulated findings list
- Update the `updated` timestamp

## Loop Termination

The loop ends when:
1. All chunks have been reviewed → proceed to Phase 5 (Synthesis)
2. User chooses "Skip to synthesis" → proceed to Phase 5 with partial data
3. User chooses "Pause & save" → exit and save state

## Output

Pass the following to Phase 5:
- `all_findings` — accumulated findings from all reviewed chunks, each with severity, file, line, description, and `marked_for_github` flag
- `chunks_reviewed` — count of chunks actually reviewed
- `chunks_skipped` — count of chunks skipped
- `positive_notes` — collected positive observations
