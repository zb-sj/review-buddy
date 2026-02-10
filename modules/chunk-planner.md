# Chunk Planner (Phase 3)

You are performing Phase 3 of a chunked PR review. Your goal is to intelligently group the changed files into reviewable chunks that respect logical boundaries, dependency order, and cognitive load limits.

## Prerequisites

- Phase 1 (Context Assembly) completed — `changed_files` list available
- Phase 2 (Comments Digest) completed — `comments_by_file` available

## Steps

### 1. Classify Each File by Category

Assign each changed file to exactly one category based on its path and name:

| Category | Priority | Patterns |
|----------|----------|----------|
| `types` | 1 | `*.d.ts`, `types.ts`, `types/*.ts`, `interfaces.ts`, `*.types.ts`, `models/*.ts`, `schema.*` |
| `config` | 2 | `*.config.*`, `.env*`, `tsconfig*`, `package.json`, `*.yml`, `*.yaml`, `Dockerfile`, `*.toml` |
| `docs` | 2 | `*.md`, `*.mdx`, `docs/*`, `README*`, `CHANGELOG*`, `LICENSE` |
| `style` | 2 | `*.css`, `*.scss`, `*.less`, `*.styled.*`, `*.module.css`, `tailwind*` |
| `utility` | 3 | `utils/*`, `helpers/*`, `lib/*`, `shared/*`, `common/*` |
| `core-logic` | 4 | `services/*`, `core/*`, `engine/*`, `api/*`, `server/*`, `controllers/*`, `middleware/*` |
| `component` | 5 | `components/*`, `pages/*`, `views/*`, `screens/*`, `layouts/*`, `app/*` |
| `test` | 6 | `*.test.*`, `*.spec.*`, `__tests__/*`, `test/*`, `tests/*`, `*.stories.*` |

Priority determines review order — lower number = reviewed first.

If a file doesn't match any pattern, assign it to `core-logic` (priority 4) as a safe default.

### 2. Build Dependency Graph

Scan the diff content of each file for import/require statements to build a dependency graph among the changed files:

1. For each changed file, extract imports from the diff's added/modified lines:
   - `import ... from './relative/path'`
   - `import ... from '../relative/path'`
   - `require('./relative/path')`
   - `from relative.module import ...` (Python)
2. Resolve relative imports to absolute paths
3. Check if the imported file is also in the changed files list
4. Build directed edges: importing file → imported file (dependency)

This graph helps ensure you review depended-upon files before their consumers.

### 3. Determine Review Order

Combine category priority with dependency order:

1. Start with a **topological sort** of the dependency graph (dependencies before dependents)
2. Within the same dependency level, sort by **category priority** (types first, tests last)
3. Within the same category, sort by **directory** (co-located files adjacent)
4. Within the same directory, sort by **file name** (alphabetical)

If the dependency graph has cycles, break cycles by category priority (lower priority file goes first).

### 4. Group into Chunks

Target chunk size: **300 LOC** (acceptable range: 200–450 LOC).
LOC = additions + deletions for each file (total change size).

#### Grouping Algorithm:

```
chunks = []
current_chunk = { files: [], loc: 0 }

for each file in sorted_order:
    file_loc = file.additions + file.deletions

    # Large file: gets its own chunk
    if file_loc > 450:
        if current_chunk.files is not empty:
            chunks.append(current_chunk)
            current_chunk = { files: [], loc: 0 }
        chunks.append({ files: [file], loc: file_loc })
        continue

    # Would this file push chunk over 450?
    if current_chunk.loc + file_loc > 450 and current_chunk.files is not empty:
        chunks.append(current_chunk)
        current_chunk = { files: [], loc: 0 }

    # Affinity check: prefer keeping same-directory or same-category files together
    if current_chunk.files is not empty:
        last_file = current_chunk.files[-1]
        same_dir = dirname(file.path) == dirname(last_file.path)
        same_cat = file.category == last_file.category

        # If adding file would exceed 450 but it has strong affinity, allow up to 500
        if current_chunk.loc + file_loc > 450 and (same_dir or same_cat):
            if current_chunk.loc + file_loc <= 500:
                # Allow slight oversize for affinity
                pass
            else:
                chunks.append(current_chunk)
                current_chunk = { files: [], loc: 0 }

    current_chunk.files.append(file)
    current_chunk.loc += file_loc

# Don't forget the last chunk
if current_chunk.files is not empty:
    # Tiny trailing chunk? Merge with previous if possible
    if current_chunk.loc < 100 and chunks is not empty:
        prev = chunks[-1]
        if prev.loc + current_chunk.loc <= 500:
            prev.files.extend(current_chunk.files)
            prev.loc += current_chunk.loc
        else:
            chunks.append(current_chunk)
    else:
        chunks.append(current_chunk)
```

### 5. Handle Edge Cases

- **Single-file PR**: Skip chunking entirely. Display: "Single file change — reviewing directly."
- **Tiny PR (< 100 total LOC)**: Suggest quick mode. Display: "This PR is small ({N} LOC). Consider using `--quick` for a faster review."
- **Huge PR (50+ files)**: Display warning: "⚠️ This PR changes {N} files. The review will be divided into {M} chunks and may take a while. Consider focusing on specific areas with `--focus`."
- **Only deleted files**: Note that review will focus on verifying safe removal (no broken imports, no lost functionality).
- **Only renamed/moved files**: Note that review will focus on verifying rename correctness and updated imports.

### 6. Display Chunk Plan

Present the plan in this format:

```
## Review Plan

{edge_case_message_if_any}

| Chunk | Files | Category | LOC | Est. Time | Existing Comments |
|-------|-------|----------|-----|-----------|-------------------|
| 1 | `types/index.d.ts`, `models/user.ts` | types | 120 | ~2 min | 0 |
| 2 | `utils/format.ts`, `utils/validate.ts` | utility | 280 | ~5 min | 2 (1 open) |
| 3 | `core/engine.ts` | core-logic | 450 | ~8 min | 5 (2 open, 1 verify) |
| 4 | `components/Form.tsx`, `components/List.tsx` | component | 310 | ~5 min | 1 (resolved) |
| 5 | `__tests__/engine.test.ts` | test | 200 | ~3 min | 0 |
| **Total** | **{N} files** | | **{LOC}** | **~{time} min** | **{comments}** |

Review order: types → utilities → core logic → components → tests
```

Time estimate heuristic: ~1 min per 60 LOC, minimum 2 min per chunk.

The "Existing Comments" column uses data from `comments_by_file` to show how many existing review comments touch each chunk's files, with a breakdown of open/verify threads.

### 7. User Gate

Ask the user how to proceed:

**Options:**
1. **Start from chunk 1** — Begin sequential review
2. **Skip to chunk N** — Jump to a specific chunk (show chunk numbers)
3. **Review all at once** — Skip chunking, review everything in one pass (not recommended for large PRs)
4. **Cancel** — Stop the review

Use `AskUserQuestion` with these options.

### 8. Output

Store the following for use by subsequent phases:
- `chunks` — ordered list of chunks, each with: files, total LOC, category summary, estimated time
- `chunk_plan_display` — the formatted table for state persistence
- `user_start_chunk` — which chunk to start from (default: 1)
- `file_categories` — map of file path → category for reference
