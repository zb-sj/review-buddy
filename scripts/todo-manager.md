# Todo Manager (Agnostic)

This module manages the `.review-buddy/review-todo.md` file, which serves as the persistent, agnostic "database" for review tasks and progress.

## Storage
**File Path:** `.review-buddy/review-todo.md` (relative to project root)

## Format (GFM Markdown)
The file must use standard GitHub Flavored Markdown task lists.

```markdown
# Review Progress: [PR Name/Number]

## 🤖 Review Flow
- [ ] Phase 1: Context Assembly
- [ ] Phase 2: Comments Digest
- [ ] Phase 3: Chunk Planning
- [ ] Phase 4: Chunk Review <!-- current_chunk: 0, total_chunks: 0 -->
- [ ] Phase 5: Synthesis & Posting

## 📝 Ad-hoc Tasks
- [ ] Task description <!-- id: timestamp -->
```

## Operations

### 1. Initialize (`init`)
Create the initial `review-todo.md` after Phase 3 (Chunk Planning).
- Populate the `Review Flow` section.
- Add sub-tasks for each chunk: `- [ ] Chunk 1: [Files]`.
- Set the `total_chunks` metadata in the Phase 4 comment.

### 2. Update Progress (`update`)
Mark tasks as complete by replacing `[ ]` with `[x]`.
- When a chunk is finished:
  - Mark that chunk's task as `[x]`.
  - Update `current_chunk` in the metadata comment.
- If all chunks are done, mark "Phase 4: Chunk Review" as `[x]`.

### 3. Add Ad-hoc Task (`add`)
Append a new item to the `## 📝 Ad-hoc Tasks` section.
- Generate a unique ID (e.g., `<!-- id: 1700000000 -->`) for programatic reference.

### 4. Sync State (`sync`)
Read the `review-todo.md` file to determine "What is next?".
- The first `[ ]` item in the file is the next priority.
- If `/review-buddy --continue` is called, the agent uses this file to resume.

## Agnostic Tooling
Agents should use standard file tools to manage this:
- **Read**: `read_file`
- **Write/Edit**: `replace` (to change `[ ]` to `[x]`) or `write_file`.
- **Cleanup**: Delete the file upon successful completion of Phase 5.
